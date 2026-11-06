# Memory Management – ARC

[⬅ Back to index](../README.md)

Covers: ARC, strong/weak/unowned, retain cycles in classes, closures and delegates, debugging leaks, autorelease pools, value types.

---

### 1. What is ARC? How does it work?

<details>
<summary>Answer</summary>

**Automatic Reference Counting** manages the memory of **class instances**. The compiler inserts `retain`/`release` calls at compile time:
- Each **strong reference** increments the instance's reference count.
- When a reference goes away (variable out of scope, set to nil, owner deallocated), the count is decremented.
- When the count reaches **zero**, the instance is **deallocated immediately** (`deinit` is called).

Unlike garbage collection, ARC is **deterministic** (no GC pauses), but it **can't break reference cycles** on its own. Value types (structs, enums) aren't reference counted, although any class references they hold are.

</details>

### 2. ARC vs Garbage Collection?

<details>
<summary>Answer</summary>

| ARC (iOS) | GC (Android/JVM) |
|---|---|
| Compile-time inserted retain/release | Runtime collector traces reachability |
| Deterministic, immediate deallocation | Non-deterministic, periodic collection |
| No pauses | GC pauses (though concurrent) |
| **Cannot handle retain cycles** → needs `weak`/`unowned` | Handles cycles (unreachable cycles get collected) |
| Small overhead on every reference change (atomic ops) | Overhead concentrated in GC cycles |

</details>

### 3. strong vs weak vs unowned?

<details>
<summary>Answer</summary>

- **strong** (default) – keeps the object alive by incrementing the reference count.
- **weak** – doesn't increment the count. It's **always an optional `var`** and is **automatically set to nil** when the object deallocates. Use it when the referenced object can be deallocated first (delegates, parent references, `[weak self]`).
- **unowned** – doesn't increment the count and **isn't optional**. It assumes the object **always outlives** the reference. **Crashes** if accessed after deallocation. Use it when the lifetimes are guaranteed (e.g. a `CreditCard` always has a `Customer` that outlives it). `unowned(unsafe)` skips even the safety check.

</details>

### 4. What is a retain cycle? Give an example and fix it.

<details>
<summary>Answer</summary>

A retain cycle is two or more objects holding **strong references to each other**. Their counts never reach zero, so they **leak**.

```swift
class Person { var apartment: Apartment?; deinit { print("Person freed") } }
class Apartment { var tenant: Person?; deinit { print("Apt freed") } }   // ❌ strong both ways

let p = Person(); let a = Apartment()
p.apartment = a; a.tenant = p   // cycle → neither deinit is called
```

**Fix:** make one side weak: `weak var tenant: Person?`.

</details>

### 5. How do closures cause retain cycles? How do you fix them?

<details>
<summary>Answer</summary>

Closures **capture `self` strongly**. If `self` also stores the closure (directly or indirectly), you get a cycle: self → closure → self.

```swift
class ProfileViewModel {
    var onUpdate: (() -> Void)?
    var name = ""
    func bind() {
        onUpdate = { print(self.name) }            // ❌ cycle
        onUpdate = { [weak self] in                // ✅
            guard let self else { return }
            print(self.name)
        }
    }
}
```

Not every closure needs `[weak self]`. **Non-escaping** closures (`map`, `forEach`) and closures that aren't stored by self (e.g. `UIView.animate`, `DispatchQueue.main.async`) don't create cycles. They only extend self's lifetime briefly.

</details>

### 6. Why should delegates be `weak`?

<details>
<summary>Answer</summary>

Typically a parent (e.g. a view controller) **owns** a child (a table view or custom view) strongly, and the child refers back to the parent through its delegate. If the delegate were strong, you'd have a cycle. So the delegate protocol is class-bound and the property is weak:

```swift
protocol CardViewDelegate: AnyObject { func didTap() }
class CardView: UIView { weak var delegate: CardViewDelegate? }
```

</details>

### 7. `[weak self]` vs `[unowned self]` – when to use which?

<details>
<summary>Answer</summary>

- **`[weak self]`** – the safe default. Use it whenever self might be deallocated before the closure runs (network callbacks, timers, notifications). Unwrap it with `guard let self else { return }`.
- **`[unowned self]`** – only when the closure **can't outlive** self (e.g. a lazy property closure on the same object). It's slightly cheaper, but crashes if you're wrong.

When in doubt, use **weak**.

</details>

### 8. What other common sources of leaks are there in iOS?

<details>
<summary>Answer</summary>

- **Timers**: `Timer.scheduledTimer(target: self…)` retains its target, and the run loop retains the timer. Use the block API with `[weak self]` and `invalidate()` it.
- **NotificationCenter** block observers that capture self strongly (store the token and remove it).
- **Combine**: a `sink` capturing self strongly while self stores the `AnyCancellable` → use `[weak self]`, or `assign(to:)` with `@Published`.
- **Tasks**: a long-running `Task { self.loop() }` keeps self alive until it completes. Cancel it in `deinit`/`onDisappear`, or capture weakly.
- **Parent ↔ child view controllers or coordinators** with strong back-references.
- `CADisplayLink` (retains its target), URLSession with a delegate (the session retains its delegate strongly until it's invalidated).
- Singletons holding view controllers.

</details>

### 9. How do you detect memory leaks?

<details>
<summary>Answer</summary>

- **Xcode Memory Graph Debugger** – pause the app and inspect the object graph. Leaked cycles are flagged with purple "!" markers.
- **Instruments → Leaks and Allocations** – track allocations over time, using generation (mark generation) analysis to spot growth.
- **`deinit` logging** – `deinit { print("\(Self.self) deinit") }` to check that screens are deallocated when dismissed.
- Unit tests that assert deallocation: `addTeardownBlock { [weak sut] in XCTAssertNil(sut) }`.
- Memory warnings and the Xcode memory gauge.

</details>

### 10. What is an autorelease pool? When do you need `autoreleasepool {}`?

<details>
<summary>Answer</summary>

Objective-C objects marked autorelease are released when the current **autorelease pool drains**. The main run loop drains a pool every iteration. In a **tight loop** that creates many temporary ObjC-bridged objects (e.g. processing thousands of images with `UIImage`/`NSData`), memory can spike before the pool drains. Wrap the loop body in `autoreleasepool { }` to release them each iteration. It's less relevant for pure Swift code.

</details>

### 11. Do structs suffer from retain cycles?

<details>
<summary>Answer</summary>

Structs can't form reference cycles **with themselves** (they're copied, not referenced). But a struct that holds a **closure or class reference** can participate in a cycle, e.g. a class stores a struct whose closure captures the class strongly.

</details>

### 12. What happens in `deinit`? When is it not called?

<details>
<summary>Answer</summary>

`deinit` runs right before a class instance is deallocated. Use it to invalidate timers, remove observers, cancel tasks and close resources. It's **not** called if the object is leaked (retain cycle), or when the app is terminated (the process is simply killed). Structs don't have a deinit (noncopyable `~Copyable` structs are the exception).

</details>

### 13. What is the difference between memory leak and abandoned memory?

<details>
<summary>Answer</summary>

- A **leak** is memory that's no longer referenced but never freed (a retain cycle with no outside references). Instruments' Leaks tool finds these.
- **Abandoned memory** is memory that's **still referenced** but never used again, e.g. an ever-growing cache or array. Leaks doesn't show it, so you need Allocations generation analysis.

</details>

### 14. How does the system handle low memory on iOS?

<details>
<summary>Answer</summary>

iOS has **no swap** for app memory. Under memory pressure, the system sends **memory warnings** (`applicationDidReceiveMemoryWarning`, `didReceiveMemoryWarning`, `UIApplication.didReceiveMemoryWarningNotification`). Free caches when you get one (`NSCache` evicts automatically). If pressure continues, **Jetsam** kills background apps, and even the foreground app if it exceeds its memory limit (this shows up as a jetsam event, not a crash log). Use `os_proc_available_memory()` to check how much headroom is left.

</details>

### 15. NSCache vs Dictionary for caching?

<details>
<summary>Answer</summary>

`NSCache` is **thread-safe**, **evicts automatically** under memory pressure, and supports `countLimit`/`totalCostLimit`, and it doesn't copy its keys. A plain `Dictionary` is none of these. Use `NSCache` for image and data memory caches. (NSCache requires class types, so keys are `NSString`/`NSURL` or wrapped values.)

</details>

---

[⬅ Previous: Swift Advanced](02-swift-advanced.md) · [⬅ Back to index](../README.md) · [Next: GCD & Operations ➡](04-gcd-operations.md)
