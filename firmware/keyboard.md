---
title: Keyboard Emulation
description: USB HID layouts, automatic typing, and on-device layout persistence.
---

ZeroKeyUSB emulates a USB keyboard to type credentials directly into the host. It supports multiple layouts and remembers the selection across reboots.

## Layout Selection

`KeyboardLayoutOption` in `zerokey-globals.h` enumerates the available layouts (US, Danish, German, Spanish, French, Hungarian, Italian, Portuguese, Swedish). `currentKeyboardLayout` persists at EEPROM address `0x003E` and is cycled through **Settings → Keyboard**.

```cpp title="zerokey-menu.cpp::settingsLanguaje"
if (currentKeyboardLayout < LAYOUT_SV_SE) {
  currentKeyboardLayout = (KeyboardLayoutOption)(currentKeyboardLayout + 1);
} else {
  currentKeyboardLayout = LAYOUT_EN_US;
}
zerokeyEeprom.writeKeyboardLayout(currentKeyboardLayout);
updateSettingsLanguageMenuItem();
```

`zerokeyEeprom.writeKeyboardLayout()` writes the selected enum value to EEPROM, and `readKeyboardLayout()` restores it during boot.

## USB HID Typing

`ZerokeyUtils::typeCredential()` begins the HID session with the chosen layout and types site, user, or password buffers at 30 ms intervals per character.

```cpp title="zerokey-utils.cpp::ZerokeyUtils::typeCredential"
void ZerokeyUtils::beginKeyboardForLayout() {
  switch (currentKeyboardLayout) {
    case LAYOUT_EN_US: Keyboard.begin(KeyboardLayout_en_US); break;
    case LAYOUT_DA_DK: Keyboard.begin(KeyboardLayout_da_DK); break;
    case LAYOUT_DE_DE: Keyboard.begin(KeyboardLayout_de_DE); break;
    // ...other layouts...
    default: Keyboard.begin(KeyboardLayout_en_US); break;
  }
  delay(10);
}
```

`typeCredential()` expands the username buffer when it encounters `@`, automatically inserting the current site after the character to support email-style logins. It can type the serial number, username, password, or both username and password separated by a tab.

`typeNumber()` formats integers with leading zeros (up to 10 digits) and types them over HID—useful when sending TOTP codes.

## On-Screen Keyboards

When editing credentials on the device, `ZerokeyDisplay::renderEditScreen()` presents three 32-character on-screen keyboards (`keyboard1–keyboard3`) stored in PROGMEM. Touch input updates `keyboardIndex` and writes characters into `currentSite`, `currentUser`, or `currentPass`. Left, Right, and Center presses navigate between cursor controls, random generation, delete, and the three keyboard pages.

## Error Handling

If a USB error occurs, `ZerokeyUtils::throwErrorScreen()` displays the message on the OLED, prints it to `SerialUSB` when unlocked, and returns the UI to the PIN screen.

:::tip
Switch layouts from the menu before plugging the device into a system with a non-US keyboard. The firmware reinitialises the HID library with the new layout immediately and persists the choice across power cycles.
:::
