---
title: Firmware Architecture
description: Runtime layout, startup sequence, and module responsibilities in ZerokeyOS.
---

The firmware that powers ZeroKeyUSB is split into tightly focused modules managed by a lightweight main loop. The MCU spends most of its time polling the touch controller, refreshing the display, and invoking helpers to decrypt or type credentials on demand.

## Startup Flow

`setup()` delegates all hardware initialisation to `ZerokeySetup::startup()` and then waits for user interaction in the main loop.

```cpp title="ZerokeyOS.ino::setup and loop"
void setup() {
  zerokeySetup.startup();
}

void loop() {
  zerokeyIo.handleButtonChecker();
  delay(30);
}
```

`ZerokeySetup::startup()` handles:

- Initialising I²C and the SSD1306 OLED at address `0x3C`.
- Resetting TS06 registers and applying the configured sensitivity.
- Loading the stored keyboard layout and screen orientation from EEPROM address `0x003E` and `0x0001`.
- Restoring the last trusted TOTP epoch (`zerokeyEeprom.readLastTotpEpoch`) and synchronising the runtime counter.
- Reading the configuration flag (`0x0000`) to decide between the setup wizard (`programPosition = SETUP`) and the PIN screen.

If the device is already configured, `zerokeySecurity.waitFromEeprom()` enforces any exponential PIN delay before the PIN entry screen is drawn.

## Module Map

| Module | Responsibility |
|--------|----------------|
| `zerokey-security` | AES-128-CBC encryption/decryption, PIN signature storage, failed-attempt counters, CSV backup/import. |
| `zerokey-eeprom` | Raw page reads/writes, TOTP metadata, persisted keyboard layout, last TOTP epoch storage. |
| `zerokey-display` | Screen rendering, including scrolling credential fields and TOTP prompts. |
| `zerokey-io` | Touch controller polling, debounce, and host epoch handling. |
| `zerokey-utils` | USB HID typing, random credential generation, screen orientation toggles. |
| `zerokey-menu` | Hierarchical menu system (Backup, Settings, Danger Zone, Info). |
| `zerokey-totp` | Secret storage, Base32 parsing, and SHA-1/SHA-256/SHA-512 HOTP/TOTP generation. |
| `zerokey-info` | Device serial number derivation and software version reporting. |

Each module exposes a minimal surface through headers under `ZerokeyOS/` so new features can hook into the existing control flow without modifying the main loop.

## Program Positions

The `programPosition` global (declared in `zerokey-globals.h`) enumerates all UI states: PIN entry, main site/user/pass views, edit keyboards, menu screens, and TOTP modes. Input handlers in `zerokey-io.cpp` switch between these states and trigger redraws through `zerokeyDisplay.drawScreen()`.

Menu navigation is driven by `ZerokeyMenu`. The main menu contains Backup, Settings, Danger Zone, and Info groups, each with their own submenus. Selecting an action such as “Export” invokes the corresponding callback (`zerokeySecurity.backupAllCredentials()`, `zerokeySecurity.eraseAll()`, etc.) before returning to the menu.

## Timing and Refresh

- The main loop pauses 30 ms between polls to balance responsiveness and power consumption.
- `zerokeyIo.handleButtonChecker()` refreshes scrolling fields every 100 ms when idle and updates TOTP codes every 200 ms while in the TOTP view.
- Long press animations and epoch updates happen inside the same polling loop, ensuring consistent behaviour even when the MCU spends long periods waiting for host input.
