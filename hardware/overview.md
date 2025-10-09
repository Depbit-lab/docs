---
title: Hardware Overview
description: Summary of the ZeroKeyUSB electronics stack and peripheral configuration.
---

ZeroKeyUSB combines an Arduino Zero (ATSAMD21G18A) with external peripherals that provide secure storage, capacitive navigation, and a compact OLED UI. The firmware initialises each component during startup and exposes abstractions through dedicated modules.

## Core Components

| Subsystem | Device | Notes |
|-----------|--------|-------|
| MCU | Arduino Zero (SAMD21) | Runs `ZerokeyOS` with USB HID support via `Keyboard.h` and layout headers. |
| Persistent storage | M24C64-WMN6TP (64-kbit I²C EEPROM) | Uses 2-byte addressing, 32-byte pages, and a total capacity of 8 KiB. |
| Touch input | TS06 capacitive sensor | Exposed over I²C at address `0xD2 >> 1` with up to six channels (five primary directions plus an optional custom pad). |
| Display | 128×32 SSD1306 OLED | Controlled through the Adafruit_SSD1306 driver at address `0x3C`. |

### Display geometry

The firmware defines the screen geometry and shared resources in `zerokey-globals.h`:

```cpp title="zerokey-globals.h::Display constants"
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 32
#define OLED_RESET 4
extern Adafruit_SSD1306 display;
```

The OLED renders an 8-character auto-scrolling window for each credential field. Long values scroll every 500 ms, while short values remain stationary.

### Touch controller wiring

`ZerokeySetup::startup()` writes `0x00` to TS06 registers `0x05` and `0x06`, then applies the highest detection threshold (`TS06_MIN_SENSITIVITY = 0x3F`) to channels `0x00–0x02`. The runtime reads channel status bytes from register `0x25` through `ZerokeyIo::readRegister()` and implements debounce and long-press logic in software.

### EEPROM layout

The EEPROM provides a 96-byte legacy configuration block followed by encrypted credential slots. Constants declared in `zerokey-memorymap.h` compute addresses and page indices for each slot.

```cpp title="zerokey-memorymap.h::credentialAddress"
constexpr uint16_t EEPROM_TOTAL_SIZE = 8192;
constexpr uint16_t EEPROM_PAGE_SIZE = 32;
constexpr uint8_t CREDENTIAL_PAGES = 4;
constexpr uint16_t CREDENTIAL_SIZE = EEPROM_PAGE_SIZE * CREDENTIAL_PAGES;
constexpr uint16_t MAX_CREDENTIALS = 62;
inline uint16_t credentialAddress(uint16_t index) {
  return EEPROM_CREDENTIAL_BASE + (index * CREDENTIAL_SIZE);
}
```

Each credential uses three encrypted pages (site, user, password). The fourth page is reserved for the optional TOTP secret, and a two-byte metadata entry stores the selected hash algorithm and secret length.

## Power and Reliability Considerations

- The EEPROM driver delays 10 ms after page writes to respect the chip’s internal programming time.
- Credential erase routines fill buffers with `0xFF` before encrypting to maintain deterministic padding.
- Screen orientation and touch inversion flags live at EEPROM address `0x0001`; the firmware rotates the display and flips controls automatically if the bit is set.
