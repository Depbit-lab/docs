---
title: Getting Started
description: Flash the firmware, provision the master PIN, and perform the first unlock.
---

This guide walks through flashing ZeroKeyUSB onto an Arduino Zero, performing the first boot wizard, and verifying that credentials unlock correctly.

## Flash the Firmware

Follow the J-Link driven workflow included with the repository:

1. Install the SEGGER J-Link Software Pack on Windows and connect the programmer over SWD.
2. Wire `SWDIO`, `SWCLK`, `RESET`, and `GND` between the J-Link and the SAMD21 as listed in `Flashing_tool/README.md`.
3. Run `flash_all.bat` to push both the bootloader (`bootloader.hex`) and the current firmware (`firmware.bin`). The script drives `flash_all.jlink` and finishes in roughly 10 seconds.

:::warning
Keep the ZeroKeyUSB powered and connected until the batch script reports success. Interrupting the run can leave the MCU without a bootloader.
:::

## First Boot Sequence

After flashing, connect the device over USB. The firmware initialises the touch controller, display, and stored configuration inside `ZerokeySetup::startup()`.

```cpp title="zerokey-setup.cpp::ZerokeySetup::startup"
void ZerokeySetup::startup() {
  Wire.begin();
  programPosition = PIN_SCREEN;
  zerokeyInfo::initialiseDeviceSerialNumber();
  // Reset and configure TS06 registers and sensitivity.
  display.begin(SSD1306_SWITCHCAPVCC, 0x3C);
  zerokeyUtils.initScreenOrientation();
  if (zerokeyEeprom.readLastTotpEpoch(savedEpoch)) {
    syncTotpEpoch(savedEpoch);
  }
  readConfigurationFlag();
}
```

When the configuration flag at EEPROM address `0x0000` is `0x00` or `0xFF`, the setup wizard appears. Use the touch arrows to step through the welcome pages, then choose a 16-digit PIN. The device:

- Generates an IV using hardware entropy and stores it at address `0x0010`.
- Encrypts a constant block with your PIN and writes the first 8 bytes to `0x0005–0x000C` as the verification signature.
- Wipes every credential slot with encrypted `0xFF` padding.

## Unlocking and Navigation

On subsequent boots the device jumps directly to the PIN screen. Enter the 16 digits using Up/Down to select a number and Right to append. Failed attempts increment a counter at EEPROM address `0x0002`. Once the count is non-zero `ZerokeySecurity::waitFromEeprom()` imposes exponential delays before new tries.

After a successful unlock:

- Swipe **Down** to view site, user, and password fields with an auto-scrolling 8-character window.
- Press **Center** to edit a field or type it over USB HID.
- Swipe **Left** to open the main menu featuring Backup, Settings, Danger Zone, and Info sections.

:::tip
When the device displays “REQTIME” on screen or over serial, open the provided `Web/zerokeyusb-time-sync.html` tool or run the Web Serial manager to push the current epoch. This step enables TOTP generation for slots with 2FA secrets.
:::

## Next Steps

- Learn more about the touch controls in [Firmware I/O](firmware/io.md).
- Explore secure storage internals in [Firmware Security](firmware/security.md).
- Configure optional Web tools in [Web Manager](web-manager/overview.md).
