---
title: EEPROM API Reference
description: Public methods for accessing the M24C64 storage driver.
---

`ZerokeyEeprom` wraps I²C access to the M24C64-WMN6TP chip. The class lives in `zerokey-eeprom.h` and `zerokey-eeprom.cpp` and exposes the following methods.

## Public Methods

| Method | Description |
|--------|-------------|
| `void readEntry(byte *entry)` | Reads a 96-byte credential (three pages) into `entry`, filling missing bytes with zero. |
| `void writeEntry(byte *entry)` | Encrypts and writes a 96-byte credential from `entry` to the current `siteIndex`. |
| `bool readEepromPage(uint32_t address, uint8_t buf[32])` | Reads one 32-byte page. Pads with zeros if fewer bytes are returned. |
| `bool writeEepromPage(uint32_t address, const uint8_t buf[32])` | Writes one 32-byte page, waiting 10 ms for completion. |
| `void writeTotpMeta(uint16_t index, uint8_t algorithm, uint8_t secret_len)` | Stores the algorithm code and secret length for slot `index`. |
| `void readTotpMeta(uint16_t index, uint8_t *algorithm, uint8_t *secret_len)` | Loads the metadata pair for slot `index`. Returns zeros on error. |
| `void writeLastTotpEpoch(uint64_t epoch)` | Stores the latest trusted epoch at `EEPROM_LAST_TOTP_EPOCH_ADDR` (`0x0040`). |
| `bool readLastTotpEpoch(uint64_t &epoch)` | Reads the persisted epoch. Returns `false` if the chip is absent or communication fails. |
| `void writeKeyboardLayout(KeyboardLayoutOption layout)` | Persists the current keyboard layout at address `0x003E`. |
| `KeyboardLayoutOption readKeyboardLayout()` | Retrieves the stored keyboard layout. Defaults to `LAYOUT_EN_US` if no byte is available. |
| `void eraseMemory()` | *Not documented in the source code.* |

## Private Helpers

`present()` probes the EEPROM by issuing a zero-length transmission and checking for an ACK. `writeEntry()` and `readEntry()` call `present()` before attempting any transfer and display an error on the OLED if the chip is missing.

## Usage Notes

- Always call `writeEepromPage()`/`readEepromPage()` instead of operating on `Wire` directly. These helpers manage the MSB/LSB ordering required by the 16-bit address bus.
- Metadata functions rely on `credentialIndexValid()` from `zerokey-memorymap.h` to guard array bounds. Ensure indexes remain below `MAX_CREDENTIALS`.
- TOTP epochs are serialised big-endian: `writeLastTotpEpoch()` shifts the value and writes the most significant byte first.

```cpp title="zerokey-eeprom.cpp::readLastTotpEpoch"
for (uint8_t i = 0; i < EEPROM_LAST_TOTP_EPOCH_SIZE && Wire.available(); ++i) {
  epoch = (epoch << 8) | static_cast<uint64_t>(Wire.read());
}
return true;
```

:::warning
Avoid calling `writeEntry()` or `writeTotpMeta()` inside interrupt contexts. Each helper performs blocking I²C transactions and delays to satisfy EEPROM write timings.
:::
