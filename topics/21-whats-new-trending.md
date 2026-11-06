# What's New & Trending in iOS

[⬅ Back to index](../README.md)

Interviewers often ask *"What's new in iOS/Swift?"* or *"Which recent APIs have you adopted?"*. This file covers the trending topics from roughly 2023 onward. Check the latest WWDC sessions and release notes before an interview.

---

## OS Releases

### 1. What were the highlights of iOS 17 for developers?

<details>
<summary>Answer</summary>

- **SwiftData** (Swift-native persistence) and the **Observation framework** (`@Observable`).
- **Interactive widgets** (buttons and toggles running App Intents), widgets on the iPad Lock Screen and StandBy.
- Swift **macros** (Swift 5.9), `#Preview`, String Catalogs (`.xcstrings`).
- SwiftUI: scroll APIs (`scrollTargetBehavior`, `scrollPosition`), `PhaseAnimator`/`KeyframeAnimator`, `containerRelativeFrame`, inspector.
- `viewIsAppearing`, custom traits and `registerForTraitChanges` in UIKit.
- `CLMonitor` / `CLLocationUpdate` async location APIs, TipKit, staged Core Data migrations, the Privacy Manifest (mandatory from 2024).

</details>

### 2. What were the highlights of iOS 18?

<details>
<summary>Answer</summary>

- **Apple Intelligence** (iOS 18.1+): Writing Tools (works automatically in standard text views), Image Playground, Genmoji, and Siri improvements driven by **App Intents**.
- **Controls** in Control Center, on the Lock Screen and the Action button (WidgetKit + App Intents).
- Home screen customization (dark and tinted icons).
- SwiftUI: zoom navigation transitions, the `Tab` API and sidebar-adaptable tab views, mesh gradients, the `@Entry` macro, custom containers.
- SwiftData: `#Index`, `#Unique`, history tracking, custom data stores.
- **Swift 6** with data-race safety, and **Swift Testing**.
- **Limited contacts access**, an accessory setup kit, locked and hidden apps, Translation framework APIs.

</details>

### 3. What is iOS 26? What's Liquid Glass?

<details>
<summary>Answer</summary>

At WWDC 2025, Apple switched to **year-based version numbers** (iOS 26, macOS Tahoe 26, and so on, all aligned), and introduced the **Liquid Glass** design language: translucent, refractive material for controls, tab bars, toolbars and navigation, with content flowing beneath them.

For developers:
- Apps built with the **iOS 26 SDK adopt Liquid Glass automatically** in standard UIKit and SwiftUI components (bars, sheets, buttons). Audit custom backgrounds and colors that may conflict with it.
- New APIs: `.glassEffect()`, `GlassEffectContainer`, `.buttonStyle(.glass)`, `UIGlassEffect`, tab bars that minimize on scroll, `.tabViewBottomAccessory`.
- Updated app icon design (layered icons, created with **Icon Composer**).
- A temporary opt-out key existed for the transition period, but it isn't intended as a long-term option.

Other iOS 26 items: the **Foundation Models framework**, a SwiftUI native `WebView`, rich-text `TextEditor`, `BGContinuedProcessingTask`, the Declared Age Range API, and Live Activities on CarPlay and Mac.

</details>

---

## AI on Apple Platforms

### 4. What is the Foundation Models framework?

<details>
<summary>Answer</summary>

The Foundation Models framework (iOS 26) gives apps access to Apple's **on-device large language model** (the one behind Apple Intelligence). It's **private, offline and free** (no API cost):

```swift
import FoundationModels

@Generable
struct TripIdea {
    @Guide(description: "A short catchy title") var title: String
    var activities: [String]
}

let session = LanguageModelSession(instructions: "You are a travel assistant.")
let idea = try await session.respond(to: "Plan a day in Kyoto", generating: TripIdea.self)
```

Features: **guided generation** (`@Generable` produces typed Swift structs instead of raw text), **streaming** partial responses, **tool calling** (the model can call your Swift functions), and multi-turn sessions. It runs only on **Apple Intelligence-capable devices** with the feature enabled, so check `SystemLanguageModel.default.availability` and design a fallback. It's good for summarization, extraction, classification and short generation, but it's not a frontier-scale model.

</details>

### 5. How else can you add ML/AI features to iOS apps?

<details>
<summary>Answer</summary>

- **Core ML** – run custom models on device (converted with `coremltools`; uses the Neural Engine).
- **Create ML** – train models (image, text, tabular classifiers) without code.
- **Vision** (text recognition/OCR, barcodes, faces, and new Swift APIs), **Natural Language**, **Speech** (`SpeechAnalyzer`, iOS 26), **Translation**, **Sound Analysis**.
- **App Intents** – expose app actions to Siri, Spotlight, Shortcuts, widgets and Apple Intelligence.
- **Cloud LLMs** through your backend (never embed provider API keys in the app).

Design considerations: privacy (on-device first), latency (streaming UI), cost, safety and guardrails, hallucination handling, and graceful fallbacks.

</details>

### 6. What are App Intents and why are they important now?

<details>
<summary>Answer</summary>

**App Intents** (iOS 16+) define your app's **actions and entities** in Swift (`struct OrderCoffeeIntent: AppIntent`). The system can then surface them in **Siri, Spotlight, Shortcuts, the Action button, Controls, interactive widgets, Live Activities**, and Apple Intelligence (including **Visual Intelligence** and onscreen awareness in recent releases). With `AppShortcutsProvider` they work with **zero user setup**. They're now the central way apps integrate with the system.

</details>

### 7. What AI tooling exists in Xcode?

<details>
<summary>Answer</summary>

Xcode 16 added **predictive code completion** running on device. Xcode 26 added **coding intelligence**: a chat assistant integrated with ChatGPT and other providers (Claude or local models via API keys), code explanation, fix-its for errors, and generated previews, documentation and tests. Be ready to talk about how you use AI tools responsibly (reviewing the output, security, not pasting secrets).

</details>

---

## Swift & Tooling

### 8. What's new in Swift 6.x?

<details>
<summary>Answer</summary>

- **Swift 6 (2024):** **compile-time data-race safety** in the Swift 6 language mode, typed throws (`throws(MyError)`), noncopyable generics, `Synchronization` (`Mutex`, `Atomic`), a unified Foundation (`swift-foundation`), and Embedded Swift.
- **Swift 6.1:** trailing commas everywhere, `nonisolated` on types and extensions, improvements to package traits.
- **Swift 6.2 (2025): "Approachable concurrency"**: opt-in **default main-actor isolation** for app modules, `nonisolated async` functions run on the caller's actor by default, and `@concurrent` to explicitly offload work. Also **`InlineArray`** (fixed-size arrays), **`Span`** (safe, fast access to contiguous memory), strict memory safety checking (opt in), raw identifiers, WebAssembly support, and Swift Testing improvements (exit tests, attachments).

</details>

### 9. What is Swift Testing and why are teams adopting it?

<details>
<summary>Answer</summary>

Apple's modern, open-source testing framework (Xcode 16): `@Test`, `#expect`/`#require`, parameterized tests, traits and tags, and parallel execution by default. It's more expressive than XCTest and gives better failure messages. It coexists with XCTest, so you can migrate incrementally. See [Testing Q3](17-testing.md).

</details>

### 10. What's the state of SwiftUI vs UIKit in 2025–26?

<details>
<summary>Answer</summary>

**SwiftUI is the default for new apps and features.** Apple's new APIs and designs (Liquid Glass, widgets, Live Activities, visionOS) are SwiftUI-first. UIKit is still fully supported, and it's common in large existing codebases and for very custom or performance-critical UI. Most production apps are **hybrid** (UIHostingController / representables). Interviewers expect you to know both, and how to migrate incrementally.

</details>

### 11. What is visionOS / spatial computing?

<details>
<summary>Answer</summary>

visionOS is Apple's OS for **Apple Vision Pro**. Apps use **SwiftUI** windows and volumes, **RealityKit** for 3D content, and ARKit. A **Full/Immersive Space** surrounds the user. Existing iPhone and iPad apps can run as compatible apps. Build with adaptive SwiftUI layouts so they translate well. Recent releases added spatial widgets and better SwiftUI 3D layout APIs.

</details>

### 12. What is Kotlin Multiplatform's relevance to iOS developers?

<details>
<summary>Answer</summary>

KMP lets teams **share business logic** (networking, persistence, domain) written in Kotlin with iOS, while the UI stays native SwiftUI (or uses Compose Multiplatform for iOS, which is stable since 2025). iOS developers consume the shared code as a framework (via SPM or CocoaPods). **Swift Export** (in progress) improves interop beyond the Objective-C bridging. Know its trade-offs: build tooling, debugging and interop rough edges, compared with the benefit of one shared codebase for the logic.

</details>

### 13. What App Store and policy changes should iOS developers know?

<details>
<summary>Answer</summary>

- **Privacy Manifests** and required-reason APIs (since 2024); signatures for third-party SDKs.
- **Account deletion** inside the app is required.
- **EU Digital Markets Act**: alternative app marketplaces and web distribution in the EU, plus alternative payment terms. US court rulings (2025) allow **external purchase links** in US storefronts. These rules keep changing, so check the current guidelines.
- **Accessibility Nutrition Labels** on App Store product pages (announced 2025).
- **Declared Age Range API** to help apps comply with child-safety laws.
- **Minimum SDK requirements**: uploads must be built with a recent Xcode/SDK (Apple raises this every spring: the iOS 18 SDK / Xcode 16 from April 2025, and the iOS 26 SDK / Xcode 26 from April 2026).

</details>

### 14. What are passkeys and why are they trending?

<details>
<summary>Answer</summary>

Passkeys are phishing-resistant, passwordless sign-in credentials synced through iCloud Keychain, supported by Apple, Google and Microsoft. More apps are moving to them as their default login. iOS 26 adds APIs for passkey creation during account sign-up and for credential import/export. See [Security Q11](18-security-privacy.md).

</details>

### 15. How do you stay up to date with iOS development?

<details>
<summary>Answer</summary>

**WWDC sessions** (watch the "What's new in Swift/SwiftUI/UIKit" sessions every year), Apple Developer News and release notes, the Swift Evolution proposals and the Swift Forums, the Swift.org blog, newsletters (iOS Dev Weekly, SwiftLee, Hacking with Swift), Point-Free, Swift by Sundell, podcasts, and building small side projects with new APIs. In the interview, mention a **specific recent API you tried** and what you learned.

</details>


### 16. How has Swift changed since 2014?

<details>
<summary>Answer</summary>

- **Swift 1–2 (2014–15):** introduction; `guard`, `defer`, error handling (`throws`), protocol extensions (POP). Open-sourced in December 2015.
- **Swift 3 (2016):** the "Grand Renaming" API guidelines, a big source-breaking cleanup.
- **Swift 4 (2017):** **Codable**, smart key paths, multi-line strings.
- **Swift 5 (2019):** **ABI stability** (the runtime ships with the OS), `Result`, raw strings. 5.1: module stability, **opaque types (`some`)**, property wrappers, function builders → **SwiftUI**. 5.5: **async/await, actors, structured concurrency**. 5.7: `if let x` shorthand, `any`, primary associated types, regex literals. 5.9: **macros**, `if`/`switch` expressions, parameter packs, ownership (`consuming`/`borrowing`), noncopyable types.
- **Swift 6 (2024):** **compile-time data-race safety**, typed throws, Embedded Swift, the Swift Testing framework, a unified Foundation. **6.2 (2025):** approachable concurrency, `InlineArray`, `Span`, WebAssembly support.

The trend: **safety** (memory, then concurrency), **performance** (ownership, noncopyable types), **expressiveness** (macros, result builders), and growth beyond Apple platforms (Linux, Windows, Android, embedded, server).

</details>

### 17. Native vs Flutter vs React Native vs Kotlin Multiplatform – what would you choose?

<details>
<summary>Answer</summary>

| | Native (Swift) | Flutter | React Native | KMP (+ Compose Multiplatform) |
|---|---|---|---|---|
| Language | Swift | Dart | JS/TS | Kotlin |
| UI | UIKit/SwiftUI (true native, Liquid Glass "for free") | Own renderer (Impeller); not native widgets | Native components through the new architecture | Native SwiftUI, **or** shared Compose UI |
| Sharing | None | UI + logic | UI + logic | Logic first, optional UI |
| New iOS APIs | Day one | Through plugins | Through native modules | Swift interop / expect-actual |
| Adoption | — | Usually a rewrite | Full app or brownfield | **Incremental** |

A good answer: "Native gives the best UX, performance and immediate access to new platform features (widgets, Live Activities, App Intents, Liquid Glass). Cross-platform reduces cost when the UI is mostly shared and the team has the skills. KMP is a pragmatic middle ground for sharing business logic while keeping SwiftUI."

</details>

---

[⬅ Previous: System Design](20-system-design.md) · [⬅ Back to index](../README.md) · [Next: Coding & Behavioral ➡](22-coding-scenarios.md)
