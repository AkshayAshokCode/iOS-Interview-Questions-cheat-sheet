# Objective-C Essentials

[⬅ Back to index](../README.md)

Many companies still have Objective-C in their codebases, and interviews for those roles include ObjC questions. Covers: Swift vs ObjC, the runtime & messaging, properties & attributes, memory (MRC/ARC), categories & extensions, blocks, KVC/KVO, selectors, NSError, interop.

---

### 1. Swift vs Objective-C?

<details>
<summary>Answer</summary>

| Objective-C | Swift |
|---|---|
| A superset of C with Smalltalk-style **dynamic messaging** | A modern, **statically typed**, compiled language |
| Messages to `nil` are no-ops (silent bugs) | **Optionals** make nil explicit |
| Header (`.h`) + implementation (`.m`) files | Single `.swift` files, module namespaces |
| Mostly reference types (NSObject) | **Value types** (struct, enum) are first-class |
| Very dynamic: swizzling, KVO, `performSelector` | Mostly static dispatch (faster, safer); dynamic only with `@objc dynamic` |
| No generics (only lightweight generics for interop) | Full generics, protocols with associated types |
| Blocks | Closures |
| Manual or ARC memory management | ARC |
| Mature, stable ABI since forever | ABI-stable since Swift 5; concurrency, macros, SwiftUI |

They interoperate: a **bridging header** exposes ObjC to Swift, and the generated `Module-Swift.h` exposes `@objc` Swift to ObjC.

</details>

### 2. How does message sending work in the Objective-C runtime?

<details>
<summary>Answer</summary>

`[object doSomething:arg]` compiles to **`objc_msgSend(object, @selector(doSomething:), arg)`**. At runtime:
1. The runtime follows the object's **`isa` pointer** to its class.
2. It looks up the selector in the class's **method cache**, then in its **method list**, walking up the **superclass chain**.
3. When it finds the **IMP** (a function pointer to the implementation), it calls it and caches it.
4. If nothing is found, **message forwarding** kicks in: `+resolveInstanceMethod:` (add a method dynamically) → `-forwardingTargetForSelector:` (redirect to another object) → `-methodSignatureForSelector:` + `-forwardInvocation:` (full forwarding). If all of these fail → `doesNotRecognizeSelector:` → crash: **"unrecognized selector sent to instance"**.

Because dispatch happens at runtime, ObjC supports **swizzling**, KVO (isa-swizzling) and proxies. Classes are objects too (metaclasses).

</details>

### 3. What happens when you send a message to `nil`?

<details>
<summary>Answer</summary>

**Nothing: no crash.** Messaging `nil` returns `nil`/`0`/`NO` (or a zeroed struct for most struct returns). It's convenient (`[nil count]` is 0), but it **hides bugs**: an uninitialized object silently does nothing. Swift's optionals make this explicit.

</details>

### 4. What are property attributes? (`atomic`/`nonatomic`, `strong`/`weak`/`copy`/`assign`, `readonly`)

<details>
<summary>Answer</summary>

```objc
@property (nonatomic, strong) NSArray *items;
@property (nonatomic, copy) NSString *name;
@property (nonatomic, weak) id<MyDelegate> delegate;
@property (nonatomic, assign) NSInteger count;
@property (nonatomic, readonly) BOOL isLoaded;
```

- **`atomic`** (the default) – the synthesized getter and setter are thread-safe **for that single property access** (you always get a whole value). It does **not** make the object thread-safe (e.g. a check-then-act sequence isn't atomic), and it's slower. **`nonatomic`** – no locking, faster. It's almost always used on iOS.
- **`strong`** – owning reference (+1 retain). The default for objects under ARC.
- **`weak`** – non-owning, and **zeroing** (set to nil when the object deallocates). Used for delegates and IBOutlets.
- **`copy`** – stores a **copy** of the assigned value. Use it for **`NSString`, `NSArray`, `NSDictionary` and blocks**, so a mutable subclass (`NSMutableString`) passed in can't change behind your back.
- **`assign`** – a plain assignment for **primitives** (`NSInteger`, `CGFloat`, `BOOL`). Using `assign` on objects creates dangling pointers.
- **`unsafe_unretained`** – like weak but not zeroing (dangerous).
- **`readonly` / `readwrite`** – with a `readwrite` redeclaration in a class extension to make the property internally writable.

</details>

### 5. What is the difference between `copy` and `retain`/`strong`?

<details>
<summary>Answer</summary>

- **`retain`/`strong`** – increments the **reference count of the same object**, so both references point to **one instance**. If it's mutable, changes are visible through both.
- **`copy`** – creates a **new object** (through `copyWithZone:` from `NSCopying`) and owns that. For immutable classes like `NSString`, `copy` usually just retains (an optimization). For `NSMutableString`, it creates an immutable snapshot.
- `mutableCopy` returns a mutable copy.

</details>

### 6. What are `@synthesize` and `@dynamic`? `_name` vs `self.name`?

<details>
<summary>Answer</summary>

- **`@synthesize name = _name;`** tells the compiler to generate the **backing instance variable** and the accessors. Since Xcode 4.4 this happens **automatically** (auto-synthesis), so it's only needed for custom ivar names, or for properties declared in protocols.
- **`@dynamic name;`** tells the compiler **not** to generate accessors, because they'll be provided **at runtime** (Core Data's `NSManagedObject` properties, or methods resolved dynamically).
- **`_name`** accesses the **ivar directly**: no getter/setter, **no KVO notifications**, no lazy loading or custom logic, no copy semantics. Use it in `init`, `dealloc` and inside custom accessors.
- **`self.name`** goes through the **accessor methods** (KVO-compliant, respects `copy`/`strong`, overridable). Use it everywhere else.

</details>

### 7. Categories vs class extensions?

<details>
<summary>Answer</summary>

- **Category** (`@interface NSString (Validation)`) adds **methods to an existing class**, even one without source access, in a separate file. It **can't add stored instance variables** (only through **associated objects**: `objc_setAssociatedObject`). If two categories define the same method, which one wins is undefined, so prefix category methods (`xyz_isValidEmail`).
- **Class extension** (`@interface MyClass ()`, an "anonymous category") is declared in the `.m` file. It **can add properties and ivars** and private methods, and redeclare `readonly` properties as `readwrite`. It requires the class's source.
- Swift's **extensions** are like categories: they add methods and computed properties and protocol conformance, but no stored properties.

</details>

### 8. What are blocks? How do you avoid retain cycles with them?

<details>
<summary>Answer</summary>

Blocks are Objective-C **closures**: `void (^completion)(NSData *data, NSError *error)`. They **capture** variables from the surrounding scope (by value, unless the variable is marked **`__block`**, which makes it mutable inside the block). Store them with the `copy` attribute (they start on the stack and are copied to the heap).

**Retain cycle:** `self` → stored block → `self`. The fix is the **weak–strong dance**:

```objc
__weak typeof(self) weakSelf = self;
self.onUpdate = ^{
    __strong typeof(weakSelf) strongSelf = weakSelf;
    if (!strongSelf) return;
    [strongSelf reload];
};
```

Referencing an **ivar** (`_name`) inside a block also captures `self` implicitly.

</details>

### 9. What is the difference between MRC and ARC? What did `retain`/`release`/`autorelease` do?

<details>
<summary>Answer</summary>

Before ARC (iOS 5), developers managed memory **manually (MRC)**:
- `alloc`/`new`/`copy`/`mutableCopy` → you **own** the object (+1).
- `retain` → +1; `release` → −1 (dealloc at 0).
- `autorelease` → release **later**, when the current **autorelease pool drains** (used to return objects from methods without leaking).
- `dealloc` → release your ivars, then `[super dealloc]`.

**ARC** inserts these calls at compile time. You can't call them yourself under ARC (`dealloc` can still be implemented for cleanup, without calling super). The same rules still matter for understanding ownership, `@autoreleasepool` and bridging (`__bridge`, `__bridge_transfer` with Core Foundation objects, which ARC doesn't manage).

</details>

### 10. Does `performSelector:withObject:afterDelay:` retain the object?

<details>
<summary>Answer</summary>

**Yes.** It schedules a timer on the current run loop that **retains the receiver (target) and the argument** until the selector runs (or the request is cancelled with `cancelPreviousPerformRequestsWithTarget:`). That can **keep a view controller alive** longer than expected. Also, `performSelector:` with a selector unknown at compile time triggers an ARC warning ("may cause a leak"), because ARC can't know the method's memory semantics.

</details>

### 11. What are selectors? What is `@selector` / `#selector`?

<details>
<summary>Answer</summary>

A **selector** (`SEL`) is the **name of a method** used for runtime lookup, not the implementation. `@selector(buttonTapped:)` in ObjC, and `#selector(buttonTapped(_:))` in Swift (the method must be `@objc`). Selectors are used for target-action, `NSTimer`, `performSelector`, `respondsToSelector:` (checking optional delegate methods) and notifications. Calling a selector that doesn't exist crashes with "unrecognized selector".

</details>

### 12. How do KVC and KVO work in Objective-C?

<details>
<summary>Answer</summary>

- **KVC:** `[obj valueForKey:@"name"]`, `setValue:forKeyPath:`, collection operators (`@sum`, `@avg`, `@distinctUnionOfObjects`). The runtime searches for accessor methods, then ivars, by naming convention.
- **KVO:** `addObserver:forKeyPath:options:context:` + `observeValueForKeyPath:ofObject:change:context:`, and **you must remove the observer** (crashes before iOS 11 otherwise). Implemented by **isa-swizzling**: the runtime creates a hidden subclass (`NSKVONotifying_MyClass`) whose setters call `willChangeValueForKey:`/`didChangeValueForKey:`. Changes made by **direct ivar access aren't observed** unless you call those methods manually.

</details>

### 13. How does error handling work in Objective-C (`NSError`)?

<details>
<summary>Answer</summary>

ObjC uses **`NSError **` out-parameters** for recoverable errors (exceptions, `@try/@catch`, are only for **programmer errors** and are rarely caught):

```objc
NSError *error = nil;
BOOL ok = [data writeToURL:url options:0 error:&error];
if (!ok) { NSLog(@"%@ (%ld) %@", error.domain, (long)error.code, error.localizedDescription); }
```

`NSError` = **domain** (string) + **code** (integer) + **userInfo** (`NSLocalizedDescriptionKey`, `NSUnderlyingErrorKey`). Check the **return value**, not the error pointer. In Swift, these methods are imported as **`throws`**, and Swift `Error`s bridge to `NSError`.

</details>

### 14. What is `instancetype` vs `id`?

<details>
<summary>Answer</summary>

- **`id`** – a pointer to **any** Objective-C object, with no compile-time type checking.
- **`instancetype`** – a **related result type**: "an instance of the class this method is called on (or its subclass)". It's used for initializers and factory methods (`+ (instancetype)userWithName:`), so `[MyUser userWithName:]` is typed `MyUser *`, which enables compiler checks and correct Swift imports (`Self`).

</details>

### 15. How do you make Objective-C code work well with Swift?

<details>
<summary>Answer</summary>

- **Nullability annotations** (`nullable`, `nonnull`, `NS_ASSUME_NONNULL_BEGIN/END`). Without them, Swift imports everything as implicitly unwrapped optionals (`String!`).
- **Lightweight generics** (`NSArray<NSString *> *`), so Swift sees `[String]` instead of `[Any]`.
- **`NS_SWIFT_NAME`** for Swift-friendly names, **`NS_ENUM`/`NS_OPTIONS`** for proper Swift enums and option sets, `NS_REFINED_FOR_SWIFT`, `NS_SWIFT_UNAVAILABLE`.
- `NS_DESIGNATED_INITIALIZER` for correct initializer inheritance.
- **Swift → ObjC:** classes must inherit from `NSObject` (or be `@objc`), and members marked `@objc`/`@objcMembers`. Swift-only features (structs, enums with associated values, generics, async functions with non-ObjC types) **can't be exposed**.

</details>

### 16. Is it faster to look up an item in an NSArray or an NSSet?

<details>
<summary>Answer</summary>

**NSSet** (or Swift `Set`): `containsObject:` is a **hash lookup, O(1) on average** (it needs proper `hash`/`isEqual:` implementations). **NSArray** `containsObject:` is a **linear scan, O(n)**. Arrays keep order and allow duplicates; sets are unordered and unique. **Iterating** over all elements is O(n) for both (arrays are slightly faster thanks to contiguous storage).

</details>

---

[⬅ Previous: Coding & Behavioral](22-coding-scenarios.md) · [⬅ Back to index](../README.md) · [Next: Accessibility & Localization ➡](24-accessibility-localization.md)
