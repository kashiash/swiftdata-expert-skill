# Performance Optimization

## Batch Operations

### Batch Insert

```swift
func batchInsert(_ users: [UserData]) {
    modelContext.autosaveEnabled = false
    
    for userData in users {
        let user = User(name: userData.name, age: userData.age)
        modelContext.insert(user)
    }
    
    try? modelContext.save()
    modelContext.autosaveEnabled = true
}
```

**Performance:** 10-100x faster than individual inserts with autosave

### Batch Update

```swift
func batchUpdate(users: [User], newStatus: String) {
    modelContext.autosaveEnabled = false
    
    for user in users {
        user.status = newStatus
    }
    
    try? modelContext.save()
    modelContext.autosaveEnabled = true
}
```

### Batch Delete

```swift
func batchDelete(olderThan date: Date) {
    let descriptor = FetchDescriptor<User>(
        predicate: #Predicate { $0.createdAt < date }
    )
    
    guard let users = try? modelContext.fetch(descriptor) else { return }
    
    modelContext.autosaveEnabled = false
    users.forEach { modelContext.delete($0) }
    try? modelContext.save()
    modelContext.autosaveEnabled = true
}
```

## Fetch Optimization

### Fetch Limit

```swift
var descriptor = FetchDescriptor<User>(
    sortBy: [SortDescriptor(\.createdAt, order: .reverse)]
)
descriptor.fetchLimit = 20
let recentUsers = try? modelContext.fetch(descriptor)
```

**Use for:** Pagination, "load more" patterns

### Fetch Offset

```swift
func loadPage(_ page: Int, pageSize: Int = 20) {
    var descriptor = FetchDescriptor<User>(
        sortBy: [SortDescriptor(\.name)]
    )
    descriptor.fetchOffset = page * pageSize
    descriptor.fetchLimit = pageSize
    
    let users = try? modelContext.fetch(descriptor)
}
```

### Prefetching Relationships

```swift
var descriptor = FetchDescriptor<Post>()
descriptor.relationshipKeyPathsForPrefetching = [
    \.author,
    \.comments,
    \.tags
]
let posts = try? modelContext.fetch(descriptor)

// author, comments, tags loaded immediately
// No additional queries needed
```

**Use when:** You know you'll access relationships

### Partial Property Fetch

```swift
var descriptor = FetchDescriptor<User>()
descriptor.propertiesToFetch = [\.name, \.email]
let users = try? modelContext.fetch(descriptor)

// Only name and email loaded
// Other properties faulted
```

**Use for:** List views, search results

## Faulting

### Understanding Faults

```swift
// User loaded as fault (minimal data)
let user = users.first!

// Accessing property fires fault
let name = user.name // Full object loaded now

// Relationships also faulted
let posts = user.posts // Posts loaded on access
```

### Controlling Faults

```swift
// Prefetch to avoid faults
var descriptor = FetchDescriptor<User>()
descriptor.relationshipKeyPathsForPrefetching = [\.posts]

// Or use propertiesToFetch
descriptor.propertiesToFetch = [\.name, \.email, \.age]
```

## Query Optimization

### Use Predicates Efficiently

```swift
// ❌ SLOW: Load all, filter in memory
@Query var users: [User]
var activeUsers: [User] {
    users.filter { $0.isActive }
}

// ✅ FAST: Filter in database
@Query(filter: #Predicate<User> { $0.isActive })
var activeUsers: [User]
```

### Index Important Properties

```swift
// No built-in index support yet
// But Core Data indexes still work under the hood
// SwiftData uses same storage format
```

### Avoid N+1 Queries

```swift
// ❌ SLOW: N+1 queries
@Query var posts: [Post]

List(posts) { post in
    Text(post.author.name) // Query per post!
}

// ✅ FAST: Prefetch relationships
var descriptor = FetchDescriptor<Post>()
descriptor.relationshipKeyPathsForPrefetching = [\.author]
```

## Memory Management

### Release Objects

```swift
// Objects held in context until saved or rolled back
modelContext.rollback() // Releases unsaved objects

// Or create temporary context
let tempContext = ModelContext(container)
// Use tempContext
// Deallocates when out of scope
```

### Background Processing

```swift
func processLargeDataset() async {
    await Task.detached {
        let backgroundContext = ModelContext(container)
        
        // Process in background
        let descriptor = FetchDescriptor<User>()
        let users = try? backgroundContext.fetch(descriptor)
        
        users?.forEach { user in
            // Process user
            user.processedAt = .now
        }
        
        try? backgroundContext.save()
    }.value
}
```

### Chunked Processing

```swift
func processInChunks(batchSize: Int = 100) {
    var offset = 0
    var hasMore = true
    
    while hasMore {
        var descriptor = FetchDescriptor<User>()
        descriptor.fetchOffset = offset
        descriptor.fetchLimit = batchSize
        
        guard let users = try? modelContext.fetch(descriptor),
              !users.isEmpty else {
            hasMore = false
            break
        }
        
        // Process chunk
        users.forEach { user in
            // Process user
        }
        
        try? modelContext.save()
        offset += batchSize
    }
}
```

## Count Optimization

### Use fetchCount

```swift
// ❌ SLOW: Fetch all objects
let users = try? modelContext.fetch(FetchDescriptor<User>())
let count = users?.count ?? 0

// ✅ FAST: Count in database
let descriptor = FetchDescriptor<User>()
let count = try? modelContext.fetchCount(descriptor)
```

### Filtered Count

```swift
let descriptor = FetchDescriptor<User>(
    predicate: #Predicate { $0.isActive }
)
let activeCount = try? modelContext.fetchCount(descriptor)
```

## External Storage

### Large Binary Data

```swift
@Model
class Photo {
    var title: String
    
    // Stored externally, not in database
    @Attribute(.externalStorage) var imageData: Data
    
    init(title: String, imageData: Data) {
        self.title = title
        self.imageData = imageData
    }
}
```

**Use for:** Images, videos, documents > 100KB

### Benefits

- Database stays small
- Faster queries
- Better memory usage
- Automatic cleanup on delete

## Caching Strategies

### Cache FetchDescriptors

```swift
class DataManager {
    static let activeUsersDescriptor = FetchDescriptor<User>(
        predicate: #Predicate { $0.isActive },
        sortBy: [SortDescriptor(\.name)]
    )
    
    func loadActiveUsers(context: ModelContext) -> [User] {
        (try? context.fetch(Self.activeUsersDescriptor)) ?? []
    }
}
```

### Cache Query Results

```swift
class UserCache {
    private var cachedUsers: [User]?
    private var cacheTime: Date?
    private let cacheTimeout: TimeInterval = 300 // 5 minutes
    
    func getUsers(context: ModelContext) -> [User] {
        if let cached = cachedUsers,
           let time = cacheTime,
           Date().timeIntervalSince(time) < cacheTimeout {
            return cached
        }
        
        let descriptor = FetchDescriptor<User>()
        let users = (try? context.fetch(descriptor)) ?? []
        
        cachedUsers = users
        cacheTime = .now
        
        return users
    }
    
    func invalidate() {
        cachedUsers = nil
        cacheTime = nil
    }
}
```

## Monitoring Performance

### Measure Fetch Time

```swift
func measureFetch() {
    let start = Date()
    
    let descriptor = FetchDescriptor<User>()
    let users = try? modelContext.fetch(descriptor)
    
    let duration = Date().timeIntervalSince(start)
    print("Fetch took \(duration)s, loaded \(users?.count ?? 0) users")
}
```

### Profile Queries

```swift
func profileQuery() {
    let descriptor = FetchDescriptor<User>(
        predicate: #Predicate { $0.age > 18 }
    )
    
    let start = Date()
    let users = try? modelContext.fetch(descriptor)
    let fetchTime = Date().timeIntervalSince(start)
    
    print("Query: \(descriptor)")
    print("Results: \(users?.count ?? 0)")
    print("Time: \(fetchTime)s")
}
```

## Best Practices

### 1. Disable Autosave for Batch Operations

```swift
// ❌ SLOW
for i in 1...1000 {
    modelContext.insert(User(name: "User \(i)"))
    // Saves 1000 times!
}

// ✅ FAST
modelContext.autosaveEnabled = false
for i in 1...1000 {
    modelContext.insert(User(name: "User \(i)"))
}
try? modelContext.save() // Save once
modelContext.autosaveEnabled = true
```

### 2. Use Predicates, Not In-Memory Filtering

```swift
// ❌ SLOW
@Query var users: [User]
var adults: [User] {
    users.filter { $0.age >= 18 }
}

// ✅ FAST
@Query(filter: #Predicate<User> { $0.age >= 18 })
var adults: [User]
```

### 3. Prefetch Relationships

```swift
// ❌ SLOW: N+1 queries
@Query var posts: [Post]
// Each post.author triggers query

// ✅ FAST: Prefetch
var descriptor = FetchDescriptor<Post>()
descriptor.relationshipKeyPathsForPrefetching = [\.author]
```

### 4. Use Fetch Limits

```swift
// ❌ SLOW: Load everything
@Query var users: [User]

// ✅ FAST: Load what you need
var descriptor = FetchDescriptor<User>()
descriptor.fetchLimit = 50
```

### 5. Use External Storage for Large Data

```swift
// ❌ SLOW: Large data in database
@Model
class Photo {
    var imageData: Data // Slows queries
}

// ✅ FAST: External storage
@Model
class Photo {
    @Attribute(.externalStorage) var imageData: Data
}
```

### 6. Background Processing

```swift
// ❌ SLOW: Block main thread
func importData() {
    // Heavy processing on main thread
}

// ✅ FAST: Background thread
func importData() async {
    await Task.detached {
        let context = ModelContext(container)
        // Process in background
    }.value
}
```

### 7. Use fetchCount, Not fetch + count

```swift
// ❌ SLOW
let users = try? modelContext.fetch(FetchDescriptor<User>())
let count = users?.count ?? 0

// ✅ FAST
let count = try? modelContext.fetchCount(FetchDescriptor<User>())
```

## Performance Checklist

- [ ] Autosave disabled for batch operations
- [ ] Predicates used instead of in-memory filtering
- [ ] Relationships prefetched when needed
- [ ] Fetch limits applied for large datasets
- [ ] External storage for large binary data
- [ ] Background contexts for heavy processing
- [ ] fetchCount used instead of fetch + count
- [ ] Partial property fetch for list views
- [ ] Chunked processing for large datasets
- [ ] FetchDescriptors cached and reused

## Common Performance Issues

### Issue: Slow List Scrolling

```swift
// Problem: Loading too much data
@Query var users: [User]

// Solution: Limit and paginate
var descriptor = FetchDescriptor<User>()
descriptor.fetchLimit = 50
```

### Issue: High Memory Usage

```swift
// Problem: All objects in memory
let users = try? modelContext.fetch(FetchDescriptor<User>())

// Solution: Process in chunks
func processInChunks() {
    var offset = 0
    let batchSize = 100
    
    while true {
        var descriptor = FetchDescriptor<User>()
        descriptor.fetchOffset = offset
        descriptor.fetchLimit = batchSize
        
        guard let batch = try? modelContext.fetch(descriptor),
              !batch.isEmpty else { break }
        
        // Process batch
        offset += batchSize
    }
}
```

### Issue: Slow Saves

```swift
// Problem: Autosave on every change
for i in 1...1000 {
    modelContext.insert(User(name: "User \(i)"))
}

// Solution: Batch save
modelContext.autosaveEnabled = false
for i in 1...1000 {
    modelContext.insert(User(name: "User \(i)"))
}
try? modelContext.save()
modelContext.autosaveEnabled = true
```

### Issue: N+1 Queries

```swift
// Problem: Query per relationship
@Query var posts: [Post]
List(posts) { post in
    Text(post.author.name) // Query!
}

// Solution: Prefetch
var descriptor = FetchDescriptor<Post>()
descriptor.relationshipKeyPathsForPrefetching = [\.author]
```


## Concurrency

### Thread Safety Rules

**Critical:**
- ModelContext is NOT thread-safe
- Cannot share ModelContext across threads
- Each Task needs its own ModelContext
- ModelContainer CAN be shared

### Background Context

```swift
let container = try ModelContainer(for: User.self)

Task.detached {
    let backgroundContext = ModelContext(container)
    
    // Perform operations
    for i in 1...1000 {
        backgroundContext.insert(User(name: "User \(i)"))
    }
    
    try? backgroundContext.save()
}
```

### Background Insert

```swift
func importUsers(_ data: [UserData]) async {
    await Task.detached {
        let context = ModelContext(container)
        context.autosaveEnabled = false
        
        for userData in data {
            let user = User(name: userData.name, age: userData.age)
            context.insert(user)
        }
        
        try? context.save()
    }.value
}
```

### Background Fetch

```swift
func fetchLargeDataset() async -> [User] {
    await Task.detached {
        let context = ModelContext(container)
        let descriptor = FetchDescriptor<User>(
            sortBy: [SortDescriptor(\.name)]
        )
        descriptor.fetchLimit = 100000
        
        return (try? context.fetch(descriptor)) ?? []
    }.value
}

// Use in view
.task {
    let users = await fetchLargeDataset()
    // Update UI on main actor
    await MainActor.run {
        self.users = users
    }
}
```

### Background Update

```swift
func updateAllUsers() async {
    await Task.detached {
        let context = ModelContext(container)
        let descriptor = FetchDescriptor<User>()
        let users = try? context.fetch(descriptor)
        
        context.autosaveEnabled = false
        users?.forEach { user in
            user.lastUpdated = .now
        }
        try? context.save()
    }.value
}
```

### Background Delete

```swift
func deleteOldRecords() async {
    await Task.detached {
        let context = ModelContext(container)
        let cutoff = Date.now.addingTimeInterval(-86400 * 30)
        let descriptor = FetchDescriptor<User>(
            predicate: #Predicate { $0.createdAt < cutoff }
        )
        
        let users = try? context.fetch(descriptor)
        context.autosaveEnabled = false
        users?.forEach { context.delete($0) }
        try? context.save()
    }.value
}
```

### Main Actor Operations

```swift
@MainActor
func updateUI(with users: [User]) {
    // Safe to update UI
    self.users = users
}

func fetchAndDisplay() async {
    let context = ModelContext(container)
    let descriptor = FetchDescriptor<User>()
    let users = try? context.fetch(descriptor)
    
    await MainActor.run {
        self.users = users ?? []
    }
}
```

### Actor-Isolated Context

```swift
actor DataProcessor {
    private let context: ModelContext
    
    init(container: ModelContainer) {
        self.context = ModelContext(container)
    }
    
    func processData(_ data: [UserData]) async throws {
        context.autosaveEnabled = false
        
        for userData in data {
            let user = User(name: userData.name)
            context.insert(user)
        }
        
        try context.save()
    }
}

// Usage
let processor = DataProcessor(container: container)
try await processor.processData(userData)
```

### Concurrent Operations

```swift
func importMultipleSources() async {
    await withTaskGroup(of: Void.self) { group in
        group.addTask {
            let context = ModelContext(container)
            // Import source 1
        }
        
        group.addTask {
            let context = ModelContext(container)
            // Import source 2
        }
        
        group.addTask {
            let context = ModelContext(container)
            // Import source 3
        }
    }
}
```

### Progress Tracking

```swift
@Observable
class ImportManager {
    var progress: Double = 0
    var isImporting = false
    
    func importData(_ data: [UserData]) async {
        isImporting = true
        let total = Double(data.count)
        
        await Task.detached {
            let context = ModelContext(self.container)
            context.autosaveEnabled = false
            
            for (index, userData) in data.enumerated() {
                let user = User(name: userData.name)
                context.insert(user)
                
                await MainActor.run {
                    self.progress = Double(index + 1) / total
                }
            }
            
            try? context.save()
        }.value
        
        isImporting = false
    }
}
```

### Chunked Processing

```swift
func processInChunks(data: [UserData], chunkSize: Int = 100) async {
    for chunk in data.chunked(into: chunkSize) {
        await Task.detached {
            let context = ModelContext(container)
            context.autosaveEnabled = false
            
            for userData in chunk {
                let user = User(name: userData.name)
                context.insert(user)
            }
            
            try? context.save()
        }.value
    }
}

extension Array {
    func chunked(into size: Int) -> [[Element]] {
        stride(from: 0, to: count, by: size).map {
            Array(self[$0..<Swift.min($0 + size, count)])
        }
    }
}
```

## Concurrency Best Practices

1. **Always create new context for background work**
   ```swift
   // ✅ CORRECT
   Task.detached {
       let context = ModelContext(container)
   }
   
   // ❌ WRONG
   Task.detached {
       modelContext.insert(user) // Crash!
   }
   ```

2. **Pass container, not context**
   ```swift
   // ✅ CORRECT
   func process(container: ModelContainer) {
       let context = ModelContext(container)
   }
   
   // ❌ WRONG
   func process(context: ModelContext) {
       // Context not thread-safe
   }
   ```

3. **Update UI on main actor**
   ```swift
   // ✅ CORRECT
   await MainActor.run {
       self.users = fetchedUsers
   }
   
   // ❌ WRONG
   self.users = fetchedUsers // May crash
   ```

4. **Disable autosave for batch operations**
   ```swift
   // ✅ CORRECT
   context.autosaveEnabled = false
   // ... operations
   try context.save()
   
   // ❌ WRONG
   // ... operations (saves after each)
   ```

5. **Use Task.detached for heavy work**
   ```swift
   // ✅ CORRECT - Doesn't block main thread
   Task.detached {
       let context = ModelContext(container)
       // Heavy work
   }
   
   // ❌ WRONG - Blocks main thread
   Task {
       // Heavy work on main actor
   }
   ```

## Common Concurrency Mistakes

```swift
// ❌ WRONG: Sharing context
let sharedContext = ModelContext(container)
Task { sharedContext.insert(user1) }
Task { sharedContext.insert(user2) } // Crash!

// ✅ CORRECT: Separate contexts
Task {
    let context = ModelContext(container)
    context.insert(user1)
}
Task {
    let context = ModelContext(container)
    context.insert(user2)
}

// ❌ WRONG: UI update in background
Task.detached {
    self.users = fetchedUsers // Crash!
}

// ✅ CORRECT: UI update on main actor
Task.detached {
    let users = fetchedUsers
    await MainActor.run {
        self.users = users
    }
}

// ❌ WRONG: Passing context to actor
actor Processor {
    func process(context: ModelContext) { } // Not thread-safe
}

// ✅ CORRECT: Passing container
actor Processor {
    func process(container: ModelContainer) {
        let context = ModelContext(container)
    }
}
```

## Performance Debugging

### Enable SwiftData Logging

Add to scheme environment variables:
```
SWIFTDATA_DEBUG = 1  // Basic logging
SWIFTDATA_DEBUG = 2  // SQL queries
OS_ACTIVITY_MODE = disable  // Reduce noise
```

### Manual Timing

```swift
let start = CFAbsoluteTimeGetCurrent()
// Operation
let elapsed = CFAbsoluteTimeGetCurrent() - start
print("Time: \(elapsed)s")
```

### Time Profiler

1. Product → Profile → Time Profiler
2. Look for `modelContext.save()` spikes
3. Look for `modelContext.fetch()` bottlenecks
4. Check main thread blocking
