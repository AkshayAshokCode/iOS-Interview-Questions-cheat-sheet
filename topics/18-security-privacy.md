# Security & Privacy

[⬅ Back to index](../README.md)

Covers: secure storage, Data Protection, Keychain & Secure Enclave, biometrics, network security, permissions model, App Tracking Transparency, privacy manifests, jailbreak detection, App Attest, passkeys, OWASP risks.

---

## Security

### 1. How do you store sensitive data securely on iOS?

<details>
<summary>Answer</summary>

- **Keychain** for tokens, passwords and keys, with the right accessibility level (`…ThisDeviceOnly` keeps items out of backups).
- **Data Protection** for files (`.completeFileProtection`: files are encrypted while the device is locked).
- **CryptoKit** (AES-GCM, `SymmetricKey`) to encrypt larger data, with the key stored in the Keychain or Secure Enclave.
- **Never** store secrets in UserDefaults, plists, source code, logs or Core Data without encryption.

</details>

### 2. What is iOS Data Protection?

<details>
<summary>Answer</summary>

Files are encrypted with keys derived from the **device passcode** + hardware keys. Protection classes:
- `.complete` – inaccessible while the device is locked.
- `.completeUnlessOpen` – files already open stay accessible after locking (for finishing writes).
- `.completeUntilFirstUserAuthentication` (**default**) – accessible after the first unlock following a boot.
- `.none`.

Choose the class based on whether background tasks need access to the file while the device is locked.

</details>

### 3. What is the Secure Enclave?

<details>
<summary>Answer</summary>

A **hardware security coprocessor** isolated from the main processor. It stores biometric data, and it can **generate and store private keys (P-256) that can never be exported**. Operations like signing happen inside the enclave. Use it through CryptoKit's `SecureEnclave.P256` or the Keychain with `kSecAttrTokenIDSecureEnclave`. It's ideal for device-bound authentication keys.

</details>

### 4. How do you implement Face ID / Touch ID authentication?

<details>
<summary>Answer</summary>

Use the **LocalAuthentication** framework:

```swift
let context = LAContext()
var error: NSError?
guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else { return }
let success = try await context.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
                                               localizedReason: "Unlock your account")
```

You need `NSFaceIDUsageDescription` in Info.plist. For **real security**, don't just check a boolean (it can be bypassed on jailbroken devices). Protect a **Keychain item with access control** (`.biometryCurrentSet`), so the secret is only released after a successful biometric check. `.biometryCurrentSet` also invalidates the item when the enrolled biometrics change.

</details>

### 5. How do you secure network communication?

<details>
<summary>Answer</summary>

- HTTPS only (**ATS** enforces TLS 1.2+).
- **Certificate or public key pinning** for sensitive apps (`NSPinnedDomains` or a URLSession delegate).
- Short-lived tokens; refresh tokens kept in the Keychain.
- **No secrets in the app binary**: they can be extracted with tools like `strings` or class-dump. Use your backend as a proxy.
- Validate server responses, and disable verbose logging in release builds.

See [Networking Q13–14](13-networking.md).

</details>

### 6. How do you protect API keys in an iOS app?

<details>
<summary>Answer</summary>

Assume **anything in the app bundle can be read**. Options:
- Keep the secret **on your server**: the app calls your backend, and the backend calls the third party.
- Restrict keys by bundle ID or platform where the provider supports it.
- **App Attest / DeviceCheck** so the backend can verify that requests come from your genuine app.
- Obfuscating keys (XOR, splitting) only slows attackers down.
- Keep keys out of git (`.xcconfig` files listed in `.gitignore`, CI secrets).

</details>

### 7. What are App Attest and DeviceCheck?

<details>
<summary>Answer</summary>

- **App Attest** (`DCAppAttestService`) – generates a key in the Secure Enclave and **attests to Apple** that it belongs to a genuine instance of your app. Your server then verifies **assertions** on sensitive requests. It fights fraud, modified apps and bots.
- **DeviceCheck** – stores **2 bits per device** on Apple's servers (e.g. "already claimed a promo"), and the data persists across reinstalls.

</details>

### 8. How do you detect jailbroken devices? Should you?

<details>
<summary>Answer</summary>

Heuristics: the presence of files such as `/Applications/Cydia.app` and `/bin/bash`, being able to write outside the sandbox, the `cydia://` URL scheme opening, suspicious dylibs being loaded (Frida, Substrate), or a debugger being attached (`sysctl` `P_TRACED`). **All of these can be bypassed**, so treat them as risk signals (banking or DRM apps), combine them with **server-side** checks (App Attest), and don't rely on them alone.

</details>

### 9. What are common iOS security vulnerabilities (OWASP Mobile)?

<details>
<summary>Answer</summary>

Insecure data storage (secrets in UserDefaults or logs), insecure communication (no TLS validation, ATS disabled), hardcoded credentials, weak authentication (client-side-only checks), insufficient cryptography (custom crypto, static IVs), sensitive data in **screenshots/app switcher snapshots** (blur sensitive screens on `sceneWillResignActive`), the **pasteboard** (use `UIPasteboard` expiration or local-only), keyboard caching (`isSecureTextEntry`, `autocorrectionType = .no`), unvalidated **deep link** parameters, WebView JavaScript bridges exposing native functions, and vulnerable third-party SDKs.

</details>

### 10. How do you secure WKWebView usage?

<details>
<summary>Answer</summary>

Load only trusted HTTPS content, validate navigation in `decidePolicyFor navigationAction`, keep `WKScriptMessageHandler` bridges minimal and validate the messages, disable JavaScript when it isn't needed (`WKWebpagePreferences.allowsContentJavaScript`), never inject untrusted strings into `evaluateJavaScript`, and use `WKWebsiteDataStore.nonPersistent()` for sensitive sessions. Never use `UIWebView` (removed).

</details>

### 11. What are passkeys? How do you implement them?

<details>
<summary>Answer</summary>

Passkeys are **passwordless, phishing-resistant credentials** based on WebAuthn/FIDO2 public-key cryptography. The private key is stored in **iCloud Keychain** (synced end-to-end encrypted across the user's devices), and the server stores only the public key. Implement them with **AuthenticationServices** (`ASAuthorizationPlatformPublicKeyCredentialProvider`) + **Associated Domains** (`webcredentials:`). iOS 26 adds an account creation API that makes it easier to create passkeys at sign-up. Also mention **Sign in with Apple**: under Guideline 4.8, apps that use third-party or social login must also offer an *equivalent privacy-focused login option*. Since 2024 that no longer has to be Sign in with Apple specifically, but it's the usual choice.

</details>

---

## Privacy

### 12. How does the iOS permission model work?

<details>
<summary>Answer</summary>

Sensitive resources (camera, microphone, photos, location, contacts, Bluetooth, motion, health, local network, tracking) need **user authorization at runtime**, and a **usage description string** in Info.plist (otherwise the app **crashes** when it accesses the resource). Best practices:
- Ask **in context**, when the user triggers the feature.
- Optionally show a pre-permission explanation screen first.
- Check the status before asking, and handle **denied/restricted** states gracefully (link to Settings with `UIApplication.openSettingsURLString`).
- **You can only show the system prompt once**: after a denial, only the user can change it in Settings.

</details>

### 13. What are the location and photo permission options?

<details>
<summary>Answer</summary>

- **Location:** When In Use / Always / **Allow Once**, and **Precise vs Approximate** (iOS 14; `accuracyAuthorization`, `requestTemporaryFullAccuracyAuthorization`). Always permission is requested after When In Use.
- **Photos:** full access, **limited access** (selected photos, iOS 14; `PHPickerViewController` / `PhotosPicker` need **no permission at all**), or add-only access. iOS 18 also added **limited contacts access**.

</details>

### 14. What is App Tracking Transparency (ATT)?

<details>
<summary>Answer</summary>

Since iOS 14.5, apps **must ask permission** (`ATTrackingManager.requestTrackingAuthorization`) before **tracking users across apps and websites** owned by other companies, or accessing the **IDFA**. You need `NSUserTrackingUsageDescription`. If the user denies (or never sees the prompt), the IDFA is all zeros. Fingerprinting to get around it is prohibited. Attribution alternatives: **SKAdNetwork / AdAttributionKit**.

</details>

### 15. What is a Privacy Manifest?

<details>
<summary>Answer</summary>

`PrivacyInfo.xcprivacy` (required since **May 2024** for App Store submissions) declares:
- the **data types collected** and their purposes,
- whether the app **tracks** users, and its tracking domains,
- **Required Reason APIs** used (e.g. `UserDefaults`, file timestamps, system boot time, disk space) with an approved reason code.

**Commonly used third-party SDKs** must include their own manifests (and signatures). Xcode aggregates them into a **privacy report**, which helps you fill in the App Store **privacy nutrition labels**.

</details>

### 16. How do you protect user privacy in your app's design?

<details>
<summary>Answer</summary>

Data minimization (collect only what's needed), on-device processing where possible, clear consent, an **account deletion option in the app** (an App Store requirement for apps with account creation), secure deletion, anonymizing analytics, no PII in logs (`os_log` privacy markers such as `\(email, privacy: .private)`), and respecting system settings (Limit IP tracking, Private Relay).

</details>

### 17. How do you hide sensitive content in the app switcher?

<details>
<summary>Answer</summary>

When the scene resigns active (`sceneWillResignActive`, or `scenePhase == .inactive`), overlay a blur or placeholder view, and remove it when the scene becomes active again. The system takes the snapshot for the app switcher when the app goes to the background.

</details>


---

## Cryptography Basics

### 18. Hashing vs encryption vs encoding? How do you compute a secure hash in Swift?

<details>
<summary>Answer</summary>

| | Purpose | Reversible? | Key? | Examples |
|---|---|---|---|---|
| **Encoding** | Change the representation (transport/storage) | Yes, by anyone | No | Base64, URL encoding, UTF-8 |
| **Encryption** | **Confidentiality** | Yes, with the key | Yes | AES-GCM, ChaChaPoly, RSA/ECC |
| **Hashing** | **Integrity/fingerprint**, one-way | No | No (HMAC uses one) | SHA-256, SHA-512 |

```swift
import CryptoKit
let digest = SHA256.hash(data: Data("hello".utf8))
let hex = digest.map { String(format: "%02x", $0) }.joined()

let key = SymmetricKey(size: .bits256)
let mac = HMAC<SHA256>.authenticationCode(for: data, using: key)   // keyed integrity
```

Base64 is **not** security. MD5/SHA-1 are broken for security purposes (`Insecure.MD5` exists in CryptoKit only for legacy compatibility). Passwords should be hashed **on the server** with slow, salted algorithms (bcrypt/Argon2), never stored on the device.

</details>

### 19. Symmetric vs asymmetric encryption? How does CryptoKit help?

<details>
<summary>Answer</summary>

- **Symmetric** (AES-GCM, ChaChaPoly) – the **same key** encrypts and decrypts. **Fast**, so it's used for bulk data (encrypting files or a local DB). The challenge is protecting and sharing the key (keep it in the Keychain or Secure Enclave).
- **Asymmetric** (P-256/Curve25519, RSA) – a **public key** encrypts or verifies, and a **private key** decrypts or signs. Slower, but it solves **key exchange** and enables **signatures** (code signing, passkeys, App Attest).
- **Hybrid in practice:** TLS and end-to-end messaging use asymmetric key agreement (ECDH) to derive a **symmetric session key**, then AES for the data.

**CryptoKit** (iOS 13+) gives a safe, Swift-native API: `AES.GCM.seal/open`, `ChaChaPoly`, `Curve25519.KeyAgreement`, `P256.Signing`, `HKDF`, `SecureEnclave.P256` (non-exportable hardware keys), and **post-quantum** algorithms (ML-KEM, ML-DSA) added in iOS 26. Never implement your own crypto.

</details>

---

[⬅ Previous: Testing](17-testing.md) · [⬅ Back to index](../README.md) · [Next: Xcode, Build & Release ➡](19-xcode-build-release.md)
