# Models and Attributes

## Basic Model Definition

```swift
import SwiftData

@Model
class User {
    var name: String
    var age: Int
    var createdAt: Date
    
    init(name: String, age: Int, createdAt: Date = .now) {
        self.name = name
        self.age = age
        self.createdAt = createdAt
    }
}
```

**Rules:**
- Must be class (not struct)
- Must have @Model macro
- Must have init() with all properties
- Default values recommended where sensible

## Supported Property Types

```swift
@Model
class DataTypes {
    // Basic types
    var string: String
    var int: Int
    var double: Double
    var bool: Bool
    var date: Date
    var data: Data
    var uuid: UUID
    
    // Optional types
    var optionalString: String?
    var optionalInt: Int?
    
    // Collections
    var strings: [String]
    var integers: [Int]
    
    // Codable types
    var customStruct: CustomStruct // Must conform to Codable
    var customEnum: CustomEnum // Must conform to Codable
    
    init(/* ... */) { /* ... */ }
}

struct CustomStruct: Codable {
    var value: String
}

enum CustomEnum: String, Codable {
    case option1, option2
}
```

## Attributes

### Unique Constraint

```swift
@Model
class User {
    @Attribute(.unique) var email: String
    var name: String
    
    init(email: String, name: String) {
        self.email = email
        self.name = name
    }
}
```

**Warning:** Cannot use with CloudKit sync!

### External Storage

```swift
@Model
class Photo {
    var title: String
    @Attribute(.externalStorage) var imageData: Data
    
    init(title: String, imageData: Data) {
        self.title = title
        self.imageData = imageData
    }
}
```

**Use for:** Large binary data (images, videos, documents)

## Computed Properties

Extend models with computed properties for UI formatting:

```swift
extension User {
    var displayName: String {
        "\(firstName) \(lastName)"
    }
    
    var ageGroup: String {
        switch age {
        case 0..<18: return "Minor"
        case 18..<65: return "Adult"
        default: return "Senior"
        }
    }
}

extension Photo {
    var thumbnail: UIImage {
        if let data = imageData, let image = UIImage(data: data) {
            return image
        }
        return UIImage(systemName: "photo")!
    }
}
```

**Rules:**
- Not persisted to storage
- Cannot be used in @Query sort or filter
- Observable - trigger UI updates
- Perfect for formatting data for views

**Common Mistakes:**
```swift
// ❌ Wrong - trying to sort by computed property
@Query(sort: \User.displayName) var users // Crash!

// ✅ Correct - sort by stored property
@Query(sort: \User.firstName) var users
```

## Ephemeral vs Transient Properties

### @Attribute(.ephemeral)

Temporary property that is observable but not persisted:

```swift
@Model
class Item {
    var name: String
    @Attribute(.ephemeral) var isSelected: Bool = false
    @Attribute(.ephemeral) var isExpanded: Bool = false
    
    init(name: String) {
        self.name = name
    }
}

// Usage in view
struct ItemRow: View {
    let item: Item
    
    var body: some View {
        HStack {
            Text(item.name)
            if item.isSelected {
                Image(systemName: "checkmark")
            }
        }
        .onTapGesture {
            item.isSelected.toggle() // UI updates automatically
        }
    }
}
```

**Use for:**
- UI state (selection, expansion, hover)
- Temporary flags
- View-specific state

**Limitations:**
- May not work reliably in predicates
- Reset when app restarts

### @Transient Properties

```swift
@Model
class User {
    var firstName: String
    var lastName: String
    
    @Transient var fullName: String {
        "\(firstName) \(lastName)"
    }
    
    init(firstName: String, lastName: String) {
        self.firstName = firstName
        self.lastName = lastName
    }
}
```

**Use for:** Computed properties not stored in database

### Original Property Name

```swift
@Model
class User {
    @Attribute(originalName: "username") var name: String
    
    init(name: String) {
        self.name = name
    }
}
```

**Use for:** Renaming properties without migration

## Custom Transformers

```swift
import SwiftData

@Model
class User {
    @Attribute(.transformable(by: ColorTransformer.self))
    var favoriteColor: UIColor
    
    init(favoriteColor: UIColor) {
        self.favoriteColor = favoriteColor
    }
}

class ColorTransformer: ValueTransformer {
    override class func transformedValueClass() -> AnyClass {
        return NSData.self
    }
    
    override class func allowsReverseTransformation() -> Bool {
        return true
    }
    
    override func transformedValue(_ value: Any?) -> Any? {
        guard let color = value as? UIColor else { return nil }
        return try? NSKeyedArchiver.archivedData(
            withRootObject: color,
            requiringSecureCoding: true
        )
    }
    
    override func reverseTransformedValue(_ value: Any?) -> Any? {
        guard let data = value as? Data else { return nil }
        return try? NSKeyedUnarchiver.unarchivedObject(
            ofClass: UIColor.self,
            from: data
        )
    }
}
```

## Enums in Models

```swift
@Model
class Task {
    var title: String
    var status: Status
    
    enum Status: String, Codable {
        case todo, inProgress, done
    }
    
    init(title: String, status: Status = .todo) {
        self.title = title
        self.status = status
    }
}
```

**Rules:**
- Must conform to Codable
- Raw value recommended (String, Int)

## Default Values

```swift
@Model
class User {
    var name: String
    var createdAt: Date
    var isActive: Bool
    var loginCount: Int
    
    init(
        name: String,
        createdAt: Date = .now,
        isActive: Bool = true,
        loginCount: Int = 0
    ) {
        self.name = name
        self.createdAt = createdAt
        self.isActive = isActive
        self.loginCount = loginCount
    }
}
```

**Best practice:** Provide defaults for all non-essential properties

## Model Identifiers

```swift
@Model
class User {
    var name: String
    
    // Automatic: persistentModelID (PersistentIdentifier)
    // Conforms to Identifiable automatically
    
    init(name: String) {
        self.name = name
    }
}

// Usage
let id = user.persistentModelID
let user2 = modelContext.model(for: id) as? User
```

## Common Patterns

### Timestamps

```swift
@Model
class Post {
    var title: String
    var content: String
    var createdAt: Date
    var updatedAt: Date
    
    init(title: String, content: String) {
        self.title = title
        self.content = content
        self.createdAt = .now
        self.updatedAt = .now
    }
    
    func touch() {
        updatedAt = .now
    }
}
```

### Soft Delete

```swift
@Model
class User {
    var name: String
    var deletedAt: Date?
    
    var isDeleted: Bool {
        deletedAt != nil
    }
    
    init(name: String) {
        self.name = name
    }
    
    func softDelete() {
        deletedAt = .now
    }
}

// Query non-deleted
@Query(filter: #Predicate<User> { $0.deletedAt == nil })
var activeUsers: [User]
```

### Validation

```swift
@Model
class User {
    var email: String
    var age: Int
    
    init(email: String, age: Int) throws {
        guard email.contains("@") else {
            throw ValidationError.invalidEmail
        }
        guard age >= 0 && age <= 150 else {
            throw ValidationError.invalidAge
        }
        self.email = email
        self.age = age
    }
}

enum ValidationError: Error {
    case invalidEmail
    case invalidAge
}
```

## CloudKit Considerations

```swift
@Model
class User {
    // ❌ Cannot use with CloudKit
    // @Attribute(.unique) var email: String
    
    // ✅ Use optional instead
    var email: String?
    var name: String?
    var age: Int?
    
    init(email: String? = nil, name: String? = nil, age: Int? = nil) {
        self.email = email
        self.name = name
        self.age = age
    }
}
```

**CloudKit rules:**
- All properties must be optional OR have defaults
- No @Attribute(.unique)
- All relationships must be optional

## Common Mistakes

```swift
// ❌ WRONG: Struct
@Model
struct User { // Compile error
    var name: String
}

// ✅ CORRECT: Class
@Model
class User {
    var name: String
    init(name: String) { self.name = name }
}

// ❌ WRONG: Missing init
@Model
class User {
    var name: String
    // Compile error: no initializer
}

// ✅ CORRECT: Has init
@Model
class User {
    var name: String
    init(name: String) { self.name = name }
}

// ❌ WRONG: Unsupported type
@Model
class User {
    var callback: () -> Void // Not Codable
}

// ✅ CORRECT: Codable types only
@Model
class User {
    var name: String
    init(name: String) { self.name = name }
}
```


## Advanced Custom Types

### Enum Requirements

Must conform to: String, Codable, CaseIterable

```swift
@Model
class Task {
    var title: String
    var status: Status = .pending
    var priority: Priority = .medium
    
    enum Status: String, Codable, CaseIterable {
        case pending = "Pending"
        case inProgress = "In Progress"
        case completed = "Completed"
    }
    
    enum Priority: String, Codable, CaseIterable {
        case low = "Low"
        case medium = "Medium"
        case high = "High"
    }
    
    init(title: String) {
        self.title = title
    }
}

// Usage with Picker
Picker("Status", selection: $task.status) {
    ForEach(Task.Status.allCases, id: \.self) { status in
        Text(status.rawValue).tag(status)
    }
}
```

**Requirements:**
- `String` - raw value for storage
- `Codable` - persistence
- `CaseIterable` - SwiftUI iteration (Picker, ForEach)

### Struct Binding Workaround

Cannot directly bind optional struct properties:

```swift
@Model
class Location {
    var name: String
    var address: Address?
    
    struct Address: Codable {
        var street: String
        var city: String
        var country: String
    }
    
    init(name: String) {
        self.name = name
    }
}

// ❌ Wrong - cannot bind optional struct property
TextField("City", text: $location.address?.city) // Error!

// ✅ Correct - copy to @State first
struct LocationEditor: View {
    @Bindable var location: Location
    @State private var city: String = ""
    
    var body: some View {
        TextField("City", text: $city)
            .task {
                city = location.address?.city ?? ""
            }
            .onChange(of: city) { _, newValue in
                if location.address == nil {
                    location.address = Location.Address(
                        street: "",
                        city: "",
                        country: ""
                    )
                }
                location.address?.city = newValue
            }
    }
}
```

## PhotosPicker Integration

Store images with external storage:

```swift
import PhotosUI

@Model
class Photo {
    var title: String
    @Attribute(.externalStorage) var imageData: Data?
    
    init(title: String) {
        self.title = title
    }
}

// View with PhotosPicker
struct PhotoEditor: View {
    @Bindable var photo: Photo
    @State private var selection: PhotosPickerItem?
    
    var body: some View {
        VStack {
            if let data = photo.imageData,
               let uiImage = UIImage(data: data) {
                Image(uiImage: uiImage)
                    .resizable()
                    .scaledToFit()
            }
            
            PhotosPicker(selection: $selection, matching: .images) {
                Label("Select Photo", systemImage: "photo")
            }
            .task(id: selection) {
                if let data = try? await selection?.loadTransferable(type: Data.self) {
                    photo.imageData = data
                }
            }
        }
    }
}
```

**Best Practices:**
- Always use @Attribute(.externalStorage) for images
- Handle nil imageData gracefully
- Use .task(id:) for async loading
- Consider compression for large images

## Mock Data Preparation

### Preview Assets

Store test images in Preview Assets.xcassets (not included in release builds):

```swift
extension Photo {
    static var preview: Photo {
        let photo = Photo(title: "Sample")
        // Image from Preview Assets
        if let image = UIImage(resource: .samplePhoto) {
            photo.imageData = image.pngData()
            // or: photo.imageData = image.jpegData(compressionQuality: 0.8)
        }
        return photo
    }
    
    static var previews: [Photo] {
        [
            Photo.preview,
            Photo(title: "Landscape"),
            Photo(title: "Portrait")
        ]
    }
}
```

### Preview Container

```swift
@MainActor
let previewContainer: ModelContainer = {
    let schema = Schema([Photo.self])
    let config = ModelConfiguration(isStoredInMemoryOnly: true)
    let container = try! ModelContainer(for: schema, configurations: [config])
    
    // Add mock data
    for photo in Photo.previews {
        container.mainContext.insert(photo)
    }
    
    return container
}()

// Usage in preview
#Preview {
    PhotoList()
        .modelContainer(previewContainer)
}
```

**Rules:**
- Preview Assets folder contents not in release builds
- Use PNG for lossless, JPEG for smaller size
- Create static preview data on model extensions
- Use in-memory container for previews

## PersistentModel Protocol

All @Model classes automatically conform to PersistentModel:

### Observable

Models are automatically observable - UI updates when data changes:

```swift
@Model
class Counter {
    var count: Int = 0
}

struct CounterView: View {
    let counter: Counter
    
    var body: some View {
        VStack {
            Text("Count: \(counter.count)")
            Button("Increment") {
                counter.count += 1 // UI updates automatically
            }
        }
    }
}
```

No need for @Published, @StateObject, or manual observation.

### Identifiable

Models are automatically Identifiable with PersistentIdentifier:

```swift
@Model
class Item {
    var name: String
    // Automatic: var id: PersistentIdentifier
}

// Use directly in List/ForEach
List(items) { item in // No need for id: \.self
    Text(item.name)
}
```

**PersistentIdentifier:**
- Stable across app launches
- Unique per model instance
- Used internally by SwiftData
- Can be stored and used for lookups

```swift
// Store identifier
let itemID = item.id

// Later, fetch by identifier
if let item = modelContext.model(for: itemID) as? Item {
    print(item.name)
}
```
