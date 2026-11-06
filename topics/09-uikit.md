# UIKit – View Controllers, Views, Auto Layout, Lists

[⬅ Back to index](../README.md)

Covers: view controller lifecycle, view lifecycle & rendering, Auto Layout, UITableView/UICollectionView, diffable data sources, navigation, gestures, animations, keyboard handling.

---

## View Controller

### 1. Explain the UIViewController lifecycle.

<details>
<summary>Answer</summary>

1. `init(coder:)` / `init(nibName:bundle:)` – created (from a storyboard or in code).
2. `loadView()` – creates `self.view`. Override it only to provide a custom root view in code (and don't call super then).
3. **`viewDidLoad()`** – called **once** after the view is loaded into memory. One-time setup: subviews, constraints, bindings, initial data loading.
4. **`viewWillAppear(_:)`** – before the view becomes visible, **every time**. Refresh data, start observers, configure the navigation bar.
5. `viewIsAppearing(_:)` (iOS 17, back-deployed to 13) – the view is in the hierarchy with **accurate geometry and traits**. The best place for appearance updates that depend on size.
6. `viewWillLayoutSubviews()` / **`viewDidLayoutSubviews()`** – each layout pass (bounds are final here; may be called many times).
7. **`viewDidAppear(_:)`** – visible on screen. Start animations, analytics and expensive work.
8. **`viewWillDisappear(_:)`** / **`viewDidDisappear(_:)`** – leaving the screen. Stop timers and observers, save state.
9. `deinit` – deallocated (if it's never called, you probably have a leak).

`viewDidUnload` was removed in iOS 6. Views are no longer unloaded on memory warnings.

</details>

### 2. Where should you put code that depends on view size?

<details>
<summary>Answer</summary>

**Not** in `viewDidLoad`: the view hasn't been laid out yet, so its bounds are the storyboard or default size. Use `viewIsAppearing`, `viewDidLayoutSubviews` (beware: it's called many times), `viewWillTransition(to:with:)` for rotation, or `traitCollectionDidChange` / `registerForTraitChanges` (iOS 17) for size class changes. Better still, use Auto Layout so you don't need manual frame math.

</details>

### 3. What are child view controllers / container view controllers?

<details>
<summary>Answer</summary>

Child view controllers let you **compose** screens from smaller, reusable view controllers (which helps avoid Massive View Controllers). Add one in this order:

```swift
addChild(child)
view.addSubview(child.view)   // + constraints
child.didMove(toParent: self)
```

Remove one with `willMove(toParent: nil)` → `view.removeFromSuperview()` → `removeFromParent()`. System containers include `UINavigationController`, `UITabBarController`, `UISplitViewController` and `UIPageViewController`.

</details>

### 4. How do you pass data between view controllers?

<details>
<summary>Answer</summary>

- **Forward:** set properties or inject through the initializer (preferred, since dependencies are explicit). With storyboards, use `prepare(for:sender:)` or `instantiateViewController(identifier:creator:)` (which allows initializer injection).
- **Backward:** delegate protocol, closure callback, or a shared view model/store.
- **App-wide:** NotificationCenter (sparingly), or shared state objects.

With coordinators, the coordinator wires up the data flow between screens.

</details>

### 5. push vs present (modal)?

<details>
<summary>Answer</summary>

- **push** (`navigationController.pushViewController`) – drill-down hierarchical navigation inside a `UINavigationController`, with a back button and swipe-back.
- **present** (`present(_:animated:)`) – modal presentation for a **separate task** (compose, login, picker). Presentation styles: `.pageSheet` (the default card, swipe to dismiss), `.fullScreen`, `.formSheet`, `.overFullScreen`, `.popover`. Sheet detents through `UISheetPresentationController` (`.medium()`, `.large()`, custom).

</details>

---

## Views & Rendering

### 6. Explain the UIView layout & drawing cycle.

<details>
<summary>Answer</summary>

On each run loop iteration, before rendering, UIKit performs three passes:
1. **Update constraints** – `updateConstraints()` (bottom-up). Triggered by `setNeedsUpdateConstraints()`.
2. **Layout** – `layoutSubviews()` (top-down) sets the frames. Triggered by `setNeedsLayout()` (deferred to the next cycle) or `layoutIfNeeded()` (**immediately**, often used inside animation blocks).
3. **Display** – `draw(_:)` for custom drawing. Triggered by `setNeedsDisplay()`.

Then Core Animation commits the layer tree to the **render server**, which composites it on the GPU. You never call `layoutSubviews()` or `draw(_:)` directly.

</details>

### 7. `setNeedsLayout()` vs `layoutIfNeeded()`?

<details>
<summary>Answer</summary>

- `setNeedsLayout()` – marks the view as needing layout. The layout happens on the **next update cycle**, so several changes are batched.
- `layoutIfNeeded()` – forces **any pending layout immediately**, synchronously. It's typically used to animate constraint changes:

```swift
heightConstraint.constant = 200
UIView.animate(withDuration: 0.3) { self.view.layoutIfNeeded() }
```

</details>

### 8. UIView vs CALayer?

<details>
<summary>Answer</summary>

Every UIView is **backed by a CALayer** (`view.layer`). The **layer** handles rendering and animation: content, borders, corner radius, shadows, transforms (Core Animation). The **view** adds event handling (`UIResponder`), Auto Layout, accessibility and gesture recognizers. Use plain CALayers (`CAShapeLayer`, `CAGradientLayer`, `CAEmitterLayer`) for lightweight visuals that don't need touch handling.

</details>

### 9. What is off-screen rendering? How do you avoid it?

<details>
<summary>Answer</summary>

Some effects force the GPU to render into a separate buffer before compositing, which is expensive and causes scrolling jank. Common causes: **`cornerRadius` + `masksToBounds`** on layers with content, **shadows without a `shadowPath`**, masks, group opacity, and `shouldRasterize` misuse.

Fixes: set `layer.shadowPath`, use `cornerCurve`/pre-rounded images, avoid masks in scrolling cells, and check with the **Instruments Core Animation** debug options (Color Off-screen Rendered).

</details>

### 10. How do you create a custom view?

<details>
<summary>Answer</summary>

Subclass `UIView`, implement `init(frame:)` and `init?(coder:)`, add subviews and constraints in a `setup()` method, expose a `configure(with:)` API, override `intrinsicContentSize` if it has a natural size, and use `draw(_:)` or layers for custom drawing. Use `@IBDesignable`/`@IBInspectable` for Interface Builder previews (or `#Preview` with UIKit in Xcode 15+). Make it accessible (labels, traits).

</details>

---

## Auto Layout

### 11. What is Auto Layout? How does it work?

<details>
<summary>Answer</summary>

Auto Layout is a **constraint-based** layout system. You describe relationships (`view.leading = superview.leading + 16`), and the **Cassowary** constraint solver computes the frames. It adapts to screen sizes, rotation, Dynamic Type and localization (RTL, long strings).

Ways to create constraints: **anchors** (`NSLayoutConstraint.activate([...])`), Interface Builder, `UIStackView` (which creates constraints for you), and the Visual Format Language (old). Always set `translatesAutoresizingMaskIntoConstraints = false` on views you constrain in code.

</details>

### 12. What are intrinsic content size, content hugging and compression resistance?

<details>
<summary>Answer</summary>

- **Intrinsic content size** – a view's natural size based on its content (a UILabel's text size, a UIButton's title + insets), so it doesn't need explicit width or height constraints.
- **Content hugging priority** – how much the view **resists growing** beyond its intrinsic size. Higher = "hug my content".
- **Compression resistance priority** – how much the view **resists shrinking** below its intrinsic size. Higher = "don't truncate me".

Example: two labels side by side. Give the one that should stay fully visible a higher compression resistance, and the one that shouldn't stretch a higher hugging priority.

</details>

### 13. What are constraint priorities and ambiguous/unsatisfiable layouts?

<details>
<summary>Answer</summary>

Priorities range from 1 to 1000 (required). Optional constraints (< 1000) are satisfied when possible.
- **Unsatisfiable** – conflicting required constraints. The console logs "Unable to simultaneously satisfy constraints", and UIKit breaks one of them. Fix it by lowering a priority or removing the conflict.
- **Ambiguous** – not enough constraints to determine a unique layout. Debug with the **View Debugger** or `hasAmbiguousLayout` / `exerciseAmbiguityInLayout()`.

Give constraints identifiers to make the logs readable.

</details>

### 14. What is UIStackView? When do you use it?

<details>
<summary>Answer</summary>

A UIStackView arranges its arranged subviews in a row or column (axis, distribution, alignment, spacing) and **generates the constraints for you**. Hiding an arranged subview automatically collapses its space. It's great for forms and toolbars. Avoid deeply nested stack views in hot scrolling cells (they create many constraints).

</details>

### 15. Frame-based layout vs Auto Layout — performance?

<details>
<summary>Answer</summary>

Auto Layout has a solver cost (much improved since iOS 12, roughly linear for independent constraints). Manual frames in `layoutSubviews` are faster, which can matter for very complex, frequently laid-out cells. Most apps use Auto Layout; optimize only after profiling.

</details>

---

## Lists

### 16. How does UITableView cell reuse work?

<details>
<summary>Answer</summary>

Instead of creating a cell for each row, the table view keeps a **reuse queue** per identifier. Cells that scroll off-screen go into the queue, and `dequeueReusableCell(withIdentifier:for:)` returns a recycled cell (or creates one). Memory stays proportional to the **visible** rows.

Pitfalls: **reset all state** in `prepareForReuse()` or during configuration (images, hidden flags, selection), and cancel async image loads so a reused cell doesn't show the wrong image (check the identifier when the image arrives, or cancel the task in `prepareForReuse`).

</details>

### 17. UITableView vs UICollectionView? What is compositional layout?

<details>
<summary>Answer</summary>

- **UITableView** – a single-column vertical list with sections.
- **UICollectionView** – any layout (grids, carousels, custom) through `UICollectionViewLayout`.
- **Compositional Layout** (iOS 13) – builds complex layouts from **items → groups → sections**, with orthogonal scrolling sections (App Store-style carousels) and `UICollectionLayoutListConfiguration` (table-like lists in a collection view).

Modern advice: use a collection view + compositional layout + a diffable data source for new UIKit screens.

</details>

### 18. What is a Diffable Data Source?

<details>
<summary>Answer</summary>

`UITableViewDiffableDataSource` / `UICollectionViewDiffableDataSource` (iOS 13) replace index-based data source methods with **snapshots**:

```swift
var snapshot = NSDiffableDataSourceSnapshot<Section, Item.ID>()
snapshot.appendSections([.main])
snapshot.appendItems(items.map(\.id))
dataSource.apply(snapshot, animatingDifferences: true)
```

The framework **computes the differences** and animates inserts, deletes and moves, which avoids the classic `NSInternalInconsistencyException` ("invalid number of rows"). Items must be `Hashable` (Apple recommends using **identifiers**, then `reconfigureItems` for content updates).

</details>

### 19. How do you implement self-sizing cells?

<details>
<summary>Answer</summary>

Use Auto Layout constraints from the content to all edges of the cell's `contentView` (an unbroken vertical chain), `tableView.rowHeight = UITableView.automaticDimension` (the default), and a reasonable `estimatedRowHeight`. In a collection view, use an estimated dimension in compositional layout (`.estimated(44)`).

</details>

### 20. How do you make lists scroll smoothly?

<details>
<summary>Answer</summary>

Reuse cells, keep `cellForRow` cheap (no synchronous I/O, image decoding or heavy formatting), **decode and downsample images off the main thread**, cache heights and formatted strings, avoid off-screen rendering, use **prefetching** (`UITableViewDataSourcePrefetching`), set opaque backgrounds, and profile with Instruments (Animation Hitches, Time Profiler).

</details>

---

## Interaction & Animation

### 21. What are gesture recognizers?

<details>
<summary>Answer</summary>

Objects that detect gestures (`UITapGestureRecognizer`, `UIPanGestureRecognizer`, `UIPinch…`, `UILongPress…`, `UISwipe…`, `UIRotation…`, `UIScreenEdgePan…`) and send actions. Each one goes through **states**: possible → began → changed → ended / cancelled / failed. Coordinate multiple recognizers with `require(toFail:)` and the `UIGestureRecognizerDelegate` method `shouldRecognizeSimultaneouslyWith`.

</details>

### 22. What animation options exist in UIKit?

<details>
<summary>Answer</summary>

- `UIView.animate(withDuration:)`, including the spring variants.
- **`UIViewPropertyAnimator`** – interruptible, scrubbable and reversible (interactive animations).
- **Core Animation** (`CABasicAnimation`, `CAKeyframeAnimation`) – layer-level control. Remember the model layer vs presentation layer distinction (set the final value).
- `UIView.transition`, custom view controller transitions (`UIViewControllerAnimatedTransitioning`), and **UIKit Dynamics** (physics).

</details>

### 23. How do you handle the keyboard covering text fields?

<details>
<summary>Answer</summary>

- iOS 15+: constrain the content to **`view.keyboardLayoutGuide.topAnchor`**, so it moves automatically.
- Older: observe `UIResponder.keyboardWillShowNotification`, read the keyboard frame, and adjust the scroll view's `contentInset` and `verticalScrollIndicatorInsets`.
- SwiftUI handles the keyboard safe area automatically.
- Dismiss the keyboard with `view.endEditing(true)` or `scrollView.keyboardDismissMode = .interactive`.

</details>

### 24. What are size classes and trait collections?

<details>
<summary>Answer</summary>

A `UITraitCollection` describes the environment: **horizontal/vertical size class** (compact or regular), display scale, user interface style (light/dark), Dynamic Type content size category, layout direction. Adapt the layout when traits change. iOS 17 adds `registerForTraitChanges(_:handler:)` (replacing `traitCollectionDidChange`) and **custom traits**.

</details>

### 25. How do you support Dark Mode and Dynamic Type?

<details>
<summary>Answer</summary>

- **Dark Mode:** semantic system colors (`.label`, `.systemBackground`) or asset-catalog colors with dark variants. Resolve `CGColor`s again on trait changes (layers don't update automatically).
- **Dynamic Type:** `UIFont.preferredFont(forTextStyle:)` + `adjustsFontForContentSizeCategory = true`, multi-line labels (`numberOfLines = 0`), and layouts that adapt at accessibility sizes (switch a stack view's axis). `UIFontMetrics` scales custom fonts.

</details>

### 26. Storyboards vs XIBs vs programmatic UI?

<details>
<summary>Answer</summary>

| Storyboards | XIBs | Programmatic |
|---|---|---|
| Visual flow and segues | Visual, a single reusable view | Full control, code review-friendly |
| **Merge conflicts** in teams, slow with many screens | Fewer conflicts | No merge-conflict pain, easier DI |
| Hard to inject dependencies (improved in iOS 13 with `creator:`) | — | Verbose without helpers |

Many teams use programmatic UIKit or SwiftUI for scalability. A common answer: "XIBs or code for reusable components, and coordinators instead of segues."

</details>

### 27. How do you use SwiftUI in UIKit and vice versa?

<details>
<summary>Answer</summary>

- **SwiftUI in UIKit:** `UIHostingController(rootView:)` as a child view controller, or `UIHostingConfiguration` for collection and table view cells (iOS 16).
- **UIKit in SwiftUI:** `UIViewRepresentable` / `UIViewControllerRepresentable` (`makeUIView`, `updateUIView`, and a `Coordinator` for delegates).

This is the standard approach for incremental migration.

</details>

---

[⬅ Previous: iOS Fundamentals](08-ios-fundamentals.md) · [⬅ Back to index](../README.md) · [Next: SwiftUI ➡](10-swiftui.md)
