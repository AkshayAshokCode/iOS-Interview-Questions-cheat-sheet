# Coding, Scenario & Behavioral Questions

[⬅ Back to index](../README.md)

Covers: iOS-flavoured coding problems from live-coding rounds, output questions, debugging scenarios, and behavioral / project questions.

---

## Coding Problems

### 1. Implement an LRU cache in Swift.

<details>
<summary>Answer</summary>

```swift
final class LRUCache<Key: Hashable, Value> {
    private final class Node {
        let key: Key; var value: Value
        weak var prev: Node?; var next: Node?   // weak back-link avoids retain cycles
        init(key: Key, value: Value) { self.key = key; self.value = value }
    }
    private let capacity: Int
    private var map: [Key: Node] = [:]
    private var head: Node?   // most recent
    private var tail: Node?   // least recent

    init(capacity: Int) { precondition(capacity > 0); self.capacity = capacity }

    func get(_ key: Key) -> Value? {
        guard let node = map[key] else { return nil }
        moveToHead(node)
        return node.value
    }

    func set(_ value: Value, for key: Key) {
        if let node = map[key] { node.value = value; moveToHead(node); return }
        let node = Node(key: key, value: value)
        map[key] = node
        insertAtHead(node)
        if map.count > capacity, let lru = tail { remove(lru); map[lru.key] = nil }
    }

    private func insertAtHead(_ n: Node) {
        n.next = head; n.prev = nil
        head?.prev = n; head = n
        if tail == nil { tail = n }
    }
    private func remove(_ n: Node) {
        n.prev?.next = n.next; n.next?.prev = n.prev
        if head === n { head = n.next }
        if tail === n { tail = n.prev }
        n.prev = nil; n.next = nil
    }
    private func moveToHead(_ n: Node) { remove(n); insertAtHead(n) }
}
```

O(1) get and set. To make it thread-safe, wrap the state in an **actor** or protect it with a lock. Interviewers like it when you point out that `prev` is `weak`: with strong links in both directions, neighbouring nodes would form retain cycles and leak.

</details>

### 2. Implement a debounced search.

<details>
<summary>Answer</summary>

**Swift Concurrency (SwiftUI):**
```swift
@MainActor @Observable
final class SearchModel {
    var query = "" { didSet { scheduleSearch() } }
    private(set) var results: [Item] = []
    private var searchTask: Task<Void, Never>?
    private let service: SearchService
    init(service: SearchService) { self.service = service }

    private func scheduleSearch() {
        searchTask?.cancel()
        let q = query
        searchTask = Task {
            try? await Task.sleep(for: .milliseconds(300))   // debounce
            guard !Task.isCancelled, !q.isEmpty else { return }
            results = (try? await service.search(q)) ?? []
        }
    }
}
```
Or in SwiftUI: `.task(id: query) { try? await Task.sleep(for: .milliseconds(300)); await search() }`, which cancels the previous task automatically.

**Combine:** `$query.debounce(for: .milliseconds(300), scheduler: RunLoop.main).removeDuplicates().map { service.searchPublisher($0) }.switchToLatest()`.

</details>

### 3. Fetch multiple APIs in parallel and combine the results.

<details>
<summary>Answer</summary>

```swift
func loadDashboard() async throws -> Dashboard {
    async let profile = api.profile()
    async let orders = api.orders()
    async let offers = try? api.offers()        // optional: failure tolerated
    return try await Dashboard(profile: profile, orders: orders, offers: offers ?? [])
}
```

For a dynamic list of requests, use `withThrowingTaskGroup`, and limit concurrency by adding new tasks only as earlier ones finish.

</details>

### 4. Write a thread-safe counter / cache using an actor.

<details>
<summary>Answer</summary>

```swift
actor Counter {
    private(set) var value = 0
    func increment() -> Int { value += 1; return value }
}

// Lock-based alternative (sync API):
final class AtomicCounter: @unchecked Sendable {
    private let lock = NSLock(); private var value = 0
    func increment() -> Int { lock.withLock { value += 1; return value } }
}
```

</details>

### 5. Convert a completion-handler API to async/await.

<details>
<summary>Answer</summary>

Use `withCheckedThrowingContinuation`, and resume it **exactly once**. See [Swift Concurrency Q15](05-swift-concurrency.md).

</details>

### 6. Implement a generic network request function with retry.

<details>
<summary>Answer</summary>

```swift
func withRetry<T>(maxAttempts: Int = 3, initialDelay: Duration = .milliseconds(500),
                  _ operation: () async throws -> T) async throws -> T {
    var delay = initialDelay
    for attempt in 1...maxAttempts {
        do { return try await operation() }
        catch let error as URLError where attempt < maxAttempts && error.code != .cancelled {
            try await Task.sleep(for: delay)
            delay *= 2                                     // exponential backoff
        }
    }
    fatalError("unreachable")
}
```

Mention retrying only **transient** errors, adding jitter, respecting cancellation (`Task.sleep` throws if the task is cancelled), and keeping requests idempotent.

</details>

### 7. Build a reusable image view that loads from a URL with caching (UIKit).

<details>
<summary>Answer</summary>

```swift
final class RemoteImageView: UIImageView {
    private var task: Task<Void, Never>?
    func setImage(url: URL, placeholder: UIImage? = nil) {
        task?.cancel()
        image = placeholder
        task = Task { [weak self] in
            guard let image = try? await ImageLoader.shared.image(for: url),
                  !Task.isCancelled else { return }
            self?.image = image
        }
    }
    func cancel() { task?.cancel() }   // call from prepareForReuse
}
```

`ImageLoader` is an actor with an `NSCache` and in-flight task deduplication (see [System Design Q2](20-system-design.md)).

</details>

### 8. Common Swift output questions.

<details>
<summary>Answer</summary>

```swift
struct Point { var x = 0 }
var a = Point(); var b = a; b.x = 5
print(a.x)                 // 0 – value type copied

class Box { var x = 0 }
let c = Box(); let d = c; d.x = 5
print(c.x)                 // 5 – shared reference
```

```swift
var counter = 0
let closure = { [counter] in print(counter) }
counter = 10
closure()                  // 0 – captured value at creation (capture list)
let closure2 = { print(counter) }
counter = 20
closure2()                 // 20 – captured by reference
```

```swift
let arr = [1, 2, 3].lazy.map { n -> Int in print("map \(n)"); return n * 2 }
print(arr.first!)          // "map 1" then 2 – lazy evaluation
```

```swift
DispatchQueue.main.async { print("A") }
print("B")
// B, A  (when run on main thread)
```

</details>

### 9. Find the bug in this code.

<details>
<summary>Answer</summary>

```swift
class ProfileViewController: UIViewController {
    var viewModel = ProfileViewModel()
    override func viewDidLoad() {
        super.viewDidLoad()
        viewModel.onUpdate = { self.nameLabel.text = self.viewModel.name }   // ❌ 1
        URLSession.shared.dataTask(with: url) { data, _, _ in
            self.nameLabel.text = String(data: data!, encoding: .utf8)          // ❌ 2, 3
        }.resume()
        timer = Timer.scheduledTimer(timeInterval: 1, target: self,
                                     selector: #selector(tick), userInfo: nil, repeats: true)  // ❌ 4
    }
}
```

1. **Retain cycle**: the VC owns the view model, which owns the closure, which captures the VC strongly. Use `[weak self]`.
2. **UI updated off the main thread**: the URLSession callback runs on a background queue.
3. **Force unwrapping** `data!` crashes on a network error.
4. The **Timer retains its target**, so the VC is never deallocated. Use the block-based API with `[weak self]` and invalidate the timer.

</details>

### 10. Implement pagination in a view model.

<details>
<summary>Answer</summary>

```swift
@MainActor @Observable
final class FeedModel {
    private(set) var items: [Post] = []
    private(set) var isLoading = false
    private var cursor: String?
    private var hasMore = true
    private let api: FeedAPI
    init(api: FeedAPI) { self.api = api }

    func loadNextPageIfNeeded(current item: Post?) async {
        guard let item else { return await loadNextPage() }
        let thresholdIndex = items.index(items.endIndex, offsetBy: -5, limitedBy: items.startIndex) ?? 0
        if items.firstIndex(where: { $0.id == item.id }) == thresholdIndex { await loadNextPage() }
    }

    private func loadNextPage() async {
        guard !isLoading, hasMore else { return }
        isLoading = true; defer { isLoading = false }
        do {
            let page = try await api.feed(after: cursor)
            items += page.posts.filter { new in !items.contains { $0.id == new.id } }
            cursor = page.nextCursor
            hasMore = page.nextCursor != nil
        } catch { /* expose error state */ }
    }
}
// In the view: .task { await model.loadNextPageIfNeeded(current: post) } on each row
```

</details>

### 11. What DSA problems should you practice for iOS interviews?

<details>
<summary>Answer</summary>

Practice in Swift: two-sum (dictionary), valid parentheses (stack), reverse a linked list, merge intervals, top-K frequent (heap or sort), BFS/DFS on grids, binary search, sliding window (longest substring), group anagrams, LRU cache, flattening nested arrays, and string manipulation (Swift String indexing!). Know their complexities, and Swift specifics like `Character` vs `String.Index`.

</details>

---

## Scenario / Debugging Questions

### 12. Users report the app is slow to launch. How do you investigate?

<details>
<summary>Answer</summary>

1. **Measure**: Organizer launch metrics, MetricKit, the Instruments **App Launch** template on an older device.
2. **Check pre-main**: the number of dynamic frameworks, static initializers and `+load` methods.
3. **Check post-main**: the work in `didFinishLaunching` and the first view controller (SDK initialization, synchronous I/O, large storyboards, network calls on the main thread).
4. **Fix**: defer and lazy-load SDKs, static linking or mergeable libraries, a simpler first screen with cached data.
5. **Verify** with `XCTApplicationLaunchMetric` in CI, and monitor in production.

</details>

### 13. The app crashes only in production and you can't reproduce it. What do you do?

<details>
<summary>Answer</summary>

Look at the symbolicated crash reports (Organizer, Crashlytics): the exception type, the thread, and which devices, OS versions and app versions are affected. Look for patterns (memory pressure → jetsam, a background launch, a specific locale, a race condition). Add breadcrumbs and logs, try to reproduce with sanitizers (TSan/ASan), on low-end devices, with slow networks, or by relaunching from a background state. Ship the fix with a regression test, and roll out through a phased release.

</details>

### 14. A table view scrolls with stutters. How do you fix it?

<details>
<summary>Answer</summary>

Profile with Instruments (Animation Hitches, Time Profiler). Common fixes: move image decoding and resizing off the main thread, cache formatted strings and heights, avoid off-screen rendering (`shadowPath`, no masks), reduce the Auto Layout complexity in cells, use prefetching, and avoid synchronous Core Data fetches in `cellForRow`.

</details>

### 15. How would you migrate a large UIKit app to SwiftUI?

<details>
<summary>Answer</summary>

Incrementally:
1. Start with **new features** and leaf screens (settings, detail pages) using `UIHostingController`.
2. Build a **SwiftUI design system** of shared components.
3. Move state into observable view models that both UIKit and SwiftUI can use.
4. Keep navigation in UIKit coordinators at first, then migrate flows to `NavigationStack` when it makes sense.
5. Keep complex or performance-critical UIKit screens through representables until there's a reason to rewrite them.
6. Track crash and performance metrics, and raise the minimum iOS version strategically (Observation needs iOS 17).

</details>

### 16. How do you prevent a payment from being submitted twice?

<details>
<summary>Answer</summary>

Disable the button and show progress (state-driven UI), use a client-generated **idempotency key** that the server deduplicates on, persist the pending state so it survives termination, don't automatically retry non-idempotent requests without that key, and confirm the final status with the server (StoreKit: rely on `Transaction.updates` and finish transactions only after delivering the content).

</details>

---

## Behavioral & Project Questions

### 17. "Tell me about yourself."

<details>
<summary>Answer</summary>

60–90 seconds: **current role** (app, scale, your ownership) → **relevant achievements with numbers** ("cut launch time by 40%", "crash-free users from 99.2% to 99.8%", "led the SwiftUI migration of the checkout") → **why this role**. Keep it focused on iOS and impact.

</details>

### 18. "What's the most challenging bug or project you've worked on?" (STAR)

<details>
<summary>Answer</summary>

Use **STAR**: **Situation** → **Task** → **Action** (your technical steps in detail: tools, hypotheses, the fix) → **Result** (measurable impact plus what you learned or changed in the process).

Stories to prepare: a hard crash or memory leak, a performance win, adopting Swift 6 concurrency, an offline sync design, an App Store rejection you resolved, a disagreement about architecture, mentoring a teammate.

</details>

### 19. Common behavioral questions to prepare.

<details>
<summary>Answer</summary>

- A conflict with a teammate, designer or PM, and how you resolved it.
- A mistake that reached production and what you did about it.
- How you give and receive code review feedback.
- How you balance tech debt with feature work.
- How you estimate work and handle deadlines.
- A time you took ownership beyond your role.
- Why you're leaving your current role (stay positive).
- **Questions for the interviewer:** team size and structure, SwiftUI/UIKit split, release cadence, testing culture, the biggest technical challenge right now, and how success is measured in the first 6 months.

</details>

---

[⬅ Previous: What's New & Trending](21-whats-new-trending.md) · [⬅ Back to index](../README.md)
