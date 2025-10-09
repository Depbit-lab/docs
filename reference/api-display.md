---
title: Display API Reference
description: Rendering helpers provided by ZerokeyDisplay for the 128×32 OLED UI.
---

`ZerokeyDisplay` orchestrates all OLED rendering. It works with the global `Adafruit_SSD1306 display` instance defined in `zerokey-globals.h` and exposes drawing helpers for each UI state.

## High-Level Rendering

| Method | Purpose |
|--------|---------|
| `void drawScreen()` | Switches on `programPosition` to render the appropriate screen (PIN, main view, edit keyboards, menus, TOTP). Calls `zerokeydisplay()` to flush the buffer. |
| `void zerokeydisplay()` | Pushes the back buffer to the OLED (`display.display()`). |
| `void wipeScreen()` | Clears the OLED, resets the cursor, and restores default text size and colour. |

`drawScreen()` delegates to more specific renderers:

```cpp title="zerokey-display.cpp::ZerokeyDisplay::drawScreen"
switch (programPosition) {
  case MAIN_SITE:
  case MAIN_USER:
  case MAIN_PASS:
  case MAIN_2FA:
    renderMainScreen();
    break;
  case EDIT_LEFT_CURSOR:
  case EDIT_RIGHT_CURSOR:
  case EDIT_RAND:
  case EDIT_BACK:
  case EDIT_KB1:
  case EDIT_KB2:
  case EDIT_KB3:
    renderEditScreen();
    break;
  case TOTP_DATE_ENTRY:
    renderTotpDateEntry();
    break;
  case TOTP_TIME_ENTRY:
    renderTotpTimeEntry();
    break;
  case TOTP_SHOW_CODE:
    renderTotpCodeScreen();
    break;
  default:
    zerokeyUtils.throwErrorScreen(F("ERROR DISP"));
    break;
}
```

## Main View and Scrolling

`renderMainScreen()` draws icons and uses `drawScrollingField()` to show site, user, password, or TOTP status. Each field scrolls within an 8-character window when the text exceeds the window size. `refreshMainScrollIfNeeded()` re-renders the screen every 500 ms while idle to advance the window.

## PIN and Setup Screens

- `renderPinScreen()` – Displays the PIN frame, obfuscated digits, and navigation hints.
- `renderHelloScreen(int page)` – Shows the multi-step setup wizard during first boot.
- `renderIndicator(const char *indicator)` – Draws a vertical label (e.g., “MENU”) along the left edge.
- `renderKeyCreationScreen()` / `renderReadyScreen()` – Provide status messages during configuration.

## TOTP Helpers

- `renderTotpDateEntry()` and `renderTotpTimeEntry()` render editable date/time fields with highlighted cursors.
- `renderTotpCodeScreen()` centres the six-digit code, prints the remaining lifetime, and is refreshed every 200 ms while active.

## Editing Interface

`renderEditScreen()` prints the current field characters, inverts the cursor, and shows soft buttons (`<`, `>`, `RND`, `BACK`, `KB1–KB3`). The on-screen keyboards pull characters from `keyboard1`, `keyboard2`, and `keyboard3` arrays in `zerokey-globals.h`.

## Additional Utilities

- `void drawInvertedBitmap(int x, int y, const uint8_t *bitmap, int w, int h)` – Inverts a bitmap (e.g., user/password icons) before drawing.
- `void fillRect(int16_t x, int16_t y, int16_t w, int16_t h, uint16_t color)` – Placeholder helper (currently unused).

:::warning
`fillRect()` is a stub that does not draw pixels. Use the Adafruit GFX primitives directly (`display.fillRect`) when you need to render rectangles outside the existing helpers.
:::
