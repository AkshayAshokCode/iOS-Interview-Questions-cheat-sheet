# Swift – Fundamentals

[⬅ Back to index](../README.md)

Covers: language basics, value vs reference types, optionals, collections, functions & closures, enums, error handling, access control.

---

## Language Basics

### 1. What is Swift? What are its key features?

<details>
<summary>Answer</summary>

Swift is Apple's modern, **type-safe, compiled** language for iOS, macOS, watchOS, tvOS and visionOS. It's also used on Linux and the server. Key features:
- **Safety**: optionals instead of null pointers, bounds-checked arrays, definite initialization, no implicit type conversions.
- **Value semantics**: structs and enums are first-class; **protocol-oriented programming**.
- **Performance**: compiled with LLVM; ARC instead of a garbage collector.
- **Modern syntax**: type inference, closures, generics, pattern matching.
- **Concurrency built in**: async/await, actors, compile-time data-race safety (Swift 6).
- Interoperates with Objective-C (and, more recently, C++).

</details>

### 2. `let` vs `var`?

<details>
<summary>Answer</summary>

- `let` – a constant; it can be assigned only once.
- `var` – a variable that can be reassigned.

For **value types** (struct), `let` makes the **whole value immutable**: you can't change its properties. For **reference types** (class), `let` only fixes the *reference*; the object's `var` properties can still change.

</details>

### 3. What is type inference and type safety in Swift?

<details>
<summary>Answer</summary>

**Type inference:** the compiler works out the type from the value (`let x = 10` is an `Int`). **Type safety:** you can't pass a `String` where an `Int` is expected, and there are **no implicit conversions** (`Double(intValue)` must be written explicitly). Types are fixed at compile time.

</details>

### 4. What are the basic data types?

<details>
<summary>Answer</summary>

`Int` (and `Int8`…`Int64`, `UInt`), `Double`, `Float`, `Bool`, `Character`, `String`, plus the collections `Array`, `Dictionary` and `Set`, and tuples. All of these are **structs** (value types) in Swift, not primitives.

</details>

---

## Value vs Reference Types

### 5. Struct vs Class – what are the differences?

<details>
<summary>Answer</summary>

| Struct | Class |
|---|---|
| **Value type**: copied on assignment | **Reference type**: shared reference |
| No inheritance (can conform to protocols) | Single inheritance |
| Memberwise initializer generated | Must write initializers (no memberwise init) |
| No deinit | Has `deinit` |
| Stored inline/stack when possible, no ARC overhead for the value itself | Heap-allocated, reference counted (ARC) |
| `mutating` keyword needed to modify self in methods | Methods can modify properties freely |
| Thread-safer by default (independent copies) | Shared mutable state → data races |
| Identity: no `===` | Identity check with `===` |

**Apple's guidance:** use **structs by default**. Use classes when you need **identity**, inheritance, shared mutable state, Objective-C interop, or deinit.

</details>

### 6. What is Copy-on-Write (CoW)?

<details>
<summary>Answer</summary>

Standard-library collections (`Array`, `Dictionary`, `Set`, `String`) are value types but store their elements in a heap buffer. On assignment, **both copies share the buffer**. It's copied only **when one copy is mutated** and the buffer isn't uniquely referenced. So copying a large array is O(1) until someone writes to it.

You can implement CoW for your own types with `isKnownUniquelyReferenced(&storage)`. Note: your own structs **don't** get CoW automatically. They're copied member by member.

</details>

### 7. Where are structs and classes stored — stack or heap?

<details>
<summary>Answer</summary>

Generally, **class instances live on the heap** (reference counted), and **structs live on the stack or inline** inside their container. But it depends: a struct captured by an escaping closure, stored in an existential (`any Protocol`) larger than the inline buffer, or a property of a class ends up on the heap. The compiler can also stack-promote class instances that don't escape. Interviewers mostly want "value types avoid heap allocation and ARC overhead in common cases".

</details>

### 8. When would you choose a class over a struct?

<details>
<summary>Answer</summary>

- When you need **shared identity** (a single `NetworkManager`, a cache, a view model observed by many views).
- When you need **inheritance** (subclassing `UIViewController`).
- For **Objective-C interop** (`@objc`, `NSObject` subclasses, KVO).
- When you need `deinit` to clean up resources.
- For `@Observable` view models in SwiftUI (Observation requires classes).

</details>

### 9. What is the `mutating` keyword?

<details>
<summary>Answer</summary>

A struct's or enum's methods can't modify `self` by default. Mark a method `mutating` to allow it; the method then works on the value in place (it can even assign a new `self`). You can't call mutating methods on a `let` instance.

</details>

---

## Optionals

### 10. What are optionals? How do you unwrap them?

<details>
<summary>Answer</summary>

`Optional<T>` (written `T?`) is an **enum with `.some(T)` and `.none`**. It represents a value that might be absent. Non-optional types can never be `nil`.

Ways to unwrap:
- **Optional binding:** `if let name { }`, `guard let name else { return }` (the shorthand form is from Swift 5.7).
- **Nil-coalescing:** `name ?? "Guest"`.
- **Optional chaining:** `user?.address?.city` (returns an optional; stops at the first `nil`).
- **Force unwrap:** `name!` crashes if it's nil. Avoid it.
- **Pattern matching:** `if case let .some(x) = value`, `switch`.
- `map`/`flatMap` on optionals.

</details>

### 11. `if let` vs `guard let`?

<details>
<summary>Answer</summary>

- **`if let`** – the unwrapped value is available **inside** the `if` block.
- **`guard let`** – handles the failure case up front with an `else` that **must exit** the scope (`return`, `throw`, `break`). The unwrapped value is available **after** the guard for the rest of the scope. It avoids the "pyramid of doom" and makes early exits clear.

</details>

### 12. What is an implicitly unwrapped optional (`String!`)?

<details>
<summary>Answer</summary>

It's an optional that's **automatically force-unwrapped** when you use it, so it crashes if it's nil. It's used when a value is nil only briefly during initialization and guaranteed to be set afterwards, e.g. `@IBOutlet weak var label: UILabel!`. Avoid it otherwise.

</details>

### 13. What is optional chaining?

<details>
<summary>Answer</summary>

`a?.b?.c()` calls through optionals. If any link is nil, the whole expression returns nil instead of crashing, and the result is always optional. Example: `let count = user?.posts?.count ?? 0`.

</details>

---

## Collections, Functions & Closures

### 14. Array vs Set vs Dictionary?

<details>
<summary>Answer</summary>

- **Array** – ordered, allows duplicates, O(1) access by index, O(n) `contains`.
- **Set** – unordered, unique elements, **O(1) average `contains`**. Elements must be `Hashable`.
- **Dictionary** – key–value pairs with unique `Hashable` keys and O(1) average lookup. Unordered (use an array of keys if you need an order).

</details>

### 15. Explain `map`, `compactMap`, `flatMap`, `filter`, `reduce`.

<details>
<summary>Answer</summary>

```swift
let nums = ["1", "2", "x"]
nums.map { Int($0) }          // [Optional(1), Optional(2), nil]
nums.compactMap { Int($0) }   // [1, 2]  – removes nils
[[1, 2], [3]].flatMap { $0 }  // [1, 2, 3] – flattens
[1, 2, 3].filter { $0 > 1 }   // [2, 3]
[1, 2, 3].reduce(0, +)        // 6
```

Others: `sorted(by:)`, `forEach`, `first(where:)`, `contains(where:)`, `allSatisfy`, `zip`, `enumerated()`, `Dictionary(grouping:by:)`. Use `.lazy` for lazy evaluation of long chains.

</details>

### 16. What are closures? What is trailing closure syntax?

<details>
<summary>Answer</summary>

Closures are self-contained blocks of code that can **capture** constants and variables from their surrounding context (like lambdas). Functions are a special case of closures with a name.

```swift
let sorted = names.sorted { $0 < $1 }   // trailing closure + shorthand args
```

If the last parameter is a closure, it can be written after the parentheses (trailing closure). Multiple trailing closures are supported too (common in SwiftUI).

</details>

### 17. Escaping vs non-escaping closures?

<details>
<summary>Answer</summary>

- **Non-escaping** (the default): the closure is called **before the function returns**. The compiler can optimize it, and you don't need to write `self.` explicitly.
- **`@escaping`**: the closure **outlives the function call**: it's stored in a property or called asynchronously (completion handlers). You must write `self` explicitly (to make you think about **retain cycles**) and often capture `[weak self]`.

```swift
func fetch(completion: @escaping (Result<Data, Error>) -> Void) {
    URLSession.shared.dataTask(with: url) { data, _, error in /* later */ }.resume()
}
```

</details>

### 18. What is a capture list?

<details>
<summary>Answer</summary>

`{ [weak self, count] in ... }` controls **how values are captured**:
- `[weak self]` / `[unowned self]` break retain cycles.
- `[count]` captures the **value at closure creation** instead of a live reference to the variable.

Without a capture list, closures capture variables **by reference** (they see later changes) and capture class instances **strongly**.

</details>

### 19. What is `@autoclosure`?

<details>
<summary>Answer</summary>

It wraps an argument expression in a closure automatically, so it's **evaluated lazily**, only if needed. `assert(condition(), "msg")` and the `??` operator use it (the default value is only computed when the left side is nil).

</details>

### 20. What are inout parameters?

<details>
<summary>Answer</summary>

`inout` lets a function modify a parameter that the caller passes with `&` (copy-in, copy-out semantics):

```swift
func increment(_ x: inout Int) { x += 1 }
increment(&value)
```

</details>

### 21. What are variadic parameters, default parameters, and argument labels?

<details>
<summary>Answer</summary>

`func log(_ items: Any..., separator: String = " ")`: `_` removes the external label, `items` is variadic (received as an array), and `separator` has a default value. External labels make call sites read like English: `move(from: a, to: b)`.

</details>

---

## Enums, Errors, Access Control

### 22. What are enums with associated values and raw values?

<details>
<summary>Answer</summary>

```swift
enum Direction: String { case north, south }          // raw values (String/Int)
enum NetworkState {
    case loading
    case success(Data)                                  // associated values
    case failure(Error)
}
switch state {
case .loading: showSpinner()
case .success(let data): render(data)
case .failure(let error): show(error)
}
```

Swift enums are **algebraic data types**: they can have methods, computed properties, conform to protocols, and be `indirect` (recursive). A `switch` must be **exhaustive**. `CaseIterable` provides `allCases`.

</details>

### 23. How does error handling work in Swift?

<details>
<summary>Answer</summary>

- Errors conform to the `Error` protocol (usually enums).
- Functions that can fail are marked `throws` and called with `try` inside `do { } catch { }`.
- `try?` – converts the result to an optional (nil on error). `try!` – crashes on error.
- **Typed throws** (Swift 6): `func load() throws(NetworkError)`.
- `Result<Success, Failure>` – models success or failure as a value (common with completion handlers).
- `defer { }` – runs when the scope exits (cleanup).
- `rethrows` – the function only throws if its closure argument throws.

</details>

### 24. What are the access control levels in Swift?

<details>
<summary>Answer</summary>

From most to least open:
- `open` – accessible **and subclassable/overridable** outside the module.
- `public` – accessible outside the module, but **not subclassable/overridable** there.
- `package` (Swift 5.9) – visible within the same **Swift package**.
- `internal` (**default**) – visible within the same module.
- `fileprivate` – visible within the same file.
- `private` – visible within the enclosing declaration (and its extensions in the same file).

</details>

### 25. What is `defer`?

<details>
<summary>Answer</summary>

`defer` runs a block of code **when the current scope exits**, however it exits (return, throw, break). It's used for cleanup: closing files, unlocking locks, `isLoading = false`. Multiple defers run in **reverse order**.

</details>

### 26. What are tuples and when do you use them?

<details>
<summary>Answer</summary>

Tuples group multiple values without defining a type: `func minMax() -> (min: Int, max: Int)`. They're good for lightweight, local return values. For anything public or reused, define a struct.

</details>

### 27. What are computed vs stored properties, lazy properties and property observers?

<details>
<summary>Answer</summary>

- **Stored** – holds a value.
- **Computed** – `var area: Double { width * height }`. No storage; has a getter and optionally a setter.
- **`lazy var`** – initialized on **first access** (useful for expensive objects or to refer to `self` during setup). It's **not thread-safe** and must be a `var`.
- **Observers** – `willSet` / `didSet` run on changes to stored properties (not during init).

</details>

### 28. What are static vs class properties and methods?

<details>
<summary>Answer</summary>

Both belong to the **type**, not to instances. `static` members **can't be overridden**. `class` members (only available in classes) **can be overridden** by subclasses. A `static let` is **lazily initialized and thread-safe**, which is how Swift singletons work (`static let shared = Manager()`).

</details>

### 29. What is `typealias` and what is `Any` vs `AnyObject`?

<details>
<summary>Answer</summary>

- `typealias Completion = (Result<User, Error>) -> Void` gives an existing type another name.
- `Any` – an instance of **any type** (structs, enums, functions, classes).
- `AnyObject` – an instance of **any class type** only. It's used to constrain protocols to classes (`protocol Delegate: AnyObject`), so that the delegate can be held `weak`.

</details>

### 30. What are extensions?

<details>
<summary>Answer</summary>

Extensions add functionality to existing types (even ones you don't own): computed properties, methods, initializers, subscripts, nested types and **protocol conformance**. They **can't add stored properties** or override existing methods. Use them to organize code (one extension per protocol conformance) and to provide default protocol implementations.

</details>

---

[⬅ Back to index](../README.md) · [Next: Swift Advanced ➡](02-swift-advanced.md)
