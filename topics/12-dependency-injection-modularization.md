# Dependency Injection, SPM & Modularization

[⬅ Back to index](../README.md)

Covers: DI types, composition root, DI containers, protocol-based injection, Swift Package Manager vs CocoaPods vs Carthage, static vs dynamic frameworks, XCFrameworks.

---

### 1. What is Dependency Injection? Why use it?

<details>
<summary>Answer</summary>

DI means **passing an object's dependencies in from outside** instead of creating them inside:

```swift
// ❌ Hidden, untestable
final class ProfileViewModel { private let api = APIClient() }
// ✅ Injected through protocol
final class ProfileViewModel {
    private let service: ProfileServiceProtocol
    init(service: ProfileServiceProtocol) { self.service = service }
}
```

Benefits: **testability** (inject mocks), loose coupling, configurability (staging vs prod), and explicit dependencies.

</details>

### 2. What are the types of DI?

<details>
<summary>Answer</summary>

- **Initializer (constructor) injection** – preferred. Dependencies are required and can be immutable (`let`).
- **Property injection** – `var service: Service!` set after creation. Needed for storyboard-created view controllers, but risks being forgotten.
- **Method injection** – passed into the specific method that needs it.
- **Ambient context** – the SwiftUI `@Environment`, task-local values.

</details>

### 3. What is a composition root?

<details>
<summary>Answer</summary>

The **single place where the object graph is built**, usually at app start (App/SceneDelegate, or an `AppDependencies` container) or in coordinators and factories. Concrete types are chosen here (`URLSessionAPIClient`, `CoreDataStore`), and everything else depends on protocols. It keeps "new-ing" in one place.

</details>

### 4. DI container vs manual DI? Which libraries exist?

<details>
<summary>Answer</summary>

- **Manual DI** (initializers + factories) – compile-time safe, no magic. Enough for most apps.
- **Containers** – register and resolve dependencies:
  - **Swinject** (runtime resolution, so a missing registration crashes at runtime),
  - **Factory** (lightweight, compile-time-safe key paths, popular with SwiftUI),
  - **Needle** (Uber, compile-time generated),
  - **swift-dependencies** (Point-Free, used by TCA, `@Dependency`).

Trade-off: containers reduce wiring boilerplate, but they can hide dependencies and move errors to runtime.

</details>

### 5. How do you inject dependencies into storyboard view controllers?

<details>
<summary>Answer</summary>

Before iOS 13 you used property injection in `prepare(for:sender:)`. Since iOS 13, use `storyboard.instantiateViewController(identifier:creator:)` with a custom `init?(coder:service:)`, which gives you **initializer injection**. Or go programmatic or SwiftUI.

</details>

### 6. How do you make code testable with protocols?

<details>
<summary>Answer</summary>

Define protocols at the boundaries (network, persistence, time, analytics, location), inject them, and provide **mock/stub/fake** implementations in tests:

```swift
protocol Clock { var now: Date { get } }
struct SystemClock: Clock { var now: Date { Date() } }
struct FixedClock: Clock { let now: Date }
```

Even `URLSession` can be abstracted (a protocol with `data(for:)`), or intercepted with a `URLProtocol` subclass.

</details>

---

## Dependencies & Packaging

### 7. Swift Package Manager vs CocoaPods vs Carthage?

<details>
<summary>Answer</summary>

| SPM | CocoaPods | Carthage |
|---|---|---|
| **Apple's official**, integrated into Xcode | Third-party (Ruby) | Third-party |
| `Package.swift`, no workspace changes | Generates a `.xcworkspace`, modifies the project | Builds frameworks; you link them manually |
| Source-based, supports binary targets (XCFrameworks), resources, plugins and macros | Huge legacy library ecosystem | Decentralized, minimal |
| The recommended choice today | **CocoaPods trunk is read-only from late 2026**: no new pods or updates | Rarely used now |

Interview answer: "SPM for new projects; migrate existing CocoaPods dependencies to SPM."

</details>

### 8. What does a Package.swift look like?

<details>
<summary>Answer</summary>

```swift
// swift-tools-version: 6.0
import PackageDescription
let package = Package(
    name: "Networking",
    platforms: [.iOS(.v16)],
    products: [.library(name: "Networking", targets: ["Networking"])],
    dependencies: [.package(url: "https://github.com/apple/swift-log", from: "1.5.0")],
    targets: [
        .target(name: "Networking", dependencies: [.product(name: "Logging", package: "swift-log")]),
        .testTarget(name: "NetworkingTests", dependencies: ["Networking"])
    ]
)
```

`Package.resolved` pins the exact versions (commit it for apps). Use semantic version ranges, `.exact`, branches or revisions.

</details>

### 9. Static vs dynamic libraries/frameworks?

<details>
<summary>Answer</summary>

- **Static library/framework** – the code is **copied into the app binary at link time**. Faster launch (nothing to load at runtime), and dead-code stripping is possible. The same code duplicated into the app and its extensions increases total size.
- **Dynamic framework** – loaded **at runtime by dyld**. It can be **shared** between the app and its extensions (smaller total size), but **each dynamic framework adds launch time**.
- SPM targets are **static by default** (unless the product is declared `.dynamic`).
- **Mergeable libraries** (Xcode 15) build dynamic in debug (fast iteration) and merge into the binary in release (fast launch).

</details>

### 10. What is an XCFramework?

<details>
<summary>Answer</summary>

A bundle containing a framework or library built for **multiple platforms and architectures** (device arm64, simulator arm64/x86_64, macOS, Catalyst). It replaced "fat" universal frameworks, which couldn't contain arm64 for both device and simulator. Use it to distribute **closed-source binary SDKs**, and ship it via SPM `binaryTarget`. XCFrameworks can be signed (Xcode 15).

</details>

### 11. How do modules communicate without depending on each other?

<details>
<summary>Answer</summary>

- **Interface modules** (protocols only) that feature modules depend on, with implementations injected by the app target.
- The app's **coordinator/router** wires the features together.
- **Deep link / URL routing** between features.
- Shared models in a `Core` module.

This avoids circular dependencies and keeps features buildable in isolation.

</details>

### 12. What are SPM plugins and macros?

<details>
<summary>Answer</summary>

- **Build tool plugins** – run during the build (code generation, e.g. SwiftGen, protobuf; linting with SwiftLint).
- **Command plugins** – run on demand (`swift package format`).
- **Macro targets** – Swift macros are distributed as packages (implemented with SwiftSyntax).

</details>

### 13. What is the `package` access modifier?

<details>
<summary>Answer</summary>

`package` (Swift 5.9) makes a declaration visible to all modules **in the same Swift package**, but not to clients outside it. That's useful for sharing internals between targets of a modular package without making them `public`.

</details>

### 14. How do you speed up builds in a large iOS project?

<details>
<summary>Answer</summary>

- **Modularize** (parallel builds, incremental compilation).
- Add explicit types in complex expressions (type-checker cost), and find slow functions with `-Xfrontend -warn-long-function-bodies`.
- Use the **Build Timeline** / build timing summary.
- Avoid heavy build-phase scripts, or add input/output files to them so they're skipped when nothing changed.
- Prefer binary dependencies for heavy third-party code, and use explicitly built modules (Xcode 16) and compilation caching.
- Debug builds: `-Onone`, incremental compilation mode, no dSYM. Use distributed or remote caching (e.g. Tuist, Bazel) in large organizations.

</details>

---

[⬅ Previous: Architecture](11-architecture.md) · [⬅ Back to index](../README.md) · [Next: Networking ➡](13-networking.md)
