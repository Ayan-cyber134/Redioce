# SecureAuth

**A military-grade, fully offline TOTP/HOTP authenticator for Android 8.0+**

SecureAuth is a privacy-first two-factor authentication app that never touches the network, encrypts everything at rest with hardware-backed keys, and provides authenticated encrypted backups protected by a one-time 256-bit recovery key.

---

## Table of Contents

- [Features](#features)
- [Security Architecture](#security-architecture)
- [Screenshots](#screenshots)
- [Installation](#installation)
- [Usage](#usage)
- [Backup & Restore](#backup--restore)
- [Threat Model](#threat-model)
- [Contributing](#contributing)
- [License](#license)

---

## Features

### Core 2FA
- **TOTP (RFC 6238)** — Time-based one-time passwords with SHA1/SHA256/SHA512, 6–8 digits, configurable period
- **HOTP (RFC 4226)** — Counter-based one-time passwords with manual or automatic counter increment
- **QR code scanning** — CameraX + ML Kit for offline barcode detection
- **Manual entry** — Full `otpauth://` URI parsing and manual key entry with strict validation
- **Google Authenticator compatible** — Codes verify against any TOTP RFC 6238 implementation

### Security
- **Encrypted database** — SQLCipher (AES-256) with a Keystore-backed passphrase
- **Hardware-backed keys** — Android Keystore with StrongBox preference on supported devices
- **Hardware attestation** — Unforgeable boot state verification via TEE-signed certificates
- **Encrypted backups** — Argon2id key derivation + AES-256-GCM, unique 256-bit key per backup
- **Biometric app lock** — Class 3 biometrics with device credential fallback
- **Auto-lock** — Configurable re-lock timeout when backgrounded
- **Screenshot blocking** — `FLAG_SECURE` on all activities
- **Clipboard guard** — Auto-clearing OTP codes with sensitivity flag on Android 13+
- **Root/Frida/Xposed detection** — Heuristic checks with hardware-backed authoritative signal

### Privacy
- **Zero network access** — The `INTERNET` permission is not declared; the app is physically incapable of exfiltration
- **No telemetry** — No analytics, no crash reporting, no ads, no trackers
- **No cloud backup** — `allowBackup="false"` and explicit data extraction exclusions
- **Fully offline** — All operations run on-device

### User Experience
- **Material You dynamic theming** — Adapts to the system wallpaper on Android 12+
- **Blur codes** — Privacy mode that hides codes until tapped (blur on Android 12+, hide on older)
- **Search** — Real-time filtering by issuer or label
- **Sort options** — By issuer, recently used, or manual
- **Biometric enrollment awareness** — No prompt for users without biometrics enrolled

---

## Security Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   UI (Jetpack Compose)                  │
│          Material 3 · Dynamic Color · Edge-to-Edge       │
├─────────────────────────────────────────────────────────┤
│                  ViewModels (Hilt)                      │
│     StateFlow · Coroutines · Hilt Navigation            │
├─────────────────────────────────────────────────────────┤
│                Domain (Pure Kotlin)                     │
│    TOTP/HOTP · otpauth:// parser · Backup crypto        │
├─────────────────────────────────────────────────────────┤
│                      Data Layer                         │
│   Room + SQLCipher (AES-256) · DataStore · Tink AEAD    │
├─────────────────────────────────────────────────────────┤
│                    Platform Layer                       │
│  Android Keystore · StrongBox · BiometricPrompt · TEE   │
└─────────────────────────────────────────────────────────┘
```

### Cryptographic Primitives

| Purpose | Algorithm | Parameters |
|---|---|---|
| Database encryption | AES-256-GCM (SQLCipher) | 256-bit key, hardware-backed passphrase |
| Backup encryption | AES-256-GCM | 128-bit tag, 12-byte nonce, 32-byte salt |
| Backup key derivation | Argon2id | 64 MB memory, 3 iterations, 4-way parallelism |
| Keystore wrapping | AES-256-GCM (Tink) | Hardware-backed master key |
| OTP generation | HMAC-SHA1/256/512 | RFC 6238 / RFC 4226 |

### Key Storage

| Key | Storage | Protection |
|---|---|---|
| Tink master key | Android Keystore | Hardware-backed (StrongBox/TEE) |
| Database passphrase | Encrypted with Tink, stored in SharedPreferences | Keystore-protected ciphertext |
| TOTP/HOTP secrets | SQLCipher database | AES-256 at rest |
| Backup key | Never stored | Displayed once to user |

---

## Screenshots

> Screenshots coming soon.

---

## Installation

### From GitHub Releases

1. Download the latest `SecureAuth-vX.Y.Z.apk` from the [Releases page](../../releases)
2. Enable "Install unknown apps" for your browser or file manager
3. Open the APK and tap **Install**
4. Launch SecureAuth and grant camera permission when adding accounts by QR

### From Google Play

> Not yet published. Coming soon.

### Requirements

- **Android 8.0 (API 26)** or higher
- **ARM64 or ARMv7** processor
- **Camera** (optional, for QR scanning)
- **Biometric hardware** (optional, for app lock)

---


## Usage

### Adding an Account

**By QR code:**
1. Tap the **+** button
2. Choose **Scan QR Code**
3. Point your camera at the QR code provided by your service

**Manually:**
1. Tap the **+** button
2. Choose **Enter Key Manually**
3. Fill in:
   - **Type** — TOTP (time-based) or HOTP (counter-based)
   - **Account name** — a label for your reference
   - **Issuer** — the service name
   - **Secret key** — the Base32-encoded secret
   - **Algorithm** — usually SHA1
   - **Digits** — usually 6
   - **Period** — usually 30 seconds

### Generating Codes

Codes are generated automatically and refreshed every 30 seconds (or your configured period). Tap the copy icon to copy a code to the clipboard — it auto-clears after 30 seconds.

### Deleting an Account

Tap the trash icon on any account card. Deletion is permanent and cannot be undone.

---

## Backup & Restore

### Creating a Backup

1. Open **Settings → Export Encrypted Backup**
2. Tap **Create Encrypted Backup**
3. **Save the recovery key immediately** — it is shown only once
4. Choose one of:
   - **Copy** — copies the key to clipboard (persists until overwritten)
   - **Save** — saves the `.sab` file to your chosen location
   - **Share** — shares the file via the system share sheet

### Restoring from Backup

1. Open **Settings → Import Encrypted Backup**
2. Select your `.sab` file
3. Enter the 256-bit recovery key you saved during export
4. Tap **Decrypt and Import**

### Backup Security

- **Encryption**: AES-256-GCM with 128-bit authentication tag
- **Key derivation**: Argon2id (64 MB memory, 3 iterations, 4-way parallelism)
- **Key space**: 2^256 ≈ 1.16 × 10^77 combinations
- **Salt**: 32 random bytes per backup
- **Nonce**: 12 random bytes per encryption
- **Wrong password behavior**: Returns null; no oracle for attackers

**If you lose the recovery key, the backup cannot be recovered.** There is no backdoor, no master key, and no recovery service.

---

## Threat Model

### What SecureAuth Protects Against

| Threat | Mitigation |
|---|---|
| Network-based attacks | No `INTERNET` permission |
| Backup theft | AES-256-GCM + Argon2id with 256-bit key |
| Database extraction | SQLCipher AES-256 with Keystore-backed passphrase |
| Screenshot/recording | `FLAG_SECURE` on all activities |
| Clipboard snooping | Auto-clear + `IS_SENSITIVE` flag |
| Shoulder surfing | Blur codes mode |
| Casual root | Heuristic + attestation detection |
| Key theft from device | Android Keystore / StrongBox hardware isolation |

### What SecureAuth Cannot Protect Against

SecureAuth is a **software authenticator**. Its guarantees are limited by the security of the underlying Android platform. The following threats are **out of scope**:

- **Rooted device with Frida/Objection attached** — An attacker with root, physical access, and dynamic instrumentation tooling can hook the app's crypto routines and extract secrets from memory. This is true of every software authenticator, including Aegis, Google Authenticator, and Microsoft Authenticator.
- **Compromised TEE firmware** — If the device's Trusted Execution Environment is compromised, hardware attestation and Keystore guarantees are void.
- **Malicious OS updates** — A custom ROM that bypasses the bootloader chain invalidates hardware attestation.
- **Physical coercion** — No cryptographic system protects against someone forcing you to unlock your device.
- **Weak backup key storage** — If you store the recovery key in an insecure location (e.g., plaintext notes, email drafts), that location becomes the weakest link.

### Honest Security Rating

| Aspect | Rating | Notes |
|---|---|---|
| Secrets at rest | **Very Good** | SQLCipher + Keystore-backed passphrase |
| Backup encryption | **Excellent** | Argon2id + AES-256-GCM + 256-bit key |
| Network attack surface | **Excellent** | Zero network access |
| Tamper detection | **Good** | Hardware attestation is authoritative |
| Resistance to rooted attackers | **Limited** | Inherent to all software authenticators |

---

## Contributing

Contributions are welcome. Before opening a pull request:

1. Read [SECURITY.md](SECURITY.md) to understand the security model
2. Follow the existing code style (Kotlin, Compose, Hilt conventions)
3. Add tests for security-critical changes
4. Update the README if you add user-facing features

For security vulnerabilities, **do not open a public issue**. See [SECURITY.md](SECURITY.md).

---

## License

SecureAuth is released under the **GNU General Public License v3.0**. See [LICENSE](LICENSE) for the full text.

```
Copyright (C) 2026 SecureAuth Contributors

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
GNU General Public License for more details.
```
