# ZeroKey USB User Manual

## Overview
ZeroKey USB is a hardware credential manager running ZerokeyOS. It uses a 128×32 OLED display and five capacitive touch pads – Up, Down, Left, Right and Center – for navigation. Each credential stores a site, username and password of up to 16 characters and is protected by a master PIN.

## First-Time Setup
1. Plug the device into a USB port. When no configuration flag is found the setup wizard starts automatically.
2. Move through the introduction screens with **Right** to continue or **Left** to go back.
3. When prompted to **Enter Master PIN**, use **Up/Down** to change the digit, **Right** to insert it and **Left** to erase.
4. Press **Center** to confirm. The device stores a signature of the new PIN, generates an encryption key and then switches to the main screen.

## Startup and PIN Entry
On subsequent boots the splash screen is followed by the PIN screen. Enter the digits as above and confirm with **Center**. After each wrong attempt the device shows a countdown and enforces an increasing delay before allowing another try.

## Main Screen Navigation
- The vertical indicator on the left shows the current credential index.
- **Left/Right** move between credential slots; long‑press skips ten at a time.
- **Down/Up** cycle through the `Site`, `User` and `Pass` fields.
- **Center (short press)** types credentials over USB:
  - from **Index**: types username, a tab and password;
  - from **User**: types only the username;
  - from **Pass**: types only the password.
- To open the menu from the main screen, go to slot `0` and press **Left**.

## Editing Credentials
1. Highlight `Site`, `User` or `Pass` and long‑press **Center** to enter edit mode.
2. Use **Left/Right** to move the cursor or **Down** to reach `Rand` and `Back` options.
3. Press **Right** to open the on‑screen keyboard. Use **Left/Right** to select characters, **Up/Down** to change keyboard page and **Center** to insert. Press **Left** at the top‑left corner to exit the keyboard.
4. Choosing `Rand` fills the field with a random 16‑character value. Selecting `Back` saves the entry and returns to the main screen.

## Menu System
From the main index at slot `0`, press **Left** to open the menu. Navigate with **Up/Down**, select with **Center** and return with **Left**.

### Backup
- **Export** – sends all credentials in CSV format over USB.
- **Import** – waits for data from the web tool and writes it to memory.

### Settings
- **Rotate Screen** – toggles display orientation.
- **Keyboard** – cycles through available keyboard layouts.
- **About** – shows device and project information.

### Danger Zone
- **Reset PIN** – prompts for a new master PIN.
- **Delete Credentials** – wipes all stored entries.
- **Factory Reset** – clears memory and resets configuration; unplug and reconnect to rerun the setup wizard.

## Web Tool
Open `Web/Memory manager  web tool.html` in a Web Serial‑compatible browser.
1. Click **Link Device** to connect.
2. Choose **Export** on the device to populate the table with existing credentials.
3. Edit entries and click **Save Changes** to send them back during an **Import** operation. Each field accepts a maximum of 16 characters.

## Troubleshooting
- **Device not detected** – use a data‑capable USB cable and a compatible browser.
- **Wrong PIN lockout** – wait for the countdown to finish or perform a factory reset.
- **Web tool errors** – ensure that the browser has Web Serial permission.

## FAQ
**What happens if I forget my PIN?**
Use **Factory Reset** to restart the setup wizard; this erases all credentials.

**Can the device type my credentials automatically?**
Yes. Press **Center** on the main screen to type the stored username and password, or select a field to type it individually.

**How do I change the keyboard layout?**
Open **Settings → Keyboard**; each selection cycles to the next layout.
