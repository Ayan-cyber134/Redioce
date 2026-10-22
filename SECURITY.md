# Security Policy

## Supported Versions

Only the latest stable release receives security updates.

| Version | Supported |
|---|---|
| 2.x.x | ✅ |
| 1.x.x | ❌ |

---

## Reporting a Vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

If you believe you have found a security vulnerability in Redioce, please report it privately via GitHub's [private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing/privately-reporting-a-security-vulnerability) feature or by emailing the maintainers at:

```
support@connecto.int.yt
```

Please include:

- A clear description of the vulnerability
- Steps to reproduce
- The affected version(s)
- Any proof-of-concept or exploit code
- Your assessment of the impact

### What to Expect

| Timeline | Response |
|---|---|
| **Within 72 hours** | Acknowledgement of your report |
| **Within 7 days** | Initial assessment and severity classification |
| **Within 30 days** | Fix or mitigation plan |
| **After fix release** | Public disclosure with credit (if you wish) |

We follow a **90-day coordinated disclosure** policy. If a fix is not available within 90 days of the initial report, we will publicly disclose the vulnerability with a note that no patch is available.

---

## Scope

### In Scope

- Cryptographic implementation flaws (TOTP, HOTP, backup encryption, database encryption)
- Key storage weaknesses (Keystore misuse, key leakage)
- Authentication bypass (biometric lock, app lock)
- Data exfiltration (any path that leaks secrets)
- Backup format vulnerabilities (tampering, weak key derivation)
- Privilege escalation (from unprivileged app to SecureAuth data)
- Hardware attestation bypass

### Out of Scope

The following are **not** considered vulnerabilities in Redioce because they are inherent to any software authenticator running on Android:

- **Attacks requiring a rooted device with physical access and dynamic instrumentation (Frida, Objection, etc.)** — If an attacker can hook the app's process, they can extract secrets from memory. This is true of Aegis, Google Authenticator, Microsoft Authenticator, and every other software 2FA app.
- **Attacks requiring a compromised TEE or bootloader** — If the device's Trusted Execution Environment is compromised, hardware-backed guarantees are void by definition.
- **Attacks requiring the user's device PIN or biometric enrollment** — Physical access to an unlocked device with the user's credentials is not a software defect.
- **Theoretical attacks with no demonstrated PoC** — Speculative "this could be a problem" reports without a working proof of concept will be acknowledged but may be closed as informational.
- **Issues in upstream dependencies (Tink, Bouncy Castle, Argon2, SQLCipher, Room)** — Report these to the respective upstream projects. We will patch when upstream releases a fix.
- **Self-XSS or social engineering** — Attacks requiring the user to be tricked into running malicious code.
- **Denial of service requiring physical access** — E.g., "an attacker with the device in hand can factory reset it."

---

## Threat Model

### Assets Protected

1. **TOTP/HOTP secrets** — Long-term shared secrets used to generate one-time codes
2. **Generated codes** — Short-lived 6–8 digit codes
3. **Backup recovery keys** — 256-bit keys that decrypt exported backups
4. **Database encryption passphrase** — Protects the SQLCipher database

### Trust Boundaries

- **The Android OS and its kernel** — Trusted
- **The device's Trusted Execution Environment (TEE)** — Trusted
- **The Android Keystore / StrongBox** — Trusted
- **The Android camera and QR scanning pipeline** — Trusted for input, but input is validated
- **Any other app on the device** — Untrusted
- **The network** — Untrusted (and unreachable)
- **The file system outside the app sandbox** — Untrusted
- **Cloud backup services** — Untrusted (and disabled)

### Security Guarantees

Redioce guarantees, on a **non-rooted device with a locked bootloader**:

1. Secrets stored in the SQLCipher database cannot be read without first unlocking the device and passing the biometric prompt
2. Backup files cannot be decrypted without the correct 256-bit recovery key
3. No data leaves the device through any network channel
4. Screenshots and screen recordings cannot capture app content
5. Clipboard contents auto-clear within the configured timeout

Redioce **does not guarantee**:

1. Protection against a determined attacker with root, physical access, and dynamic instrumentation tooling
2. Protection against a compromised TEE or bootloader
3. Protection against a malicious OS update or custom ROM
4. Protection against physical coercion of the user
5. Recovery of a backup if the recovery key is lost

---

## Cryptographic Design

### Backup Encryption

**Format:** `MAGIC(4) | VERSION(1) | SALT(32) | NONCE(12) | CIPHERTEXT+TAG`

- **MAGIC**: `"BKUP"` (4 bytes)
- **VERSION**: `0x02` (1 byte)
- **SALT**: 32 bytes from `SecureRandom`
- **NONCE**: 12 bytes from `SecureRandom`
- **Ciphertext**: AES-256-GCM output, includes 128-bit authentication tag

**Key derivation:** Argon2id with the following parameters (OWASP 2025 recommendations):

| Parameter | Value |
|---|---|
| Memory | 64 MB |
| Iterations | 3 |
| Parallelism | 4 |
| Output length | 32 bytes |

**Why Argon2id?** It is a memory-hard function, resistant to GPU, ASIC, and FPGA attacks. Unlike PBKDF2 (which is compute-bound and can be parallelized cheaply on GPUs), Argon2id requires substantial memory per guess, making large-scale parallel cracking economically infeasible.

**Why AES-256-GCM?** It provides authenticated encryption in a single pass — confidentiality, integrity, and authenticity. The 128-bit authentication tag makes forgery computationally infeasible (2^-128 probability of a successful forgery attempt).

### Database Encryption

**Engine:** SQLCipher 4.6.1

**Cipher:** AES-256-CBC with HMAC-SHA512 for authentication

**Key derivation:** PBKDF2-HMAC-SHA512 with 256,000 iterations (SQLCipher 4.x default)

**Passphrase management:**
1. On first launch, 32 random bytes are generated via `SecureRandom`
2. The passphrase is encrypted with Tink AEAD (AES-256-GCM)
3. The Tink master key lives in the Android Keystore (StrongBox preferred, TEE fallback)
4. Only the encrypted ciphertext is stored in `SharedPreferences`
5. At runtime, the ciphertext is decrypted and passed to SQLCipher

**Why this design?** The passphrase never appears in plaintext on disk. An attacker who extracts the SharedPreferences file gets ciphertext they cannot decrypt without the Keystore key, which cannot be extracted from a device with a locked bootloader.

### Keystore Master Key

**Engine:** Google Tink `AndroidKeysetManager`

**Key template:** `AES256_GCM`

**Key URI:** `android-keystore://secureauth_master_key`

**StrongBox preference:** On Android 9+ (API 28), the key is stored in StrongBox if the device supports it; otherwise, the TEE-backed Keystore is used.

**OEM fallback:** On devices where the Keystore throws `UnrecoverableKeyException` (a known OEM bug), the app falls back to a software-backed keyset.

---

## Known Limitations

### Rooted Devices

Redioce detects root via multiple heuristics and via hardware attestation. On a rooted device:

- Heuristic checks (file paths, Magisk package, `su` binary) are bypassable via Magisk DenyList, Shamiko, or Frida hooks
- Hardware attestation via TEE is **not** bypassable on a device with a locked bootloader, because the TEE signs the boot state with a hardware key the normal world never sees

**Current policy:** The app shows a warning but allows continued use. This is consistent with Aegis Authenticator's behavior. A future release may offer a "block on root" option for enterprise deployments.

### Memory Extraction

When the app is running, secrets are decrypted into the process's memory. On a rooted device with Frida or Objection, an attacker can hook:

- `DatabaseKeyManager.getPassphrase()` — to capture the SQLCipher passphrase
- `TotpGenerator.generate()` — to capture the generated code or the secret
- `ClipGuard.copyAndClear()` — to capture codes before they are copied

**Mitigation:** We use `FLAG_SECURE`, root detection, and Frida port scanning. None of these stop a determined attacker with root. This is inherent to all software authenticators.

### Backup Key Loss

If the user loses their backup recovery key, the backup is **permanently unrecoverable**. There is no backdoor, no master key, and no recovery service. This is by design — any recovery mechanism would be an attack vector.

---

## Security Best Practices for Users

1. **Use a strong device PIN or password.** Biometric lock only works if the device itself is locked.
2. **Keep your device updated.** Install Android security patches as soon as they are available.
3. **Do not root your device.** Rooting disables the security guarantees that SecureAuth relies on.
4. **Store your backup recovery key securely.** Use a password manager or a physical safe. Do not email it to yourself or store it in plaintext notes.
5. **Verify the app's signature.** Only install Redioce from official sources (GitHub Releases or Google Play).
6. **Disable OEM Unlocking** in Developer Options if you do not need it.
7. **Do not enable USB debugging** unless you are actively developing.
8. **Verify the app's integrity check** in Settings shows "Device Integrity Verified."

---

## Security-Related Resources

- [OWASP Mobile Application Security Verification Standard (MASVS)](https://mas.owasp.org/MASVS/)
- [OWASP Mobile Application Security Testing Guide (MASTG)](https://mas.owasp.org/MASTG/)
- [RFC 6238 — TOTP](https://datatracker.ietf.org/doc/html/rfc6238)
- [RFC 4226 — HOTP](https://datatracker.ietf.org/doc/html/rfc4226)
- [Android Keystore System](https://developer.android.com/privacy-and-security/keystore)
- [Android Key Attestation](https://developer.android.com/privacy-and-security/security-key-attestation)
- [Google Tink](https://developers.google.com/tink)
- [SQLCipher Design](https://www.zetetic.net/sqlcipher/design/)

---

## Acknowledgements

We thank the following projects for their foundational work:

- **Google Tink** — Hardware-backed AEAD
- **SQLCipher** — Database encryption
- **Argon2Kt** — Argon2id bindings for Android
- **Bouncy Castle** — ASN.1 parsing for attestation
- **Quickie** — QR code scanning
- **kotlin-onetimepassword** — RFC 4226/6238 implementations

---

## Version History of This Document

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2026-10-12 | Initial security policy |
