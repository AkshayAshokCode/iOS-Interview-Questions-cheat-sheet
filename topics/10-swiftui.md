# SwiftUI

[⬅ Back to index](../README.md)

Covers: fundamentals, view identity & lifecycle, state management (@State, @Binding, @Observable, @Environment, legacy ObservableObject), data flow, navigation, lists & performance, layout, modifiers, animations, interop, testing.

---

## Fundamentals

### 1. What is SwiftUI? Declarative vs imperative?

<details>
<summary>Answer</summary>

SwiftUI is Apple's **declarative UI framework** (iOS 13+) for all Apple platforms. You describe **what the UI should look like for a given state**, and SwiftUI updates it when the state changes. In imperative UIKit, you create views and **mutate** them manually (`label.text = …`).

Benefits: less code, a **single source of truth** (fewer inconsistent-UI bugs), live **Previews**, cross-platform, and built-in support for Dark Mode, Dynamic Type, accessibility and animations. Trade-offs: some advanced customizations still need UIKit, and behaviour can differ between OS versions.

</details>

### 2. Why are SwiftUI views structs? What is `body`?

<details>
<summary>Answer</summary>

Views are **lightweight value-type descriptions** of UI, not the rendered views themselves. They're cheap to create and get recreated often. SwiftUI diffs these descriptions and updates the underlying render tree (the "attribute graph").
- `body` is a computed property that returns `some View`. It's called whenever the view's dependencies change, so **keep it fast and free of side effects**.
- Because views are structs, persistent state must live in SwiftUI-managed storage (`@State`, `@StateObject`), not in plain properties.

</details>

### 3. What does `some View` mean?

<details>
<summary>Answer</summary>

It's an **opaque return type**: "a specific concrete View type that the compiler knows, but the caller doesn't need to". The real type is a deeply nested generic type (`ModifiedContent<VStack<TupleView<…>>, …>`). This gives static typing and performance without having to spell out the type. All branches must return the same type, which is why `@ViewBuilder` wraps if/else branches in a `_ConditionalContent`. Use `AnyView` only as a last resort, because it erases the type and hurts diffing.

</details>

### 4. What is view identity? Structural vs explicit identity?

<details>
<summary>Answer</summary>

SwiftUI uses identity to decide whether a view is the **same view** across updates (so its state and animations are kept) or a **new view** (its state is reset, with an insertion/removal transition).
- **Structural identity** – the view's type and position in the view hierarchy. Note that `if/else` creates **two different identities**.
- **Explicit identity** – `id` in `ForEach` (Identifiable) or the `.id(_:)` modifier. Changing `.id` forces a **reset** of the view and its state.

Prefer **modifiers with conditional values** (`.opacity(isOn ? 1 : 0)`) over `if/else` branches when you want to keep identity and state.

</details>

### 5. What is the SwiftUI view lifecycle?

<details>
<summary>Answer</summary>

- **Lifetime of identity**: from when a view with a given identity appears in the hierarchy until it's removed. `@State` lives for that duration.
- `onAppear` / `onDisappear` – called when the view appears or disappears (they can run multiple times, e.g. in tabs or navigation).
- **`.task { }`** – starts async work on appear and **cancels it automatically on disappear**. `.task(id: value)` restarts when the value changes.
- `onChange(of:)` – reacts to value changes.
- The `body` struct may be **recreated many times**. Don't do work in `init`.

</details>

---

## State & Data Flow

### 6. What property wrappers manage state in SwiftUI?

<details>
<summary>Answer</summary>

| Wrapper | Owns data? | Use |
|---|---|---|
| `@State` | ✅ | Private, local **value** state of a view (and, on iOS 17+, an `@Observable` model instance the view owns) |
| `@Binding` | ❌ | **Two-way reference** to state owned by a parent |
| `@Observable` class (iOS 17+) | — | Shared model; views track only the properties they read |
| `@Bindable` | ❌ | Create bindings (`$model.name`) to properties of an `@Observable` object |
| `@Environment` | ❌ | Read system values (`\.colorScheme`, `\.dismiss`) or injected objects (`@Environment(Model.self)`) |
| `@StateObject` (legacy) | ✅ | Own an `ObservableObject`, created once |
| `@ObservedObject` (legacy) | ❌ | Observe an `ObservableObject` passed in from outside |
| `@EnvironmentObject` (legacy) | ❌ | `ObservableObject` injected through the environment |
| `@AppStorage` / `@SceneStorage` | ✅ | UserDefaults / per-scene restoration |
| `@FocusState`, `@GestureState`, `@Namespace` | — | Focus, transient gesture state, matched geometry |

</details>

### 7. `@State` vs `@Binding`?

<details>
<summary>Answer</summary>

```swift
struct Parent: View {
    @State private var isOn = false                   // source of truth
    var body: some View { ToggleRow(isOn: $isOn) }    // pass a binding with $
}
struct ToggleRow: View {
    @Binding var isOn: Bool                           // reads & writes parent's state
    var body: some View { Toggle("Wi-Fi", isOn: $isOn) }
}
```

`@State` should be **private** and is initialized only once per view identity (later `init` values are ignored). `@Binding` doesn't own storage.

</details>

### 8. What is the Observation framework (`@Observable`)? How is it better than ObservableObject?

<details>
<summary>Answer</summary>

`@Observable` (iOS 17, a macro) makes a class observable:

```swift
@Observable final class ProfileModel {
    var name = ""
    var posts: [Post] = []
    func load() async { ... }
}
struct ProfileView: View {
    @State private var model = ProfileModel()      // view owns it
    var body: some View { Text(model.name) }       // tracks ONLY `name`
}
```

Advantages over `ObservableObject` + `@Published`:
- **Fine-grained tracking**: a view re-renders only when the properties it **actually reads in `body`** change. With `ObservableObject`, **any** `@Published` change invalidates every observing view.
- No `@Published` needed; it also works with optionals and nested observable objects.
- Simpler wrappers: `@State` (own), a plain property (pass in), `@Bindable` (bindings), `@Environment(Type.self)` (inject).

</details>

### 9. `@StateObject` vs `@ObservedObject`? (Classic bug)

<details>
<summary>Answer</summary>

- **`@StateObject`** – the view **creates and owns** the object. It's created **once** per view identity and survives `body` recomputations.
- **`@ObservedObject`** – the object is **owned elsewhere** and passed in.

**Bug:** `@ObservedObject var vm = ViewModel()` inside a view. Every time the parent re-renders, the view struct is recreated, so a **new ViewModel** is created and the state is lost. Use `@StateObject` when the view creates the object. (With `@Observable`, the equivalent rule is to use `@State`.)

</details>

### 10. How does data flow work in SwiftUI? What is a single source of truth?

<details>
<summary>Answer</summary>

Every piece of data has **one owner** (the source of truth), and everyone else reads it or gets a **binding** to it. Data flows **down** (properties, environment), and changes flow **up** (bindings, closures, model methods). Duplicating state (copying a value into a child's `@State`) causes things to go out of sync. Keep the state as high as needed, and as low as possible.

</details>

### 11. What is the Environment? How do you inject dependencies with it?

<details>
<summary>Answer</summary>

The environment holds values that propagate **down the view tree** implicitly: system values (`\.colorScheme`, `\.locale`, `\.dismiss`, `\.openURL`) and custom ones:

```swift
extension EnvironmentValues { @Entry var analytics: Analytics = .live }  // iOS 18 @Entry macro
ContentView().environment(\.analytics, .mock)
@Environment(\.analytics) private var analytics
```

`@Observable` objects can be injected with `.environment(model)` and read with `@Environment(Model.self)`. It's good for app-wide dependencies, but explicit init injection is more visible.

</details>

### 12. Where should business logic live in SwiftUI? Is MVVM still needed?

<details>
<summary>Answer</summary>

Keep views focused on layout. Put logic in an **`@Observable` model or view model** (testable, no UI code), and data access in services or repositories. Some argue SwiftUI's views already act as view models ("MV" architecture), while others use MVVM or TCA. A good answer: "Simple screens can use `@State` and small models; complex screens get an observable view model with injected dependencies; data access lives in a separate layer." See [Architecture](11-architecture.md).

</details>

---

## Navigation, Lists, Layout

### 13. How does navigation work in SwiftUI (NavigationStack)?

<details>
<summary>Answer</summary>

`NavigationView` is **deprecated**. Use **`NavigationStack`** (iOS 16) with **value-based** navigation:

```swift
@State private var path = NavigationPath()   // or [Route]
NavigationStack(path: $path) {
    List(products) { p in NavigationLink(p.name, value: p) }
        .navigationDestination(for: Product.self) { ProductDetail(product: $0) }
}
// programmatic: path.append(product); path.removeLast(); path = NavigationPath() (pop to root)
```

Benefits: **programmatic navigation**, deep linking (build the path from a URL), and state restoration (`NavigationPath.CodableRepresentation`). `NavigationSplitView` handles multi-column layouts on iPad and Mac. Modals: `.sheet(item:)`, `.fullScreenCover`, `.popover`, `.alert`, `.confirmationDialog`, `.inspector`.

</details>

### 14. List vs LazyVStack vs VStack?

<details>
<summary>Answer</summary>

- **VStack** – creates **all** its children immediately. Fine for small, fixed content.
- **LazyVStack** (inside a ScrollView) – creates children **as they appear**, but doesn't recycle views. Fully customizable styling.
- **List** – backed by UIKit/AppKit collection views, so it **reuses** rows. Gives you swipe actions, selection, edit mode, sections, refresh, and system styling. Best for large data sets with platform behaviour.

`LazyVGrid`/`LazyHGrid` are for grids. Use stable **`id`s** in `ForEach` (never array indices for mutable data).

</details>

### 15. How does SwiftUI layout work?

<details>
<summary>Answer</summary>

There are three steps:
1. The **parent proposes a size** to the child.
2. The **child chooses its own size** (it may ignore the proposal; e.g. `Text` uses its ideal size, `Image` is fixed unless it's `.resizable()`).
3. The **parent places the child** within its coordinate space (according to alignment).

Modifiers like `.frame` and `.padding` wrap views and take part in the negotiation. Tools: `Spacer`, `layoutPriority`, `fixedSize()`, `GeometryReader` (reads the proposed size; use it sparingly), `containerRelativeFrame` (iOS 17), `ViewThatFits`, and the **`Layout` protocol** (iOS 16) for custom layouts.

</details>

### 16. Why does modifier order matter?

<details>
<summary>Answer</summary>

Each modifier **wraps** the view in a new view, so the order changes the result:

```swift
Text("Hi").padding().background(.red)   // red includes padding
Text("Hi").background(.red).padding()   // red only behind text
```

Modifiers are applied from the inside out. Custom modifiers use `ViewModifier` + `.modifier()` (or a `View` extension).

</details>

### 17. How do animations work in SwiftUI?

<details>
<summary>Answer</summary>

- **Implicit:** `.animation(.spring, value: isExpanded)` animates changes caused by that value.
- **Explicit:** `withAnimation(.easeInOut) { isExpanded.toggle() }`.
- **Transitions** for insertion and removal: `.transition(.slide)`, `.move(edge:)`, `.asymmetric`.
- `matchedGeometryEffect` (hero animations), `PhaseAnimator` / `KeyframeAnimator` (iOS 17), the `Animatable` protocol for custom animatable data, and navigation **zoom transitions** (iOS 18).

SwiftUI interpolates between the **before and after states**. Identity changes (`if/else`) trigger transitions instead of interpolation.

</details>

---

## Performance, Interop, Testing

### 18. How do you optimize SwiftUI performance?

<details>
<summary>Answer</summary>

- Keep `body` **cheap and pure**: no formatting allocations, sorting or I/O inside it. Precompute in the model.
- Use **`@Observable`** for fine-grained updates, and split large views into smaller subviews (each subview gets its own invalidation scope).
- Pass only the data a view needs, not whole models.
- Use lazy containers and `List` for big data sets, with **stable identity** (`id`).
- Avoid `AnyView`, and avoid conditional `if/else` that breaks identity unnecessarily.
- Avoid heavy `GeometryReader` usage, and use `.drawingGroup()` for complex vector drawing.
- Debug with `Self._printChanges()` inside `body` (it prints why the view updated), the **SwiftUI instrument** in Instruments (Xcode 26) and the View Body / hitches instruments.

</details>

### 19. How do you use UIKit inside SwiftUI?

<details>
<summary>Answer</summary>

`UIViewRepresentable` / `UIViewControllerRepresentable`:

```swift
struct MapViewWrapper: UIViewRepresentable {
    @Binding var region: MKCoordinateRegion
    func makeUIView(context: Context) -> MKMapView {
        let map = MKMapView(); map.delegate = context.coordinator; return map
    }
    func updateUIView(_ map: MKMapView, context: Context) { map.setRegion(region, animated: true) }
    func makeCoordinator() -> Coordinator { Coordinator(self) }
    final class Coordinator: NSObject, MKMapViewDelegate { /* delegate callbacks → update binding */ }
}
```

The **Coordinator** handles delegates and target-actions. Going the other way, use `UIHostingController` (SwiftUI inside UIKit).

</details>

### 20. How do you handle async work in SwiftUI?

<details>
<summary>Answer</summary>

Use **`.task { await model.load() }`**: it runs on appear, is **auto-cancelled** on disappear, and is main-actor isolated by default. `.task(id:)` re-runs when the ID changes (search queries). `.refreshable { }` gives pull-to-refresh. Show loading, error and empty states from an enum state in the model.

</details>

### 21. How do you test SwiftUI views?

<details>
<summary>Answer</summary>

- Test **models/view models** with unit tests. Most of the logic should be there.
- Use **Previews** (`#Preview`) with mock data in several states (loading, error, Dynamic Type, dark mode).
- **Snapshot tests** (e.g. swift-snapshot-testing) catch visual regressions.
- **XCUITest** for end-to-end flows (use `accessibilityIdentifier`).
- Libraries like ViewInspector for introspection-based unit tests.

</details>

### 22. What are PreferenceKeys and how do they differ from Environment?

<details>
<summary>Answer</summary>

The environment passes data **down** the tree. **PreferenceKeys** pass data **up**, from children to ancestors (e.g. a child reports its size or a title). `navigationTitle` is implemented with preferences. You read them with `.onPreferenceChange`. For geometry, iOS 18's `onGeometryChange` is often simpler.

</details>

### 23. What are common SwiftUI mistakes?

<details>
<summary>Answer</summary>

- Using `@ObservedObject` for objects the view creates (use `@StateObject`/`@State`).
- Doing heavy work in `body` or in view `init`.
- Using array indices as `ForEach` ids.
- Making `@State` non-private or initializing it from parameters (it won't update when the parameter changes).
- Using `if/else` when you meant to toggle a modifier (it resets state and animations).
- Overusing `AnyView` and `GeometryReader`.
- Starting tasks in `onAppear` without cancelling them (use `.task`).
- Huge views (split them into subviews).

</details>

### 24. What's new in SwiftUI recently?

<details>
<summary>Answer</summary>

- **iOS 17:** Observation (`@Observable`), SwiftData integration, `scrollPosition`/`scrollTargetBehavior`, phase and keyframe animators, `containerRelativeFrame`, inspector.
- **iOS 18:** `@Entry` macro, zoom navigation transitions, mesh gradients, `onScrollGeometryChange`, custom containers (`ForEach(subviews:)`), new `Tab` API.
- **iOS 26:** the **Liquid Glass** design (`.glassEffect()`, automatic adoption in system bars and controls), a native `WebView`, rich-text editing with `AttributedString` in `TextEditor`, 3D charts, and SwiftUI performance instruments.

See [What's New](21-whats-new-trending.md).

</details>

---

[⬅ Previous: UIKit](09-uikit.md) · [⬅ Back to index](../README.md) · [Next: Architecture ➡](11-architecture.md)
