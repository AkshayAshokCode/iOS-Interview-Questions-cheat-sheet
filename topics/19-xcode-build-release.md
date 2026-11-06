# Xcode, Build System & App Store Release

[⬅ Back to index](../README.md)

Covers: targets/schemes/configurations, xcconfig, build phases, compiler, code signing & provisioning, entitlements, archiving, TestFlight, App Store review, phased release, CI/CD, versioning, debugging tools.

---

### 1. Project vs Target vs Scheme vs Configuration?

<details>
<summary>Answer</summary>

- **Project** – the container of files, targets and build settings.
- **Target** – a **product** to build (app, extension, framework, test bundle), with its own build settings, phases and Info.plist.
- **Build configuration** – a named set of settings (`Debug`, `Release`, plus custom ones like `Staging`).
- **Scheme** – **which targets to build and how** for each action (Run, Test, Profile, Analyze, Archive), which configuration each action uses, environment variables and launch arguments, diagnostics (sanitizers), and the test plan.

</details>

### 2. How do you manage multiple environments (Dev/Staging/Prod)?

<details>
<summary>Answer</summary>

- Create **build configurations** (Debug-Dev, Debug-Staging, Release-Prod) and **schemes** for each environment.
- Put the per-environment values in **`.xcconfig` files** (`API_BASE_URL`, `PRODUCT_BUNDLE_IDENTIFIER` suffix, app icon set, display name), then expose them through Info.plist (`$(API_BASE_URL)`) and read them with `Bundle.main.object(forInfoDictionaryKey:)`.
- Use **compilation conditions** (`#if DEBUG`, custom `STAGING` flags) sparingly.
- Different bundle IDs let you install several environments side by side.

</details>

### 3. What are build phases?

<details>
<summary>Answer</summary>

The ordered steps a target goes through to build: **Dependencies** (other targets) → **Run Script** (e.g. SwiftLint, code generation; declare input and output files so they're skipped when nothing changed) → **Compile Sources** → **Link Binary With Libraries** → **Copy Bundle Resources** → **Embed Frameworks/App Extensions**. Xcode 15+ enables **user script sandboxing** by default (scripts must declare their inputs and outputs).

</details>

### 4. How does the Swift compilation process work?

<details>
<summary>Answer</summary>

Swift source → **parsing** (AST) → **semantic analysis** / type checking → **SIL** (Swift Intermediate Language: optimizations, ARC optimization, devirtualization, generic specialization) → **LLVM IR** → machine code (object files) → the **linker** (ld/ld-prime) produces the executable → it's **code signed**. Optimization levels: `-Onone` (debug), `-O` (speed), `-Osize`. **Whole Module Optimization** compiles the entire module together for better optimization.

</details>

### 5. How does code signing work? What is a provisioning profile?

<details>
<summary>Answer</summary>

Code signing proves **who built the app** and that it **hasn't been modified**. iOS refuses to run unsigned or tampered code.
- **Certificate** (Apple Development / Apple Distribution) – your identity. The private key lives in your keychain.
- **App ID / Bundle ID** – identifies the app, along with its enabled **capabilities**.
- **Entitlements** – the capabilities the binary requests (push, App Groups, iCloud, Keychain sharing, Associated Domains).
- **Provisioning profile** – ties these together: **certificate(s) + App ID + entitlements + devices** (for development and Ad Hoc). Types: Development, Ad Hoc, App Store, Enterprise (in-house).

At install time, iOS checks the signature, profile and entitlements. Use **automatic signing** for small teams, or **fastlane match** / manual signing in CI.

</details>

### 6. What happens when you archive and upload an app?

<details>
<summary>Answer</summary>

**Product → Archive** builds with the Release configuration and creates an `.xcarchive` (the app + **dSYMs**). In the **Organizer**, you validate it and **distribute** it: App Store Connect (TestFlight/App Store), Ad Hoc, Enterprise or Development. App Store Connect then processes the build (re-signing, app thinning, and symbol upload if enabled) and it becomes available for TestFlight and review.

</details>

### 7. What is TestFlight?

<details>
<summary>Answer</summary>

Apple's beta testing platform.
- **Internal testers** (up to 100 App Store Connect users) – builds are available right after processing, with no review.
- **External testers** (up to 10,000, invited by email or public link) – the first build of a version needs **Beta App Review**.
- Builds expire after 90 days. Testers send feedback and screenshots, and crashes are reported in the Organizer.

</details>

### 8. What happens in App Store Review? Common rejection reasons?

<details>
<summary>Answer</summary>

Apple reviews against the **App Store Review Guidelines**: safety, performance, business, design, legal. Common rejections:
- Crashes and bugs (**Guideline 2.1**), incomplete metadata or placeholder content.
- Missing or vague **permission usage descriptions**.
- **In-app purchase** violations (digital goods must use IAP, with regional exceptions that keep changing, e.g. external links in the US).
- No account deletion option, missing **privacy policy** or inaccurate privacy labels, a missing **Privacy Manifest**.
- Login required for no reason, or no demo account provided for review.
- Copycat or spam apps, minimal functionality ("just a website wrapper").
- Hidden features, or private API usage.

</details>

### 9. What release strategies does App Store Connect offer?

<details>
<summary>Answer</summary>

- **Manual or automatic release** after approval, or release on a scheduled date.
- **Phased release** for automatic updates over **7 days** (1% → 2% → 5% → 10% → 20% → 50% → 100%). It can be paused (for up to 30 days in total) if crash rates spike. Users can still update manually.
- **Expedited review** for critical bug fixes.
- Remote **feature flags** and **force-update** mechanisms (a minimum supported version from your backend) because, unlike the web, you **can't roll back** a shipped binary.

</details>

### 10. What is the difference between version and build number?

<details>
<summary>Answer</summary>

- **Version** (`CFBundleShortVersionString`, `MARKETING_VERSION`) – user-facing, e.g. `2.3.0` (semantic versioning).
- **Build** (`CFBundleVersion`, `CURRENT_PROJECT_VERSION`) – must be **unique and increasing** for each upload within a version. CI usually sets it automatically (`agvtool`, the CI build number).

</details>

### 11. What does an iOS CI/CD pipeline look like?

<details>
<summary>Answer</summary>

Tools: **Xcode Cloud**, GitHub Actions (macOS runners), Bitrise, CircleCI, Jenkins, plus **fastlane** for automation.

On every PR: SwiftLint/SwiftFormat → build → **unit tests** (+ coverage) → snapshot tests → UI tests (maybe nightly) → static analysis.
On merge or tag: bump the build number → archive with **signing from match** or App Store Connect API keys → upload to **TestFlight** (`fastlane pilot`) → upload dSYMs to the crash reporter → release notes → submit for review (`deliver`) → phased release.

Keep secrets (signing keys, API keys) in the CI secret store.

</details>

### 12. What debugging tools in Xcode should you know?

<details>
<summary>Answer</summary>

- Breakpoints: conditional, symbolic, **exception and Swift error breakpoints**, and actions (log without stopping).
- **LLDB** (`po`, `p`, `v`, `expr`, `bt`, `frame variable`).
- **View Hierarchy Debugger** (Auto Layout issues, hidden views), **Memory Graph Debugger** (leaks).
- Environment Overrides (Dynamic Type, dark mode, accessibility), the network link conditioner, and location simulation.
- **Sanitizers** (Address, Thread, Undefined Behavior), the **Main Thread Checker**, Zombie Objects.
- `os_log` / `Logger` with the **Console** app.
- Instruments.

</details>

### 13. What is the difference between `print`, `NSLog`, `os_log`/`Logger`?

<details>
<summary>Answer</summary>

- `print` – stdout only, visible in the Xcode console. Not captured on devices.
- `NSLog` – goes to the system log, but it's slow and synchronous.
- **`Logger` (OSLog)** – fast and structured, with levels (`debug`, `info`, `notice`, `error`, `fault`), subsystems and categories, and **privacy redaction** of dynamic values by default. Viewable in Console.app and in Xcode 15+'s structured console. This is the recommended choice.

</details>

### 14. What are linker issues you might face?

<details>
<summary>Answer</summary>

"Undefined symbols" (a missing framework or target membership), "duplicate symbols" (the same static library linked twice), missing `-ObjC` flags for categories in static ObjC libraries, architecture mismatches for simulator vs device (fixed with XCFrameworks), and "image not found" at runtime (a dynamic framework wasn't embedded).

</details>

### 15. What are App Store Connect features beyond publishing?

<details>
<summary>Answer</summary>

App Analytics, sales and trends, customer reviews (and replies), **in-app purchases and subscriptions** (with StoreKit 2 on the client), promo and offer codes, **Product Page Optimization** (A/B testing store pages), **Custom Product Pages**, in-app events, and the App Store Connect API for automation.

</details>

### 16. What is StoreKit 2?

<details>
<summary>Answer</summary>

StoreKit 2 is the Swift-concurrency based in-app purchase API (iOS 15+): `Product.products(for:)`, `product.purchase()`, **`Transaction.updates`** (listen at launch for renewals, refunds and purchases made on other devices), `Transaction.currentEntitlements`, JWS-signed transactions that can be **verified on the device**, and the App Store Server API and notifications on the backend. Test locally with **StoreKit configuration files** in Xcode, and in the sandbox. SwiftUI offers ready-made views (`StoreView`, `SubscriptionStoreView`).

</details>

---

[⬅ Previous: Security & Privacy](18-security-privacy.md) · [⬅ Back to index](../README.md) · [Next: iOS System Design ➡](20-system-design.md)
