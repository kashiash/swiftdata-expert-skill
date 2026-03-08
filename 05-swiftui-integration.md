# SwiftUI Integration

## @Query Basics

```swift
import SwiftUI
import SwiftData

struct ContentView: View {
    @Query var users: [User]
    @Environment(\.modelContext) var modelContext
    
    var body: some View {
        List(users) { user in
            Text(user.name)
        }
    }
}
```

**Key points:**
- Only works in SwiftUI views
- Automatically updates on data changes
- Uses main context from environment

## @Bindable for Editing

```swift
struct EditUserView: View {
    @Bindable var user: User
    
    var body: some View {
        Form {
            TextField("Name", text: $user.name)
            TextField("Email", text: $user.email)
        }
    }
}
```

**Use @Bindable when:**
- Need two-way binding to SwiftData object
- Editing properties in forms
- Object passed from parent view

## Dynamic Sorting

### Setup

```swift
// Child view with query
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

// Parent view with sort control
struct ContentView: View {
    @State private var sortOrder = SortDescriptor(\User.name)
    
    var body: some View {
        VStack {
            Picker("Sort", selection: $sortOrder) {
                Text("Name").tag(SortDescriptor(\User.name))
                Text("Age").tag(SortDescriptor(\User.age, order: .reverse))
            }
            
            UserListView(sort: sortOrder)
        }
    }
}
```

### Multiple Sort Options

```swift
struct ContentView: View {
    @State private var sortOrder = SortDescriptor(\User.name)
    
    var body: some View {
        NavigationStack {
            UserListView(sort: sortOrder)
                .toolbar {
                    Menu("Sort", systemImage: "arrow.up.arrow.down") {
                        Picker("Sort", selection: $sortOrder) {
                            Text("Name")
                                .tag(SortDescriptor(\User.name))
                            Text("Age")
                                .tag(SortDescriptor(\User.age, order: .reverse))
                            Text("Date")
                                .tag(SortDescriptor(\User.createdAt, order: .reverse))
                        }
                    }
                }
        }
    }
}
```

## Dynamic Filtering

### Search

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

### Combined Sort and Filter

```swift
struct UserListView: View {
    @Query var users: [User]
    
    init(sort: SortDescriptor<User>, searchText: String) {
        _users = Query(
            filter: #Predicate {
                if searchText.isEmpty {
                    return true
                } else {
                    return $0.name.localizedStandardContains(searchText)
                }
            },
            sort: [sort]
        )
    }
    
    var body: some View {
        List(users) { user in
            Text(user.name)
        }
    }
}

struct ContentView: View {
    @State private var sortOrder = SortDescriptor(\User.name)
    @State private var searchText = ""
    
    var body: some View {
        NavigationStack {
            UserListView(sort: sortOrder, searchText: searchText)
                .searchable(text: $searchText)
                .toolbar {
                    Menu("Sort", systemImage: "arrow.up.arrow.down") {
                        Picker("Sort", selection: $sortOrder) {
                            Text("Name").tag(SortDescriptor(\User.name))
                            Text("Age").tag(SortDescriptor(\User.age))
                        }
                    }
                }
        }
    }
}
```

## List Operations

### Delete

```swift
struct ContentView: View {
    @Query var users: [User]
    @Environment(\.modelContext) var modelContext
    
    var body: some View {
        List {
            ForEach(users) { user in
                Text(user.name)
            }
            .onDelete(perform: deleteUsers)
        }
    }
    
    func deleteUsers(_ indexSet: IndexSet) {
        for index in indexSet {
            modelContext.delete(users[index])
        }
    }
}
```

### Move

```swift
struct ContentView: View {
    @Query(sort: \User.order) var users: [User]
    @Environment(\.modelContext) var modelContext
    
    var body: some View {
        List {
            ForEach(users) { user in
                Text(user.name)
            }
            .onMove(perform: moveUsers)
        }
    }
    
    func moveUsers(from source: IndexSet, to destination: Int) {
        var updatedUsers = users
        updatedUsers.move(fromOffsets: source, toOffset: destination)
        
        for (index, user) in updatedUsers.enumerated() {
            user.order = index
        }
    }
}
```

## Navigation

### NavigationLink with Value

```swift
struct ContentView: View {
    @Query var users: [User]
    
    var body: some View {
        NavigationStack {
            List(users) { user in
                NavigationLink(value: user) {
                    Text(user.name)
                }
            }
            .navigationDestination(for: User.self) { user in
                EditUserView(user: user)
            }
        }
    }
}
```

### Programmatic Navigation

```swift
struct ContentView: View {
    @Query var users: [User]
    @Environment(\.modelContext) var modelContext
    @State private var path = [User]()
    
    var body: some View {
        NavigationStack(path: $path) {
            List(users) { user in
                NavigationLink(value: user) {
                    Text(user.name)
                }
            }
            .navigationDestination(for: User.self) { user in
                EditUserView(user: user)
            }
            .toolbar {
                Button("Add", systemImage: "plus", action: addUser)
            }
        }
    }
    
    func addUser() {
        let user = User(name: "New User")
        modelContext.insert(user)
        path = [user] // Navigate to new user
    }
}
```

## Forms and Editing

### Basic Form

```swift
struct EditUserView: View {
    @Bindable var user: User
    
    var body: some View {
        Form {
            TextField("Name", text: $user.name)
            TextField("Email", text: $user.email)
            
            DatePicker("Birth Date", selection: $user.birthDate, displayedComponents: .date)
            
            Picker("Role", selection: $user.role) {
                Text("User").tag(UserRole.user)
                Text("Admin").tag(UserRole.admin)
            }
        }
        .navigationTitle("Edit User")
    }
}
```

### Validation

```swift
struct EditUserView: View {
    @Bindable var user: User
    @State private var showError = false
    @Environment(\.dismiss) var dismiss
    
    var body: some View {
        Form {
            TextField("Name", text: $user.name)
            TextField("Email", text: $user.email)
        }
        .navigationTitle("Edit User")
        .toolbar {
            Button("Save") {
                if validate() {
                    dismiss()
                } else {
                    showError = true
                }
            }
        }
        .alert("Invalid Data", isPresented: $showError) {
            Button("OK") { }
        } message: {
            Text("Please check your input")
        }
    }
    
    func validate() -> Bool {
        !user.name.isEmpty && user.email.contains("@")
    }
}
```

## Animations

### Animated Changes

```swift
@Query(sort: \User.name, animation: .default) var users: [User]
```

### Custom Animation

```swift
@Query(sort: \User.name, animation: .spring(duration: 0.3)) var users: [User]
```

### Conditional Animation

```swift
List(users) { user in
    Text(user.name)
}
.animation(.default, value: users)
```

## Observation and Updates

### Automatic Updates

```swift
struct UserDetailView: View {
    @Bindable var user: User
    
    var body: some View {
        VStack {
            Text(user.name) // Updates automatically
            Text("\(user.age)") // Updates automatically
        }
    }
}
```

### Manual Refresh

```swift
struct ContentView: View {
    @Query var users: [User]
    @Environment(\.modelContext) var modelContext
    
    var body: some View {
        List(users) { user in
            Text(user.name)
        }
        .refreshable {
            // Reload data from server
            await fetchUsers()
        }
    }
    
    func fetchUsers() async {
        // Fetch and update
    }
}
```

## Environment Access

### ModelContext

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

### Custom Environment

```swift
// Define key
struct UserFilterKey: EnvironmentKey {
    static let defaultValue: UserFilter = .all
}

extension EnvironmentValues {
    var userFilter: UserFilter {
        get { self[UserFilterKey.self] }
        set { self[UserFilterKey.self] = newValue }
    }
}

// Use in view
struct ContentView: View {
    @State private var filter: UserFilter = .all
    
    var body: some View {
        UserListView()
            .environment(\.userFilter, filter)
    }
}

struct UserListView: View {
    @Environment(\.userFilter) var filter
    @Query var users: [User]
    
    init() {
        // Can't access environment in init
        _users = Query()
    }
    
    var filteredUsers: [User] {
        users.filter { filter.matches($0) }
    }
    
    var body: some View {
        List(filteredUsers) { user in
            Text(user.name)
        }
    }
}
```

## UIKit Integration

### UICollectionView

```swift
class UserViewController: UIViewController {
    var container: ModelContainer?
    var collectionView: UICollectionView!
    var dataSource: UICollectionViewDiffableDataSource<Section, User>?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        container = try? ModelContainer(for: User.self)
        setupCollectionView()
        setupDataSource()
        loadUsers()
    }
    
    func setupCollectionView() {
        let config = UICollectionLayoutListConfiguration(appearance: .plain)
        let layout = UICollectionViewCompositionalLayout.list(using: config)
        
        collectionView = UICollectionView(frame: view.bounds, collectionViewLayout: layout)
        collectionView.autoresizingMask = [.flexibleWidth, .flexibleHeight]
        view.addSubview(collectionView)
        
        collectionView.register(UICollectionViewListCell.self, forCellWithReuseIdentifier: "Cell")
    }
    
    func setupDataSource() {
        dataSource = UICollectionViewDiffableDataSource<Section, User>(
            collectionView: collectionView
        ) { collectionView, indexPath, user in
            let cell = collectionView.dequeueReusableCell(
                withReuseIdentifier: "Cell",
                for: indexPath
            )
            
            var content = UIListContentConfiguration.cell()
            content.text = user.name
            cell.contentConfiguration = content
            
            return cell
        }
    }
    
    func loadUsers() {
        let descriptor = FetchDescriptor<User>()
        let users = (try? container?.mainContext.fetch(descriptor)) ?? []
        
        var snapshot = NSDiffableDataSourceSnapshot<Section, User>()
        snapshot.appendSections([.main])
        snapshot.appendItems(users)
        dataSource?.apply(snapshot, animatingDifferences: false)
    }
}

enum Section {
    case main
}
```

## Common Patterns

### Pull to Refresh

```swift
struct ContentView: View {
    @Query var users: [User]
    @Environment(\.modelContext) var modelContext
    
    var body: some View {
        List(users) { user in
            Text(user.name)
        }
        .refreshable {
            await refreshUsers()
        }
    }
    
    func refreshUsers() async {
        // Fetch from server
        let newUsers = await fetchFromServer()
        
        // Update local data
        for userData in newUsers {
            if let existing = users.first(where: { $0.id == userData.id }) {
                existing.name = userData.name
            } else {
                let user = User(name: userData.name)
                modelContext.insert(user)
            }
        }
    }
}
```

### Empty State

```swift
struct ContentView: View {
    @Query var users: [User]
    
    var body: some View {
        Group {
            if users.isEmpty {
                ContentUnavailableView(
                    "No Users",
                    systemImage: "person.slash",
                    description: Text("Add your first user to get started")
                )
            } else {
                List(users) { user in
                    Text(user.name)
                }
            }
        }
    }
}
```

### Sectioned List

```swift
struct ContentView: View {
    @Query(sort: \User.name) var users: [User]
    
    var groupedUsers: [String: [User]] {
        Dictionary(grouping: users) { user in
            String(user.name.prefix(1))
        }
    }
    
    var body: some View {
        List {
            ForEach(groupedUsers.keys.sorted(), id: \.self) { key in
                Section(key) {
                    ForEach(groupedUsers[key] ?? []) { user in
                        Text(user.name)
                    }
                }
            }
        }
    }
}
```

## Common Mistakes

```swift
// ❌ WRONG: Using @Query in non-view
class ViewModel {
    @Query var users: [User] // Doesn't work
}

// ✅ CORRECT: Use FetchDescriptor
class ViewModel {
    func loadUsers(context: ModelContext) {
        let descriptor = FetchDescriptor<User>()
        let users = try? context.fetch(descriptor)
    }
}

// ❌ WRONG: Accessing environment in init
struct UserListView: View {
    @Environment(\.modelContext) var modelContext
    
    init() {
        modelContext.insert(User()) // Crash!
    }
}

// ✅ CORRECT: Use onAppear
struct UserListView: View {
    @Environment(\.modelContext) var modelContext
    
    var body: some View {
        List { }
            .onAppear {
                modelContext.insert(User())
            }
    }
}

// ❌ WRONG: Not using @Bindable
struct EditView: View {
    var user: User
    
    var body: some View {
        TextField("Name", text: $user.name) // Error!
    }
}

// ✅ CORRECT: Use @Bindable
struct EditView: View {
    @Bindable var user: User
    
    var body: some View {
        TextField("Name", text: $user.name)
    }
}
```


## Observable Objects Pattern

### Basic Observable Class

```swift
import SwiftData

@Observable
class DataManager {
    var items: [Item] = []
    private var modelContext: ModelContext
    
    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }
    
    func fetchItems() async throws {
        let descriptor = FetchDescriptor<Item>(
            sortBy: [SortDescriptor(\.name)]
        )
        items = try modelContext.fetch(descriptor)
    }
    
    func addItem(name: String) {
        let item = Item(name: name)
        modelContext.insert(item)
    }
    
    func deleteItem(_ item: Item) {
        modelContext.delete(item)
    }
    
    func updateItem(_ item: Item, name: String) {
        item.name = name
    }
}
```

### Using in SwiftUI

```swift
struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @State private var dataManager: DataManager?
    
    var body: some View {
        List(dataManager?.items ?? []) { item in
            Text(item.name)
        }
        .task {
            dataManager = DataManager(modelContext: modelContext)
            try? await dataManager?.fetchItems()
        }
        .toolbar {
            Button("Add", systemImage: "plus") {
                dataManager?.addItem(name: "New Item")
            }
        }
    }
}
```

### With Search and Filter

```swift
@Observable
class UserManager {
    var users: [User] = []
    var searchText = ""
    private var modelContext: ModelContext
    
    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }
    
    var filteredUsers: [User] {
        if searchText.isEmpty {
            return users
        }
        return users.filter { $0.name.localizedStandardContains(searchText) }
    }
    
    func fetchUsers() async throws {
        let descriptor = FetchDescriptor<User>(
            sortBy: [SortDescriptor(\.name)]
        )
        users = try modelContext.fetch(descriptor)
    }
}

struct UserListView: View {
    @State private var manager: UserManager?
    @Environment(\.modelContext) private var modelContext
    
    var body: some View {
        List(manager?.filteredUsers ?? []) { user in
            Text(user.name)
        }
        .searchable(text: Binding(
            get: { manager?.searchText ?? "" },
            set: { manager?.searchText = $0 }
        ))
        .task {
            manager = UserManager(modelContext: modelContext)
            try? await manager?.fetchUsers()
        }
    }
}
```

### Passing Models to Subviews

```swift
struct ItemListView: View {
    @State private var manager: DataManager?
    
    var body: some View {
        List(manager?.items ?? []) { item in
            NavigationLink(value: item) {
                Text(item.name)
            }
        }
        .navigationDestination(for: Item.self) { item in
            EditItemView(item: item)
        }
    }
}

struct EditItemView: View {
    @Bindable var item: Item
    
    var body: some View {
        Form {
            TextField("Name", text: $item.name)
        }
    }
}
```

### Background Operations

```swift
@Observable
class ImportManager {
    var progress: Double = 0
    var isImporting = false
    private var modelContext: ModelContext
    
    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }
    
    func importData(_ data: [ItemData]) async {
        isImporting = true
        let total = Double(data.count)
        
        for (index, itemData) in data.enumerated() {
            let item = Item(name: itemData.name)
            modelContext.insert(item)
            progress = Double(index + 1) / total
        }
        
        try? modelContext.save()
        isImporting = false
    }
}
```

### Error Handling

```swift
@Observable
class DataManager {
    var items: [Item] = []
    var errorMessage: String?
    var isLoading = false
    private var modelContext: ModelContext
    
    init(modelContext: ModelContext) {
        self.modelContext = modelContext
    }
    
    func fetchItems() async {
        isLoading = true
        errorMessage = nil
        
        do {
            let descriptor = FetchDescriptor<Item>()
            items = try modelContext.fetch(descriptor)
        } catch {
            errorMessage = error.localizedDescription
        }
        
        isLoading = false
    }
}

struct ContentView: View {
    @State private var manager: DataManager?
    
    var body: some View {
        Group {
            if manager?.isLoading == true {
                ProgressView()
            } else if let error = manager?.errorMessage {
                Text("Error: \(error)")
            } else {
                List(manager?.items ?? []) { item in
                    Text(item.name)
                }
            }
        }
        .task {
            manager = DataManager(modelContext: modelContext)
            await manager?.fetchItems()
        }
    }
}
```

### Multiple Managers

```swift
struct ContentView: View {
    @State private var userManager: UserManager?
    @State private var postManager: PostManager?
    @Environment(\.modelContext) private var modelContext
    
    var body: some View {
        TabView {
            UserListView(manager: userManager)
                .tabItem { Label("Users", systemImage: "person") }
            
            PostListView(manager: postManager)
                .tabItem { Label("Posts", systemImage: "doc") }
        }
        .task {
            userManager = UserManager(modelContext: modelContext)
            postManager = PostManager(modelContext: modelContext)
            
            try? await userManager?.fetchUsers()
            try? await postManager?.fetchPosts()
        }
    }
}
```

## When to Use Observable Objects

**Use Observable Objects when:**
- Complex business logic beyond simple CRUD
- Need to share state across multiple views
- Async operations with progress tracking
- Error handling and loading states
- Background processing
- Computed properties based on data

**Use @Query directly when:**
- Simple list/detail views
- Direct data display
- No complex filtering logic
- No async operations needed
