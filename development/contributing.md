---
title: Contributing
description: Workflow and expectations for submitting changes to ZeroKeyUSB.
---

We welcome contributions to ZeroKeyUSB. Follow these guidelines to keep the firmware, tools, and documentation consistent.

## Workflow

1. Fork the repository and create a feature branch from `main`.
2. Make focused commits using the imperative mood (e.g., `Add TOTP screen refresh`). Keep subject lines under 50 characters and leave a blank line before the body.
3. Run relevant checks: compile `ZerokeyOS` for the Arduino Zero, lint JavaScript/HTML files with `prettier`, and format C/C++ sources with `clang-format` as described in `CONTRIBUTING.md`.
4. Push your branch and open a pull request describing the problem and solution. Reference issues with `Fixes #123` when applicable.
5. Respond to review feedback promptly and update the pull request until it is approved.

## Code Style

- Match the surrounding style and use four spaces for indentation.
- Keep firmware modules modular—expose new APIs through headers in `ZerokeyOS/`.
- Reuse helpers such as `zerokeyEeprom.writeEepromPage()` or `zerokeySecurity.encryptPage()` instead of duplicating I²C or AES logic.
- When touching the UI, ensure new states integrate with `programPosition` and update `zerokeyDisplay.drawScreen()`.

## Testing Checklist

- **Firmware** – Compile for the `arduino:samd:arduino_zero_native` FQBN. If you modify touch or TOTP code, verify interactions on hardware when possible.
- **Web tools** – Test Web Serial flows in a Chromium-based browser. Ensure the credential manager still imports/exports CSV files correctly and the time sync tool responds to `REQTIME`.
- **Documentation** – Add front matter to new pages and keep Mintlify callouts formatted correctly.

## Reporting Issues

Use the [GitHub issue tracker](https://github.com/ZeroKeyUSB/zerokeybeta/issues) to report bugs or request features. Include reproduction steps, firmware version (`zerokeyInfo::SOFTWARE_VERSION`), and hardware revisions where relevant.

Thank you for helping improve ZeroKeyUSB!
