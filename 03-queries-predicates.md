# Queries and Predicates

## Basic @Query

```swift
import SwiftUI
import SwiftData

struct ContentView: View {
    @Query var users: [User]
    
    var body: some View {
        List(users) { user in
            Text(user.name)
        }
    }
}
```

**Rules:**
- Only works in SwiftUI views
- Automatically updates when data changes
- Runs on view appear

## Sorting

### Single Property

```swift
// Ascending
@Query(sort: \User.name) var users: [User]

// Descending
@Query(sort: \User.age, order: .reverse) var users: [User]
```

### Multiple Properties

```swift
@Query(sort: [
    SortDescriptor(\User.age, order: .reverse),
    SortDescriptor(\User.name)
]) var users: [User]
```

### Localized Sorting

```swift
@Query(sort: [
    SortDescriptor(\User.name, comparator: .localizedStandard)
]) var users: [User]
```

## Filtering with #Predicate

### Basic Comparisons

```swift
// Equal
@Query(filter: #Predicate<User> { $0.age == 30 })
var users: [User]

// Not equal
@Query(filter: #Predicate<User> { $0.age != 30 })
var users: [User]

// Greater than
@Query(filter: #Predicate<User> { $0.age > 18 })
var users: [User]

// Less than or equal
@Query(filter: #Predicate<User> { $0.age <= 65 })
var users: [User]
```

### String Operations

```swift
// Contains (case-insensitive)
@Query(filter: #Predicate<User> {
    $0.name.localizedStandardContains("john")
}) var users: [User]

// Starts with
@Query(filter: #Predicate<User> {
    $0.email.starts(with: "admin")
}) var users: [User]

// Ends with
@Query(filter: #Predicate<User> {
    $0.email.hasSuffix("@example.com")
}) var users: [User]
```

### Logical Operators

```swift
// AND
@Query(filter: #Predicate<User> {
    $0.age > 18 && $0.age < 65
}) var users: [User]

// OR
@Query(filter: #Predicate<User> {
    $0.role == "admin" || $0.role == "moderator"
}) var users: [User]

// NOT
@Query(filter: #Predicate<User> {
    !$0.isDeleted
}) var users: [User]
```

### Date Comparisons

```swift
// After date
@Query(filter: #Predicate<User> {
    let cutoff = Date.now.addingTimeInterval(-86400 * 30)
    return $0.createdAt > cutoff
}) var recentUsers: [User]

// Before date
@Query(filter: #Predicate<User> {
    let now = Date.now
    return $0.expiresAt < now
}) var expiredUsers: [User]
```

### Optional Properties

```swift
// Is nil
@Query(filter: #Predicate<User> {
    $0.deletedAt == nil
}) var activeUsers: [User]

// Is not nil
@Query(filter: #Predicate<User> {
    $0.profileImage != nil
}) var usersWithImages: [User]
```

### Array/Collection Checks

```swift
// Array contains
@Query(filter: #Predicate<User> {
    $0.tags.contains("premium")
}) var premiumUsers: [User]

// Array is empty
@Query(filter: #Predicate<User> {
    $0.posts.isEmpty
}) var usersWithoutPosts: [User]

// Array count
@Query(filter: #Predicate<User> {
    $0.posts.count > 10
}) var activePosters: [User]
```

### Relationship Queries

```swift
// Has relationship
@Query(filter: #Predicate<Post> {
    $0.author != nil
}) var postsWithAuthors: [Post]

// Relationship property
@Query(filter: #Predicate<Post> {
    $0.author?.name == "Alice"
}) var alicesPosts: [Post]

// Relationship count
@Query(filter: #Predicate<User> {
    $0.posts.count >= 5
}) var prolificAuthors: [User]
```

## Dynamic Queries

### Dynamic Sorting

```swift
struct UserListView: View {
    @Query var users: [User]
    
    init(sort: SortDescriptor<User>) {
        _users = Query(sort: [sort])
    }
    
    var body: some View {
        List(users) { user in
            Text(user.name)
        }
    }
}

struct ContentView: View {
    @State private var sortOrder = SortDescriptor(\User.name)
    
    var body: some View {
        VStack {
            Picker("Sort", selection: $sortOrder) {
                Text("Name").tag(SortDescriptor(\User.name))
                Text("Age").tag(SortDescriptor(\User.age))
            }
            
            UserListView(sort: sortOrder)
        }
    }
}
```

### Dynamic Filtering

```swift
struct UserListView: View {
    @Query var users: [User]
    
    init(searchText: String) {
        _users = Query(filter: #Predicate {
            if searchText.isEmpty {
                return true
            } else {
                return $0.name.localizedStandardContains(searchText)
            }
        })
    }
    
    var body: some View {
        List(users) { user in
            Text(user.name)
        }
    }
}

struct ContentView: View {
    @State private var searchText = ""
    
    var body: some View {
        NavigationStack {
            UserListView(searchText: searchText)
                .searchable(text: $searchText)
        }
    }
}
```

### Combined Dynamic Query

```swift
struct UserListView: View {
    @Query var users: [User]
    
    init(sort: SortDescriptor<User>, searchText: String, minAge: Int) {
        _users = Query(filter: #Predicate {
            let matchesSearch = searchText.isEmpty ||
                $0.name.localizedStandardContains(searchText)
            let matchesAge = $0.age >= minAge
            return matchesSearch && matchesAge
        }, sort: [sort])
    }
    
    var body: some View {
        List(users) { user in
            Text("\(user.name), \(user.age)")
        }
    }
}
```

## FetchDescriptor

### Basic Usage

```swift
@Environment(\.modelContext) var modelContext

func loadUsers() {
    let descriptor = FetchDescriptor<User>()
    let users = try? modelContext.fetch(descriptor)
}
```

### With Sorting

```swift
let descriptor = FetchDescriptor<User>(
    sortBy: [SortDescriptor(\.name)]
)
```

### With Predicate

```swift
let descriptor = FetchDescriptor<User>(
    predicate: #Predicate { $0.age > 18 },
    sortBy: [SortDescriptor(\.name)]
)
```

### Fetch Limit

```swift
var descriptor = FetchDescriptor<User>(
    sortBy: [SortDescriptor(\.createdAt, order: .reverse)]
)
descriptor.fetchLimit = 10
let recentUsers = try? modelContext.fetch(descriptor)
```

### Fetch Offset

```swift
var descriptor = FetchDescriptor<User>(
    sortBy: [SortDescriptor(\.name)]
)
descriptor.fetchOffset = 20
descriptor.fetchLimit = 10
// Pagination: skip 20, take 10
```

### Prefetching Relationships

```swift
var descriptor = FetchDescriptor<Post>()
descriptor.relationshipKeyPathsForPrefetching = [\.author, \.comments]
let posts = try? modelContext.fetch(descriptor)
// author and comments loaded immediately
```

### Faulting

```swift
var descriptor = FetchDescriptor<User>()
descriptor.propertiesToFetch = [\.name, \.email]
// Only load specific properties
```

## Aggregate Queries

### Count

```swift
let descriptor = FetchDescriptor<User>(
    predicate: #Predicate { $0.age > 18 }
)
let count = try? modelContext.fetchCount(descriptor)
```

### Exists

```swift
let descriptor = FetchDescriptor<User>(
    predicate: #Predicate { $0.email == "admin@example.com" }
)
let exists = (try? modelContext.fetchCount(descriptor)) ?? 0 > 0
```

## Complex Predicates

### Multiple Conditions

```swift
@Query(filter: #Predicate<User> {
    $0.age >= 18 &&
    $0.age <= 65 &&
    $0.isActive &&
    !$0.isDeleted &&
    $0.email != nil
}) var eligibleUsers: [User]
```

### Nested Conditions

```swift
@Query(filter: #Predicate<User> {
    ($0.role == "admin" || $0.role == "moderator") &&
    $0.isActive &&
    $0.lastLogin > Date.now.addingTimeInterval(-86400 * 7)
}) var activeStaff: [User]
```

### Using External Values

```swift
struct UserListView: View {
    let minAge: Int
    let maxAge: Int
    
    @Query var users: [User]
    
    init(minAge: Int, maxAge: Int) {
        self.minAge = minAge
        self.maxAge = maxAge
        
        _users = Query(filter: #Predicate {
            $0.age >= minAge && $0.age <= maxAge
        })
    }
    
    var body: some View {
        List(users) { user in
            Text(user.name)
        }
    }
}
```

## Animation

```swift
// Animate query changes
@Query(sort: \User.name, animation: .default) var users: [User]

// Custom animation
@Query(sort: \User.name, animation: .spring()) var users: [User]

// No animation
@Query(sort: \User.name, animation: nil) var users: [User]
```

## Common Patterns

### Search with Debounce

```swift
struct ContentView: View {
    @State private var searchText = ""
    @State private var debouncedSearch = ""
    
    var body: some View {
        UserListView(searchText: debouncedSearch)
            .searchable(text: $searchText)
            .onChange(of: searchText) { oldValue, newValue in
                Task {
                    try? await Task.sleep(for: .milliseconds(300))
                    if searchText == newValue {
                        debouncedSearch = newValue
                    }
                }
            }
    }
}
```

### Filtered by Enum

```swift
enum UserRole: String, Codable {
    case admin, user, guest
}

@Query(filter: #Predicate<User> {
    $0.role == UserRole.admin
}) var admins: [User]
```

### Date Range

```swift
struct EventListView: View {
    let startDate: Date
    let endDate: Date
    
    @Query var events: [Event]
    
    init(startDate: Date, endDate: Date) {
        self.startDate = startDate
        self.endDate = endDate
        
        _users = Query(filter: #Predicate {
            $0.date >= startDate && $0.date <= endDate
        })
    }
}
```

## Common Mistakes

```swift
// ❌ WRONG: Using @Query outside SwiftUI view
class ViewModel {
    @Query var users: [User] // Doesn't work!
}

// ✅ CORRECT: Use FetchDescriptor
class ViewModel {
    let modelContext: ModelContext
    
    func loadUsers() {
        let descriptor = FetchDescriptor<User>()
        let users = try? modelContext.fetch(descriptor)
    }
}

// ❌ WRONG: Modifying query property
@Query var users: [User]
users = newUsers // Cannot assign

// ✅ CORRECT: Modify data, query updates automatically
modelContext.insert(newUser)

// ❌ WRONG: Case-sensitive search
@Query(filter: #Predicate<User> {
    $0.name.contains("john") // Misses "John"
})

// ✅ CORRECT: Case-insensitive search
@Query(filter: #Predicate<User> {
    $0.name.localizedStandardContains("john")
})
```

## Performance Tips

1. **Use fetch limits** for large datasets
2. **Prefetch relationships** when needed
3. **Use propertiesToFetch** to load only required fields
4. **Avoid complex predicates** in tight loops
5. **Cache FetchDescriptors** for reuse
6. **Use fetchCount** instead of fetch + count
7. **Debounce search** to reduce queries
