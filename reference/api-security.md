---
title: Security API Reference
description: Methods exposed by ZerokeySecurity for PIN handling and credential encryption.
---

`ZerokeySecurity` implements the cryptographic core of ZeroKeyUSB. Use the following methods to manage credentials, PINs, and backups.

## Credential Operations

| Method | Description |
|--------|-------------|
| `void lock()` | Encrypts `currentSite`, `currentUser`, and `currentPass` into EEPROM using AES-128-CBC. Trims trailing spaces to `0xFF` before encryption. |
| `void unlock()` | Reads the encrypted pages for `siteIndex`, decrypts them, and populates the current buffers. |
| `void eraseAll()` | Overwrites every credential slot with encrypted `0xFF` pages and clears TOTP metadata. Displays a countdown on the OLED. |
| `bool encryptPage(const uint8_t *plain, uint8_t out[32])` | AES-encrypts a 32-byte buffer using the current PIN-derived key and stored IV. |
| `bool decryptPage(const uint8_t in[32], uint8_t out[32])` | AES-decrypts a 32-byte buffer using the same key/IV. |

## PIN Lifecycle

| Method | Description |
|--------|-------------|
| `void storeSignature()` | Encrypts `constantEncryptedBlock` with the current PIN, stores the first 8 bytes at `0x0005–0x000C`, and runs the configuration routine. |
| `bool verifySignature()` | Recomputes the signature and compares it with the stored bytes. Increments the failed-attempt counter and calls `waitFromEeprom()` on mismatch. |
| `bool generateAndStoreIV()` | Generates a new IV from hardware entropy, writes it to `0x0010`, and calls `eraseAll()` to invalidate old data. |
| `bool loadIVfromEEPROM()` | Reads the IV from EEPROM, regenerating it if uninitialised or unreadable. |
| `uint8_t readFailedAttemptsCounter()` | Returns the counter stored at EEPROM `0x0002`. |
| `void writeFailedAttemptsCounter(uint8_t value)` | Persists the counter to EEPROM. |
| `void incrementFailedAttemptsCounter()` | Increments (saturating at 255) and stores the counter. |
| `void waitFromEeprom()` | Applies an exponential backoff based on the counter and displays the remaining wait time. |

## Backup Utilities

| Method | Description |
|--------|-------------|
| `void backupAllCredentials()` | Prints `MAXSITES` followed by CSV lines with decrypted data and optional TOTP secrets. |
| `void loadAllbackupCredentials()` | Reads CSV-formatted records from `SerialUSB`, encrypts them with `lock()`, and writes them to EEPROM. |
| `String bufferToString(const char *buf)` | Converts a 16-byte buffer into a printable string, skipping `0xFF` padding. |

## Usage Considerations

- All encryption helpers copy `pinArray` into the AES key before use. Ensure the array is populated with 16 digits.
- `loadIVfromEEPROM()` calls `zerokeyUtils.throwErrorScreen()` if it cannot recover a valid IV during unlock operations.
- Backups require `SerialUSB` to be active; call `zerokeyUtils.markSerialUnlocked()` after successful PIN verification before invoking them.

```cpp title="zerokey-security.cpp::ZerokeySecurity::incrementFailedAttemptsCounter"
uint8_t currentVal = readFailedAttemptsCounter();
if (currentVal < 255) {
  currentVal++;
} else {
  currentVal = 255;
}
writeFailedAttemptsCounter(currentVal);
```

:::warning
`eraseAll()` and `storeSignature()` display blocking countdowns. Avoid invoking them from asynchronous contexts—they assume exclusive control over the display and input state.
:::
