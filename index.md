---
title: ZeroKeyUSB Overview
description: Core capabilities of the Arduino Zero based hardware password manager.
---

ZeroKeyUSB is an offline credential vault built on an Arduino Zero (ATSAMD21G18A) that stores encrypted records inside an external M24C64-WMN6TP EEPROM. The firmware boots into a touch-driven interface, unlocks with a 16-digit PIN, and types credentials over USB HID.

## Highlights

- **Hardware security** – Credentials and TOTP material reside in the 64-kbit I²C EEPROM. Each block is encrypted with AES-128 in CBC mode using a key derived from the master PIN and an IV stored inside the chip.
- **Touch-first navigation** – A TS06 capacitive controller drives the five primary channels (Left, Right, Up, Down, Center) with software debounce, long-press detection, and channel lockout.
- **Fast credential recall** – Once unlocked, the device decrypts site, username, password, and optional TOTP pages on demand and can emit them through USB keyboard emulation in multiple layouts.
- **Web-assisted management** – Browser tools under `Web/` use the Web Serial API to back up, edit, and restore the database or push the current epoch for TOTP validation.

## Hardware at a Glance

| Component | Role | Firmware references |
|-----------|------|---------------------|
| ATSAMD21G18A (Arduino Zero) | Main MCU running `ZerokeyOS` | `ZerokeyOS/ZerokeyOS.ino`
| M24C64-WMN6TP 64-kbit EEPROM | Persistent encrypted storage | `ZerokeyOS/zerokey-memorymap.h`, `zerokey-eeprom.cpp`
| TS06 capacitive controller | Five-channel touch input | `ZerokeyOS/zerokey-io.cpp`
| 128×32 SSD1306 OLED | Status UI with auto-scrolling fields | `ZerokeyOS/zerokey-display.cpp`

:::tip
Keep the device powered during credential imports or mass erase operations—the firmware writes each 32-byte EEPROM page sequentially and expects uninterrupted power.
:::

## Secure Data Model

Each credential occupies 128 bytes (four 32-byte pages). The first three pages hold the encrypted site, user, and password buffers. The optional fourth page carries the encrypted TOTP secret, while two bytes in the configuration area describe the TOTP algorithm and secret length.

On first boot the firmware guides the user through setup, gathers entropy for the IV, encrypts a constant block with the new PIN to store a verification signature, and then wipes every credential slot. Subsequent boots enforce exponential PIN retry delays backed by the failed-attempt counter stored at EEPROM address `0x0002`.

## Where to Go Next

- [Getting started](getting-started.md) explains flashing, setup, and first unlock.
- [Hardware](hardware/overview.md) covers the PCB stack, EEPROM map, and touch system.
- [Firmware](firmware/architecture.md) documents the runtime, security model, and I/O subsystems.
- [Reference](reference/api-eeprom.md) lists callable firmware APIs for advanced integrations.
