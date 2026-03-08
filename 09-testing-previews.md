# Testing and Previews

## SwiftUI Previews

### Basic Preview with In-Memory Container

```swift
#Preview {
    let config = ModelConfiguration(isStoredInMemoryOnly: true)
    let container = try! ModelContainer(for: User.self, configurations: config)
    
    let user = User(name: "Alice", age: 30)
    container.mainContext.insert(user)
    
    return EditUserView(user: user)
        .modelContainer(container)
}
```

### Preview with Multiple Objects

```swift
#Preview {
    let config = ModelConfiguration(isStoredInMemoryOnly: true)
    let container = try! ModelContainer(for: User.self, configurations: config)
    
    for i in 1...10 {
        let user = User(name: "User \(i)", age: 20 + i)
        container.mainContext.insert(user)
    }
    
    return ContentView()
        .modelContainer(container)
}
```

### Preview with Relationships

```swift
#Preview {
    let config = ModelConfiguration(isStoredInMemoryOnly: true)
    let container = try! ModelContainer(
        for: User.self, Post.self,
        configurations: config
    )
    
    let user = User(name: "Alice")
    let post1 = Post(title: "First Post", author: user)
    let post2 = Post(title: "Second Post", author: user)
    
    container.mainContext.insert(user)
    container.mainContext.insert(post1)
    container.mainContext.insert(post2)
    
    return UserDetailView(user: user)
        .modelContainer(container)
}
```

### Reusable Preview Container

```swift
@MainActor
class PreviewContainer {
    static let shared: ModelContainer = {
        do {
            let config = ModelConfiguration(isStoredInMemoryOnly: true)
            let container = try ModelContainer(
                for: User.self, Post.self,
                configurations: config
            )
            
            // Add sample data
            let user1 = User(name: "Alice", age: 30)
            let user2 = User(name: "Bob", age: 25)
            
            container.mainContext.insert(user1)
            container.mainContext.insert(user2)
            
            return container
        } catch {
            fatalError("Failed to create preview container: \(error)")
        }
    }()
}

#Preview {
    ContentView()
        .modelContainer(PreviewContainer.shared)
}
```

## Unit Testing

### Test Setup

```swift
import XCTest
import SwiftData
@testable import MyApp

class UserTests: XCTestCase {
    var container: ModelContainer!
    var context: ModelContext!
    
    override func setUp() async throws {
        let config = ModelConfiguration(isStoredInMemoryOnly: true)
        container = try ModelContainer(for: User.self, configurations: config)
        context = ModelContext(container)
    }
    
    override func tearDown() async throws {
        container = nil
        context = nil
    }
}
```

### Test CRUD Operations

```swift
func testCreateUser() throws {
    // Create
    let user = User(name: "Alice", age: 30)
    context.insert(user)
    try context.save()
    
    // Verify
    let descriptor = FetchDescriptor<User>()
    let users = try context.fetch(descriptor)
    
    XCTAssertEqual(users.count, 1)
    XCTAssertEqual(users.first?.name, "Alice")
    XCTAssertEqual(users.first?.age, 30)
}

func testUpdateUser() throws {
    // Create
    let user = User(name: "Alice", age: 30)
    context.insert(user)
    try context.save()
    
    // Update
    user.name = "Alice Smith"
    user.age = 31
    try context.save()
    
    // Verify
    let descriptor = FetchDescriptor<User>()
    let users = try context.fetch(descriptor)
    
    XCTAssertEqual(users.first?.name, "Alice Smith")
    XCTAssertEqual(users.first?.age, 31)
}

func testDeleteUser() throws {
    // Create
    let user = User(name: "Alice", age: 30)
    context.insert(user)
    try context.save()
    
    // Delete
    context.delete(user)
    try context.save()
    
    // Verify
    let descriptor = FetchDescriptor<User>()
    let users = try context.fetch(descriptor)
    
    XCTAssertEqual(users.count, 0)
}
```

### Test Queries

```swift
func testQueryWithPredicate() throws {
    // Create test data
    let alice = User(name: "Alice", age: 30)
    let bob = User(name: "Bob", age: 25)
    let charlie = User(name: "Charlie", age: 35)
    
    context.insert(alice)
    context.insert(bob)
    context.insert(charlie)
    try context.save()
    
    // Query
    let descriptor = FetchDescriptor<User>(
        predicate: #Predicate { $0.age > 28 }
    )
    let users = try context.fetch(descriptor)
    
    // Verify
    XCTAssertEqual(users.count, 2)
    XCTAssertTrue(users.contains(where: { $0.name == "Alice" }))
    XCTAssertTrue(users.contains(where: { $0.name == "Charlie" }))
}

func testQueryWithSorting() throws {
    // Create test data
    let alice = User(name: "Alice", age: 30)
    let bob = User(name: "Bob", age: 25)
    
    context.insert(alice)
    context.insert(bob)
    try context.save()
    
    // Query
    let descriptor = FetchDescriptor<User>(
        sortBy: [SortDescriptor(\.name)]
    )
    let users = try context.fetch(descriptor)
    
    // Verify
    XCTAssertEqual(users.first?.name, "Alice")
    XCTAssertEqual(users.last?.name, "Bob")
}
```

### Test Relationships

```swift
func testOneToManyRelationship() throws {
    // Create
    let user = User(name: "Alice")
    let post1 = Post(title: "First", author: user)
    let post2 = Post(title: "Second", author: user)
    
    context.insert(user)
    context.insert(post1)
    context.insert(post2)
    try context.save()
    
    // Verify
    XCTAssertEqual(user.posts.count, 2)
    XCTAssertEqual(post1.author?.name, "Alice")
    XCTAssertEqual(post2.author?.name, "Alice")
}

func testCascadeDelete() throws {
    // Create
    let user = User(name: "Alice")
    let post = Post(title: "Test", author: user)
    
    context.insert(user)
    context.insert(post)
    try context.save()
    
    // Delete user
    context.delete(user)
    try context.save()
    
    // Verify post deleted (if cascade delete rule)
    let descriptor = FetchDescriptor<Post>()
    let posts = try context.fetch(descriptor)
    
    XCTAssertEqual(posts.count, 0)
}
```

### Test Validation

```swift
func testValidation() throws {
    // Test invalid data
    XCTAssertThrowsError(try User(email: "invalid")) { error in
        XCTAssertTrue(error is ValidationError)
    }
    
    // Test valid data
    XCTAssertNoThrow(try User(email: "valid@example.com"))
}
```

## Integration Testing

### Test with Real Container

```swift
class IntegrationTests: XCTestCase {
    var container: ModelContainer!
    var context: ModelContext!
    
    override func setUp() async throws {
        // Use real file-based container
        let storeURL = FileManager.default.temporaryDirectory
            .appending(path: "test.store")
        
        let config = ModelConfiguration(url: storeURL)
        container = try ModelContainer(for: User.self, configurations: config)
        context = ModelContext(container)
    }
    
    override func tearDown() async throws {
        // Clean up
        let storeURL = FileManager.default.temporaryDirectory
            .appending(path: "test.store")
        try? FileManager.default.removeItem(at: storeURL)
        
        container = nil
        context = nil
    }
    
    func testPersistence() throws {
        // Create and save
        let user = User(name: "Alice", age: 30)
        context.insert(user)
        try context.save()
        
        // Create new context
        let newContext = ModelContext(container)
        
        // Verify data persisted
        let descriptor = FetchDescriptor<User>()
        let users = try newContext.fetch(descriptor)
        
        XCTAssertEqual(users.count, 1)
        XCTAssertEqual(users.first?.name, "Alice")
    }
}
```

## Performance Testing

### Measure Fetch Performance

```swift
func testFetchPerformance() throws {
    // Create test data
    for i in 1...1000 {
        let user = User(name: "User \(i)", age: 20 + (i % 50))
        context.insert(user)
    }
    try context.save()
    
    // Measure
    measure {
        let descriptor = FetchDescriptor<User>()
        _ = try? context.fetch(descriptor)
    }
}
```

### Measure Insert Performance

```swift
func testInsertPerformance() throws {
    measure {
        context.autosaveEnabled = false
        
        for i in 1...1000 {
            let user = User(name: "User \(i)", age: 20)
            context.insert(user)
        }
        
        try? context.save()
        context.autosaveEnabled = true
    }
}
```

## Mock Data

### Sample Data Generator

```swift
class SampleData {
    static func createUsers(count: Int, in context: ModelContext) {
        for i in 1...count {
            let user = User(
                name: "User \(i)",
                age: Int.random(in: 18...80),
                email: "user\(i)@example.com"
            )
            context.insert(user)
        }
        try? context.save()
    }
    
    static func createUsersWithPosts(in context: ModelContext) {
        for i in 1...5 {
            let user = User(name: "User \(i)")
            context.insert(user)
            
            for j in 1...3 {
                let post = Post(
                    title: "Post \(j) by User \(i)",
                    author: user
                )
                context.insert(post)
            }
        }
        try? context.save()
    }
}

// Usage in tests
func testWithSampleData() throws {
    SampleData.createUsers(count: 10, in: context)
    
    let descriptor = FetchDescriptor<User>()
    let users = try context.fetch(descriptor)
    
    XCTAssertEqual(users.count, 10)
}
```

## Testing Best Practices

### 1. Use In-Memory Storage

```swift
// ✅ FAST: In-memory
let config = ModelConfiguration(isStoredInMemoryOnly: true)

// ❌ SLOW: File-based
let config = ModelConfiguration()
```

### 2. Clean Up After Tests

```swift
override func tearDown() async throws {
    // Clear all data
    let descriptor = FetchDescriptor<User>()
    let users = try? context.fetch(descriptor)
    users?.forEach { context.delete($0) }
    try? context.save()
    
    container = nil
    context = nil
}
```

### 3. Test Edge Cases

```swift
func testEmptyQuery() throws {
    let descriptor = FetchDescriptor<User>()
    let users = try context.fetch(descriptor)
    XCTAssertEqual(users.count, 0)
}

func testQueryWithNoMatches() throws {
    let user = User(name: "Alice", age: 30)
    context.insert(user)
    try context.save()
    
    let descriptor = FetchDescriptor<User>(
        predicate: #Predicate { $0.age > 100 }
    )
    let users = try context.fetch(descriptor)
    XCTAssertEqual(users.count, 0)
}
```

### 4. Test Concurrent Access

```swift
func testConcurrentAccess() async throws {
    await withTaskGroup(of: Void.self) { group in
        for i in 1...10 {
            group.addTask {
                let context = ModelContext(self.container)
                let user = User(name: "User \(i)", age: 20)
                context.insert(user)
                try? context.save()
            }
        }
    }
    
    let descriptor = FetchDescriptor<User>()
    let users = try context.fetch(descriptor)
    XCTAssertEqual(users.count, 10)
}
```

## UI Testing

### Test SwiftUI Views

```swift
import XCTest

class UITests: XCTestCase {
    var app: XCUIApplication!
    
    override func setUp() {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launchArguments = ["testing"]
        app.launch()
    }
    
    func testAddUser() {
        // Tap add button
        app.buttons["Add User"].tap()
        
        // Enter name
        let nameField = app.textFields["Name"]
        nameField.tap()
        nameField.typeText("Alice")
        
        // Save
        app.buttons["Save"].tap()
        
        // Verify
        XCTAssertTrue(app.staticTexts["Alice"].exists)
    }
}
```

### Test with Mock Data

```swift
// In App
@main
struct MyApp: App {
    var container: ModelContainer
    
    init() {
        let isTesting = CommandLine.arguments.contains("testing")
        
        let config = ModelConfiguration(
            isStoredInMemoryOnly: isTesting
        )
        container = try! ModelContainer(
            for: User.self,
            configurations: config
        )
        
        if isTesting {
            SampleData.createUsers(count: 10, in: container.mainContext)
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

## Debugging Tests

### Print Context State

```swift
func printContextState() {
    print("Has changes: \(context.hasChanges)")
    print("Inserted: \(context.insertedModelsArray.count)")
    print("Updated: \(context.changedModelsArray.count)")
    print("Deleted: \(context.deletedModelsArray.count)")
}
```

### Verify Save

```swift
func verifySave() throws {
    XCTAssertTrue(context.hasChanges, "Context should have changes")
    
    try context.save()
    
    XCTAssertFalse(context.hasChanges, "Context should be clean after save")
}
```

## Common Test Patterns

### Setup and Teardown

```swift
class UserTests: XCTestCase {
    var container: ModelContainer!
    var context: ModelContext!
    
    override func setUp() async throws {
        let config = ModelConfiguration(isStoredInMemoryOnly: true)
        container = try ModelContainer(for: User.self, configurations: config)
        context = ModelContext(container)
    }
    
    override func tearDown() async throws {
        container = nil
        context = nil
    }
}
```

### Helper Methods

```swift
extension UserTests {
    func createUser(name: String, age: Int) -> User {
        let user = User(name: name, age: age)
        context.insert(user)
        return user
    }
    
    func fetchAllUsers() throws -> [User] {
        try context.fetch(FetchDescriptor<User>())
    }
    
    func clearAllUsers() throws {
        let users = try fetchAllUsers()
        users.forEach { context.delete($0) }
        try context.save()
    }
}
```

### Async Testing

```swift
func testAsyncOperation() async throws {
    let user = User(name: "Alice", age: 30)
    context.insert(user)
    try context.save()
    
    // Simulate async operation
    try await Task.sleep(for: .seconds(1))
    
    let users = try context.fetch(FetchDescriptor<User>())
    XCTAssertEqual(users.count, 1)
}
```

## Checklist

Before releasing:

- [ ] Unit tests for all models
- [ ] Tests for CRUD operations
- [ ] Tests for queries and predicates
- [ ] Tests for relationships
- [ ] Tests for validation
- [ ] Integration tests
- [ ] Performance tests
- [ ] UI tests
- [ ] Edge case tests
- [ ] Concurrent access tests
- [ ] Preview data working
- [ ] Mock data generators
- [ ] Test cleanup working
- [ ] All tests passing
