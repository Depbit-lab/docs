---
title: Input Handling
description: Touch controller mapping, debounced navigation, and host communication logic.
---

ZeroKeyUSB relies on `ZerokeyIo` to translate capacitive touches into UI actions. The class polls the TS06 controller, enforces debounce, and invokes handlers that update `programPosition`, move between credentials, or trigger USB output.

## Channel Mapping

The TS06 exposes six bits even though only five physical pads are required. Channels are interpreted as:

| Channel | Action |
|---------|--------|
| 0 | Up |
| 1 | Custom (context-sensitive) |
| 2 | Right |
| 3 | Center |
| 4 | Down |
| 5 | Left |

`handleChannelPress()` remaps directions when screen inversion is active and forwards the event to the appropriate handler.

```cpp title="zerokey-io.cpp::ZerokeyIo::handleChannelPress"
if (invertControls) {
  switch (channel) {
    case 0: channel = 4; break;
    case 4: channel = 0; break;
    case 2: channel = 5; break;
    case 5: channel = 2; break;
  }
}
switch (channel) {
  case 0: upButtonPressed(); break;
  case 1: customButtonPressed(); break;
  case 2: rightButtonPressed(); break;
  case 3: centerButtonPressed(); break;
  case 4: downButtonPressed(); break;
  case 5: leftButtonPressed(); break;
}
zerokeyDisplay.drawScreen();
```

## Navigation Logic

- **Left/Right** – Scroll through credentials. Long presses jump ±10 slots using `adjustSiteIndexWithMenuFallback()`.
- **Up/Down** – Move between site, user, password, and TOTP views or increment/decrement digits during PIN entry.
- **Center** – Confirm input, enter edit mode, or emit data over USB HID (`zerokeyUtils.typeCredential`).
- **Custom** – Acts as a “cancel” button for TOTP entry screens, returning to the main view and resetting date/time inputs.

While editing, the firmware renders on-screen keyboards (`EDIT_KB1–EDIT_KB3`) stored in PROGMEM and updates fields using the `keyboardIndex` cursor.

## Host Interaction

`handleIncomingHostEpoch()` listens to `SerialUSB` for a Unix epoch after the firmware sends `REQTIME`. Valid timestamps update `estatus`, set `totpTimeSyncedFromHost`, and persist the value via `zerokeyEeprom.writeLastTotpEpoch()`.

`zerokeySecurity.backupAllCredentials()` and `loadAllbackupCredentials()` use the same serial link. When exporting, the device prints `MAXSITES` followed by CSV rows. When importing, it expects the host to send the record count plus `index,site,user,pass[,totp]` entries. Touch navigation remains responsive because `handleButtonChecker()` continues to poll between serial reads.

## Visual Feedback

The OLED highlights the active field with inverted colours. `zerokeyDisplay.refreshMainScrollIfNeeded()` scrolls fields longer than eight characters in 500 ms steps, and `drawLongPressProgress()` overlays a circular indicator during long presses. TOTP mode redraws every 200 ms to display the latest code and countdown.

:::tip
If a touch input appears stuck, toggle screen orientation from **Settings → Rotate Screen**. The firmware flips both the OLED orientation and input mapping by toggling the flag at EEPROM address `0x0001`.
:::
