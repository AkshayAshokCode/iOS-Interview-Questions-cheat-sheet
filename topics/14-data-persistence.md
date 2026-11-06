# Data Persistence – UserDefaults, Keychain, Core Data, SwiftData

[⬅ Back to index](../README.md)

Covers: storage options, UserDefaults, Keychain, file system, Core Data stack, contexts & concurrency, migrations, SwiftData, SQLite/GRDB, CloudKit sync, choosing a store.

---

### 1. What are the persistence options in iOS?

<details>
<summary>Answer</summary>

| Need | Option |
|---|---|
| Small preferences/flags | **UserDefaults** / `@AppStorage` |
| Secrets (tokens, passwords) | **Keychain** |
| Documents, images, JSON blobs | **File system** (FileManager + Codable) |
| Structured, relational, queryable data | **SwiftData** (iOS 17+) / **Core Data** / SQLite (GRDB) |
| Sync across a user's devices | **CloudKit** (`NSPersistentCloudKitContainer` / SwiftData with CloudKit), `NSUbiquitousKeyValueStore` |
| Shared with extensions | **App Groups** container |

</details>

### 2. What is UserDefaults? What shouldn't you store there?

<details>
<summary>Answer</summary>

UserDefaults is a key-value store for **small** property-list values (settings, onboarding flags). It's stored as an **unencrypted plist** inside the app container and loaded into memory.

**Don't store:** tokens or passwords (use the Keychain), large data (it slows down app launch), or complex object graphs. It's thread-safe, and you use `UserDefaults(suiteName:)` to share with app groups.

</details>

### 3. What is the Keychain?

<details>
<summary>Answer</summary>

The Keychain is an **encrypted, secure store** managed by the system for small secrets: passwords, tokens, keys and certificates.
- Use the `SecItemAdd` / `SecItemCopyMatching` / `SecItemUpdate` / `SecItemDelete` APIs, or a wrapper library (KeychainAccess).
- **Accessibility levels** control when the item is readable, e.g. `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` (works for background tasks) or `WhenUnlocked`. `ThisDeviceOnly` means the item isn't included in backups.
- **Access control**: require biometrics or the passcode (`SecAccessControlCreateWithFlags` with `.biometryCurrentSet`).
- It can be shared between your apps through **Keychain access groups**.
- Keychain items **persist after the app is deleted** (handle stale data on reinstall, e.g. with a first-launch flag).

</details>

### 4. How do you save files? Which directory should you use?

<details>
<summary>Answer</summary>

Use `FileManager` + `Data.write(to:options:)` (use `.atomic` writes, plus `.completeFileProtection` for data protection). Directories:
- **Documents** – user data, backed up.
- **Application Support** – app data, backed up and hidden from the user.
- **Caches** – re-creatable data; can be purged.
- **tmp** – temporary.

Exclude large re-downloadable files from iCloud backup with `isExcludedFromBackup`. Use `URL.documentsDirectory` (iOS 16+).

</details>

### 5. What is Core Data? Is it a database?

<details>
<summary>Answer</summary>

Core Data is an **object graph and persistence framework**, not a database. It manages model objects, their relationships, change tracking, undo, validation, faulting and migrations. It usually persists to **SQLite**, but it can use other stores (binary, in-memory). It's mature (since iOS 3) and integrates with CloudKit.

</details>

### 6. Explain the Core Data stack.

<details>
<summary>Answer</summary>

- **NSManagedObjectModel** – the schema (`.xcdatamodeld`): entities, attributes, relationships.
- **NSPersistentStoreCoordinator** – connects the model to the persistent store(s) (SQLite).
- **NSManagedObjectContext** – a "scratchpad" where you create, fetch, modify and delete objects, then **`save()`** to persist the changes.
- **NSPersistentContainer** (iOS 10) – wraps all of the above: `viewContext` (main queue) plus `newBackgroundContext()` / `performBackgroundTask`.

</details>

### 7. How does Core Data handle concurrency?

<details>
<summary>Answer</summary>

**Managed objects and contexts are not thread-safe.** Each context is bound to a queue:
- `viewContext` – main queue, for the UI.
- Background contexts – private queue, for imports and heavy work.

Rules:
- Access a context only inside **`perform { }` / `performAndWait { }`** (or `await context.perform { }`).
- **Never pass `NSManagedObject`s between contexts or threads**. Pass the **`NSManagedObjectID`** and refetch with `context.object(with:)`.
- Merge changes into the view context with `automaticallyMergesChangesFromParent = true`, and set a merge policy (`NSMergeByPropertyObjectTrumpMergePolicy`).
- Enable the debug argument `-com.apple.CoreData.ConcurrencyDebug 1` to catch violations.

</details>

### 8. What is faulting in Core Data?

<details>
<summary>Answer</summary>

A **fault** is a placeholder object whose data **isn't loaded yet**. Core Data fetches the actual values from the store when you first access a property ("firing the fault"). This saves memory with large object graphs. Watch out for **N+1 fetches** when looping over relationships: set `relationshipKeyPathsForPrefetching` and `fetchBatchSize`, or use `returnsObjectsAsFaults = false` when you know you'll use everything.

</details>

### 9. What is NSFetchedResultsController?

<details>
<summary>Answer</summary>

It manages the results of a fetch request for table and collection views, **monitors changes** in the context, and provides section info. With diffable data sources, use `controller(_:didChangeContentWith snapshot:)`. In SwiftUI, the equivalent is **`@FetchRequest`** (Core Data) or **`@Query`** (SwiftData).

</details>

### 10. How do you handle Core Data migrations?

<details>
<summary>Answer</summary>

When the model changes, add a **new model version** (never edit the shipped version) and set it as current.
- **Lightweight migration** – automatic for simple changes (adding optional attributes or entities, renaming with a renaming ID). Enabled by default with NSPersistentContainer.
- **Heavyweight / custom migration** – mapping models + `NSEntityMigrationPolicy` for complex transformations.
- **Staged migrations** (iOS 17) – `NSStagedMigrationManager` for multi-step migrations with custom code between stages.

Test upgrades from **every** shipped version.

</details>

### 11. How do you do batch operations efficiently?

<details>
<summary>Answer</summary>

`NSBatchInsertRequest`, `NSBatchUpdateRequest` and `NSBatchDeleteRequest` work **directly on the store**, without loading objects into memory, so they're very fast for big imports or deletes. Because they bypass the context, you need to **merge the changes** manually (`NSManagedObjectContext.mergeChanges(fromRemoteContextSave:into:)`) or use persistent history tracking.

</details>

### 12. What is SwiftData?

<details>
<summary>Answer</summary>

SwiftData (iOS 17) is a **Swift-native persistence framework built on Core Data's engine**, using macros:

```swift
@Model final class Trip {
    var name: String
    var startDate: Date
    @Relationship(deleteRule: .cascade) var stops: [Stop] = []
    init(name: String, startDate: Date) { self.name = name; self.startDate = startDate }
}

@main struct TripsApp: App {
    var body: some Scene { WindowGroup { ContentView() }.modelContainer(for: Trip.self) }
}

struct TripList: View {
    @Query(sort: \Trip.startDate) private var trips: [Trip]
    @Environment(\.modelContext) private var context
    var body: some View {
        List(trips) { Text($0.name) }
        Button("Add") { context.insert(Trip(name: "Goa", startDate: .now)) }   // autosaves
    }
}
```

- **ModelContainer** (the store), **ModelContext** (track and save changes; it **autosaves**), `@Query` (live fetch for SwiftUI), `#Predicate` (type-safe queries), `FetchDescriptor`.
- **`@ModelActor`** for background work; pass `PersistentIdentifier`s between actors.
- Supports CloudKit sync, `VersionedSchema` + `SchemaMigrationPlan` for migrations, `#Unique`, `#Index` (iOS 18), and model inheritance (iOS 26).

</details>

### 13. SwiftData vs Core Data?

<details>
<summary>Answer</summary>

| SwiftData | Core Data |
|---|---|
| Code-first models with `@Model` (no `.xcdatamodeld`) | Visual model editor + `NSManagedObject` subclasses |
| Deep SwiftUI integration (`@Query`), Swift concurrency | Works with UIKit and SwiftUI; queue-based concurrency |
| iOS 17+ only; newer, so fewer advanced features | iOS 3+; mature, with every feature (batch requests, fine-grained control, persistent history) |
| `#Predicate` type-safe queries | `NSPredicate` strings |

Choose **SwiftData** for new SwiftUI apps targeting iOS 17+. Choose **Core Data** when you need older OS support, its advanced features, or have existing Core Data stores. They can coexist on the same store.

</details>

### 14. What about SQLite directly, GRDB or Realm?

<details>
<summary>Answer</summary>

- **SQLite** (C API) – full control, but lots of boilerplate.
- **GRDB** – a popular Swift SQLite toolkit: type-safe records, migrations, observation, good concurrency. Good when you want SQL control.
- **Realm** – an object database. **Its Atlas Device Sync was deprecated by MongoDB**, so it's less recommended for new projects.

</details>

### 15. How do you sync data with iCloud?

<details>
<summary>Answer</summary>

- **`NSPersistentCloudKitContainer`** (Core Data) or **SwiftData with CloudKit** – automatic syncing of the private database. There are constraints: all properties must be optional or have defaults, no unique constraints, relationships must be optional.
- **CloudKit** directly (`CKContainer`, `CKRecord`) for custom sync or public and shared databases.
- `NSUbiquitousKeyValueStore` for small key-value settings.

Handle conflicts, account changes and quota limits.

</details>

### 16. How do you share data between an app and its extensions/widgets?

<details>
<summary>Answer</summary>

Enable an **App Group** and use the shared container (`FileManager.containerURL(forSecurityApplicationGroupIdentifier:)`) for files and databases (point the Core Data or SwiftData store URL there), `UserDefaults(suiteName:)` for small data, and Keychain access groups for secrets. Call `WidgetCenter.shared.reloadTimelines` after data changes.

</details>

### 17. How do you design an offline-first data layer?

<details>
<summary>Answer</summary>

The **local store is the source of truth**: the UI observes the database (`@Query` or FRC). The network only syncs into the database. Writes go to the database first with a sync status, and a sync engine pushes pending changes when online (background tasks or app foreground). Resolve conflicts with timestamps or versions. See [System Design](20-system-design.md).

</details>


### 18. What is NSFetchRequest? How do NSPredicate and NSSortDescriptor work?

<details>
<summary>Answer</summary>

An **`NSFetchRequest<T>`** describes **what to fetch** from Core Data: an entity, plus:
- **`predicate`** (`NSPredicate`) – filtering, e.g. `NSPredicate(format: "age >= %d AND name CONTAINS[cd] %@", 18, query)` (`[cd]` = case- and diacritic-insensitive). Always use format arguments (`%@`), never string concatenation.
- **`sortDescriptors`** (`NSSortDescriptor(keyPath: \User.name, ascending: true)`) – ordering. Also used for sorting plain arrays (`NSArray.sortedArray(using:)`). The Swift-native equivalent is `SortDescriptor` / `KeyPathComparator`.
- `fetchLimit`, `fetchOffset` (paging), **`fetchBatchSize`** (loads objects in batches, saving memory), `propertiesToFetch`, `resultType` (`.countResultType`, `.dictionaryResultType`), `relationshipKeyPathsForPrefetching`.

Run it with `try context.fetch(request)` inside the context's queue. SwiftData uses `FetchDescriptor` + `#Predicate` instead.

</details>

### 19. When would you use Core Data instead of UserDefaults? How do you persist an array of custom objects?

<details>
<summary>Answer</summary>

Use **Core Data/SwiftData** when the data is **structured, large, relational or queried** (filtering, sorting, paging), changes often, or needs migrations or sync. Use **UserDefaults** for **small settings and flags** only, since the whole plist is loaded into memory at launch.

To persist an array of custom objects **simply** (small to medium size):

```swift
struct Note: Codable { let id: UUID; var text: String }
let url = URL.applicationSupportDirectory.appending(path: "notes.json")
try JSONEncoder().encode(notes).write(to: url, options: .atomic)
let loaded = try JSONDecoder().decode([Note].self, from: Data(contentsOf: url))
```

Create the directory if it doesn't exist. For larger or queryable data, move to SwiftData, Core Data or GRDB.

</details>

---

[⬅ Previous: Networking](13-networking.md) · [⬅ Back to index](../README.md) · [Next: Background Tasks & Notifications ➡](15-background-notifications.md)
