---
title: Security System
description: AES-128 CBC encryption, IV lifecycle, and PIN enforcement in ZeroKeyUSB.
---

The security module centres around AES-128 in CBC mode. Every credential page and TOTP secret is encrypted using a key derived from the 16-digit master PIN and a 16-byte IV stored inside the EEPROM.

## Key and IV Management

`pinArray` holds the current PIN digits and is copied byte-for-byte into the AES key before each operation. The IV lives at EEPROM address `0x0010` and is validated on load—if it reads as all `0x00` or `0xFF`, the firmware regenerates it using hardware entropy from an unconnected ADC pin (`ENTROPY_PIN = A0`).

```cpp title="zerokey-security.cpp::ZerokeySecurity::loadIVfromEEPROM"
uint8_t bytesRead = Wire.requestFrom(eepromAddress, (uint8_t)16);
if (bytesRead != 16) {
  regenerateIVFromEntropy(iv);
  return true;
}
bool allFF = true;
bool all00 = true;
for (int i = 0; i < 16; i++) {
  iv[i] = Wire.read();
  if (iv[i] != 0xFF) allFF = false;
  if (iv[i] != 0x00) all00 = false;
}
if (allFF || all00) {
  regenerateIVFromEntropy(iv);
}
```

Generating a fresh IV (`generateAndStoreIV()`) also triggers `eraseAll()` to wipe credentials encrypted with the previous IV.

## PIN Verification

During setup, `storeSignature()` encrypts a constant 16-byte block (`constantEncryptedBlock`) and stores the first eight bytes at EEPROM addresses `0x0005–0x000C`. Subsequent logins call `verifySignature()` to reproduce the ciphertext and compare it to the stored value.

```cpp title="zerokey-security.cpp::ZerokeySecurity::verifySignature"
for (int i = 0; i < 16; i++) {
  key[i] = pinArray[i];
}
if (!loadIVfromEEPROM()) {
  zerokeyUtils.throwErrorScreen(F("Missing IV"));
  return false;
}
int encryptedLength = aesLib.encrypt(block, 16, encryptedBlock, key, 128, iv);
if (encryptedLength != 16) {
  return false;
}
for (int i = 0; i < 8; i++) {
  if (computedSignature[i] != storedSignature[i]) {
    incrementFailedAttemptsCounter();
    zerokeySecurity.waitFromEeprom();
    return false;
  }
}
writeFailedAttemptsCounter(0);
return true;
```

The failed-attempt counter at `0x0002` is incremented on mismatches and resets to zero after a successful unlock.

## Exponential Lockout

`waitFromEeprom()` multiplies a five-second base delay by `2^(attempts-1)` (capped at 2,560 seconds). During the countdown the OLED displays “Wrong PIN” and the remaining seconds.

## Credential Encryption

`lock()` and `unlock()` operate on three 16-byte buffers (`currentSite`, `currentUser`, `currentPass`). Before encryption, trailing spaces are converted to `0xFF` padding. Each field is copied into a 32-byte page filled with `0xFF`, encrypted, and written via `zerokeyEeprom.writeEepromPage()`.

```cpp title="zerokey-security.cpp::ZerokeySecurity::lock"
memset(plain, 0xFF, 32);
memcpy(plain, (field == 0 ? currentSite : field == 1 ? currentUser : currentPass), 16);
if (aesLib.encrypt(plain, 32, encrypted, key, 128, iv) != 32) {
  return;
}
uint32_t addr = credentialPageAddress(siteIndex, field);
zerokeyEeprom.writeEepromPage(addr, encrypted);
```

`unlock()` reverses the process: it reads, decrypts, and copies the first 16 bytes back into the field buffers.

## Database Maintenance

- **Erase all** – `eraseAll()` encrypts a 32-byte block of `0xFF` for every page in all 62 slots, then clears the TOTP metadata (`zerokeyEeprom.writeTotpMeta(index, 0, 0)`).
- **Backups** – `backupAllCredentials()` iterates through every slot, decrypts it with `unlock()`, prints CSV lines over `SerialUSB`, and exports TOTP metadata when available.
- **Imports** – `loadAllbackupCredentials()` parses CSV input, fills buffers with `0xFF`, writes credentials with `lock()`, and stores TOTP secrets parsed from Base32 or `otpauth://` URIs.

:::warning
Never disconnect power while `eraseAll()` or `loadAllbackupCredentials()` is running. Both routines rewrite multiple EEPROM pages in sequence, and interruption may leave records partially encrypted.
:::
