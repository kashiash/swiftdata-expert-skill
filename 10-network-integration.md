# Network Integration

Integrating SwiftData with URLSession for fetching and storing data from APIs.

## Basic Pattern

```swift
@Model
class Post: Codable {
    enum CodingKeys: CodingKey {
        case id, title, body
    }
    
    let id: Int
    var title: String
    var body: String
    
    required init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(Int.self, forKey: .id)
        title = try container.decode(String.self, forKey: .title)
        body = try container.decode(String.self, forKey: .body)
    }
    
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
        try container.encode(title, forKey: .title)
        try container.encode(body, forKey: .body)
    }
}
```

## Fetch and Save

```swift
func fetchPosts() async throws {
    guard let url = URL(string: "https://api.example.com/posts") else {
        throw URLError(.badURL)
    }
    
    let (data, _) = try await URLSession.shared.data(from: url)
    let posts = try JSONDecoder().decode([Post].self, from: data)
    
    await MainActor.run {
        for post in posts {
            modelContext.insert(post)
        }
    }
}
```

## Update vs Insert (Upsert)

```swift
func syncPosts() async throws {
    let (data, _) = try await URLSession.shared.data(from: url)
    let fetchedPosts = try JSONDecoder().decode([Post].self, from: data)
    
    await MainActor.run {
        let descriptor = FetchDescriptor<Post>()
        let existingPosts = try? modelContext.fetch(descriptor)
        let existingIDs = Set(existingPosts?.map(\.id) ?? [])
        
        for post in fetchedPosts {
            if existingIDs.contains(post.id) {
                // Update existing
                if let existing = existingPosts?.first(where: { $0.id == post.id }) {
                    existing.title = post.title
                    existing.body = post.body
                }
            } else {
                // Insert new
                modelContext.insert(post)
            }
        }
    }
}
```

## Image Download

```swift
@Model
class Article {
    var title: String
    var imageURL: String?
    @Attribute(.externalStorage) var imageData: Data?
    
    init(title: String, imageURL: String? = nil) {
        self.title = title
        self.imageURL = imageURL
    }
}

func downloadImage(for article: Article) async throws {
    guard let urlString = article.imageURL,
          let url = URL(string: urlString) else { return }
    
    let (data, _) = try await URLSession.shared.data(from: url)
    
    await MainActor.run {
        article.imageData = data
    }
}
```

## Background Sync

```swift
func backgroundSync() async throws {
    let container = modelContainer
    let backgroundContext = ModelContext(container)
    
    let (data, _) = try await URLSession.shared.data(from: url)
    let posts = try JSONDecoder().decode([Post].self, from: data)
    
    for post in posts {
        backgroundContext.insert(post)
    }
    
    try backgroundContext.save()
}
```

## Error Handling

```swift
enum NetworkError: Error {
    case invalidURL
    case decodingFailed
    case networkFailed
    case saveFailed
}

func fetchWithErrorHandling() async throws {
    do {
        guard let url = URL(string: apiURL) else {
            throw NetworkError.invalidURL
        }
        
        let (data, response) = try await URLSession.shared.data(from: url)
        
        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.networkFailed
        }
        
        let posts = try JSONDecoder().decode([Post].self, from: data)
        
        await MainActor.run {
            for post in posts {
                modelContext.insert(post)
            }
        }
        
        try modelContext.save()
        
    } catch is DecodingError {
        throw NetworkError.decodingFailed
    } catch {
        throw error
    }
}
```

## Cache Strategy

```swift
func fetchWithCache(forceRefresh: Bool = false) async throws {
    let descriptor = FetchDescriptor<Post>()
    let cached = try? modelContext.fetch(descriptor)
    
    if !forceRefresh && !cached.isEmpty {
        // Use cached data
        return
    }
    
    // Fetch fresh data
    let (data, _) = try await URLSession.shared.data(from: url)
    let posts = try JSONDecoder().decode([Post].self, from: data)
    
    await MainActor.run {
        // Clear old data
        cached?.forEach { modelContext.delete($0) }
        
        // Insert fresh data
        for post in posts {
            modelContext.insert(post)
        }
    }
}
```

## Pagination

```swift
func fetchPage(page: Int, pageSize: Int = 20) async throws {
    var components = URLComponents(string: "https://api.example.com/posts")
    components?.queryItems = [
        URLQueryItem(name: "page", value: "\(page)"),
        URLQueryItem(name: "limit", value: "\(pageSize)")
    ]
    
    guard let url = components?.url else {
        throw URLError(.badURL)
    }
    
    let (data, _) = try await URLSession.shared.data(from: url)
    let posts = try JSONDecoder().decode([Post].self, from: data)
    
    await MainActor.run {
        for post in posts {
            modelContext.insert(post)
        }
    }
}
```

## Observable Manager Pattern

```swift
@Observable
class NetworkManager {
    var posts: [Post] = []
    var isLoading = false
    var error: Error?
    
    private let modelContext: ModelContext
    
    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }
    
    func fetchPosts() async {
        isLoading = true
        error = nil
        
        do {
            let url = URL(string: "https://api.example.com/posts")!
            let (data, _) = try await URLSession.shared.data(from: url)
            let fetchedPosts = try JSONDecoder().decode([Post].self, from: data)
            
            await MainActor.run {
                for post in fetchedPosts {
                    modelContext.insert(post)
                }
                
                let descriptor = FetchDescriptor<Post>()
                posts = (try? modelContext.fetch(descriptor)) ?? []
            }
        } catch {
            await MainActor.run {
                self.error = error
            }
        }
        
        await MainActor.run {
            isLoading = false
        }
    }
}

// Usage in view
struct PostsView: View {
    @Environment(\.modelContext) var modelContext
    @State private var manager: NetworkManager?
    
    var body: some View {
        List(manager?.posts ?? []) { post in
            Text(post.title)
        }
        .task {
            manager = NetworkManager(modelContext: modelContext)
            await manager?.fetchPosts()
        }
    }
}
```

## Common Patterns

### Codable + @Model
```swift
// Separate DTO and Model
struct PostDTO: Codable {
    let id: Int
    let title: String
    let body: String
}

@Model
class Post {
    let id: Int
    var title: String
    var body: String
    
    init(from dto: PostDTO) {
        self.id = dto.id
        self.title = dto.title
        self.body = dto.body
    }
}
```

### Timestamp Tracking
```swift
@Model
class Post {
    var title: String
    var lastSynced: Date?
    
    func needsSync(threshold: TimeInterval = 3600) -> Bool {
        guard let lastSynced else { return true }
        return Date().timeIntervalSince(lastSynced) > threshold
    }
}
```

### Batch Processing
```swift
func syncLargeDataset() async throws {
    let batchSize = 100
    var offset = 0
    
    while true {
        let url = URL(string: "https://api.example.com/posts?offset=\(offset)&limit=\(batchSize)")!
        let (data, _) = try await URLSession.shared.data(from: url)
        let posts = try JSONDecoder().decode([Post].self, from: data)
        
        if posts.isEmpty { break }
        
        await MainActor.run {
            for post in posts {
                modelContext.insert(post)
            }
        }
        
        offset += batchSize
    }
}
```

## Best Practices

1. **Always decode on background thread** - URLSession operations are async
2. **Insert on MainActor** - ModelContext operations need main thread
3. **Use @Attribute(.externalStorage)** - For images and large data
4. **Implement upsert logic** - Check existing before insert
5. **Handle errors gracefully** - Network can fail
6. **Cache when appropriate** - Reduce network calls
7. **Use background context** - For large sync operations
8. **Track sync timestamps** - Know when data is stale

## Common Mistakes

```swift
// ❌ Wrong - inserting on background thread
Task {
    let posts = try await fetchPosts()
    for post in posts {
        modelContext.insert(post) // Crash!
    }
}

// ✅ Correct - insert on MainActor
Task {
    let posts = try await fetchPosts()
    await MainActor.run {
        for post in posts {
            modelContext.insert(post)
        }
    }
}

// ❌ Wrong - storing image URL as Data
@Model class Article {
    var imageData: Data? // Should be URL string
}

// ✅ Correct - separate URL and data
@Model class Article {
    var imageURL: String?
    @Attribute(.externalStorage) var imageData: Data?
}

// ❌ Wrong - no error handling
func fetch() async {
    let data = try! await URLSession.shared.data(from: url)
}

// ✅ Correct - proper error handling
func fetch() async throws {
    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        // process
    } catch {
        // handle error
        throw error
    }
}
```
