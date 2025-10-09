---
title: TS06 Touch Sensor
description: Capacitive input handling, debounce, and long-press logic for the TS06 controller.
---

The TS06 capacitive controller exposes up to six channels over I²C (Left, Right, Up, Down, Center, and an optional custom pad). ZeroKeyUSB polls the status register at `0x25` to detect touches, debounce them, and distinguish between short and long presses.

## Configuration

`ZerokeySetup::startup()` performs the initial configuration:

- Writes `0x00` to registers `0x05` and `0x06` to reset channel state.
- Applies `TS06_MIN_SENSITIVITY = 0x3F` to sensitivity registers `0x00–0x02`.
- Waits 30 ms before enabling the main loop.

## Polling Loop

`ZerokeyIo::handleButtonChecker()` executes in the main loop (`ZerokeyOS.ino::loop`) and handles debounce, channel locking, long presses, and TOTP refreshes.

```cpp title="zerokey-io.cpp::ZerokeyIo::handleButtonChecker"
uint8_t rawStatus = readRegister(0x25);
unsigned long now = millis();
updateTotpEpochFromMillis();
if (lockedChannel != -1 && now >= lockReleaseTime) {
  lockedChannel = -1;
}
for (uint8_t channel = 0; channel < NUM_CHANNELS; channel++) {
  uint8_t mask = (1 << channel);
  bool rawPressed = (rawStatus & mask);
  if (lockedChannel != -1 && channel != lockedChannel) {
    rawPressed = false;
  }
  if (rawPressed) {
    if (rawPressStartTime[channel] == 0) {
      rawPressStartTime[channel] = now;
    }
    if ((now - rawPressStartTime[channel]) >= DEBOUNCE_MS) {
      filteredStatus |= mask;
    }
  } else {
    rawPressStartTime[channel] = 0;
  }
}
```

### Debounce and Channel Lockout

- **Debounce** – `DEBOUNCE_MS = 50` ensures touches last at least 50 ms before counting as pressed.
- **Channel lockout** – When a channel activates, `lockedChannel` blocks other pads for `CHANNEL_LOCKOUT_MS = 150` ms to avoid multi-touch jitter.

### Long Press Detection

`LONG_PRESS_THRESHOLD = 800` ms defines when a press becomes a long press. During the hold, a circular progress indicator rendered by `drawLongPressProgress()` expands from radius 2 to 16 pixels.

```cpp title="zerokey-io.cpp::ZerokeyIo::drawLongPressProgress"
int radius = 2 + (int)(progress * 14.0f);
int centerX = 64;
int centerY = 32;
display.fillRect(48, 16, 32, 32, BLACK);
display.fillCircle(centerX, centerY, radius, WHITE);
```

Short presses call direction handlers such as `leftButtonPressed()` and `centerButtonPressed()` to drive navigation, editing, and USB typing. Long presses map to bulk actions—for example, holding Left or Right jumps by ten slots, while holding Center in a credential view switches to edit mode.

### Host Time Handling

While scanning, `handleButtonChecker()` also calls `handleIncomingHostEpoch()` so the device can react immediately when a host responds to `REQTIME` with a Unix epoch. Successful reception updates `estatus`, sets `waitingForHostTime = false`, and redraws the TOTP UI.

:::warning
Do not disconnect the device while a long press is in progress. The loop re-renders the OLED repeatedly and expects the touch controller to remain powered; removing power can leave the I²C bus mid-transaction.
:::
