# OOP, SOLID & Design Patterns

[⬅ Back to index](../README.md)

Covers: OOP pillars, SOLID with iOS examples, and the design patterns used across Apple frameworks (Delegation, Singleton, Observer, KVO, Notification, Target-Action, Factory, Builder, Adapter, Facade, Decorator, Strategy, Coordinator…).

---

## OOP & SOLID

### 1. What are the four pillars of OOP?

<details>
<summary>Answer</summary>

1. **Encapsulation** – hide state behind an interface (`private(set) var`).
2. **Abstraction** – expose *what*, hide *how* (protocols such as `UserServiceProtocol`).
3. **Inheritance** – reuse through subclassing (`UIViewController` subclasses).
4. **Polymorphism** – one interface, many implementations (method overriding, protocol conformances).

In Swift, prefer **composition and protocols** over deep inheritance.

</details>

### 2. Explain SOLID with iOS examples.

<details>
<summary>Answer</summary>

- **S – Single Responsibility:** avoid the "Massive View Controller". Move networking into a service, formatting into a view model or formatter, and navigation into a coordinator.
- **O – Open/Closed:** add a new payment method by adding a type that conforms to `PaymentMethod`, instead of editing a `switch`.
- **L – Liskov Substitution:** any `DataStore` conformance (Core Data, in-memory, mock) must behave correctly wherever a `DataStore` is expected. A subclass that throws `fatalError("not supported")` violates it.
- **I – Interface Segregation:** small protocols (`UITableViewDataSource` vs `UITableViewDelegate`, `Encodable` vs `Decodable`), and optional delegate methods to avoid fat interfaces.
- **D – Dependency Inversion:** a view model depends on `UserRepository` (a protocol), not on `URLSession` or Core Data, and receives it through its initializer.

</details>

### 3. Composition vs inheritance in Swift?

<details>
<summary>Answer</summary>

Prefer composition: protocols + extensions, injected collaborators, child view controllers, and SwiftUI view composition. Inheritance is fine for framework requirements (subclassing `UIView`/`UIViewController`), but avoid deep custom hierarchies like a `BaseViewController` that does everything.

</details>

---

## Design Patterns in iOS

### 4. What is the Delegation pattern?

<details>
<summary>Answer</summary>

One object **hands off responsibilities or events to another** through a protocol. It's **one-to-one** communication and it's everywhere in UIKit: `UITableViewDelegate/DataSource`, `UITextFieldDelegate`, `URLSessionDelegate`, `CLLocationManagerDelegate`.

```swift
protocol PickerDelegate: AnyObject { func picker(_ p: Picker, didSelect item: Item) }
final class Picker { weak var delegate: PickerDelegate? }
```

Always declare the delegate `weak` with a class-bound protocol, to avoid retain cycles.

</details>

### 5. Delegate vs Notification vs Closure vs KVO vs Combine — when to use which?

<details>
<summary>Answer</summary>

| Mechanism | Relationship | Use when |
|---|---|---|
| **Delegate** | 1:1, protocol-based | Many related callbacks, the delegate can return values (`heightForRow`) |
| **Closure/callback** | 1:1 | One or two simple callbacks (completion handlers, button actions) |
| **NotificationCenter** | 1:many, loosely coupled | Broadcasting app-wide events (keyboard shown, logout, app background) |
| **KVO** | 1:many, property changes | Observing properties of NSObject subclasses (`AVPlayer.status`, `WKWebView.estimatedProgress`) |
| **Combine/Observation** | 1:many, reactive | State streams and data binding in MVVM/SwiftUI |

</details>

### 6. What is the Singleton pattern? What are its pros and cons?

<details>
<summary>Answer</summary>

A singleton gives you **one shared instance** with global access: `static let shared = Manager()` + `private init()`. Apple examples: `UIApplication.shared`, `URLSession.shared`, `FileManager.default`, `UserDefaults.standard`.

**Pros:** easy access, one shared resource.
**Cons:** global mutable state (thread-safety issues), hidden dependencies, **hard to test** (it can't be replaced), tight coupling.
**Better:** inject the shared instance behind a protocol (`init(session: NetworkSession = URLSession.shared)`).

</details>

### 7. What is KVO (Key-Value Observing)?

<details>
<summary>Answer</summary>

KVO is an Objective-C runtime mechanism to observe changes to properties. It needs `NSObject` subclasses and `@objc dynamic` properties:

```swift
observation = player.observe(\.status, options: [.new]) { player, _ in
    if player.status == .readyToPlay { play() }
}
```

Keep the returned `NSKeyValueObservation`; when it's deallocated, observation stops. It works through **isa-swizzling** (a dynamic subclass is created at runtime). For Swift types, use Combine or Observation instead.

</details>

### 8. What is NotificationCenter? What are the pitfalls?

<details>
<summary>Answer</summary>

NotificationCenter is a **publish-subscribe** broadcast system: you `post` a `Notification.Name`, and all registered observers receive it.

Pitfalls:
- Hard to trace (implicit coupling).
- Block-based observers must be removed (keep the token) and should capture `[weak self]`.
- Notifications are delivered **synchronously on the posting thread**.
- `userInfo` is untyped.

The modern alternative is `NotificationCenter.default.notifications(named:)` as an AsyncSequence, or the Combine `publisher(for:)`.

</details>

### 9. What is the Target-Action pattern?

<details>
<summary>Answer</summary>

A control (UIButton, UISwitch) stores a **target object and a selector** and calls it when an event happens: `button.addTarget(self, action: #selector(tap), for: .touchUpInside)`. It relies on the ObjC runtime (`@objc`). Since iOS 14 you can use `UIAction` closures instead: `UIButton(primaryAction: UIAction { _ in })`.

</details>

### 10. What is the Observer pattern?

<details>
<summary>Answer</summary>

The Observer pattern is a one-to-many dependency where observers are notified when a subject changes. In iOS it's implemented by KVO, NotificationCenter, Combine (`@Published`), the Observation framework (`@Observable`), and SwiftUI's state system.

</details>

### 11. Factory, Builder and Abstract Factory in iOS?

<details>
<summary>Answer</summary>

- **Factory** – encapsulates object creation: `UIFont.systemFont(ofSize:)`, `UIButton(type:)`, a `ViewControllerFactory` used by coordinators to build screens with their dependencies.
- **Builder** – step-by-step construction: `URLComponents`, `UIButton.Configuration`, `NSAttributedString` building, `AttributedString`, or a custom `RequestBuilder`.
- **Abstract Factory** – creates families of related objects (e.g. themed UI component factories).

</details>

### 12. Adapter, Facade, Decorator, Proxy in iOS?

<details>
<summary>Answer</summary>

- **Adapter** – wraps an incompatible interface: wrapping a third-party analytics SDK behind your own `AnalyticsService` protocol, or `UIViewRepresentable` (UIKit → SwiftUI).
- **Facade** – a simple interface over a complex subsystem: a `Repository` hiding networking + caching + Core Data, or `UIImagePickerController` hiding the camera APIs.
- **Decorator** – adds behaviour without subclassing: Swift **extensions**, SwiftUI **view modifiers**, wrapping a service to add logging or caching (`CachedUserService: UserService`).
- **Proxy** – controls access: `NSProxy`, lazy-loading image proxies, a logging proxy around a network client.

</details>

### 13. Strategy, State, Command, Chain of Responsibility in iOS?

<details>
<summary>Answer</summary>

- **Strategy** – interchangeable algorithms: `UICollectionViewLayout` subclasses (compositional, flow), sorting comparators, different `ImageCachePolicy` types.
- **State** – behaviour depends on state: an enum `ViewState { loading, loaded(data), error }` driving the UI, a player state machine.
- **Command** – `UndoManager` (registering undo actions), `UIAction`, an operation queue of tasks.
- **Chain of Responsibility** – the **responder chain** (UIView → superview → UIViewController → UIWindow → UIApplication → AppDelegate), which handles touch events and actions sent to `nil` targets.

</details>

### 14. What is the Coordinator pattern?

<details>
<summary>Answer</summary>

Coordinator **moves navigation logic out of view controllers** into coordinator objects that create and present screens:

```swift
protocol Coordinator: AnyObject { func start() }
final class AuthCoordinator: Coordinator {
    let nav: UINavigationController
    var onFinish: (() -> Void)?
    init(nav: UINavigationController) { self.nav = nav }
    func start() {
        let vm = LoginViewModel(onLogin: { [weak self] in self?.onFinish?() })
        nav.pushViewController(LoginViewController(viewModel: vm), animated: true)
    }
}
```

Benefits: reusable view controllers (they don't know what comes next), centralized flows, easier deep linking, and testable navigation. Usually there's a parent `AppCoordinator` with child coordinators per flow. Remove child coordinators when they finish, to avoid leaks. In SwiftUI, a similar role is played by a router object that owns the `NavigationPath`.

</details>

### 15. What is the Repository pattern?

<details>
<summary>Answer</summary>

A repository is an abstraction over data sources (API, database, cache) that exposes domain models and hides where the data comes from. Business logic and view models depend on a protocol, which makes caching strategies and testing (fake repositories) easy.

</details>

### 16. What is MVC's "Massive View Controller" problem and how do patterns help?

<details>
<summary>Answer</summary>

In Apple's MVC, the view controller tends to take on networking, data formatting, table data sources, validation and navigation. It grows into thousands of lines and becomes untestable. Fixes: **MVVM** (presentation logic in a view model), **Coordinators** (navigation), **services and repositories** (data), **child view controllers** (compose the UI), separate data source objects, and extensions organized by responsibility. See [Architecture](11-architecture.md).

</details>

---

[⬅ Previous: Combine](06-combine.md) · [⬅ Back to index](../README.md) · [Next: iOS Fundamentals ➡](08-ios-fundamentals.md)
