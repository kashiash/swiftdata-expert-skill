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

### Transient Properties

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
