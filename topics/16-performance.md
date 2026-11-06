# Performance & Instruments

[⬅ Back to index](../README.md)

Covers: app launch time, hangs, scrolling hitches, memory & images, battery & energy, app size, Instruments tools, MetricKit & Xcode Organizer, crash debugging.

---

### 1. How do you measure and improve app launch time?

<details>
<summary>Answer</summary>

**Phases:** pre-main (dyld loading dynamic libraries, rebasing and binding, ObjC setup, static initializers) → `main` → `didFinishLaunching` → first frame rendered.

**Measure:** the Instruments **App Launch** template, the Xcode Organizer **Launch Time** metric (from real users), MetricKit, and `XCTApplicationLaunchMetric` in performance tests. Target: the **first frame within ~400 ms**. The watchdog kills apps that take far too long (about 20 s).

**Improve:**
- Fewer **dynamic frameworks** (static linking, mergeable libraries).
- Nothing heavy in `+load`, static initializers or `didFinishLaunching`: defer SDK setup and analytics, initialize lazily, and move work off the main thread.
- A simple first screen. Show cached data first, then refresh.
- Avoid synchronous I/O and network calls during launch, and avoid large storyboards for the initial screen.

**Cold** launch = the app isn't in memory (the slowest). **Warm** = parts are cached by the system. **Resume** = from the background.

</details>

### 2. What is a hang? How do you detect it?

<details>
<summary>Answer</summary>

A **hang** is when the main thread is blocked and doesn't respond to user input for **more than ~250 ms** (a "microhang" from 100 ms). Causes: synchronous network, file or database work on main, heavy computation, lock contention, layout thrashing, and synchronous `DispatchQueue.main.sync` calls from background threads.

**Detect:** Xcode **hang detection** (in development builds), Instruments **Hangs/Time Profiler**, the Organizer **Hangs** report, MetricKit `MXHangDiagnostic`, and the Thread Performance Checker. **Fix:** move work off the main thread (async/await, background actors), reduce work, and avoid main-thread `sync` calls.

</details>

### 3. What causes scrolling jank (hitches)? How do you fix it?

<details>
<summary>Answer</summary>

To hit 60/120 fps (ProMotion), each frame has **16.7 ms / 8.3 ms**. A **hitch** is a frame that's displayed late.
- **Commit hitches** (main thread) – expensive `cellForRow`/`body`, layout, text measurement, image decoding, Auto Layout complexity.
- **Render hitches** (render server/GPU) – off-screen rendering (shadows without a path, masks, rounded corners with clipping), large blended layers, heavy blur effects.

Fixes: reuse cells, precompute and cache data, **downsample and decode images in the background**, set `shadowPath`, avoid masks, use opaque views, and prefetch. Measure with the Instruments **Animation Hitches** tool, the hitch-rate metric in the Organizer, and `XCTOSSignpostMetric.scrollDecelerationMetric` in tests.

</details>

### 4. How do you handle large images efficiently?

<details>
<summary>Answer</summary>

Memory = **width × height × 4 bytes**, regardless of file size: a 4032×3024 photo uses about **48 MB** decoded. So:
- **Downsample** to the display size with ImageIO (`CGImageSourceCreateThumbnailAtIndex` with `kCGImageSourceThumbnailMaxPixelSize`), or `UIImage.preparingThumbnail(of:)`.
- Decode **off the main thread** (`byPreparingForDisplay()`).
- Cache the downsampled images in an `NSCache`, and release them on memory warnings.
- Use asset catalogs (app thinning), and vectors (SF Symbols, PDF/SVG) for icons.

</details>

### 5. Which Instruments do you use and for what?

<details>
<summary>Answer</summary>

- **Time Profiler** – CPU hotspots (call trees, heaviest stack traces).
- **Allocations** – memory growth, generation analysis (abandoned memory).
- **Leaks** – retain cycles and leaked memory.
- **Hangs / Animation Hitches** – UI responsiveness.
- **App Launch** – launch time phases.
- **Network** – HTTP traffic and timings.
- **Energy Log / Power Profiler** – battery impact.
- **Core Animation / GPU** – rendering issues.
- **SwiftUI** instrument (Xcode 26) – view body updates and their causes.
- **Swift Concurrency** – tasks and actors.
- **os_signpost** – custom intervals for your own code (`OSSignposter`).

Always profile in **release** configuration on a **real device**.

</details>

### 6. What are MetricKit and the Xcode Organizer?

<details>
<summary>Answer</summary>

- **Xcode Organizer** – aggregated, anonymized metrics from users who opted in: launch time, hangs, memory, battery, disk writes, scrolling hitches, terminations, and crash and energy reports.
- **MetricKit** – the app receives **daily metric payloads** (`MXMetricPayload`) and **diagnostics** (`MXDiagnosticPayload`: crashes, hangs, CPU exceptions, disk-write exceptions) with call stacks, which you can send to your own backend.

Combine them with Crashlytics or Sentry for real-time crash reporting.

</details>

### 7. How do you reduce battery usage?

<details>
<summary>Answer</summary>

Batch and defer networking (`isDiscretionary`, BGTaskScheduler), avoid polling (use push), use the lowest **location accuracy** that works and stop updates when you're done, avoid timers that keep waking the CPU, reduce GPU work (animations, blurs), respect **Low Power Mode** (`ProcessInfo.isLowPowerModeEnabled`), and minimize work in the background. Measure with the **Energy Log** / Xcode energy gauge, and the Organizer battery metrics.

</details>

### 8. How do you reduce app size?

<details>
<summary>Answer</summary>

- **App Thinning**: slicing (per-device variants), on-demand resources, bitcode (deprecated since Xcode 14).
- Asset catalogs; compress images (HEIC/WebP-like formats where supported); remove unused assets (tools like FengNiao or Periphery for dead code); vectors and SF Symbols.
- Avoid heavy dependencies for small features.
- Strip debug symbols in release, and enable dead-code stripping. Use `-Osize` where it makes sense.
- Analyze with the **App Size Report** (export from the archive with app thinning) and the App Store Connect size info.

</details>

### 9. How do you debug and symbolicate crashes?

<details>
<summary>Answer</summary>

- Crash logs show the **exception type** (`EXC_BAD_ACCESS` / `SIGSEGV` = a bad memory access; `EXC_BREAKPOINT` / `SIGTRAP` = a Swift runtime trap such as force unwrapping nil, an array index out of range, or `fatalError`; `0x8badf00d` = **watchdog** timeout, often a blocked main thread during launch; `0xdead10cc` = holding a file lock while suspended; jetsam = memory limit).
- **Symbolication** maps addresses to symbols using **dSYM** files. Upload the dSYMs to Crashlytics or Sentry.
- Tools: the Organizer Crashes tab, breakpoints (exception and Swift error breakpoints), **Address Sanitizer** (memory corruption), **Thread Sanitizer** (races), **Zombie Objects** (messages sent to deallocated objects), and the Main Thread Checker.

</details>

### 10. What are common causes of crashes in iOS apps?

<details>
<summary>Answer</summary>

Force unwrapping nil, out-of-bounds indexes, unsafe force casts (`as!`), UI updates from background threads, data races (dictionary mutated during enumeration), table and collection view **inconsistency exceptions** (use diffable data sources), the watchdog (slow launch or a blocked main thread), memory limits (jetsam), Core Data threading violations, unrecognized selectors (ObjC), missing privacy usage descriptions, and stack overflows from infinite recursion.

</details>

### 11. How do you write performance tests?

<details>
<summary>Answer</summary>

`measure(metrics: [XCTClockMetric(), XCTMemoryMetric(), XCTCPUMetric()]) { … }` in XCTest, with **baselines** that fail on regression. UI tests can measure launch (`XCTApplicationLaunchMetric`) and scrolling (`XCTOSSignpostMetric`). Run them in a consistent environment (a real device, release-like build) in CI.

</details>

### 12. What is the impact of Swift language choices on performance?

<details>
<summary>Answer</summary>

- Value types avoid ARC and heap allocation.
- Use `final` and `private` for static dispatch, generics and `some` instead of `any` existentials, and Whole Module Optimization.
- Avoid unnecessary copies of large structs: CoW collections help, and `borrowing`/`consuming` parameters and noncopyable types help in hot paths.
- `ContiguousArray` for non-class elements.
- Reserve capacity for arrays you'll fill in a loop.
- Avoid repeated `String` index operations (String is Unicode-correct, so indexing is O(n)).

</details>

### 13. How do you detect excessive memory usage?

<details>
<summary>Answer</summary>

Use the Xcode memory gauge and memory graph, **Allocations** with generations (repeat an action and check that memory goes back to baseline), **VM Tracker** (dirty memory, which is what counts towards the jetsam limit), the Organizer memory metrics and jetsam terminations, and `os_proc_available_memory()`. Common culprits: image caches without limits, leaked view controllers, and large data loaded all at once.

</details>


### 14. Why are caching and compression important on mobile devices?

<details>
<summary>Answer</summary>

Mobile devices have **slow, unreliable and often metered networks**, **limited battery** (the radio is one of the biggest power consumers, and every request keeps it awake for seconds), and limited memory and storage.
- **Caching** avoids repeated downloads: instant UI, offline support, fewer radio wake-ups, less server load. Use layers (memory → disk → HTTP cache) with sensible invalidation.
- **Compression** (gzip/brotli for responses, which URLSession decodes automatically; efficient image formats like HEIC/WebP; downsampled image sizes) means fewer bytes, so it's faster and cheaper in data and battery.
- Respect **Low Data Mode** (`allowsConstrainedNetworkAccess = false` for non-essential transfers) and expensive networks.

</details>

---

[⬅ Previous: Background & Notifications](15-background-notifications.md) · [⬅ Back to index](../README.md) · [Next: Testing ➡](17-testing.md)
