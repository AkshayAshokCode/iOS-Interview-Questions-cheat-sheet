# Accessibility & Localization

[⬅ Back to index](../README.md)

Covers: accessibility problems & principles, VoiceOver, labels/traits/hints, Dynamic Type, other accommodations, SwiftUI accessibility, testing; localization with String Catalogs, plurals, formatting, RTL, testing.

---

## Accessibility

### 1. What are the main problems we need to solve when making apps accessible?

<details>
<summary>Answer</summary>

- **Vision:** blind users rely on **VoiceOver**; low-vision users need **Dynamic Type** (large text), **zoom**, sufficient **contrast**, and must not depend on color alone.
- **Motor:** large hit targets (**44×44 pt** minimum), **Switch Control**, **Voice Control** ("tap Send" needs visible labels), and alternatives to complex gestures.
- **Hearing:** captions and subtitles, and visual or haptic alternatives to audio cues.
- **Cognitive:** clear and simple language, consistent navigation, no time pressure, **Reduce Motion**.

Accessibility is also a legal requirement in many markets (ADA, the European Accessibility Act from 2025), and the App Store now shows **Accessibility Nutrition Labels**.

</details>

### 2. How would you explain VoiceOver to a new developer? What do you need to configure?

<details>
<summary>Answer</summary>

**VoiceOver** is iOS's screen reader: users swipe to move between elements, and VoiceOver **speaks each element's description**; double-tap activates it. It reads the **accessibility tree**, not the pixels. For each element, provide:
- **`accessibilityLabel`** – *what* it is ("Play", "Profile photo of Anna"). Don't include the type ("button"); traits handle that.
- **`accessibilityTraits`** – *what kind* (`.button`, `.header`, `.selected`, `.image`, `.link`, `.adjustable`).
- **`accessibilityValue`** – the current value ("50%", "3 of 5 stars").
- **`accessibilityHint`** – the *result* of an action, when it's not obvious ("Opens the article").
- **`isAccessibilityElement`** – `false` for decorative images. Group containers so related info is read once (a cell reading "Anna, 3 new messages, 2 min ago").
- **Custom actions** (`accessibilityCustomActions`) instead of swipe-only actions, `accessibilityElements` for reading order, and `UIAccessibility.post(notification: .announcement / .screenChanged, …)` for dynamic updates.

</details>

### 3. How would you explain Dynamic Type? How do you support it?

<details>
<summary>Answer</summary>

**Dynamic Type** lets users choose their preferred text size system-wide (including much larger **accessibility sizes**). Apps should respect it.
- **UIKit:** `label.font = .preferredFont(forTextStyle: .body)` + `adjustsFontForContentSizeCategory = true`. For custom fonts, use `UIFontMetrics(forTextStyle: .body).scaledFont(for: customFont)`.
- **SwiftUI:** text styles (`.font(.body)`) scale automatically. Use `.font(.custom("Brand", size: 17, relativeTo: .body))` and `@ScaledMetric` for spacing and icon sizes.
- **Layout:** no fixed heights, `numberOfLines = 0`, scroll views, and **switch horizontal layouts to vertical** at accessibility sizes (`traitCollection.preferredContentSizeCategory.isAccessibilityCategory`, or `ViewThatFits`/`dynamicTypeSize` in SwiftUI).
- Test with Xcode **Environment Overrides** and Previews at different sizes.

</details>

### 4. What other accessibility accommodations do you add?

<details>
<summary>Answer</summary>

- **Reduce Motion** (`UIAccessibility.isReduceMotionEnabled` / `@Environment(\.accessibilityReduceMotion)`): replace large animations with fades.
- **Reduce Transparency**, **Increase Contrast** / Differentiate Without Color, **Bold Text**: use system colors and materials, which adapt automatically.
- **Smart Invert**: mark images and videos with `accessibilityIgnoresInvertColors`.
- **Captions/subtitles** for media (AVKit supports them), and haptics as non-audio feedback.
- **Voice Control / Full Keyboard Access:** visible text labels that match `accessibilityLabel`, plus `accessibilityInputLabels` for alternative names.
- **Large Content Viewer** for bar items at large text sizes (`UILargeContentViewerInteraction`).
- Concrete stories for interviews: "made our feed cells a single VoiceOver element with custom actions for like/share", "fixed layouts truncating at AX5 text sizes".

</details>

### 5. How do you make SwiftUI views accessible?

<details>
<summary>Answer</summary>

```swift
Image(systemName: "heart.fill").accessibilityLabel("Favorite")
Image("divider").accessibilityHidden(true)
HStack { Image(…); Text(name); Text(time) }.accessibilityElement(children: .combine)
Text("Settings").accessibilityAddTraits(.isHeader)
Slider(value: $volume).accessibilityValue("\(Int(volume * 100)) percent")
Row(item).accessibilityAction(named: "Delete") { delete(item) }
CustomStepper().accessibilityAdjustableAction { direction in … }
.accessibilitySortPriority(1)  ·  .accessibilityRepresentation { Toggle(…) }  ·  .accessibilityIdentifier("loginButton") // for UI tests
```

Standard controls (`Button`, `Toggle`, `Label`) are accessible by default. Prefer them over tap gestures on plain views, because they provide the correct traits and actions.

</details>

### 6. How do you test accessibility on iOS?

<details>
<summary>Answer</summary>

- **Manual:** turn on **VoiceOver** (the Accessibility Shortcut is a triple-click of the side button) and navigate every screen. Test the largest **Dynamic Type** sizes, **Voice Control**, Switch Control, Reduce Motion and dark mode with high contrast.
- **Accessibility Inspector** (Xcode → Open Developer Tool): inspect elements, run an **audit** (contrast, hit areas, missing labels, Dynamic Type issues).
- **Automated:** XCUITest **`try app.performAccessibilityAudit()`** (iOS 17+) runs audits in UI tests. Snapshot tests at multiple content size categories.
- Include accessibility in the **definition of done** and in code review.

</details>

---

## Localization & Internationalization

### 7. How do you localize an iOS app?

<details>
<summary>Answer</summary>

- **String Catalogs** (`.xcstrings`, Xcode 15+) are the modern approach. Xcode **automatically extracts** strings from code (`String(localized:)`, SwiftUI `Text("Hello")` literals, `LocalizedStringResource`) and from Info.plist, and tracks translation state (new, stale, translated). They replace `.strings`/`.stringsdict` files.
- In code: `String(localized: "welcome_message", defaultValue: "Welcome, \(name)!", comment: "Home screen greeting")`. SwiftUI `Text` literals are localized automatically.
- **Plurals and device variations** are built into String Catalogs ("1 item" / "%lld items", per-language plural rules like Arabic's six forms).
- Localize **Info.plist strings** (permission descriptions), images where needed, App Store metadata and the app name.
- Export and import **XLIFF/catalogs** for translators. Xcode 26 can generate comments for translators automatically.
- Never concatenate localized fragments. Use full sentences with placeholders (word order differs between languages).

</details>

### 8. How do you format dates, numbers and currencies for different locales?

<details>
<summary>Answer</summary>

**Never hardcode formats.** Use locale-aware formatters:

```swift
date.formatted(date: .abbreviated, time: .shortened)          // "Nov 6, 2026 at 3:41 PM" / "6 nov 2026, 15:41"
price.formatted(.currency(code: "INR"))                        // ₹1,234.50 – grouping per locale
0.25.formatted(.percent)                                       // 25%
Measurement(value: 5, unit: UnitLength.kilometers).formatted() // "3.1 mi" for US users
date.formatted(.relative(presentation: .named))                // "yesterday"
```

The legacy equivalents are `DateFormatter` (with `dateStyle`/`setLocalizedDateFormatFromTemplate`), `NumberFormatter`, `MeasurementFormatter` and `RelativeDateTimeFormatter`. They're **expensive to create**, so cache them. Store dates in **UTC/ISO-8601**, and display them in the user's time zone and calendar (some users use non-Gregorian calendars). Use `Locale.current` and respect the region vs language settings.

</details>

### 9. How do you support right-to-left (RTL) languages?

<details>
<summary>Answer</summary>

- Use **leading/trailing** constraints (not left/right) in Auto Layout. UIKit and SwiftUI then **mirror layouts automatically** for Arabic and Hebrew.
- SF Symbols and directional images: use symbols that flip automatically, or set `imageFlippedForRightToLeftLayoutDirection()` / asset catalog "Direction: Left to Right, Mirrors". Don't flip non-directional content (media play icons, logos, clocks).
- Text alignment `.natural`, and `semanticContentAttribute` for exceptions (e.g. a phone number field forced LTR).
- SwiftUI: `@Environment(\.layoutDirection)`, `.flipsForRightToLeftLayoutDirection(true)`.
- Test with **"Right-to-Left Pseudolanguage"** in scheme options, plus real Arabic/Hebrew locales.

</details>

### 10. How do you test localization?

<details>
<summary>Answer</summary>

- **Scheme → App Language/Region**, or launch arguments (`-AppleLanguages (de)`), to run in a specific language without changing the device.
- **Pseudolanguages**: **Double-Length** (catches truncation and fixed widths), **Accented**, **Bounded String**, **Right-to-Left**.
- **Previews** with `.environment(\.locale, Locale(identifier: "ar"))`, and snapshot tests per locale.
- Check the String Catalog for **untranslated or stale** entries, and add CI checks for missing translations.
- **Per-app language** (iOS 13+): users can pick an app-specific language in Settings. Support it by listing the localizations in the project; don't build in-app language switchers that fight the system.

</details>

---

[⬅ Previous: Objective-C Essentials](23-objective-c.md) · [⬅ Back to index](../README.md) · [Next: Company-Wise Questions ➡](25-company-wise-questions.md)
