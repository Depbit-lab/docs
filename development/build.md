---
title: Build and Flash Guide
description: Compile ZerokeyOS and deploy it to the Arduino Zero platform.
---

ZeroKeyUSB targets the Arduino Zero (SAMD21). You can build the firmware using the Arduino IDE or the Arduino CLI, then flash it with either the standard bootloader or the provided J-Link scripts.

## Prerequisites

- Arduino IDE 1.8+ or Arduino CLI 0.32+ with the **Arduino SAMD Boards** package installed.
- Libraries referenced by the sketch:
  - `Adafruit_SSD1306`
  - `Keyboard` and locale layout headers (`Keyboard_da_DK`, `Keyboard_de_DE`, etc.)
  - `AESLib`
- A Micro-USB cable for the Arduino Zero.
- Optional: SEGGER J-Link (for bare-metal flashing).

## Building with Arduino IDE

1. Open `ZerokeyOS/ZerokeyOS.ino` in the Arduino IDE.
2. Select **Tools → Board → Arduino Zero (Native USB Port)**.
3. Choose the correct port under **Tools → Port**.
4. Click **Sketch → Verify/Compile** to build the firmware.
5. Use **Sketch → Upload** to flash over the native USB bootloader.

## Building with Arduino CLI

```bash
arduino-cli compile --fqbn arduino:samd:arduino_zero_native ZerokeyOS
arduino-cli upload --fqbn arduino:samd:arduino_zero_native --port /dev/ttyACM0 ZerokeyOS
```

Replace `/dev/ttyACM0` with the serial device path exposed by your board.

## Flashing with J-Link

For a clean flash or when recovering a bricked board, use the scripts under `Flashing_tool/`:

1. Connect the J-Link to the SWD pins (`SWDIO`, `SWCLK`, `RESET`, `GND`).
2. Double-click `flash_all.bat` on Windows. The script runs `flash_all.jlink`, erasing flash, programming `bootloader.hex`, and then writing `firmware.bin`.
3. Wait for the console to report success (about 10 seconds) before disconnecting power.

## Post-Flash Checklist

- Open a serial monitor at 115200 baud to observe debug output (optional).
- Complete the on-device setup wizard to generate the IV and PIN signature.
- Verify that touch input works in both orientations. If not, toggle the screen flag from **Settings → Rotate Screen** (stored at EEPROM `0x0001`).

:::warning
Do not compile against boards that lack native USB support; the firmware depends on `Keyboard.begin()` and `SerialUSB`, which are only available on the SAMD21 native USB stack.
:::
