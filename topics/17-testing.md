# Testing – XCTest, Swift Testing, UI Tests

[⬅ Back to index](../README.md)

Covers: testing pyramid, XCTest, Swift Testing, test doubles, testing async code, network tests, UI tests, snapshot tests, TDD, code coverage, CI.

---

### 1. What types of tests do iOS apps have?

<details>
<summary>Answer</summary>

- **Unit tests** – test logic in isolation (view models, services, parsers). Fast. The majority of tests.
- **Integration tests** – several components together (repository + real Core Data in memory, API client + `URLProtocol` stub).
- **UI tests** (XCUITest) – drive the app like a user through accessibility. Slow and more flaky, so use them for critical flows only.
- **Snapshot tests** – compare rendered UI to reference images.
- **Performance tests** – `measure {}` with baselines.

The **testing pyramid**: lots of unit tests, fewer integration tests, few UI tests.

</details>

### 2. What is XCTest? Explain the test lifecycle.

<details>
<summary>Answer</summary>

XCTest is Apple's testing framework. Subclass `XCTestCase`; test methods start with `test`.
- `setUp()` / `setUpWithError()` / `async setUp()` – runs **before each** test (create the SUT).
- `tearDown()` – runs after each test (cleanup). `addTeardownBlock` for per-test cleanup.
- `class func setUp()` – once for the whole class.
- Assertions: `XCTAssertEqual`, `XCTAssertTrue`, `XCTAssertNil`, `XCTAssertThrowsError`, `XCTUnwrap`, `XCTFail`.

Each test method runs on a **new instance** of the test case class.

</details>

### 3. What is Swift Testing? How is it different from XCTest?

<details>
<summary>Answer</summary>

**Swift Testing** (Xcode 16, open source) is a modern, macro-based framework:

```swift
import Testing

@Suite("Cart")
struct CartTests {
    let cart = Cart()          // fresh instance per test (init = setUp)

    @Test("Adding item increases total")
    func addItem() {
        cart.add(.init(price: 10))
        #expect(cart.total == 10)
    }

    @Test(arguments: [0, 1, 5])
    func quantity(_ count: Int) throws {
        let item = try #require(Item.sample(count: count))
        #expect(item.count == count)
    }
}
```

Differences:
- `@Test` functions (no `test` prefix), with `struct` suites, so you can use `init`/`deinit` for setup and teardown.
- **`#expect`** – one macro instead of many `XCTAssert*` variants, with rich failure output. **`#require`** stops the test on failure (and unwraps optionals).
- **Parameterized tests** (`arguments:`), **traits** (`.tags`, `.enabled(if:)`, `.disabled`, `.timeLimit`, `.bug`).
- Tests run **in parallel by default** (in-process).
- Async-first: `confirmation { }` replaces expectations for events.

It coexists with XCTest. **UI tests and performance tests still use XCTest.**

</details>

### 4. What are test doubles? Mock vs Stub vs Fake vs Spy?

<details>
<summary>Answer</summary>

- **Dummy** – passed in to fill a parameter, never used.
- **Stub** – returns predefined data (`getUser()` returns a fixed user).
- **Spy** – records calls so you can assert on them later (`saveCallCount == 1`).
- **Mock** – preconfigured with expectations, and verifies the interactions.
- **Fake** – a working lightweight implementation (an in-memory repository).

In Swift, they're usually **hand-written conformances to protocols**, since there's no runtime mocking like Mockito. Macros or Sourcery can generate them.

</details>

### 5. How do you unit test a ViewModel?

<details>
<summary>Answer</summary>

```swift
final class MockProfileService: ProfileService {
    var result: Result<Profile, Error> = .success(.sample)
    private(set) var fetchCount = 0
    func fetch() async throws -> Profile { fetchCount += 1; return try result.get() }
}

@MainActor
struct ProfileViewModelTests {
    @Test func loadSuccess() async {
        let service = MockProfileService()
        let sut = ProfileViewModel(service: service)
        await sut.load()
        #expect(sut.state == .loaded(.sample))
        #expect(service.fetchCount == 1)
    }

    @Test func loadFailureShowsError() async {
        let service = MockProfileService(); service.result = .failure(URLError(.notConnectedToInternet))
        let sut = ProfileViewModel(service: service)
        await sut.load()
        guard case .failed = sut.state else { Issue.record("Expected failure"); return }
    }
}
```

Key points: inject protocols, test the **state outputs** for given inputs, cover the error paths, and make the async work awaitable (return the Task or make the method `async`).

</details>

### 6. How do you test asynchronous code?

<details>
<summary>Answer</summary>

- **async/await:** mark the test `async` and `await` the call directly.
- **Callbacks/delegates (XCTest):** `let exp = expectation(description:)` → `exp.fulfill()` in the callback → `await fulfillment(of: [exp], timeout: 1)` (or `wait(for:)` in sync tests).
- **Swift Testing:** `await confirmation(expectedCount: 1) { confirm in sut.onEvent = { confirm() }; … }`.
- **Combine:** collect values with `sink`, or iterate `publisher.values`.
- Control **time** by injecting a `Clock` or scheduler, never real sleeps.

</details>

### 7. How do you test networking without hitting the server?

<details>
<summary>Answer</summary>

- Test layers above the network with a **mock `APIClient` protocol**.
- Test the client itself with a **`URLProtocol` subclass** registered in an ephemeral session configuration:

```swift
final class MockURLProtocol: URLProtocol {
    nonisolated(unsafe) static var handler: ((URLRequest) throws -> (HTTPURLResponse, Data))?
    override class func canInit(with request: URLRequest) -> Bool { true }
    override class func canonicalRequest(for r: URLRequest) -> URLRequest { r }
    override func startLoading() {
        do {
            let (response, data) = try Self.handler!(request)
            client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
            client?.urlProtocol(self, didLoad: data)
            client?.urlProtocolDidFinishLoading(self)
        } catch { client?.urlProtocol(self, didFailWithError: error) }
    }
    override func stopLoading() {}
}
```

- JSON **fixtures** in the test bundle for decoding tests.

</details>

### 8. How do UI tests (XCUITest) work?

<details>
<summary>Answer</summary>

UI tests run in a **separate process** and control the app through the **accessibility layer**:

```swift
let app = XCUIApplication()
app.launchArguments = ["-uiTesting"]          // switch to mock data
app.launch()
app.textFields["emailField"].tap()
app.textFields["emailField"].typeText("a@b.com")
app.buttons["loginButton"].tap()
XCTAssertTrue(app.staticTexts["Welcome"].waitForExistence(timeout: 5))
```

Best practices: stable **`accessibilityIdentifier`s**, **launch arguments or environment** to stub the network and reset state, `waitForExistence` instead of sleeps, the **Page Object pattern** for maintainable tests, and few tests (only the critical journeys). Xcode can also record UI tests.

</details>

### 9. What is snapshot testing?

<details>
<summary>Answer</summary>

It renders a view or view controller and compares it **pixel by pixel** (or as text) to a stored reference image. It catches unintended UI changes across devices, Dynamic Type sizes, dark mode and locales. Popular library: **swift-snapshot-testing** (Point-Free). Record on a fixed simulator and OS version to avoid noise.

</details>

### 10. How do you make code testable?

<details>
<summary>Answer</summary>

Dependency injection with protocols, no singletons accessed directly (inject them), **pure functions** for logic, separating the UI from logic (view models), injecting **time, randomness, UUIDs, locale and file system**, small types with single responsibilities, and avoiding static mutable state.

</details>

### 11. What is TDD?

<details>
<summary>Answer</summary>

**Test-Driven Development**: **Red** (write a failing test) → **Green** (the minimal code to pass) → **Refactor** (clean up while keeping the tests green). Benefits: a testable design, regression safety, and executable documentation. In interviews, show that you understand it, even if you apply it pragmatically.

</details>

### 12. What is code coverage and how much is enough?

<details>
<summary>Answer</summary>

Enable "Gather coverage" in the scheme's test options, and look at the report in Xcode or export it with `xccov` (and Slather in CI). Aim for meaningful coverage of **business logic** (view models, services, parsers). 100% isn't the goal. Coverage shows untested code, not test quality.

</details>

### 13. How do you test for memory leaks in unit tests?

<details>
<summary>Answer</summary>

```swift
func trackForMemoryLeaks(_ instance: AnyObject, file: StaticString = #filePath, line: UInt = #line) {
    addTeardownBlock { [weak instance] in
        XCTAssertNil(instance, "Instance should have been deallocated. Potential memory leak.", file: file, line: line)
    }
}
```

Call it on the SUT and its collaborators in each test.

</details>

### 14. What are Test Plans and how do you run tests in CI?

<details>
<summary>Answer</summary>

**Test Plans** (`.xctestplan`) configure multiple **configurations** (languages, sanitizers, environment variables), select tests, enable parallelization and repetition (for finding flaky tests). In CI, run `xcodebuild test -scheme App -testPlan Unit -destination 'platform=iOS Simulator,name=iPhone 16'` on Xcode Cloud, GitHub Actions or Bitrise, collect the `.xcresult` bundle, and publish the results and coverage.

</details>

### 15. How do you deal with flaky tests?

<details>
<summary>Answer</summary>

Remove timing dependencies (no `sleep`; use expectations and injected clocks), isolate shared state (reset UserDefaults and the database between tests, avoid singletons), stub the network, disable animations in UI tests, make tests order-independent, and use test repetition ("run until failure") to reproduce the flakiness.

</details>

---

[⬅ Previous: Performance](16-performance.md) · [⬅ Back to index](../README.md) · [Next: Security & Privacy ➡](18-security-privacy.md)
