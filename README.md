# iOS Interview Questions – Cheat Sheet

A revision guide for iOS developer interviews: **515 questions with interview-ready answers**, split into 25 topics. It covers Swift, concurrency, UIKit and SwiftUI, architecture, system design and what's new in iOS 26 and Swift 6.

Every answer is hidden in a collapsible block, so you can **test yourself first** and then click to check.

> **📈 [Open the interactive study tracker](https://akshayashokcode.github.io/iOS-Interview-Questions-cheat-sheet/)**: mark questions ✓ Done or ↻ Revise, see your progress per topic, search every question, listen to answers read aloud, and continue where you left off.
>
> **No account needed.** Your progress saves automatically on your device. Want to keep it safe and in sync across your devices? Sign in (optional).

> Companion repo: [Android Interview Questions – Cheat Sheet](https://github.com/AkshayAshokCode/Android-Interview-Questions-cheat-sheet)

---

## How to use this repo

1. **Pick a topic** from the index below.
2. **Read the question, answer it out loud**, then expand **Answer** to compare.
3. Re-read the questions you got wrong. Use the [Last-Minute Revision](#-last-minute-revision-one-liners) list on interview day.

> Tip: interviewers care about **why** and **trade-offs** as much as **what**. Mention when you'd choose one approach over another.

---

## Topics

### 🟣 Swift Language

| # | Topic | Questions | Key areas |
|---|---|---|---|
| 01 | [Swift Fundamentals](topics/01-swift-basics.md) | 43 | Struct vs class, CoW, optionals, closures, escaping, enums, errors, access control |
| 02 | [Swift Advanced](topics/02-swift-advanced.md) | 31 | POP, associated types, `some` vs `any`, generics, dispatch, property wrappers, macros, Codable |
| 03 | [Memory Management (ARC)](topics/03-memory-management.md) | 15 | ARC, strong/weak/unowned, retain cycles, closures & delegates, leak detection |
| 23 | [Objective-C Essentials](topics/23-objective-c.md) | 16 | Messaging & runtime, property attributes, categories, blocks, MRC/ARC, KVC/KVO, Swift interop |

### 🔵 Concurrency

| # | Topic | Questions | Key areas |
|---|---|---|---|
| 04 | [GCD, Operations & Thread Safety](topics/04-gcd-operations.md) | 19 | Queues, sync/async, deadlocks, DispatchGroup, barriers, OperationQueue, locks, RunLoop |
| 05 | [Swift Concurrency](topics/05-swift-concurrency.md) | 21 | async/await, Task, TaskGroup, cancellation, actors, MainActor, Sendable, Swift 6/6.2 |
| 06 | [Combine](topics/06-combine.md) | 16 | Publishers, subjects, @Published, operators, schedulers, errors, Combine vs async/await |

### 🟢 Design & Platform

| # | Topic | Questions | Key areas |
|---|---|---|---|
| 07 | [OOP, SOLID & Design Patterns](topics/07-oop-solid-design-patterns.md) | 18 | SOLID, delegation, KVO, NotificationCenter, singleton, coordinator, responder chain |
| 08 | [iOS Fundamentals](topics/08-ios-fundamentals.md) | 26 | App states, AppDelegate vs SceneDelegate, launch sequence, sandbox, universal links |

### 🟠 UI

| # | Topic | Questions | Key areas |
|---|---|---|---|
| 09 | [UIKit](topics/09-uikit.md) | 40 | VC lifecycle, layout cycle, Auto Layout, cell reuse, diffable data sources, animations |
| 10 | [SwiftUI](topics/10-swiftui.md) | 30 | View identity, @State/@Binding/@Observable, data flow, NavigationStack, performance |
| 24 | [Accessibility & Localization](topics/24-accessibility-localization.md) | 10 | VoiceOver, Dynamic Type, SwiftUI accessibility, audits, String Catalogs, formatting, RTL |

### 🟡 Architecture & Data

| # | Topic | Questions | Key areas |
|---|---|---|---|
| 11 | [App Architecture](topics/11-architecture.md) | 16 | MVC, MVVM, VIPER, Clean, TCA, coordinators, UDF, modularization |
| 12 | [DI, SPM & Modularization](topics/12-dependency-injection-modularization.md) | 14 | DI types, composition root, SPM vs CocoaPods, static vs dynamic, XCFrameworks |
| 13 | [Networking](topics/13-networking.md) | 20 | URLSession, network layer design, Codable, token refresh, caching, pinning |
| 14 | [Data Persistence](topics/14-data-persistence.md) | 19 | UserDefaults, Keychain, Core Data stack & concurrency, SwiftData, CloudKit |
| 15 | [Background Tasks & Notifications](topics/15-background-notifications.md) | 14 | BGTaskScheduler, APNs flow, silent push, notification extensions, Live Activities |

### 🔴 Quality, Security & Delivery

| # | Topic | Questions | Key areas |
|---|---|---|---|
| 16 | [Performance & Instruments](topics/16-performance.md) | 14 | Launch time, hangs, hitches, image memory, Instruments, MetricKit, crashes |
| 17 | [Testing](topics/17-testing.md) | 16 | XCTest, Swift Testing, mocks, async tests, URLProtocol, XCUITest, snapshots |
| 18 | [Security & Privacy](topics/18-security-privacy.md) | 19 | Keychain, Secure Enclave, biometrics, App Attest, ATT, privacy manifests, passkeys |
| 19 | [Xcode, Build & Release](topics/19-xcode-build-release.md) | 18 | Schemes/configs, code signing, TestFlight, App Review, phased release, CI/CD, StoreKit 2 |

### ⭐ Interview Rounds

| # | Topic | Questions | Key areas |
|---|---|---|---|
| 20 | [iOS System Design](topics/20-system-design.md) | 16 | Framework + designs: image cache, feed, chat, offline sync, downloader, analytics SDK |
| 21 | [What's New & Trending](topics/21-whats-new-trending.md) | 17 | iOS 17/18/26, Liquid Glass, Foundation Models, App Intents, Swift 6.2, policies |
| 22 | [Coding, Scenarios & Behavioral](topics/22-coding-scenarios.md) | 24 | LRU cache, debounce, parallel fetch, find-the-bug, STAR answers |
| 25 | [Company-Wise Questions](topics/25-company-wise-questions.md) | 23 | What Apple, Google, Meta, Amazon, Uber, Netflix and Indian product companies ask; machine coding; UPI app, quick commerce, Live Activity tracking, super-app, Reels-style feed |

---

## 🗓 Suggested revision plans

**2-week plan (≈1–2 hours/day)**

| Day | Topics |
|---|---|
| 1 | 01 Swift Fundamentals |
| 2 | 02 Swift Advanced |
| 3 | 03 Memory Management + 07 OOP & Patterns + 23 Objective-C (if the role uses it) |
| 4 | 04 GCD & Operations |
| 5 | 05 Swift Concurrency |
| 6 | 06 Combine + 08 iOS Fundamentals |
| 7 | 09 UIKit |
| 8 | 10 SwiftUI + 24 Accessibility & Localization |
| 9 | 11 Architecture + 12 DI & SPM |
| 10 | 13 Networking + 14 Persistence |
| 11 | 15 Background & Notifications + 16 Performance |
| 12 | 17 Testing + 18 Security & Privacy |
| 13 | 19 Build & Release + 20 System Design |
| 14 | 21 What's New + 22 Coding & Behavioral + 25 Company-Wise (your target companies) + Last-Minute Revision |

**2 days before:** Struct vs class, ARC (03), Swift Concurrency (05), SwiftUI (10), UIKit lifecycle (09), Architecture (11), System Design (20) and the list below.

---

## ⚡ Last-Minute Revision (one-liners)

<details>
<summary>Open the quick revision list</summary>

**Swift**
- Structs are value types (copied, no inheritance, no ARC overhead for the value itself); classes are reference types (shared, inheritance, deinit, ARC). Default to structs.
- Collections are CoW: copies share storage until one of them is mutated.
- Optionals are an enum (`.some`/`.none`). Prefer `if let`/`guard let`/`??`/`?.`; avoid `!`.
- An `@escaping` closure outlives the call → write `self` explicitly, consider `[weak self]`.
- `some P` = one hidden concrete type (static dispatch, fast); `any P` = boxed existential (dynamic dispatch).
- Extension-only protocol methods are statically dispatched (the classic gotcha).
- `final`/`private`/structs → static dispatch; class overrides → vtable; `@objc dynamic` → message dispatch.
- Access levels: open > public > package > internal (default) > fileprivate > private.

**Memory**
- ARC frees an object when its strong count hits 0; it can't break cycles.
- weak = optional `var`, set to nil automatically; unowned = non-optional, crashes if the object is gone.
- Leak sources: closures stored by self, strong delegates, Timer targets, NotificationCenter blocks, Combine sinks, long Tasks.
- Detect leaks with the Memory Graph Debugger, Instruments Leaks/Allocations, and `deinit` logs.

**Concurrency**
- UI only on the main thread. `DispatchQueue.main.sync` from the main thread = deadlock.
- Serial vs concurrent is a property of the queue; sync vs async is a property of how you submit.
- DispatchGroup = wait for many tasks; barrier = concurrent reads with exclusive writes; semaphore = limit concurrency.
- async/await suspends without blocking threads (cooperative pool). Never block inside async code.
- `async let` = fixed parallel work; TaskGroup = dynamic; `Task {}` = unstructured (cancel it manually).
- Cancellation is cooperative: check `Task.checkCancellation()` / `Task.isCancelled`.
- Actors serialize access to their state. Watch for **reentrancy** across `await`.
- `@MainActor` = main thread. `Sendable` = safe to share. Swift 6 turns data races into compile errors.
- Continuations bridge callbacks to async; resume exactly once.

**UIKit**
- VC lifecycle: init → loadView → viewDidLoad (once) → viewWillAppear → viewIsAppearing → viewDidLayoutSubviews → viewDidAppear → viewWillDisappear → viewDidDisappear → deinit.
- frame = in the superview's coordinates; bounds = the view's own coordinates.
- setNeedsLayout = layout on the next cycle; layoutIfNeeded = layout now (use it in animation blocks).
- Hugging = resists growing; compression resistance = resists shrinking.
- Cell reuse: reset the state in prepareForReuse, and cancel image loads. Use diffable data sources.
- Responder chain: view → superview → VC → window → application → app delegate.

**SwiftUI**
- Views are cheap struct descriptions; `body` must be fast and pure.
- Identity (structural or explicit `.id`) controls state lifetime; `if/else` creates different identities.
- `@State` = owned local state; `@Binding` = two-way reference; `@Observable` + `@State` (owned) / `@Bindable` (bindings) / `@Environment` (injected).
- `@StateObject` (owns) vs `@ObservedObject` (passed in). Never create an object with `@ObservedObject`.
- Use `.task` for async work (cancelled automatically); `NavigationStack` + `navigationDestination` + `NavigationPath`.
- Modifier order matters; `List` reuses rows; `LazyVStack` doesn't reuse.

**Architecture & Data**
- MVVM: the view model has no UIKit and no reference to the view → testable. Coordinators own navigation.
- VIPER = View, Interactor, Presenter, Entity, Router (lots of boilerplate). TCA = State, Action, Reducer, Effect, Store.
- DI through initializers + protocols; build the graph in a composition root.
- SPM is the standard (CocoaPods trunk is going read-only). Static linking → faster launch.
- URLSession doesn't throw on 4xx/5xx; check the status code. Refresh tokens through an actor with a single in-flight task.
- Secrets → Keychain; preferences → UserDefaults; structured data → SwiftData/Core Data.
- Core Data: contexts are queue-bound; pass `NSManagedObjectID` between threads, not objects.

**Delivery & Quality**
- Push: permission → register → device token → your server → APNs → device. Silent push = `content-available: 1`, not guaranteed.
- Background: BGAppRefreshTask (short), BGProcessingTask (long, deferrable), background URLSession for transfers.
- Launch: fewer dynamic frameworks, defer SDK setup, a light first screen. Hang = main thread blocked for more than 250 ms.
- Images: decoded size = w × h × 4 bytes. Downsample with ImageIO, and decode off the main thread.
- Testing: protocols + mocks, async tests, URLProtocol stubs, Swift Testing `@Test`/`#expect`.
- Signing = certificate + App ID + entitlements + provisioning profile. TestFlight → App Review → phased release over 7 days.
- Privacy: usage descriptions, ATT for tracking, a Privacy Manifest with required-reason APIs, in-app account deletion.

</details>

---

## 📚 Official references

- [Swift Documentation](https://www.swift.org/documentation/) · [The Swift Programming Language](https://docs.swift.org/swift-book/)
- [Apple Developer Documentation](https://developer.apple.com/documentation/) · [WWDC Videos](https://developer.apple.com/videos/)
- [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/) · [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)

---

## 🤝 Contributing

Found a mistake or have a question you were asked in an interview? Open an issue or PR. Keep the format: a numbered `###` question + a `<details><summary>Answer</summary>` block, concise and interview-ready.

⭐ If this helped you prepare, consider starring the repo!
