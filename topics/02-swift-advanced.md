# Swift – Advanced

[⬅ Back to index](../README.md)

Covers: protocols & POP, generics, associated types, `some` vs `any`, method dispatch, property wrappers, result builders, macros, key paths, Codable, Equatable/Hashable, Objective-C interop.

---

## Protocols

### 1. What is Protocol-Oriented Programming (POP)?

<details>
<summary>Answer</summary>

POP (introduced at WWDC 2015) means designing around **protocols + protocol extensions + value types** instead of class inheritance:
- Protocols define capabilities, and **protocol extensions provide default implementations** (shared behaviour without a base class).
- A type can conform to many protocols (**composition**). Class inheritance only allows one parent.
- It works with **structs and enums**, so you get value semantics.
- It's easy to **mock in tests** (inject a protocol).

```swift
protocol Identifiable { var id: String { get } }
extension Identifiable { var debugID: String { "ID-\(id)" } }
```

</details>

### 2. Protocol vs class inheritance? What problems does inheritance have?

<details>
<summary>Answer</summary>

Inheritance issues: single inheritance only, **tight coupling** to the parent's implementation (the fragile base class problem), deep hierarchies ("BaseViewController" grows into a god class), reference types only, and you inherit things you don't need. Protocols offer **multiple conformances**, work with value types, and give looser coupling through composition (`typealias Codable = Encodable & Decodable`).

</details>

### 3. What are protocol extensions? What's the dispatch gotcha?

<details>
<summary>Answer</summary>

Protocol extensions add default method implementations for all conforming types. **Gotcha:** methods defined **only in the extension** (not declared as requirements in the protocol) are **statically dispatched**:

```swift
protocol Greeter { func hello() }
extension Greeter {
    func hello() { print("Default hello") }   // requirement → dynamic dispatch
    func bye() { print("Default bye") }       // NOT a requirement → static dispatch
}
struct Person: Greeter {
    func hello() { print("Person hello") }
    func bye() { print("Person bye") }
}
let g: Greeter = Person()
g.hello()  // "Person hello"
g.bye()    // "Default bye" ← surprising!
```

</details>

### 4. What are associated types?

<details>
<summary>Answer</summary>

Associated types are **placeholder types** inside a protocol that conforming types fill in. They make protocols generic:

```swift
protocol Repository {
    associatedtype Item
    func fetchAll() async throws -> [Item]
}
struct UserRepo: Repository { func fetchAll() async throws -> [User] { ... } }
```

Before Swift 5.7, protocols with associated types (PATs) couldn't be used as plain types (the "can only be used as a generic constraint" error). Now you can write `any Repository<User>` using **primary associated types** (`protocol Repository<Item>`).

</details>

### 5. `some` vs `any` – opaque types vs existentials?

<details>
<summary>Answer</summary>

- **`some Protocol` (opaque type)** – "**one specific** concrete type that conforms, chosen by the implementation and hidden from the caller". The compiler knows the real type, so it uses **static dispatch with no boxing**, which is fast. Example: `var body: some View`.
- **`any Protocol` (existential)** – "**any** conforming type, which can change at runtime". The value is stored in an **existential container** (boxed, possibly heap-allocated) and uses dynamic dispatch, so it's slower. Use it for heterogeneous collections: `[any Shape]`.

Prefer `some` (or generics), and use `any` only when you truly need different types at runtime. Since Swift 5.7 you can also write `func draw(_ s: some Shape)` as shorthand for a generic.

</details>

### 6. What is a protocol composition and class-only protocol?

<details>
<summary>Answer</summary>

- Composition: `func save(_ x: Codable & Identifiable)` requires both protocols.
- Class-only: `protocol ViewDelegate: AnyObject` limits conformance to classes, so that you can declare `weak var delegate: ViewDelegate?` (weak only works with class references).

</details>

---

## Generics

### 7. What are generics? What are generic constraints?

<details>
<summary>Answer</summary>

Generics let you write flexible, reusable code that works with any type while staying type-safe:

```swift
func swapValues<T>(_ a: inout T, _ b: inout T) { (a, b) = (b, a) }
func largest<T: Comparable>(_ items: [T]) -> T? { items.max() }
func process<C: Collection>(_ c: C) where C.Element: Hashable { }
```

Constraints (`T: Comparable`, `where` clauses) restrict which types are allowed. Swift **specializes** generics at compile time where possible, which makes them fast.

</details>

### 8. Generics vs `any Protocol` – which is faster and why?

<details>
<summary>Answer</summary>

Generics (and `some`) are usually **faster**: the compiler can **specialize** them for each concrete type (static dispatch, inlining, no boxing). `any` uses an existential container plus witness table lookups (dynamic dispatch, possibly a heap allocation). Use generics for performance-sensitive code.

</details>

---

## Dispatch & Performance

### 9. What is method dispatch in Swift? Static vs dynamic vs message dispatch?

<details>
<summary>Answer</summary>

- **Static (direct) dispatch** – the call target is known at compile time. It's the fastest and can be inlined. Used for **structs and enums**, `final` classes and methods, `private` members, and non-requirement protocol extension methods.
- **Table (dynamic) dispatch** – a lookup through a **vtable** (classes) or **witness table** (protocols) at runtime. Used for overridable class methods and protocol requirements.
- **Message dispatch** – the Objective-C runtime (`objc_msgSend`). The most dynamic: it enables swizzling and KVO. Used with `@objc dynamic`.

Performance tips: mark classes `final` when they aren't subclassed, use `private`, and enable **Whole Module Optimization** (lets the compiler infer `final`).

</details>

### 10. What does `final` do?

<details>
<summary>Answer</summary>

`final` prevents a class from being subclassed, or a method or property from being overridden. That allows **static dispatch** (a performance gain) and documents your intent.

</details>

---

## Language Features

### 11. What are property wrappers?

<details>
<summary>Answer</summary>

Property wrappers wrap the storage logic of a property in a reusable type annotated with `@propertyWrapper`, which provides `wrappedValue` (and optionally `projectedValue`, accessed with `$`):

```swift
@propertyWrapper
struct Clamped {
    private var value: Int; let range: ClosedRange<Int>
    init(wrappedValue: Int, _ range: ClosedRange<Int>) { self.range = range; value = min(max(wrappedValue, range.lowerBound), range.upperBound) }
    var wrappedValue: Int { get { value } set { value = min(max(newValue, range.lowerBound), range.upperBound) } }
}
struct Volume { @Clamped(0...100) var level = 50 }
```

Examples: SwiftUI's `@State`, `@Binding`, `@Environment`, `@AppStorage`; Combine's `@Published`; custom `@UserDefault` and `@Trimmed` wrappers.

</details>

### 12. What are result builders?

<details>
<summary>Answer</summary>

`@resultBuilder` types turn a sequence of statements into a single value, which enables **DSLs**. SwiftUI's `@ViewBuilder` is the best-known one: it combines the views in a `body` into a single view. They're built from static methods like `buildBlock`, `buildOptional` (`if`), `buildEither` (`if/else`) and `buildArray` (`for`). Other examples: RegexBuilder and HTML DSLs.

</details>

### 13. What are Swift macros?

<details>
<summary>Answer</summary>

Macros (Swift 5.9) are **compile-time code generation**. They're implemented with SwiftSyntax in a separate compiler plugin, and they're type-checked and expanded in the editor.
- **Freestanding** – `#Preview { }`, `#warning`, `#URL("...")` (expression macros).
- **Attached** – `@Observable` (generates observation tracking), `@Model` (SwiftData), `@Test` / `#expect` (Swift Testing), and custom macros for Codable keys or mocks.

They reduce boilerplate without runtime cost, and you can expand them in Xcode to see the generated code.

</details>

### 14. What are key paths?

<details>
<summary>Answer</summary>

Key paths are **type-safe references to properties**: `\User.name` has type `KeyPath<User, String>`. They're used for `users.map(\.name)`, sorting (`sorted(using: KeyPathComparator(\.age))`), KVO (`observe(\.isHidden)`), SwiftUI (`ForEach(items, id: \.id)`) and Combine (`assign(to: \.text, on: label)`). `WritableKeyPath` allows writing as well as reading.

</details>

### 15. What is Codable? How do you customize it?

<details>
<summary>Answer</summary>

`Codable = Encodable & Decodable`. The compiler synthesizes encoding and decoding for types whose properties are all Codable. Use it with `JSONDecoder` and `PropertyListEncoder`.

Customization:
- **`CodingKeys` enum** – rename or omit keys.
- `decoder.keyDecodingStrategy = .convertFromSnakeCase`, `dateDecodingStrategy = .iso8601`.
- Custom `init(from decoder:)` / `encode(to:)` for nested JSON, default values or polymorphic types (`decodeIfPresent` with a fallback).

```swift
struct User: Codable {
    let id: Int; let fullName: String
    enum CodingKeys: String, CodingKey { case id; case fullName = "full_name" }
}
```

</details>

### 16. Equatable, Hashable, Comparable, Identifiable?

<details>
<summary>Answer</summary>

- **Equatable** – `==`. It's synthesized for structs and enums whose members are all Equatable.
- **Hashable** – `hash(into:)`. Required for `Set` elements and `Dictionary` keys; equal values **must** have equal hashes.
- **Comparable** – `<`, which enables `sorted()` and `max()`.
- **Identifiable** – a stable `id`. Used by SwiftUI `List`/`ForEach` and diffing.

</details>

### 17. What are subscripts?

<details>
<summary>Answer</summary>

Subscripts provide custom `[]` access: `subscript(index: Int) -> T { get set }`. They can take multiple parameters, labels and default values. A common example is a safe array subscript: `subscript(safe i: Int) -> Element? { indices.contains(i) ? self[i] : nil }`.

</details>

### 18. What are initializers in Swift? Designated vs convenience vs failable vs required?

<details>
<summary>Answer</summary>

- **Designated** – the primary initializer. It must initialize all properties introduced by the class, then call the superclass's designated init.
- **Convenience** – a secondary initializer that must call another init in the **same** class (delegates across).
- **Failable** – `init?` returns nil on invalid input.
- **`required`** – every subclass must implement it (e.g. `required init?(coder:)` in UIKit).
- **Two-phase initialization**: phase 1 sets all stored properties (bottom-up). Phase 2 lets the initializer customize the instance and use `self`.

</details>

### 19. What is `@dynamicMemberLookup` and `@dynamicCallable`?

<details>
<summary>Answer</summary>

They allow dot syntax for members resolved at runtime through a subscript (`@dynamicMemberLookup`, often combined with key paths for type safety, e.g. wrappers that forward properties), and calling an instance like a function (`@dynamicCallable`, used for interop with dynamic languages such as Python). They're rarely needed, but interviewers sometimes ask about them.

</details>

### 20. What is Objective-C interop? What do `@objc` and `dynamic` do?

<details>
<summary>Answer</summary>

- `@objc` exposes a Swift declaration to the Objective-C runtime. It's required for `#selector`, target-action, `@IBAction`, and `@objc` protocols with optional methods.
- `dynamic` forces **message dispatch** through the ObjC runtime. It's needed for **KVO** and **method swizzling**.
- A **bridging header** exposes ObjC code to Swift. The generated `ModuleName-Swift.h` header exposes Swift to ObjC.
- Types like `String`/`NSString` and `Array`/`NSArray` bridge automatically.

</details>

### 21. What is method swizzling?

<details>
<summary>Answer</summary>

Method swizzling **swaps method implementations at runtime** using the Objective-C runtime (`method_exchangeImplementations`). It only works for `@objc dynamic` / NSObject methods. It's used by analytics SDKs (for example, to track `viewDidAppear`). It's risky: hard to debug, fragile across OS versions, and it can conflict with other libraries. Mention it as something to avoid in app code.

</details>

### 22. What are `@discardableResult`, `@available`, `@MainActor` and `@inlinable`?

<details>
<summary>Answer</summary>

- `@discardableResult` – no warning when the caller ignores the return value.
- `@available(iOS 17, *)` / `if #available(iOS 17, *)` – API availability checks.
- `@MainActor` – runs on the main thread (covered in [Swift Concurrency](05-swift-concurrency.md)).
- `@inlinable` – exposes the function body across modules so it can be inlined (library performance).

</details>

### 23. What is pattern matching in Swift?

<details>
<summary>Answer</summary>

`switch` and `if case` support enum cases with associated values, tuples, ranges (`case 0..<10`), type casts (`case let x as String`), `where` clauses, optional patterns (`case let x?`) and `~=` for custom matching:

```swift
switch (statusCode, data) {
case (200..<300, let data?): parse(data)
case (401, _): logout()
default: showError()
}
```

</details>

### 24. What is the difference between `Self` and `self`?

<details>
<summary>Answer</summary>

`self` is the current **instance** (or the type itself, inside static methods). `Self` is the current **type**, often used in protocols (`func copy() -> Self`) and in static factory methods. In a class, `Self` refers to the dynamic type, i.e. the actual subclass.

</details>

### 25. What is the difference between `as`, `as?` and `as!`?

<details>
<summary>Answer</summary>

- `as` – upcasting or bridging that's guaranteed at compile time (`view as UIView`, `"a" as NSString`).
- `as?` – a conditional downcast that returns an optional.
- `as!` – a forced downcast that crashes if it fails.

`is` checks the type without casting.

</details>

---

[⬅ Previous: Swift Fundamentals](01-swift-basics.md) · [⬅ Back to index](../README.md) · [Next: Memory Management ➡](03-memory-management.md)
