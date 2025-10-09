---
title: TOTP Engine
description: Storage, parsing, and generation of time-based one-time passwords.
---

ZeroKeyUSB provides built-in TOTP support for each credential slot. Secrets are stored encrypted inside the fourth EEPROM page and accompanied by metadata describing the algorithm and secret length.

## Secret Storage

- Each slot reserves 32 bytes (`TOTP_PAGE_INDEX = 3`) for the encrypted TOTP secret.
- Metadata lives in the configuration block beginning at `CONFIG_TOTP_META_START` with a stride of two bytes: algorithm code and secret length.
- `MAX_TOTP_SECRET_BYTES = 32` bounds the decoded secret size.

```cpp title="zerokey-totp.cpp::writeTotpSecret"
uint8_t plaintext[32];
memset(plaintext, 0, sizeof(plaintext));
memcpy(plaintext, secret_raw, secret_len);
uint8_t padValue = AES_BLOCK - secret_len;
for (uint8_t i = secret_len; i < AES_BLOCK; ++i) {
  plaintext[i] = padValue;
}
if (!zerokeySecurity.encryptPage(plaintext, ciphertext)) {
  return false;
}
zerokeyEeprom.writeEepromPage(credentialPageAddress(index, TOTP_PAGE_INDEX), ciphertext);
zerokeyEeprom.writeTotpMeta(index, algorithmCode, secret_len);
```

Secrets are encrypted using the same AES-128-CBC key and IV as credentials, ensuring a consistent security model.

## Algorithm Codes

`TotpAlgorithmCode` enumerates supported digests:

- `TOTP_ALGO_SHA1`
- `TOTP_ALGO_SHA256`
- `TOTP_ALGO_SHA512`

The parser accepts plain Base32 strings or full `otpauth://` URIs. `parseTotpProvisioningString()` strips whitespace, decodes Base32, and honours directives such as `;algo=SHA256`.

## Code Generation

`generateTotpFromSecret()` computes HOTP counters using a 30-second period (`counter = epoch_seconds / 30`). After calculating the HMAC, it applies dynamic truncation and reduces the result modulo `10^digits` (6–8 digits supported).

```cpp title="zerokey-totp.cpp::generateTotpFromSecret"
uint64_t counter = epoch_seconds / 30ULL;
uint8_t msg[8];
for (int i = 7; i >= 0; --i) {
  msg[i] = (uint8_t)(counter & 0xFF);
  counter >>= 8;
}
switch (algorithmCode) {
  case TOTP_ALGO_SHA256:
    hmacSha256(secret, secret_len, msg, sizeof(msg), hmac);
    hmacLen = 32;
    break;
  case TOTP_ALGO_SHA512:
    hmacSha512(secret, secret_len, msg, sizeof(msg), hmac);
    hmacLen = 64;
    break;
  default:
    hmacSha1(secret, secret_len, msg, sizeof(msg), hmac);
    hmacLen = 20;
    break;
}
uint8_t offset = hmac[hmacLen - 1] & 0x0F;
uint32_t code = ((uint32_t)(hmac[offset] & 0x7F) << 24) |
                ((uint32_t)(hmac[offset + 1] & 0xFF) << 16) |
                ((uint32_t)(hmac[offset + 2] & 0xFF) << 8) |
                ((uint32_t)(hmac[offset + 3] & 0xFF));
return code % pow10(digits);
```

`getTotpCodeForSlot()` wraps the flow by decrypting the secret, calling `generateTotpFromSecret()`, and returning the code to the caller.

## Epoch Handling

- `syncTotpEpoch()` stores the epoch in `estatus` and records when it was updated.
- `updateTotpEpochFromMillis()` advances the counter locally between host syncs.
- `getTotpSecondsRemaining()` returns the remaining seconds in the current 30-second window and is used to drive on-screen countdowns.

When the user requests a TOTP code, the menu sets `waitingForHostTime = true` and prints `REQTIME` over `SerialUSB`. `handleIncomingHostEpoch()` monitors serial input during the polling loop and updates `estatus` once a valid epoch is received. The timestamp is also persisted via `zerokeyEeprom.writeLastTotpEpoch()` for future boots.

## Export and Import

`formatTotpSecretForExport()` retrieves the decrypted secret, encodes it as Base32, and appends `;algo=SHA256` or `;algo=SHA512` when needed. `loadAllbackupCredentials()` recognises CSV rows with a fourth column (TOTP provisioning string) and stores the parsed secret via `writeTotpSecret()`.

:::tip
If the on-device countdown reaches zero without receiving host time, use the `zerokeyusb-time-sync.html` tool to send the epoch automatically whenever “REQTIME” appears.
:::
