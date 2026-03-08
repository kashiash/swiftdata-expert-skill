# CloudKit Sync

## Setup

### 1. Enable iCloud Capability

In Xcode:
1. Select target → Signing & Capabilities
2. Click + Capability → iCloud
3. Check CloudKit
4. Select or create CloudKit container

### 2. Enable Background Modes

1. Click + Capability → Background Modes
2. Check "Remote notifications"

### 3. That's It!

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: User.self)
        // Automatically syncs with iCloud!
    }
}
```

## CloudKit Requirements

### All Properties Must Be Optional or Have Defaults

```swift
// ❌ WRONG: Non-optional without default
@Model
class User {
    var name: String // Sync fails silently!
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

// ✅ CORRECT: Optional properties
@Model
class User {
    var name: String?
    var age: Int?
    
    init(name: String? = nil, age: Int? = nil) {
        self.name = name
        self.age = age
    }
}

// ✅ ALSO CORRECT: Default values
@Model
class User {
    var name: String
    var age: Int
    
    init(name: String = "", age: Int = 0) {
        self.name = name
        self.age = age
    }
}
```

### No Unique Constraints

```swift
// ❌ WRONG: Unique attribute
@Model
class User {
    @Attribute(.unique) var email: String? // Sync fails!
}

// ✅ CORRECT: Remove unique
@Model
class User {
    var email: String?
}
```

### All Relationships Must Be Optional

```swift
// ❌ WRONG: Required relationship
@Model
class Post {
    var title: String?
    var author: User // Sync fails!
}

// ✅ CORRECT: Optional relationship
@Model
class Post {
    var title: String?
    var author: User?
}
```

## Disable CloudKit Sync

```swift
@main
struct MyApp: App {
    var container: ModelContainer
    
    init() {
        do {
            let config = ModelConfiguration(cloudKitDatabase: .none)
            container = try ModelContainer(
                for: User.self,
                configurations: config
            )
        } catch {
            fatalError("Failed to create container")
        }
    }
    
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)
    }
}
```

## Custom CloudKit Container

```swift
let config = ModelConfiguration(
    cloudKitDatabase: .private("com.example.mycontainer")
)
container = try ModelContainer(for: User.self, configurations: config)
```

## Sync Behavior

### Automatic Sync

- Syncs on app launch
- Syncs on app background
- Syncs on data changes
- Syncs periodically

### No Manual Sync

```swift
// ❌ No API to force sync
// CloudKit decides when to sync

// ✅ Trust automatic sync
// Just modify data, sync happens automatically
```

## Conflict Resolution

### Last Write Wins

```swift
// Device A: user.name = "Alice"
// Device B: user.name = "Bob"
// Result: Last save wins (usually "Bob")
```

### Custom Resolution

```swift
// No built-in API for custom conflict resolution
// Use timestamps to track changes

@Model
class User {
    var name: String?
    var updatedAt: Date?
    
    func update(name: String) {
        self.name = name
        self.updatedAt = .now
    }
}
```

## Testing CloudKit Sync

### Test on Real Devices

```swift
// Simulator sync is unreliable
// Always test on physical devices
```

### Multiple Devices

1. Install app on Device A
2. Add data
3. Wait for sync (check CloudKit Dashboard)
4. Install app on Device B
5. Verify data appears

### CloudKit Dashboard

Monitor sync at: https://icloud.developer.apple.com/dashboard

- View records
- Check sync status
- Debug issues

## Common Issues

### Silent Sync Failures

```swift
// Problem: Sync fails, no error
// Causes:
// - Non-optional properties without defaults
// - @Attribute(.unique) used
// - Required relationships

// Solution: Follow CloudKit requirements
@Model
class User {
    var name: String? // Optional
    var email: String? // No unique
    var profile: Profile? // Optional relationship
}
```

### Data Not Syncing

```swift
// Check:
// 1. iCloud capability enabled
// 2. Background modes enabled
// 3. Signed into iCloud on device
// 4. Network connection
// 5. CloudKit Dashboard for errors
```

### Duplicate Records

```swift
// Problem: Same record appears multiple times
// Cause: Creating objects on multiple devices simultaneously

// Solution: Use unique identifiers
@Model
class User {
    var uuid: UUID
    var name: String?
    
    init(name: String? = nil) {
        self.uuid = UUID()
        self.name = name
    }
}

// Check for duplicates
func deduplicateUsers(context: ModelContext) {
    let descriptor = FetchDescriptor<User>()
    let users = try? context.fetch(descriptor)
    
    var seen = Set<UUID>()
    users?.forEach { user in
        if seen.contains(user.uuid) {
            context.delete(user) // Duplicate
        } else {
            seen.insert(user.uuid)
        }
    }
}
```

## Patterns

### Sync Status Indicator

```swift
struct ContentView: View {
    @State private var isSyncing = false
    
    var body: some View {
        VStack {
            if isSyncing {
                ProgressView("Syncing...")
            }
            
            // Content
        }
        .onReceive(NotificationCenter.default.publisher(
            for: NSPersistentCloudKitContainer.eventChangedNotification
        )) { notification in
            // Monitor sync events
            if let event = notification.userInfo?[NSPersistentCloudKitContainer.eventNotificationUserInfoKey] as? NSPersistentCloudKitContainer.Event {
                isSyncing = !event.endDate.map { _ in true } ?? false
            }
        }
    }
}
```

### Offline Support

```swift
// SwiftData handles offline automatically
// Changes queued and synced when online

@Model
class Post {
    var title: String?
    var content: String?
    var syncStatus: SyncStatus?
    
    enum SyncStatus: String, Codable {
        case pending, synced, failed
    }
}

// Track sync status manually if needed
func createPost(title: String, content: String) {
    let post = Post(title: title, content: content)
    post.syncStatus = .pending
    modelContext.insert(post)
    
    // Will sync automatically when online
}
```

### Selective Sync

```swift
// Sync only specific models
let config1 = ModelConfiguration(
    for: User.self,
    cloudKitDatabase: .private("mycontainer")
)

let config2 = ModelConfiguration(
    for: TempData.self,
    cloudKitDatabase: .none // Don't sync
)

container = try ModelContainer(
    for: User.self, TempData.self,
    configurations: config1, config2
)
```

## Migration with CloudKit

### Schema Changes

```swift
// Simple changes (add property) work automatically
@Model
class User {
    var name: String?
    var email: String?
    var age: Int? // New property - syncs automatically
}

// Complex changes need migration plan
// See 08-migrations.md
```

### Existing CloudKit Data

```swift
// SwiftData can read existing CloudKit data
// If Core Data → SwiftData migration
// CloudKit data preserved automatically
```

## Debugging

### Enable CloudKit Logging

```swift
// Add to scheme environment variables:
// -com.apple.CoreData.CloudKitDebug 1
// -com.apple.CoreData.Logging.oslog 1
```

### Check CloudKit Dashboard

1. Go to https://icloud.developer.apple.com/dashboard
2. Select your container
3. View data in Development or Production
4. Check for errors in Telemetry

### Verify Sync

```swift
func verifySyncSetup() {
    print("Container: \(container)")
    print("Main context: \(container.mainContext)")
    
    // Check configuration
    if let config = container.configurations.first {
        print("CloudKit database: \(config.cloudKitDatabase)")
    }
}
```

## Best Practices

### 1. Always Use Optional Properties

```swift
@Model
class User {
    var name: String? // Not String
    var age: Int? // Not Int
}
```

### 2. No Unique Constraints

```swift
// Use UUID for uniqueness instead
@Model
class User {
    var uuid: UUID
    var email: String?
    
    init(email: String? = nil) {
        self.uuid = UUID()
        self.email = email
    }
}
```

### 3. Test on Real Devices

```swift
// Simulator sync is unreliable
// Always test on physical devices
```

### 4. Handle Conflicts with Timestamps

```swift
@Model
class User {
    var name: String?
    var updatedAt: Date?
    
    func update(name: String) {
        self.name = name
        self.updatedAt = .now
    }
}
```

### 5. Monitor CloudKit Dashboard

```swift
// Regularly check for:
// - Sync errors
// - Duplicate records
// - Schema issues
```

## Limitations

### No Public Database

```swift
// SwiftData only supports private database
// For public/shared, use Core Data with NSPersistentCloudKitContainer
```

### No Sharing

```swift
// No built-in sharing API
// For sharing, use Core Data or CloudKit directly
```

### No Custom Zones

```swift
// SwiftData uses default zone
// For custom zones, use Core Data
```

### No Manual Sync Control

```swift
// Can't force sync
// Can't pause sync
// Can't check sync status reliably
```

## When to Use CloudKit

### ✅ Good For

- Personal data sync across devices
- Automatic backup
- Simple multi-device apps
- Private user data

### ❌ Not Good For

- Public data sharing
- Complex sharing permissions
- Real-time collaboration
- Fine-grained sync control
- Apps requiring iOS 16 support

## Alternative: Manual CloudKit

```swift
// For more control, use CloudKit directly
import CloudKit

class CloudKitManager {
    let container = CKContainer.default()
    let database: CKDatabase
    
    init() {
        database = container.privateCloudDatabase
    }
    
    func saveUser(_ user: User) async throws {
        let record = CKRecord(recordType: "User")
        record["name"] = user.name
        record["age"] = user.age
        
        try await database.save(record)
    }
    
    func fetchUsers() async throws -> [User] {
        let query = CKQuery(recordType: "User", predicate: NSPredicate(value: true))
        let results = try await database.records(matching: query)
        
        return results.matchResults.compactMap { try? $0.1.get() }.map { record in
            User(
                name: record["name"] as? String,
                age: record["age"] as? Int
            )
        }
    }
}
```

## Checklist

Before enabling CloudKit:

- [ ] iCloud capability added
- [ ] Background modes enabled
- [ ] All properties optional or have defaults
- [ ] No @Attribute(.unique) used
- [ ] All relationships optional
- [ ] Tested on real devices
- [ ] CloudKit Dashboard monitored
- [ ] Conflict resolution strategy defined
- [ ] Offline support considered
- [ ] Migration plan for schema changes
