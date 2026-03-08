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
