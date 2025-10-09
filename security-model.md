# Security Model

This document outlines the current security design of ZeroKey USB firmware.

## Cryptography

* Credentials are encrypted using the [AESLib](../ZerokeyOS/libraries/AESLib) library and AES-128 with a 16-byte key derived from the user's PIN. Each database field is decrypted with `aesLib.decrypt(...)`, and PIN-related operations encrypt with `aesLib.encrypt(...)`.
* An initialization vector (IV) is generated and stored in EEPROM to seed encryption and decryption routines.

## PIN Verification Workflow

1. **Signature creation:** A constant 16-byte block is encrypted with AES-128 using the PIN as the key. The first eight bytes of the ciphertext form the verification signature stored in EEPROM.
2. **Verification:** When a PIN is entered, the same block is encrypted again and compared to the stored signature. Any mismatch triggers an increment of the failed-attempt counter.

## Failed-Attempt Controls

* A counter at EEPROM address `0x0002` tracks consecutive failed PIN attempts. The counter saturates at 255.
* The firmware imposes a delay proportional to the counter value before accepting another attempt, providing brute-force protection.

## Backup and Export

* **Export:** `backupAllCredentials()` outputs every credential record in CSV format over the USB serial connection.
* **Import:** `loadAllbackupCredentials()` parses CSV data received over serial to restore stored entries.

## Threat Model

The device protects credentials at rest but assumes a trusted physical environment. An attacker with full device access could attempt PIN guessing or tamper with the firmware. Side-channel attacks, hardware probing, and compromised host machines are out of scope.

## Safeguarding Credentials

* Choose a strong PIN and avoid sharing it.
* Keep the hardware in a secure location to prevent physical tampering or unauthorized use.
* Store backups exported over serial on encrypted media and transfer them over trusted connections.
* Update firmware promptly when security patches become available.
