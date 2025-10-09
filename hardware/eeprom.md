---
title: EEPROM Memory Map
description: Layout of the M24C64-WMN6TP address space and access routines.
---

ZeroKeyUSB uses the 64-kbit M24C64-WMN6TP EEPROM to persist configuration and encrypted credentials. Pages are 32 bytes wide, and all transactions use 2-byte addressing over I²C.

## Address Map

| Address range | Size | Purpose |
|---------------|------|---------|
| `0x0000` | 1 byte | Configuration flag (`0x00` = unconfigured, `0x01` = ready). |
| `0x0001` | 1 byte | Screen orientation and input inversion flag. |
| `0x0002` | 1 byte | Failed PIN attempts counter. |
| `0x0003–0x0004` | 2 bytes | Reserved. |
| `0x0005–0x000C` | 8 bytes | PIN verification signature (first 8 bytes of the encrypted constant block). |
| `0x000D–0x001C` | 16 bytes | AES IV used for credential encryption. |
| `0x001D` | 1 byte | Reserved. |
| `0x001E–0x003D` | 32 bytes | Legacy Bitcoin private key slot (unused by current firmware). |
| `0x003E` | 1 byte | Persisted keyboard layout selection. |
| `0x003F–0x005F` | 33 bytes | Free / padding. |
| `0x0060–0x00DB` | 124 bytes | TOTP metadata table (`algorithm`, `secret length` per slot). |
| `0x00E0–0x1FFF` | 7968 bytes | Encrypted credential slots (128 bytes each, 62 maximum). |

:::tip
Credential pages are aligned automatically. Use `credentialPageAddress()` from `zerokey-memorymap.h` when addressing specific pages instead of hard-coding offsets.
:::

## Page Helpers

`ZerokeyEeprom` exposes helpers to read and write whole pages, ensuring the correct addressing order and write-cycle timing.

```cpp title="zerokey-eeprom.cpp::writeEepromPage"
bool ZerokeyEeprom::writeEepromPage(uint32_t address, const uint8_t buf[32]) {
  Wire.beginTransmission(eepromAddress);
  Wire.write((uint8_t)(address >> 8));
  Wire.write((uint8_t)(address & 0xFF));
  Wire.write(buf, EEPROM_PAGE_SIZE);
  byte error = Wire.endTransmission();
  delay(10);
  return error == 0;
}
```

`readEepromPage()` mirrors this logic and pads the buffer with zeros if fewer than 32 bytes are returned.

## Credential Slots

Each slot spans four pages (128 bytes):

1. **Page 0** – Encrypted site field (16 characters padded with `0xFF`).
2. **Page 1** – Encrypted username field.
3. **Page 2** – Encrypted password field.
4. **Page 3** – Encrypted TOTP secret (optional).

Metadata for each slot resides in the table starting at `CONFIG_TOTP_META_START` (`0x0060`):

```cpp title="zerokey-eeprom.cpp::writeTotpMeta"
void ZerokeyEeprom::writeTotpMeta(uint16_t index, uint8_t algorithm, uint8_t secret_len) {
  uint16_t address = CONFIG_TOTP_META_START + index * CONFIG_TOTP_META_STRIDE;
  Wire.beginTransmission(eepromAddress);
  Wire.write((uint8_t)(address >> 8));
  Wire.write((uint8_t)(address & 0xFF));
  Wire.write(algorithm);
  Wire.write(secret_len);
  Wire.endTransmission();
  delay(5);
}
```

A separate eight-byte record at `0x0040` (`EEPROM_LAST_TOTP_EPOCH_ADDR`) stores the last trusted epoch. `writeLastTotpEpoch()` serialises the 64-bit value most significant byte first.

## Layout Persistence

The chosen keyboard layout is stored at `0x003E` via `writeKeyboardLayout()` and restored on boot with `readKeyboardLayout()`. Screen orientation toggles reuse address `0x0001` through `ZerokeyUtils::toggleScreenOrientation()` and `ZerokeyUtils::initScreenOrientation()`.

### Unimplemented Helpers

- `ZerokeyEeprom::eraseMemory()` – *Not documented in the source code.*
