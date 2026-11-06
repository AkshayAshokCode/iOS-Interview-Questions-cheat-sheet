# iOS (Mobile) System Design

[⬅ Back to index](../README.md)

Covers: how to approach the round, plus worked designs: image cache/loader, news feed, chat app, offline-first sync, file downloader, analytics SDK, video streaming, and common building blocks.

---

### 1. How should you approach a mobile system design interview?

<details>
<summary>Answer</summary>

A structured approach (~45 min):

1. **Clarify requirements (5 min)**
   - Functional: the core features; agree on scope.
   - Non-functional: offline support, real-time needs, scale (users, items), latency, battery and data usage, privacy and security, accessibility, the minimum iOS version, iPad support.
2. **High-level client architecture (10 min)** – UI (SwiftUI/UIKit) → ViewModel → Repository → (API client, persistence, cache) + coordinator/router + DI + modules.
3. **API design (5 min)** – endpoints, **cursor pagination**, payloads, REST vs GraphQL vs WebSocket, auth.
4. **Data layer (10 min)** – the local schema (SwiftData/Core Data), caching policy, **single source of truth**, sync and conflict resolution.
5. **Deep dives (10 min)** – pick 2–3: pagination, image loading, offline mode, real-time updates, background sync, error handling, performance.
6. **Wrap-up** – testing, monitoring (crashes, MetricKit), analytics, feature flags, security, accessibility, localization, rollout (phased release).

**Talk through the trade-offs** for every choice.

</details>

### 2. Design an image loading and caching library (like Kingfisher/Nuke).

<details>
<summary>Answer</summary>

**Requirements:** load remote images into views; memory and disk caches; downsampling; placeholders; cancellation when cells are reused; deduplication; prefetching.

**Pipeline:**
`load(url, targetSize)` → **memory cache** (`NSCache<NSURL, UIImage>`, keyed by URL + size; cost = decoded bytes) → hit? return it.
Miss → **disk cache** (files in `Caches/`, LRU eviction by size and age, file names = hashed URL) → hit? decode in the background.
Miss → **network** (URLSession) → write the data to disk → **downsample** with ImageIO to the target size → decode in the background → store in the memory cache → deliver on the main actor.

**Concurrency design:** an `actor ImageLoader` holding a `[URL: Task<UIImage, Error>]` dictionary of **in-flight tasks**, so concurrent requests for the same URL share one download. Cancel a request when the cell is reused (cancel its Task in `prepareForReuse` or `.onDisappear`). Use priorities (visible first) and prefetching.

**Extras:** clear the memory cache on memory warnings, handle animated images and progressive JPEGs, respect HTTP cache headers, and make the caches pluggable behind protocols.

</details>

### 3. Design a news feed (like Instagram/Twitter).

<details>
<summary>Answer</summary>

- **API:** `GET /feed?cursor=abc&limit=20` → `{ posts, nextCursor }`. Cursor pagination avoids duplicates when new posts arrive.
- **Data:** posts are persisted in SwiftData or Core Data (feed order + posts + authors). The UI observes the store (offline viewing, instant launch).
- **Loading:** show the cached feed first, then refresh. Infinite scroll triggers the next page near the end. Pull-to-refresh. A "New posts" pill instead of jumping the scroll position.
- **Media:** an image pipeline with downsampling and prefetching (`UICollectionViewDataSourcePrefetching` / onAppear lookahead). Video **autoplay only for the most visible cell**, with a single reused `AVPlayer` (or a small pool).
- **Interactions:** **optimistic likes** (update locally, call the API, roll back on failure), with a queue for offline actions.
- **Performance:** a diffable data source or `List` with stable IDs, precomputed text layouts and heights, avoiding off-screen rendering.
- **Analytics:** impression tracking (> 50% visible for > 1 s), batched.

</details>

### 4. Design a chat app (like WhatsApp/iMessage).

<details>
<summary>Answer</summary>

- **Storage:** local DB as the source of truth: `Conversation`, `Message(id, clientId, conversationId, senderId, body, status, createdAt, serverSeq)`, indexed by `(conversationId, serverSeq)`.
- **Real-time:** a **WebSocket** while in the foreground (heartbeats, reconnection with exponential backoff + jitter, resyncing from the last sequence number after reconnecting). In the background, **APNs** pushes, with a **Notification Service Extension** to decrypt or enrich the notification and possibly write to the shared App Group DB.
- **Sending:** an **optimistic UI**: insert locally with status `.sending` and a client-generated UUID (**idempotency**), then send. Mark it sent on the server ack and update it with delivered/read receipts. Failed messages are retried from an outbox queue (also when the app comes back to the foreground, or through BGTaskScheduler).
- **Ordering:** server sequence numbers, deduplicated by `clientId`.
- **Media:** upload first (with a background URLSession for big files), then send a message referencing the media URL. Thumbnails, blurhash placeholders.
- **Pagination:** load older messages by cursor as the user scrolls up, while keeping the scroll position.
- **Security:** end-to-end encryption (Signal protocol), keys in the Keychain or Secure Enclave.
- **Notifications:** grouped by thread, inline reply actions, communication notifications (with the sender's avatar).

</details>

### 5. Design an offline-first app with sync.

<details>
<summary>Answer</summary>

- The **local DB is the single source of truth**. The UI only reads locally.
- Each record has `id` (a client UUID), `updatedAt`, `version`, `syncState` (synced / pendingCreate / pendingUpdate / pendingDelete), and soft-delete tombstones.
- A **sync engine** (an actor):
  1. **push** pending changes (batched, idempotent),
  2. **pull** changes since `lastSyncToken`,
  3. **merge**.
  It's triggered on app foreground, on connectivity changes (`NWPathMonitor`), by silent pushes and by BGAppRefreshTask.
- **Conflict resolution:** last-write-wins by server timestamp, per-field merges, or user resolution, depending on the domain.
- **CloudKit option:** `NSPersistentCloudKitContainer` or SwiftData + CloudKit for automatic per-user sync.
- The UI shows the sync status and handles partial failures.

</details>

### 6. Design a large file downloader (e.g. offline videos/podcasts).

<details>
<summary>Answer</summary>

- A **background URLSession** with download tasks, so downloads continue when the app is suspended or killed. Handle `handleEventsForBackgroundURLSession`.
- **Pause/resume** with `cancel(byProducingResumeData:)` → `downloadTask(withResumeData:)`.
- A queue with a max concurrent count, and a Wi-Fi-only option (`allowsCellularAccess`, `allowsExpensiveNetworkAccess`).
- Persist the download state in the DB. Move finished files from the temp location to Application Support (and exclude them from backup).
- **Progress UI** through the delegate (`didWriteData`) mapped to observable state. A Live Activity for long downloads.
- Storage checks, checksum verification, and cleanup policies (auto-delete after watching).
- For **HLS video**, use `AVAssetDownloadURLSession` (with FairPlay for DRM content).

</details>

### 7. Design an analytics SDK.

<details>
<summary>Answer</summary>

- **API:** `Analytics.track("event", properties:)`. Non-blocking and thread-safe (an actor or serial queue).
- Events are written to a **persistent queue** (a file or SQLite) so they survive termination.
- **Batch upload** every N events or T seconds, and when the app moves to the background (`beginBackgroundTask`), with gzip compression, retries with backoff, and a cap on stored events.
- Context: app version, OS, device model, session ID, anonymous user ID. **Privacy:** consent gating (ATT/GDPR), no PII, an opt-out, a Privacy Manifest for the SDK.
- SDK quality: a small binary, no heavy dependencies, **no work at launch** (lazy initialization), distributed as an XCFramework or SPM package, a stable public API, and documentation.

</details>

### 8. Design a video streaming app (like Netflix/YouTube).

<details>
<summary>Answer</summary>

- **AVPlayer** with **HLS adaptive bitrate** streaming. **FairPlay** DRM for protected content.
- Player lifecycle: preloading, `AVPlayerItem` observation (status, buffering through `isPlaybackLikelyToKeepUp`), and handling interruptions (calls) and route changes (headphones unplugged → pause).
- **Picture in Picture** (`AVPictureInPictureController`), **AirPlay**, background audio (Audio background mode), Now Playing info and remote commands (`MPNowPlayingInfoCenter`, `MPRemoteCommandCenter`).
- Offline downloads with `AVAssetDownloadURLSession`. Resume position stored locally and synced.
- Catalog browsing: compositional layout carousels, image prefetching, and caching the catalog data.
- Analytics: startup time, rebuffering ratio, bitrate switches.

</details>

### 9. What caching strategies exist on mobile?

<details>
<summary>Answer</summary>

- **Cache-first / stale-while-revalidate** – show the cache instantly, then refresh. Best for feeds and profiles.
- **Network-first, falling back to the cache** – for data that must be fresh (prices, balances).
- **Cache-only / network-only.**
- **Layers:** memory (`NSCache`), disk (DB or files), HTTP (`URLCache`).
- **Invalidation:** TTLs, ETags, server-sent invalidations, versioned keys, and clearing on logout.

</details>

### 10. How do you design deep linking?

<details>
<summary>Answer</summary>

Universal Links + a custom scheme as a fallback, and push notification payloads, all parsed into a **`Route` enum** (`.product(id)`, `.order(id)`) in a single `DeepLinkParser`. The **router/coordinator** handles routes. It checks auth state (store a pending deep link until after login) and builds the navigation stack (`NavigationPath`). It handles cold launch (launch options or connection options) as well as a running app. Validate the parameters (security).

</details>

### 11. How do you handle app versioning and forced updates?

<details>
<summary>Answer</summary>

The backend exposes a **minimum supported app version** (and a recommended one). On launch, the app compares it with its own version and shows a blocking "Update required" screen or a soft prompt. **Version your APIs** (`/v2/`) and keep older versions working as long as old app versions are in use. Use **feature flags** (Firebase Remote Config or an in-house service) as kill switches, since a release can't be rolled back.

</details>

### 12. What non-functional aspects should you always mention?

<details>
<summary>Answer</summary>

- Performance (launch, scrolling, memory), battery and data usage.
- Offline behaviour, error, empty and loading states.
- Security (Keychain, pinning, ATS) and privacy (permissions, privacy manifest, data minimization).
- **Accessibility** (VoiceOver, Dynamic Type) and **localization** (RTL, pluralization with String Catalogs).
- Observability (crash reporting, MetricKit, logging, analytics).
- Testability and modularity, app size, and the minimum OS version strategy.

</details>


---

## More System Design Problems

### 13. Design Instagram Stories (iOS).

<details>
<summary>Answer</summary>

**Requirements:** a tray of users with unseen/seen rings; a full-screen viewer that auto-advances (~5 s per image, video length for video); tap to go back or forward, long-press to pause, swipe between users; 24 h expiry; it should feel instant.

- **API:** `GET /stories/tray` with story metadata (multiple media resolutions from a CDN, durations, `seen`, `expiresAt`). Cache it locally (SwiftData or a file) for an instant tray on launch.
- **Preloading is key:** prefetch the **next 1–2 stories' media**, and the first story of the next user. Images: download + **downsample** to screen size and decode in the background. Video: `AVPlayerItem` preloading/`preferredForwardBufferDuration`, or `AVAssetDownloadURLSession` for caching.
- **Viewer:** a horizontal paging container (a `UIPageViewController`, a collection view, or `TabView(.page)` / a SwiftUI `ScrollView` with `.scrollTargetBehavior(.paging)`). Per user, segmented progress bars driven by a **timer/CADisplayLink** that pauses on touch-down, when the app goes inactive, and during buffering. A **single reused `AVPlayer`** (or a small pool).
- **Seen state:** mark it locally right away and sync in batches (a queue that works offline).
- **Memory:** release off-screen players and images, use `NSCache` limits, and clear on memory warnings.
- **Edge cases:** expired stories, network loss mid-story (show a retry), accessibility (VoiceOver reading the author and time, pausing auto-advance for VoiceOver users), analytics batching.

</details>

### 14. Design a networking library for iOS.

<details>
<summary>Answer</summary>

- **Public API:** `func send<T: Decodable>(_ endpoint: Endpoint<T>) async throws -> T`, with endpoints as values (path, method, query, headers, body, decoder).
- **Request pipeline (Chain of Responsibility):** request **adapters** (auth headers, user agent, signing) → transport (`URLSession`) → response **validators** (status codes) → **retriers** (exponential backoff with jitter for transient errors, a 401 → token refresh through an actor with a single in-flight refresh) → **decoders**.
- **Concurrency:** async/await, `Sendable` types, cancellation propagated from Task to `URLSessionTask`, optional request deduplication (an actor keyed by request).
- **Caching:** `URLCache` policies + an optional custom response cache with TTL.
- **Errors:** a typed `NetworkError` (offline, timeout, unauthorized, server(code), decoding) with the underlying error attached.
- **Observability:** pluggable loggers/metrics (`URLSessionTaskMetrics`), redacting sensitive headers.
- **Testability:** protocol-based transport (a `URLProtocol` stub or an injected session), fixtures.
- **Packaging:** an SPM package, no third-party dependencies, documented API, semantic versioning.

</details>

### 15. Design a ride-hailing app (rider side, like Uber).

<details>
<summary>Answer</summary>

- **Flows:** set pickup/drop-off (MapKit search, `MKLocalSearchCompleter`) → fare estimate → request → driver matched → **live tracking** → trip → payment (Apple Pay) and rating.
- **Location:** Core Location `CLLocationUpdate.liveUpdates()`, with **When In Use** authorization (riders rarely need Always). Increase accuracy only during active states.
- **Real-time:** a **WebSocket** during an active trip (driver location every few seconds, ETA, state changes); **APNs** for trip events when backgrounded; a **Live Activity** (Dynamic Island / Lock Screen) showing the driver's ETA, updated through ActivityKit push.
- **Map:** smooth marker animation between updates (interpolate the coordinate and heading), route polylines (`MKDirections` or server-provided).
- **State machine:** `idle → searching → driverAssigned → arriving → onTrip → completed`, owned by an actor/store and **restored on relaunch** (the server is authoritative, re-synced on reconnect).
- **Reliability:** idempotent ride requests (client UUID), reconnection with backoff, graceful offline states.
- **Battery and privacy:** stop high-frequency location after the trip, minimize data collected, and write clear permission prompts.

</details>

### 16. How do voice and video calls work in an iOS app?

<details>
<summary>Answer</summary>

- **Media:** **WebRTC** (P2P, encrypted SRTP over UDP, Opus/H.264/VP8 codecs), with **STUN** for NAT discovery and **TURN** relays as a fallback. Group calls use an **SFU** server.
- **Signaling:** your own server (WebSocket) exchanges SDP offers/answers and ICE candidates.
- **Incoming calls when the app isn't running:** a **PushKit VoIP push** wakes the app, which **must immediately report the call to CallKit** (`CXProvider.reportNewIncomingCall`), or iOS terminates the app and may stop delivering VoIP pushes.
- **CallKit** gives the native call UI (lock screen, Recents, Bluetooth/CarPlay integration). **LiveCommunicationKit** (iOS 17.4+) is an alternative in regions where CallKit is restricted.
- **Audio:** `AVAudioSession` category `.playAndRecord`, mode `.voiceChat`/`.videoChat`, handling route changes (speaker, AirPods) and interruptions. Background modes: **Voice over IP** and **Audio**.
- **Video:** camera capture (AVFoundation), **Picture in Picture** for video calls (`AVPictureInPictureController.ContentSource` with a sample buffer display layer).

</details>

---

[⬅ Previous: Xcode, Build & Release](19-xcode-build-release.md) · [⬅ Back to index](../README.md) · [Next: What's New & Trending ➡](21-whats-new-trending.md)
