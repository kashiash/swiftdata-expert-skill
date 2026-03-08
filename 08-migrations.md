# Migrations and Schema Changes

## Automatic Migrations

### Simple Changes (Automatic)

```swift
// Version 1
@Model
class User {
    var name: String
    var age: Int
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

// Version 2 - Add property (automatic)
@Model
class User {
    var name: String
    var age: Int
    var email: String? // New property - migrates automatically
    
    init(name: String, age: Int, email: String? = nil) {
        self.name = name
        self.age = age
        self.email = email
    }
}
```

**Automatic migrations work for:**
- Adding optional properties
- Adding properties with defaults
- Removing properties
- Renaming properties (with @Attribute(originalName:))

### Renaming Properties

```swift
// Version 1
@Model
class User {
    var username: String
}

// Version 2 - Rename (automatic with hint)
@Model
class User {
    @Attribute(originalName: "username") var name: String
}
```

## Manual Migrations

### Schema Versions

```swift
import SwiftData

enum UserSchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    
    static var models: [any PersistentModel.Type] {
        [User.self]
    }
    
    @Model
    class User {
        var name: String
        var age: Int
        
        init(name: String, age: Int) {
            self.name = name
            self.age = age
        }
    }
}

enum UserSchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    
    static var models: [any PersistentModel.Type] {
        [User.self]
    }
    
    @Model
    class User {
        var name: String
        var age: Int
        var email: String
        
        init(name: String, age: Int, email: String = "") {
            self.name = name
            self.age = age
            self.email = email
        }
    }
}
```

### Migration Plan

```swift
enum UserMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [UserSchemaV1.self, UserSchemaV2.self]
    }
    
    static var stages: [MigrationStage] {
        [migrateV1toV2]
    }
    
    static let migrateV1toV2 = MigrationStage.custom(
        fromVersion: UserSchemaV1.self,
        toVersion: UserSchemaV2.self,
        willMigrate: nil,
        didMigrate: { context in
            // Custom migration logic
            let users = try context.fetch(FetchDescriptor<User>())
            for user in users {
                // Set default email
                user.email = "\(user.name.lowercased())@example.com"
            }
            try context.save()
        }
    )
}
```

### Apply Migration Plan

```swift
@main
struct MyApp: App {
    var container: ModelContainer
    
    init() {
        do {
            let config = ModelConfiguration()
            container = try ModelContainer(
                for: UserSchemaV2.User.self,
                migrationPlan: UserMigrationPlan.self,
                configurations: config
            )
        } catch {
            fatalError("Failed to create container: \(error)")
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

## Migration Stages

### Lightweight Migration

```swift
static let migrateV1toV2 = MigrationStage.lightweight(
    fromVersion: UserSchemaV1.self,
    toVersion: UserSchemaV2.self
)
```

**Use for:** Simple schema changes that don't need custom logic

### Custom Migration

```swift
static let migrateV1toV2 = MigrationStage.custom(
    fromVersion: UserSchemaV1.self,
    toVersion: UserSchemaV2.self,
    willMigrate: { context in
        // Before migration
        print("Starting migration...")
    },
    didMigrate: { context in
        // After migration
        let users = try context.fetch(FetchDescriptor<User>())
        for user in users {
            // Transform data
            user.email = generateEmail(for: user)
        }
        try context.save()
    }
)
```

**Use for:** Complex transformations, data cleanup, computed values

## Common Migration Scenarios

### Add Property with Default

```swift
// V1
@Model
class User {
    var name: String
}

// V2 - Automatic
@Model
class User {
    var name: String
    var role: String = "user" // Default value
}
```

### Add Optional Property

```swift
// V1
@Model
class User {
    var name: String
}

// V2 - Automatic
@Model
class User {
    var name: String
    var email: String? // Optional
}
```

### Remove Property

```swift
// V1
@Model
class User {
    var name: String
    var oldField: String
}

// V2 - Automatic
@Model
class User {
    var name: String
    // oldField removed automatically
}
```

### Rename Property

```swift
// V1
@Model
class User {
    var username: String
}

// V2 - Automatic with hint
@Model
class User {
    @Attribute(originalName: "username")
    var name: String
}
```

### Change Property Type

```swift
// V1
enum UserSchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] { [User.self] }
    
    @Model
    class User {
        var age: String // String
        init(age: String) { self.age = age }
    }
}

// V2
enum UserSchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    static var models: [any PersistentModel.Type] { [User.self] }
    
    @Model
    class User {
        var age: Int // Int
        init(age: Int) { self.age = age }
    }
}

// Migration
static let migrateV1toV2 = MigrationStage.custom(
    fromVersion: UserSchemaV1.self,
    toVersion: UserSchemaV2.self,
    didMigrate: { context in
        let users = try context.fetch(FetchDescriptor<UserSchemaV2.User>())
        for user in users {
            // Convert String to Int
            if let ageString = user.age as? String,
               let ageInt = Int(ageString) {
                user.age = ageInt
            } else {
                user.age = 0
            }
        }
        try context.save()
    }
)
```

### Add Relationship

```swift
// V1
@Model
class User {
    var name: String
}

@Model
class Post {
    var title: String
}

// V2 - Add relationship
@Model
class User {
    var name: String
    var posts: [Post]? // New relationship
}

@Model
class Post {
    var title: String
    var author: User? // New relationship
}
```

### Split Model

```swift
// V1
enum SchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] { [User.self] }
    
    @Model
    class User {
        var name: String
        var street: String
        var city: String
        var zip: String
    }
}

// V2 - Split into User + Address
enum SchemaV2: VersionedSchema {
    static var versionIdentifier = Schema.Version(2, 0, 0)
    static var models: [any PersistentModel.Type] { [User.self, Address.self] }
    
    @Model
    class User {
        var name: String
        var address: Address?
    }
    
    @Model
    class Address {
        var street: String
        var city: String
        var zip: String
    }
}

// Migration
static let migrateV1toV2 = MigrationStage.custom(
    fromVersion: SchemaV1.self,
    toVersion: SchemaV2.self,
    didMigrate: { context in
        let users = try context.fetch(FetchDescriptor<SchemaV2.User>())
        for user in users {
            // Create address from old fields
            let address = SchemaV2.Address(
                street: user.street,
                city: user.city,
                zip: user.zip
            )
            context.insert(address)
            user.address = address
        }
        try context.save()
    }
)
```

## Testing Migrations

### Test Migration

```swift
func testMigration() throws {
    // Create V1 container
    let v1Config = ModelConfiguration(isStoredInMemoryOnly: true)
    let v1Container = try ModelContainer(
        for: UserSchemaV1.User.self,
        configurations: v1Config
    )
    
    // Add V1 data
    let user = UserSchemaV1.User(name: "Alice", age: 30)
    v1Container.mainContext.insert(user)
    try v1Container.mainContext.save()
    
    // Migrate to V2
    let v2Config = ModelConfiguration(isStoredInMemoryOnly: true)
    let v2Container = try ModelContainer(
        for: UserSchemaV2.User.self,
        migrationPlan: UserMigrationPlan.self,
        configurations: v2Config
    )
    
    // Verify V2 data
    let users = try v2Container.mainContext.fetch(
        FetchDescriptor<UserSchemaV2.User>()
    )
    XCTAssertEqual(users.count, 1)
    XCTAssertEqual(users.first?.name, "Alice")
    XCTAssertNotNil(users.first?.email)
}
```

## Migration Best Practices

### 1. Always Test Migrations

```swift
// Test with real data
// Test on device, not just simulator
// Test upgrade path from each version
```

### 2. Backup Before Migration

```swift
func backupDatabase() {
    let storeURL = URL.documentsDirectory.appending(path: "default.store")
    let backupURL = URL.documentsDirectory.appending(path: "backup.store")
    
    try? FileManager.default.copyItem(at: storeURL, to: backupURL)
}
```

### 3. Version Incrementally

```swift
// ✅ GOOD: V1 → V2 → V3
enum MigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [SchemaV1.self, SchemaV2.self, SchemaV3.self]
    }
    
    static var stages: [MigrationStage] {
        [migrateV1toV2, migrateV2toV3]
    }
}

// ❌ BAD: V1 → V3 (skipping V2)
// Users on V1 can't upgrade
```

### 4. Handle Errors Gracefully

```swift
static let migrateV1toV2 = MigrationStage.custom(
    fromVersion: SchemaV1.self,
    toVersion: SchemaV2.self,
    didMigrate: { context in
        do {
            let users = try context.fetch(FetchDescriptor<User>())
            for user in users {
                // Transform data
                user.email = generateEmail(for: user)
            }
            try context.save()
        } catch {
            print("Migration failed: \(error)")
            throw error
        }
    }
)
```

### 5. Provide Progress Feedback

```swift
static let migrateV1toV2 = MigrationStage.custom(
    fromVersion: SchemaV1.self,
    toVersion: SchemaV2.self,
    didMigrate: { context in
        let users = try context.fetch(FetchDescriptor<User>())
        let total = users.count
        
        for (index, user) in users.enumerated() {
            user.email = generateEmail(for: user)
            
            if index % 100 == 0 {
                print("Migrated \(index)/\(total) users")
            }
        }
        
        try context.save()
    }
)
```

## Troubleshooting

### Migration Fails

```swift
// Check:
// 1. Schema versions defined correctly
// 2. Migration stages in correct order
// 3. All models included in schema
// 4. Custom migration logic correct
// 5. No data corruption
```

### Data Loss

```swift
// Prevent:
// 1. Always backup before migration
// 2. Test migrations thoroughly
// 3. Use lightweight migrations when possible
// 4. Handle errors in custom migrations
```

### Performance Issues

```swift
// Optimize:
// 1. Batch process large datasets
// 2. Disable autosave during migration
// 3. Use background context for heavy work
// 4. Show progress to user

static let migrateV1toV2 = MigrationStage.custom(
    fromVersion: SchemaV1.self,
    toVersion: SchemaV2.self,
    didMigrate: { context in
        context.autosaveEnabled = false
        
        let users = try context.fetch(FetchDescriptor<User>())
        for user in users {
            // Transform
        }
        
        try context.save()
        context.autosaveEnabled = true
    }
)
```

## Common Mistakes

```swift
// ❌ WRONG: Changing model without migration
@Model
class User {
    var age: String // Was Int
}
// App crashes on launch!

// ✅ CORRECT: Define migration
enum MigrationPlan: SchemaMigrationPlan {
    // Define versions and stages
}

// ❌ WRONG: Skipping versions
static var schemas: [any VersionedSchema.Type] {
    [SchemaV1.self, SchemaV3.self] // Missing V2
}

// ✅ CORRECT: Include all versions
static var schemas: [any VersionedSchema.Type] {
    [SchemaV1.self, SchemaV2.self, SchemaV3.self]
}

// ❌ WRONG: No error handling
didMigrate: { context in
    let users = try context.fetch(FetchDescriptor<User>())
    // What if this fails?
}

// ✅ CORRECT: Handle errors
didMigrate: { context in
    do {
        let users = try context.fetch(FetchDescriptor<User>())
        // Process users
        try context.save()
    } catch {
        print("Migration error: \(error)")
        throw error
    }
}
```

## Checklist

Before releasing schema changes:

- [ ] Schema versions defined
- [ ] Migration plan created
- [ ] Migration stages implemented
- [ ] Custom logic tested
- [ ] Backup strategy in place
- [ ] Error handling added
- [ ] Progress feedback implemented
- [ ] Tested on real data
- [ ] Tested on device
- [ ] Performance acceptable
- [ ] Rollback plan defined
