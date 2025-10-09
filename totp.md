# ZeroKeyUSB TOTP Support

This firmware stores a per-account TOTP secret alongside the existing site, user, and password pages. Each credential slot now reserves four EEPROM pages (128 bytes) and tracks the TOTP algorithm and raw secret length in the configuration area.

## Provisioning from the host

Send the device a CSV record in the format `index,site,user,pass,otpauth://totp/...` when using the USB import flow. The URI must include a `secret=` parameter (base32 encoded) and may optionally include `algorithm=` (`SHA1`, `SHA256`, or `SHA512`) and `time=`/`timestamp=`/`epoch=` to seed the device clock. The TOTP menu also emits `REQTIME` over USB if no clock is available; reply with a line containing the epoch to update `estatus`. Example:

```
12,Example,alice@example.com,s3cr3t,otpauth://totp/Example:alice?secret=JBSWY3DPEHPK3PXP&algorithm=SHA256&digits=6&time=1700000000
```

The importer decodes the base32 payload, encrypts it with the account PIN key, writes the ciphertext to the fourth page of the credential slot, and updates the metadata table. If `time` (or `timestamp`/`epoch`) is present the global epoch counter is updated for immediate code generation.

## Verifying with RFC 6238

Builds compiled with `ENABLE_TOTP_SELF_TEST` run an embedded test using the SHA-256 vector from RFC 6238 (secret `"12345678901234567890123456789012"`, time step `59s`). The expected TOTP code is `46119246`. You can also call `generateTotpFromSecret()` directly in a development sketch to reproduce the vector.

## EEPROM layout

* Credential size: 4 pages × 32 bytes = 128 bytes per slot.
* Maximum credentials: 62 slots.
* TOTP metadata table: starts at `0x0060`, `CONFIG_TOTP_META_STRIDE = 2` bytes per slot (`algorithm`, `secret_len`).
* TOTP ciphertext page: `credentialPageAddress(index, 3)`.

Only ciphertext and metadata are ever written to EEPROM; plaintext secrets remain in RAM temporarily and are wiped after use. The TOTP menu entry on device uses the cached epoch (`estatus`) to generate six-digit codes and shows a countdown until the 30-second window rolls over.
