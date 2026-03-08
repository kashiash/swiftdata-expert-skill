# Relationships

## Inferred vs Explicit

### Inferred (Automatic)

```swift
@Model
class School {
    var name: String
    var students: [Student]
    
    init(name: String, students: [Student] = []) {
        self.name = name
        self.students = students
    }
}

@Model
class Student {
    var name: String
    var school: School? // Must be optional for inference
    
    init(name: String, school: School? = nil) {
        self.name = name
        self.school = school
    }
}
```

**Rules for inference:**
- One side must be optional
- SwiftData infers bidirectional relationship
- Safe for simple cases

### Explicit (Recommended)

```swift
@Model
class School {
    var name: String
    @Relationship(deleteRule: .cascade, inverse: \Student.school)
    var students: [Student]
    
    init(name: String, students: [Student] = []) {
        self.name = name
        self.students = students
    }
}

@Model
class Student {
    var name: String
    var school: School
    
    init(name: String, school: School) {
        self.name = name
        self.school = school
    }
}
```

**Best practice:** Always use @Relationship for clarity and control

## One-to-One

```swift
@Model
class Country {
    var name: String
    var capitalCity: City?
    
    init(name: String, capitalCity: City? = nil) {
        self.name = name
        self.capitalCity = capitalCity
    }
}

@Model
class City {
    var name: String
    var country: Country?
    
    init(name: String, country: Country? = nil) {
        self.name = name
        self.country = country
    }
}

// Usage
let country = Country(name: "England")
let city = City(name: "London", country: country)
modelContext.insert(city) // Inserts both
```

**Rules:**
- Both sides must be optional (Swift limitation)
- Insert only one object (other inserted automatically)

## One-to-Many

```swift
@Model
class Director {
    var name: String
    @Relationship(deleteRule: .nullify, inverse: \Movie.director)
    var movies: [Movie]
    
    init(name: String, movies: [Movie] = []) {
        self.name = name
        self.movies = movies
    }
}

@Model
class Movie {
    var title: String
    var director: Director?
    
    init(title: String, director: Director? = nil) {
        self.title = title
        self.director = director
    }
}
```

**Delete rules:**
- `.nullify` - Set to nil (default)
- `.cascade` - Delete related objects
- `.deny` - Prevent deletion if relationships exist
- `.noAction` - Do nothing (dangerous)

## Many-to-Many

```swift
@Model
class Actor {
    var name: String
    var movies: [Movie]
    
    init(name: String, movies: [Movie] = []) {
        self.name = name
        self.movies = movies
    }
}

@Model
class Movie {
    var title: String
    @Relationship(inverse: \Actor.movies) var cast: [Actor]
    
    init(title: String, cast: [Actor] = []) {
        self.title = title
        self.cast = cast
    }
}

// ✅ CORRECT: Create objects, then insert
let movie = Movie(title: "Mission: Impossible", cast: [])
let cruise = Actor(name: "Tom Cruise", movies: [movie])
let newton = Actor(name: "Thandiwe Newton", movies: [movie])

modelContext.insert(cruise)
modelContext.insert(newton)
// Movie inserted automatically

// ❌ WRONG: Manipulate before insert
let movie = Movie(title: "MI", cast: [])
let cruise = Actor(name: "Tom", movies: [])
movie.cast.append(cruise) // CRASH!
```

**Critical rules:**
- Must use @Relationship (not inferred)
- Insert objects BEFORE manipulating arrays
- Define inverse on ONE side only

## Delete Rules

### Nullify (Default)

```swift
@Model
class Author {
    var name: String
    @Relationship(deleteRule: .nullify, inverse: \Book.author)
    var books: [Book]
    
    init(name: String, books: [Book] = []) {
        self.name = name
        self.books = books
    }
}

@Model
class Book {
    var title: String
    var author: Author?
    
    init(title: String, author: Author? = nil) {
        self.title = title
        self.author = author
    }
}

// Delete author → books.author = nil
```

### Cascade

```swift
@Model
class House {
    var address: String
    @Relationship(deleteRule: .cascade, inverse: \Room.house)
    var rooms: [Room]
    
    init(address: String, rooms: [Room] = []) {
        self.address = address
        self.rooms = rooms
    }
}

@Model
class Room {
    var name: String
    var house: House
    
    init(name: String, house: House) {
        self.name = name
        self.house = house
    }
}

// Delete house → all rooms deleted
```

### Multi-level Cascade

```swift
@Model
class School {
    var name: String
    @Relationship(deleteRule: .cascade, inverse: \Student.school)
    var students: [Student]
    
    init(name: String, students: [Student] = []) {
        self.name = name
        self.students = students
    }
}

@Model
class Student {
    var name: String
    var school: School
    @Relationship(deleteRule: .cascade) var grades: [Grade]
    
    init(name: String, school: School, grades: [Grade] = []) {
        self.name = name
        self.school = school
        self.grades = grades
    }
}

@Model
class Grade {
    var subject: String
    var score: Int
    
    init(subject: String, score: Int) {
        self.subject = subject
        self.score = score
    }
}

// Delete school → students deleted → grades deleted
```

## Relationship Constraints

### Minimum/Maximum Count

```swift
@Model
class DogWalker {
    var name: String
    @Relationship(
        minimumModelCount: 1,
        maximumModelCount: 5,
        inverse: \Dog.walker
    )
    var dogs: [Dog]
    
    init(name: String, dogs: [Dog] = []) {
        self.name = name
        self.dogs = dogs
    }
}

@Model
class Dog {
    var name: String
    var walker: DogWalker?
    
    init(name: String, walker: DogWalker? = nil) {
        self.name = name
        self.walker = walker
    }
}

// Violating constraints causes silent save failure!
```

**Warning:** Constraint violations cause silent failures - always verify saves

## Accessing Relationships

### Direct Access

```swift
// Load movie with director
let movie = movies.first!
print(movie.director?.name ?? "Unknown")

// Load director with movies
let director = directors.first!
for movie in director.movies {
    print(movie.title)
}
```

### Lazy Loading

```swift
// Relationships loaded on-demand
let movie = movies.first!
// director not loaded yet

let directorName = movie.director?.name
// director loaded now
```

## Common Patterns

### Bidirectional Navigation

```swift
@Model
class Post {
    var title: String
    @Relationship(deleteRule: .cascade, inverse: \Comment.post)
    var comments: [Comment]
    
    init(title: String, comments: [Comment] = []) {
        self.title = title
        self.comments = comments
    }
}

@Model
class Comment {
    var text: String
    var post: Post
    
    init(text: String, post: Post) {
        self.text = text
        self.post = post
    }
}

// Navigate both ways
let post = posts.first!
let firstComment = post.comments.first!
let backToPost = firstComment.post
```

### Self-Referencing

```swift
@Model
class Employee {
    var name: String
    var manager: Employee?
    @Relationship(inverse: \Employee.manager)
    var reports: [Employee]
    
    init(name: String, manager: Employee? = nil, reports: [Employee] = []) {
        self.name = name
        self.manager = manager
        self.reports = reports
    }
}
```

### Optional vs Required

```swift
// Optional: User can exist without profile
@Model
class User {
    var name: String
    var profile: Profile?
    
    init(name: String, profile: Profile? = nil) {
        self.name = name
        self.profile = profile
    }
}

// Required: Comment must have post
@Model
class Comment {
    var text: String
    var post: Post
    
    init(text: String, post: Post) {
        self.text = text
        self.post = post
    }
}
```

## CloudKit Relationships

```swift
@Model
class User {
    var name: String?
    var posts: [Post]? // Must be optional for CloudKit
    
    init(name: String? = nil, posts: [Post]? = nil) {
        self.name = name
        self.posts = posts
    }
}

@Model
class Post {
    var title: String?
    var author: User? // Must be optional for CloudKit
    
    init(title: String? = nil, author: User? = nil) {
        self.title = title
        self.author = author
    }
}
```

**CloudKit rule:** All relationships must be optional

## Common Mistakes

```swift
// ❌ WRONG: Both sides non-optional
@Model
class User {
    var profile: Profile // Cannot infer
}
@Model
class Profile {
    var user: User // Cannot infer
}

// ✅ CORRECT: One side optional
@Model
class User {
    var profile: Profile?
}
@Model
class Profile {
    var user: User?
}

// ❌ WRONG: Manipulate before insert
let movie = Movie(title: "MI", cast: [])
let actor = Actor(name: "Tom", movies: [])
movie.cast.append(actor) // CRASH
modelContext.insert(movie)

// ✅ CORRECT: Insert first
let movie = Movie(title: "MI", cast: [])
let actor = Actor(name: "Tom", movies: [])
modelContext.insert(movie)
modelContext.insert(actor)
movie.cast.append(actor) // OK

// ❌ WRONG: @Relationship on both sides
@Model
class User {
    @Relationship(inverse: \Profile.user) var profile: Profile?
}
@Model
class Profile {
    @Relationship(inverse: \User.profile) var user: User? // Error!
}

// ✅ CORRECT: @Relationship on one side only
@Model
class User {
    @Relationship(inverse: \Profile.user) var profile: Profile?
}
@Model
class Profile {
    var user: User?
}
```

## Troubleshooting

### Relationship Not Updating

```swift
// Problem: Changes not reflected
user.posts.append(newPost)
// UI doesn't update

// Solution: Ensure object is inserted
modelContext.insert(newPost)
user.posts.append(newPost)
```

### Cascade Delete Not Working

```swift
// Problem: Related objects not deleted
@Model
class Parent {
    var children: [Child] // Missing delete rule
}

// Solution: Add cascade delete rule
@Model
class Parent {
    @Relationship(deleteRule: .cascade, inverse: \Child.parent)
    var children: [Child]
}
```

### Constraint Violation

```swift
// Problem: Silent save failure
walker.dogs.append(dog6) // Exceeds maximum of 5

// Solution: Check constraints before adding
if walker.dogs.count < 5 {
    walker.dogs.append(newDog)
} else {
    // Handle error
}
```
