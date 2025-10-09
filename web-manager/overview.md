---
title: Web Manager Overview
description: Browser tools for credential backups and time synchronisation.
---

ZeroKeyUSB ships with browser-based utilities located under the `Web/` directory. They use the Web Serial API to communicate with the device at 115200 baud—no drivers or native applications are required.

## Credentials Manager

`Memory manager  web tool.html` presents a Bootstrap interface for viewing and editing credentials. Key behaviours:

- **Link Device** – Requests a serial port, opens it at 115200 baud, and begins streaming lines through a `LineBreakTransformer`.
- **Read flow** – Sends `R` to the device (after the user selects **Export** on ZeroKeyUSB). The firmware responds with the record count followed by CSV lines (`index,site,user,pass[,totp]`). Each row becomes an editable table entry limited to 16 characters per field.
- **Write flow** – Clicking **Save Changes** sends the number of rows and the CSV payload expected by `ZerokeySecurity::loadAllbackupCredentials()`. The page waits for the user to trigger **Import** on the device before transmitting.
- **TOTP support** – The table includes a “TOTP Secret” column. Users can enter Base32 strings or full `otpauth://` URIs; the device parser extracts the secret and algorithm code.
- **CSV utilities** – Optional buttons enable importing/exporting CSV files for offline backups.

Buttons for row reordering and password visibility help manage large credential sets. All processing happens locally inside the browser.

## Time Synchronisation Tool

`zerokeyusb-time-sync.html` automates the epoch handshake required for TOTP generation:

- Opens a serial connection and listens for lines containing `REQTIME` or `TIME?`.
- Displays the current UTC epoch and pushes it to the device whenever the request phrase appears or the user clicks **Enviar hora UTC**.
- Logs all serial traffic in a scrollable pane for troubleshooting.

The script writes newline-terminated ASCII epochs using `TextEncoderStream`, matching the expectations of `handleIncomingHostEpoch()` in the firmware.

:::warning
Browsers only expose Web Serial on secure contexts (`https://` or `chrome://`). When testing locally, open the HTML files from disk using a Chrome or Edge build that enables `file://` access to Web Serial, or host them via `http://localhost`.
:::

## Expected Device Workflow

1. Unlock the device and navigate to **Backup → Export** (for reading) or **Backup → Import** (for writing).
2. Open the credentials manager and click **Link Device** to choose the correct serial port.
3. Trigger the corresponding action on the device. The webpage mirrors progress in its debug console.
4. For TOTP, keep the time sync tool open. It automatically responds when the firmware prints `REQTIME` from either the menu or the main TOTP screen.

Both tools rely on the same serial protocol used by the desktop backup scripts, making them safe to use alongside command-line utilities.
