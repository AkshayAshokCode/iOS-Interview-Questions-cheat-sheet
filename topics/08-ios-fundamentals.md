# iOS Fundamentals – App Lifecycle & Platform

[⬅ Back to index](../README.md)

Covers: app states & lifecycle, AppDelegate vs SceneDelegate, SwiftUI App lifecycle, launch sequence, Info.plist, bundle & sandbox, responder chain, deep links, extensions, iOS architecture layers.

---

### 1. What are the iOS app states?

<details>
<summary>Answer</summary>

- **Not running** – not launched, or terminated.
- **Inactive** – in the foreground but **not receiving events**: a transitional state (an incoming call, Control Center pulled down, app switching).
- **Active** – in the foreground and receiving events.
- **Background** – running code but not visible. You get a few seconds by default; longer only with background modes or tasks.
- **Suspended** – in memory but **not executing code**. The system can terminate it at any time **without notifying** the app (memory pressure).

</details>

### 2. What are the AppDelegate lifecycle methods?

<details>
<summary>Answer</summary>

- `application(_:willFinishLaunchingWithOptions:)` / **`didFinishLaunchingWithOptions:`** – app setup (SDKs, appearance, push registration). Keep it fast, since it affects launch time.
- `applicationDidBecomeActive` / `applicationWillResignActive`
- `applicationDidEnterBackground` / `applicationWillEnterForeground`
- `applicationWillTerminate` – not called if the app is suspended and then killed.
- Remote notifications: `didRegisterForRemoteNotificationsWithDeviceToken`.
- `application(_:configurationForConnecting:options:)` – scene configuration (iOS 13+).

With scenes, the **UI lifecycle** callbacks (active, background) move to the **SceneDelegate**. The AppDelegate keeps the **process-level** events.

</details>

### 3. AppDelegate vs SceneDelegate? Why were scenes introduced?

<details>
<summary>Answer</summary>

iOS 13 introduced **multi-window** support on iPad (and Mac Catalyst). One app process can have **multiple scenes** (UI instances, each with its own window).
- **AppDelegate** – process lifecycle: launch, push registration, background URLSession events, scene session management.
- **SceneDelegate** – per-window UI lifecycle: `scene(_:willConnectTo:options:)` (create the window and root view controller), `sceneDidBecomeActive`, `sceneWillResignActive`, `sceneDidEnterBackground`, `sceneWillEnterForeground`, `sceneDidDisconnect`, and deep links (`scene(_:openURLContexts:)`, `continue userActivity`).

At WWDC 2025, Apple announced that the UIScene lifecycle will be **mandatory**: in the major release after iOS 26, apps built with the latest SDK that don't adopt scenes won't launch. Migrate legacy AppDelegate-only apps now.

</details>

### 4. What is the SwiftUI App lifecycle?

<details>
<summary>Answer</summary>

```swift
@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate   // if needed
    @Environment(\.scenePhase) private var scenePhase
    var body: some Scene {
        WindowGroup { ContentView() }
            .onChange(of: scenePhase) { _, phase in /* .active / .inactive / .background */ }
    }
}
```

`App` + `Scene` (`WindowGroup`, `DocumentGroup`, `Settings`, `ImmersiveSpace` on visionOS) replace AppDelegate/SceneDelegate. Use `@UIApplicationDelegateAdaptor` for things that still need an app delegate (push tokens). Handle deep links with `.onOpenURL`.

</details>

### 5. What happens when an iOS app launches?

<details>
<summary>Answer</summary>

1. The user taps the icon; **SpringBoard** asks the system to launch the app process.
2. The kernel creates the process. **dyld** loads the executable and **dynamic libraries/frameworks**, rebases and binds symbols, and runs static initializers (ObjC `+load`, C++ constructors).
3. `main()` runs → `UIApplicationMain` creates `UIApplication` and the AppDelegate, and starts the **main run loop**.
4. `didFinishLaunching` runs → the scene connects → the window and root view controller are created → `viewDidLoad`, then layout.
5. The **first frame** is rendered, and the launch screen is replaced.

Optimizing launch: fewer dynamic frameworks (merge them or use static linking), no heavy work in `+load`/static initializers, deferring SDK initialization, a light first screen, and measuring with **Instruments App Launch** and Xcode Organizer launch metrics. See [Performance](16-performance.md).

</details>

### 6. What is Info.plist? Name important keys.

<details>
<summary>Answer</summary>

Info.plist is a property list with app **metadata** that the system reads:
- Bundle identifier, version (`CFBundleShortVersionString`) and build (`CFBundleVersion`), display name.
- **Privacy usage descriptions** (required, or the app crashes on access): `NSCameraUsageDescription`, `NSLocationWhenInUseUsageDescription`, `NSPhotoLibraryUsageDescription`, `NSUserTrackingUsageDescription`.
- `UIBackgroundModes`, `UIApplicationSceneManifest`, supported orientations, `UILaunchScreen`.
- `CFBundleURLTypes` (custom URL schemes), `LSApplicationQueriesSchemes` (`canOpenURL`).
- `NSAppTransportSecurity` (ATS exceptions).

Many keys can now be set in **build settings** (`INFOPLIST_KEY_*`) instead.

</details>

### 7. What is the app sandbox? What are the main directories?

<details>
<summary>Answer</summary>

Each app runs in its **own sandboxed container** and can't access other apps' files. Directories:
- **Bundle** (`Bundle.main`) – read-only app resources and code.
- **Documents/** – user-created data, **backed up** to iCloud, can be exposed in the Files app.
- **Library/Application Support/** – app-generated data (databases), backed up. It isn't visible to the user.
- **Library/Caches/** – re-creatable data. **Not backed up**, and the system may purge it when storage is low.
- **tmp/** – temporary files that can be deleted at any time.

Sharing between your own apps and extensions: **App Groups** (shared container + `UserDefaults(suiteName:)`) and the **shared Keychain**.

</details>

### 8. What is the Responder Chain?

<details>
<summary>Answer</summary>

The responder chain is a chain of `UIResponder` objects (views, view controllers, window, application) that **events travel through**. A touch first goes through **hit-testing** (`hitTest(_:with:)` / `point(inside:)` find the deepest view under the touch), which becomes the first responder for that touch. If that view doesn't handle an event, the event is passed to its `next` responder: superview → view controller → window → `UIApplication` → AppDelegate.

Also used for: **first responder** handling (the keyboard, `becomeFirstResponder`/`resignFirstResponder`), actions sent to a `nil` target, `UIKeyCommand`s, and edit menus.

</details>

### 9. How do you expand the tappable area of a button / let touches pass through a view?

<details>
<summary>Answer</summary>

- Expand it: override `point(inside:with:)` to return true for an inset rect: `bounds.insetBy(dx: -10, dy: -10).contains(point)`.
- Pass touches through: override `hitTest(_:with:)` to return `nil` when the hit view is `self`, or set `isUserInteractionEnabled = false`.

Hit-testing ignores views that are hidden, have `alpha < 0.01`, or have user interaction disabled.

</details>

### 10. What are URL schemes vs Universal Links?

<details>
<summary>Answer</summary>

- **Custom URL schemes** (`myapp://product/42`) – easy to set up, but **any app can claim the same scheme**, they're not secure, and they fail (with an error) if the app isn't installed.
- **Universal Links** (`https://example.com/product/42`) – standard web links verified through the **`apple-app-site-association`** file on your domain + the **Associated Domains** entitlement. They open the app if it's installed and the website otherwise. Secure and unique. Handled in `scene(_:continue userActivity:)` or `.onOpenURL`.

</details>

### 11. What are App Extensions? Name a few.

<details>
<summary>Answer</summary>

Extensions extend app functionality into the system. They run in a **separate process** with limited memory and APIs:
- **Widgets** (WidgetKit; also Live Activities and Control Center controls).
- **Share**, **Action**, **Notification Service** (modify push content, e.g. download images or decrypt) and **Notification Content** (custom UI).
- **Keyboard**, **Today** (deprecated), **Intents / App Intents** (Siri, Shortcuts), **Safari extensions**, **File Provider**, **Photo editing**.

They share data with the containing app through **App Groups**.

</details>

### 12. What are the iOS architecture layers?

<details>
<summary>Answer</summary>

From top to bottom:
1. **Cocoa Touch** – UIKit, SwiftUI, push notifications, gesture recognizers.
2. **Media** – Core Graphics, Core Animation, AVFoundation, Metal, Core Image.
3. **Core Services** – Foundation, Core Data, CloudKit, Core Location, networking.
4. **Core OS** – the Darwin/XNU kernel, security, Bluetooth, the file system.

</details>

### 13. What is the difference between frame and bounds?

<details>
<summary>Answer</summary>

- **frame** – the view's position and size in its **superview's** coordinate system.
- **bounds** – the view's own coordinate system: origin (usually (0,0); changing it scrolls the content, which is how UIScrollView works) and size.

When a view is **rotated or transformed**, `frame` becomes the bounding box (and is undefined for layout), while `bounds` stays the same. Use `bounds` for internal drawing and layout.

</details>

### 14. What is `UIApplication.shared` and `UIWindow`?

<details>
<summary>Answer</summary>

`UIApplication.shared` is the singleton that represents the running app: it handles event routing, opening URLs, the badge count, and registration for remote notifications. `UIWindow` is the root container for the view hierarchy and dispatches events to views. Each scene has one or more windows, and `rootViewController` sets the initial UI.

</details>

### 15. How does state restoration work?

<details>
<summary>Answer</summary>

The system may terminate a suspended app; when the user returns, they expect to be where they left off.
- **UIKit:** `NSUserActivity`-based scene restoration (`stateRestorationActivity(for:)`), or the older `restorationIdentifier` APIs.
- **SwiftUI:** `@SceneStorage` for per-scene UI state and `@AppStorage` for preferences.

Persist important data (drafts) yourself.

</details>

### 16. What is the main bundle and how do you load resources?

<details>
<summary>Answer</summary>

`Bundle.main` contains the app's code and resources. Load resources with `Bundle.main.url(forResource:withExtension:)`, `UIImage(named:)` (cached), `UIImage(contentsOfFile:)` (not cached), and `NSLocalizedString` / **String Catalogs** (`.xcstrings`) for localization. Swift packages use `Bundle.module` for their resources.

</details>

### 17. What is `Codable`'s role in `UserDefaults` / what is `@AppStorage`?

<details>
<summary>Answer</summary>

`UserDefaults` stores small property-list types (Bool, Int, String, Data, Date, arrays and dictionaries of these). To store custom types, encode them to `Data` with `JSONEncoder`. `@AppStorage("key")` is the SwiftUI property wrapper over UserDefaults; it re-renders views when the value changes. Never store secrets in UserDefaults (it's an unencrypted plist). Use the Keychain instead.

</details>


---

## Platform & Frameworks

### 18. What is Cocoa / Cocoa Touch? Why do classes start with prefixes like `NS` and `UI`?

<details>
<summary>Answer</summary>

- **Cocoa** is the macOS application framework stack (Foundation + AppKit). **Cocoa Touch** is the iOS equivalent: **Foundation + UIKit** (plus related frameworks such as notifications and gesture recognizers). It covers app lifecycle, UI, events and multitasking. It **doesn't include** lower-level frameworks like Core Graphics, Core Data or AVFoundation, which sit in other layers.
- **Prefixes:** Objective-C has **no namespaces**, so Apple prefixes classes to avoid name collisions: `NS` (NeXTSTEP, Foundation), `UI` (UIKit), `CG` (Core Graphics), `CA` (Core Animation), `CL` (Core Location), `MK` (MapKit). Apple reserves two-letter prefixes, so third-party ObjC code should use **three letters**. Swift has **module namespaces**, so Swift types don't need prefixes.

</details>

### 19. Name important Apple frameworks and what they're used for.

<details>
<summary>Answer</summary>

| Area | Frameworks |
|---|---|
| UI & graphics | UIKit, SwiftUI, **Core Graphics** (2D drawing, paths), **Core Animation** (layers, animations), **Core Image** (image filters), Metal (low-level GPU), PDFKit, Vision (image analysis) |
| Media | **AVFoundation** (camera, audio/video capture and playback, editing), AVKit (player UI), PhotoKit/PhotosUI, MediaPlayer, Core Audio, ReplayKit (screen recording/broadcast) |
| Location & maps | **Core Location** (GPS, geofencing, **iBeacons**: Bluetooth LE proximity beacons), **MapKit** |
| Data | Foundation, Core Data, SwiftData, CloudKit, Core Spotlight |
| Connectivity | Network, URLSession, **CoreBluetooth**, MultipeerConnectivity, Core NFC, WatchConnectivity |
| Games | **SpriteKit** (2D), **SceneKit** (3D, now soft-deprecated in favour of RealityKit), **GameplayKit** (AI, pathfinding, state machines, randomization), **Game Center/GameKit** (leaderboards, achievements, multiplayer) |
| AR/3D | ARKit, **RealityKit** |
| Health & home | **HealthKit** (health data with granular permissions), **HomeKit** (smart home accessories), WorkoutKit |
| Commerce | **StoreKit** (in-app purchases, subscriptions, reviews), **PassKit** (Apple Pay, Wallet passes) |
| System integration | WidgetKit, ActivityKit, App Intents, UserNotifications, BackgroundTasks, CallKit, PushKit, WeatherKit, Foundation Models |

You don't need depth in all of these. Know what each is for, and talk about the ones you've used.

</details>

### 20. What are the different ways to show web content?

<details>
<summary>Answer</summary>

1. **Open in Safari**: `UIApplication.shared.open(url)` / SwiftUI `openURL`. This leaves your app.
2. **`SFSafariViewController`** (SwiftUI: a wrapper, or `.safariView` from libraries): an in-app Safari with **shared cookies and AutoFill**, Reader mode and content blockers. Secure, with no access to the page from your code. **Best for showing external links.**
3. **`WKWebView`** (SwiftUI `WebView` in iOS 26): an **embeddable, customizable** browser engine. You control navigation, inject JavaScript, and use message handlers. Use it for hybrid content or your own HTML.
4. **`ASWebAuthenticationSession`**: for **OAuth/login flows**. It shares Safari cookies (SSO) and returns a callback URL.

`UIWebView` was removed (rejected by the App Store).

</details>

### 21. How do you work with the file system (list a directory, read and write files)?

<details>
<summary>Answer</summary>

Use **`FileManager`**:

```swift
let docs = URL.documentsDirectory                       // iOS 16+
let files = try FileManager.default.contentsOfDirectory(at: docs,
                includingPropertiesForKeys: [.fileSizeKey, .contentModificationDateKey])
try data.write(to: docs.appending(path: "notes.json"), options: [.atomic, .completeFileProtection])
let loaded = try Data(contentsOf: docs.appending(path: "notes.json"))
try FileManager.default.removeItem(at: oldURL)
FileManager.default.fileExists(atPath: url.path())
```

To **serialize custom objects**, use `Codable` + `JSONEncoder`/`PropertyListEncoder` → `Data` → write to a file. Use `enumerator(at:)` for recursive listing. Do file I/O **off the main thread** for large files.

</details>

### 22. How do you play audio or a custom sound?

<details>
<summary>Answer</summary>

- **Short UI sounds** (under 30 s, no controls): `AudioServicesPlaySystemSound` with a `SystemSoundID` created from the file. Low latency, but limited.
- **Local audio files** with control (volume, looping, progress): **`AVAudioPlayer`**.
- **Streaming or remote audio and video**: **`AVPlayer`** / `AVQueuePlayer`.
- **Advanced processing** (mixing, effects, synthesis): **`AVAudioEngine`**.
- **Always configure `AVAudioSession`** (category `.playback` to play while the silent switch is on or in the background, `.ambient` for mixing with other audio), handle **interruptions** (calls) and route changes, and enable the **Audio background mode** for background playback, with Now Playing info.

</details>

### 23. What are Handoff, Continuity and iCloud services?

<details>
<summary>Answer</summary>

- **Handoff** – start an activity on one Apple device and continue it on another signed into the same Apple ID. Apps advertise an **`NSUserActivity`** (with a type and `userInfo`, or a web URL), and the other device restores it in `scene(_:continue:)`. It also works between apps and websites (with a web page URL and associated domains).
- **Continuity** – the broader feature set: Universal Clipboard, Continuity Camera, AirDrop, iPhone Mirroring, Sidecar.
- **iCloud services:** **CloudKit** (databases and assets), **iCloud Drive / documents** (`UIDocument`, ubiquity containers), **key-value storage** (`NSUbiquitousKeyValueStore`), **iCloud Keychain** (passwords and passkeys), and Core Data/SwiftData sync through CloudKit.

</details>

### 24. What are App Clips?

<details>
<summary>Answer</summary>

App Clips are **small parts of an app** (under 15 MB; up to 50 MB for some cases since iOS 17) that users can launch **instantly without installing** the full app, from App Clip Codes, NFC tags, QR codes, Safari banners, Maps or Messages. They're used for quick tasks (pay for parking, rent a scooter, order food). They're a separate target in the same project and can use Sign in with Apple and Apple Pay. They're cleared after a period of inactivity, and can prompt the user to install the full app (data migrates through a shared App Group).

</details>

### 25. What is a sensible minimum iOS deployment target?

<details>
<summary>Answer</summary>

A common rule of thumb is **the current major version minus 1 or 2** (in late 2026: iOS 17 or iOS 18). Adoption of new iOS versions is fast: the vast majority of active devices run the last two versions. Things to weigh:
- **User analytics** (what share of your users are on older versions? Enterprise and emerging-market apps may need older targets).
- **API needs**: Observation and SwiftData need **iOS 17**; the newest SwiftUI features need recent versions.
- **The cost** of `#available` branches and testing on old OS versions.
- Hardware support: each new iOS drops older devices.

Raising the target usually lets you remove a lot of compatibility code. Decide based on data.

</details>

### 26. How does multitasking work on iOS?

<details>
<summary>Answer</summary>

- **App-level:** only the foreground app runs freely. Background apps are **suspended** quickly unless they use specific background modes (audio, location, VoIP, Bluetooth), finite tasks, BGTaskScheduler or background URLSession. The system can terminate suspended apps at any time to reclaim memory.
- **UI multitasking (iPad):** **Split View** and **Slide Over**, **Stage Manager** (resizable overlapping windows), and fully **resizable windows** in iPadOS 26. Multiple scenes (windows) per app, and **Picture in Picture** for video. Apps must support **flexible layouts** (size classes, no fixed screen sizes) and handle scene lifecycles independently.
- The **app switcher** snapshot is taken when the app goes to the background (hide sensitive data).

</details>

---

[⬅ Previous: OOP, SOLID & Design Patterns](07-oop-solid-design-patterns.md) · [⬅ Back to index](../README.md) · [Next: UIKit ➡](09-uikit.md)
