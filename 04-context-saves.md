# Context and Saves

## ModelContainer Setup

### Basic Setup

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: User.self)
    }
}
```

### Multiple Models

```swift
.modelContainer(for: [User.self, Post.self, Comment.self])
```

### Custom Configuration

```swift
@main
struct MyApp: App {
    var container: ModelContainer
    
    init() {
        do {
            let config = ModelConfiguration(
                isStoredInMemoryOnly: false,
                allowsSave: true
            )
            container = try ModelContainer(
                for: User.self,
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

### Custom Storage Location

```swift
let storeURL = URL.documentsDirectory.appending(path: "myapp.sqlite")
let config = ModelConfiguration(url: storeURL)
container = try ModelContainer(for: User.self, configurations: config)
```

### In-Memory Storage

```swift
let config = ModelConfiguration(isStoredInMemoryOnly: true)
container = try ModelContainer(for: User.self, configurations: config)
```

**Use for:** Testing, previews, temporary data

## ModelContext

### Accessing Context

```swift
struct ContentView: View {
    @Environment(\.modelContext) var modelContext
    
    var body: some View {
        Button("Add User") {
            let user = User(name: "Alice")
            modelContext.insert(user)
        }
    }
}
```

### Creating Custom Context

```swift
let container = try ModelContainer(for: User.self)
let context = ModelContext(container)
```

### Background Context

```swift
let container = try ModelContainer(for: User.self)

Task.detached {
    let backgroundContext = ModelContext(container)
    
    // Perform work on background
    let user = User(name: "Alice")
    backgroundContext.insert(user)
    try? backgroundContext.save()
}
```

## CRUD Operations

### Create

```swift
let user = User(name: "Alice", age: 30)
modelContext.insert(user)
// Auto-saved (if autosave enabled)
```

### Read

```swift
// Via @Query
@Query var users: [User]

// Via FetchDescriptor
let descriptor = FetchDescriptor<User>()
let users = try? modelContext.fetch(descriptor)

// By ID
let id: PersistentIdentifier = user.persistentModelID
let user = modelContext.model(for: id) as? User
```

### Update

```swift
user.name = "Alice Smith"
user.age = 31
// Auto-saved (if autosave enabled)
```

### Delete

```swift
modelContext.delete(user)
// Auto-saved (if autosave enabled)
```

### Batch Delete

```swift
let users = try? modelContext.fetch(FetchDescriptor<User>())
users?.forEach { modelContext.delete($0) }
```

## Saving

### Autosave (Default)

```swift
// Enabled by default
let user = User(name: "Alice")
modelContext.insert(user)
// Saved automatically at end of runloop
```

### Manual Save

```swift
// Disable autosave
.modelContainer(for: User.self, isAutosaveEnabled: false)

// Save manually
do {
    try modelContext.save()
} catch {
    print("Save failed: \(error)")
}
```

### Check for Changes

```swift
if modelContext.hasChanges {
    try? modelContext.save()
}
```

### Rollback Changes

```swift
// Discard unsaved changes
modelContext.rollback()
```

## Undo/Redo

### Enable Undo

```swift
let context = ModelContext(container)
context.undoManager = UndoManager()
```

### Undo/Redo Operations

```swift
// Undo
context.undoManager?.undo()

// Redo
context.undoManager?.redo()

// Check if can undo/redo
let canUndo = context.undoManager?.canUndo ?? false
let canRedo = context.undoManager?.canRedo ?? false
```

### Grouping Changes

```swift
context.undoManager?.beginUndoGrouping()

user.name = "Alice"
user.age = 30
user.email = "alice@example.com"

context.undoManager?.endUndoGrouping()
// All changes undone together
```

## Transaction Management

### Explicit Transaction

```swift
do {
    // Begin transaction
    let user1 = User(name: "Alice")
    let user2 = User(name: "Bob")
    
    modelContext.insert(user1)
    modelContext.insert(user2)
    
    // Commit
    try modelContext.save()
} catch {
    // Rollback on error
    modelContext.rollback()
}
```

### Batch Operations

```swift
func importUsers(_ names: [String]) {
    // Disable autosave for batch
    modelContext.autosaveEnabled = false
    
    for name in names {
        let user = User(name: name)
        modelContext.insert(user)
    }
    
    // Save once at end
    try? modelContext.save()
    
    // Re-enable autosave
    modelContext.autosaveEnabled = true
}
```

## Error Handling

### Save Errors

```swift
do {
    try modelContext.save()
} catch let error as NSError {
    switch error.code {
    case 1570: // Validation error
        print("Validation failed: \(error.localizedDescription)")
    case 133021: // Merge conflict
        print("Merge conflict: \(error.localizedDescription)")
    default:
        print("Save error: \(error)")
    }
}
```

### Validation Errors

```swift
// Common validation errors:
// - Required property is nil
// - Relationship constraint violated
// - Unique constraint violated
// - Min/max count constraint violated

// Check before saving
if user.name.isEmpty {
    // Handle validation error
    return
}

try? modelContext.save()
```

## Context Coordination

### Multiple Contexts

```swift
let container = try ModelContainer(for: User.self)

// Main context (UI)
let mainContext = container.mainContext

// Background context (import)
let backgroundContext = ModelContext(container)

Task.detached {
    // Work in background
    let user = User(name: "Alice")
    backgroundContext.insert(user)
    try? backgroundContext.save()
    
    // Main context automatically notified
}
```

### Merge Changes

```swift
// Changes from other contexts automatically merged
// No manual merge needed (unlike Core Data)
```

## Object States

### Check if Inserted

```swift
let isInserted = modelContext.insertedModelsArray.contains { $0 === user }
```

### Check if Deleted

```swift
let isDeleted = modelContext.deletedModelsArray.contains { $0 === user }
```

### Check if Modified

```swift
let isModified = modelContext.changedModelsArray.contains { $0 === user }
```

### Refresh Object

```swift
// Discard in-memory changes, reload from store
modelContext.refresh(user, mergeChanges: false)

// Keep in-memory changes, merge with store
modelContext.refresh(user, mergeChanges: true)
```

## Performance Patterns

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

### Batch Update

```swift
func batchUpdate() {
    modelContext.autosaveEnabled = false
    
    let descriptor = FetchDescriptor<User>()
    let users = try? modelContext.fetch(descriptor)
    
    users?.forEach { user in
        user.lastUpdated = .now
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
    let users = try? modelContext.fetch(descriptor)
    
    modelContext.autosaveEnabled = false
    users?.forEach { modelContext.delete($0) }
    try? modelContext.save()
    modelContext.autosaveEnabled = true
}
```

## Common Patterns

### Safe Save

```swift
func safeSave() {
    guard modelContext.hasChanges else { return }
    
    do {
        try modelContext.save()
    } catch {
        print("Save failed: \(error)")
        modelContext.rollback()
    }
}
```

### Transactional Update

```swift
func updateUser(_ user: User, name: String, age: Int) {
    let oldName = user.name
    let oldAge = user.age
    
    user.name = name
    user.age = age
    
    do {
        try modelContext.save()
    } catch {
        // Rollback on error
        user.name = oldName
        user.age = oldAge
        print("Update failed: \(error)")
    }
}
```

### Import with Progress

```swift
func importUsers(_ data: [UserData], progress: @escaping (Double) -> Void) {
    modelContext.autosaveEnabled = false
    
    let total = Double(data.count)
    for (index, userData) in data.enumerated() {
        let user = User(name: userData.name, age: userData.age)
        modelContext.insert(user)
        
        progress(Double(index + 1) / total)
    }
    
    try? modelContext.save()
    modelContext.autosaveEnabled = true
}
```

## Common Mistakes

```swift
// ❌ WRONG: Not checking for changes
try modelContext.save() // Unnecessary if no changes

// ✅ CORRECT: Check first
if modelContext.hasChanges {
    try modelContext.save()
}

// ❌ WRONG: Ignoring save errors
try? modelContext.save() // Silent failure

// ✅ CORRECT: Handle errors
do {
    try modelContext.save()
} catch {
    print("Save failed: \(error)")
    modelContext.rollback()
}

// ❌ WRONG: Using context across threads
Task.detached {
    modelContext.insert(user) // Crash!
}

// ✅ CORRECT: Create context per thread
Task.detached {
    let backgroundContext = ModelContext(container)
    backgroundContext.insert(user)
}

// ❌ WRONG: Not disabling autosave for batch
for i in 1...1000 {
    modelContext.insert(User(name: "User \(i)"))
    // Saves 1000 times!
}

// ✅ CORRECT: Disable autosave
modelContext.autosaveEnabled = false
for i in 1...1000 {
    modelContext.insert(User(name: "User \(i)"))
}
try? modelContext.save()
modelContext.autosaveEnabled = true
```

## Debugging

### Print Context State

```swift
print("Has changes: \(modelContext.hasChanges)")
print("Inserted: \(modelContext.insertedModelsArray.count)")
print("Updated: \(modelContext.changedModelsArray.count)")
print("Deleted: \(modelContext.deletedModelsArray.count)")
print("Autosave: \(modelContext.autosaveEnabled)")
```

### Verify Save

```swift
func verifySave() {
    do {
        try modelContext.save()
        print("✅ Save successful")
    } catch {
        print("❌ Save failed: \(error)")
        print("Inserted: \(modelContext.insertedModelsArray.count)")
        print("Updated: \(modelContext.changedModelsArray.count)")
        print("Deleted: \(modelContext.deletedModelsArray.count)")
    }
}
```


## Undo and Redo

### Enable UndoManager

#### Via ModelContainer (Recommended)

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: User.self, isUndoEnabled: true)
    }
}

// Access in view
@Environment(\.undoManager) var undoManager
```

#### Via ModelContext (Manual)

```swift
// Enable for specific view
.onAppear {
    modelContext.undoManager = UndoManager()
}

.onDisappear {
    modelContext.undoManager = nil
}

// Access via context
modelContext.undoManager?.undo()
```

### Undo Operations

```swift
// Undo last change
modelContext.undoManager?.undo()

// Check if can undo
if modelContext.undoManager?.canUndo == true {
    // Enable undo button
}

// Get undo action name
let actionName = modelContext.undoManager?.undoActionName
```

### Redo Operations

```swift
// Redo last undone change
modelContext.undoManager?.redo()

// Check if can redo
if modelContext.undoManager?.canRedo == true {
    // Enable redo button
}

// Get redo action name
let actionName = modelContext.undoManager?.redoActionName
```

### Action Names

```swift
// Set custom action name
func deleteUser(_ user: User) {
    modelContext.delete(user)
    modelContext.undoManager?.setActionName("Delete User")
}

// Display in UI
Button("Undo \(undoManager?.undoActionName ?? "")") {
    undoManager?.undo()
}
.disabled(undoManager?.canUndo == false)
```

### Rollback All Changes

```swift
// Rollback requires autosave disabled
modelContext.autosaveEnabled = false

// Make changes
user.name = "New Name"
user.age = 30

// Rollback all changes at once
if modelContext.hasChanges {
    modelContext.rollback()
}

// Re-enable autosave
modelContext.autosaveEnabled = true
```

**Important:** `rollback()` is NOT part of UndoManager - it discards ALL unsaved changes

### Limit Undo Levels

```swift
// Limit number of undo operations
modelContext.undoManager?.levelsOfUndo = 10

// Unlimited (default)
modelContext.undoManager?.levelsOfUndo = 0
```

### Scope Undo to Specific Views

```swift
struct EditView: View {
    @Environment(\.modelContext) var modelContext
    @Bindable var user: User
    
    var body: some View {
        Form {
            TextField("Name", text: $user.name)
        }
        .onAppear {
            // Enable undo only for this view
            modelContext.undoManager = UndoManager()
        }
        .onDisappear {
            // Disable to free memory
            modelContext.undoManager = nil
        }
    }
}
```

### Grouping Operations

```swift
// Group multiple changes into single undo
modelContext.undoManager?.beginUndoGrouping()

user.name = "Alice"
user.age = 30
user.email = "alice@example.com"

modelContext.undoManager?.endUndoGrouping()
modelContext.undoManager?.setActionName("Update User")

// All changes undone together
```

### System Gestures

```swift
// When isUndoEnabled: true, system gestures work:
// - Three-finger swipe (iPad)
// - Shake to undo (iPhone)
// - Cmd+Z / Cmd+Shift+Z (Mac)

.modelContainer(for: User.self, isUndoEnabled: true)
```

**Note:** System gestures may not work consistently in Simulator

## Undo/Redo Patterns

### Undo Button

```swift
struct ContentView: View {
    @Environment(\.undoManager) var undoManager
    
    var body: some View {
        VStack {
            // Content
            
            HStack {
                Button("Undo") {
                    undoManager?.undo()
                }
                .disabled(undoManager?.canUndo == false)
                
                Button("Redo") {
                    undoManager?.redo()
                }
                .disabled(undoManager?.canRedo == false)
            }
        }
    }
}
```

### Undo with Confirmation

```swift
func deleteUser(_ user: User) {
    let userName = user.name
    
    modelContext.delete(user)
    modelContext.undoManager?.setActionName("Delete \(userName)")
    
    // Show undo toast
    showUndoToast(message: "Deleted \(userName)")
}

func showUndoToast(message: String) {
    // Show temporary message with undo button
}
```

### Clear Undo Stack

```swift
// Remove all undo/redo operations
modelContext.undoManager?.removeAllActions()
```

## Common Mistakes

```swift
// ❌ WRONG: Using rollback with autosave enabled
modelContext.rollback() // Fails silently

// ✅ CORRECT: Disable autosave first
modelContext.autosaveEnabled = false
modelContext.rollback()
modelContext.autosaveEnabled = true

// ❌ WRONG: Forgetting to set action names
modelContext.delete(user)
// Undo button shows generic "Undo"

// ✅ CORRECT: Set descriptive names
modelContext.delete(user)
modelContext.undoManager?.setActionName("Delete User")

// ❌ WRONG: Not checking canUndo/canRedo
Button("Undo") { undoManager?.undo() }
// Button always enabled

// ✅ CORRECT: Disable when not available
Button("Undo") { undoManager?.undo() }
    .disabled(undoManager?.canUndo == false)

// ❌ WRONG: Keeping UndoManager always enabled
// Uses memory for undo stack

// ✅ CORRECT: Enable only when needed
.onAppear { modelContext.undoManager = UndoManager() }
.onDisappear { modelContext.undoManager = nil }
```

## Undo vs Rollback

| Feature | Undo | Rollback |
|---------|------|----------|
| Granularity | Single operation | All changes |
| Requires | UndoManager | Autosave disabled |
| Can redo | Yes | No |
| Action names | Yes | No |
| Memory | Uses stack | No overhead |
| Use case | User-facing | Error recovery |
