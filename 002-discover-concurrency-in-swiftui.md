# <a href="https://developer.apple.com/videos/play/wwdc2021/10019/">Discover Concurrency in SwiftUI</a>

## Overview
This session demonstrates how Swift 5.5's new concurrency features integrate with SwiftUI to build better data models and user interfaces. The example builds a space photo app that fetches images from a REST API.

## Key Concepts

### The Main Actor and SwiftUI Run Loop
- SwiftUI's run loop runs on the main actor in Swift 5.5
- The run loop receives user events, updates models, and renders views in "ticks"
- All ObservableObject updates must happen on the main actor to ensure proper view updates

### Common Concurrency Issues
**Problem**: Blocking the main actor with long-running operations causes UI hitches
```swift
// Bad: Blocks main actor
func updateItems() {
    let fetched = fetchPhotos() // Blocks here
    items = fetched
}
```

**Problem**: Updating ObservableObject from background threads can cause race conditions where SwiftUI misses state changes

**Solution**: Use `await` to yield the main actor during async operations
```swift
// Good: Yields main actor during async work
func updateItems() async {
    let fetched = await fetchPhotos() // Yields here
    items = fetched // Back on main actor
}
```

## Implementation

### 1. Data Model - SpacePhoto
```swift
/// A SpacePhoto contains information about a single day's photo record
/// including its date, a title, description, etc.
struct SpacePhoto {
    /// The title of the astronomical photo.
    var title: String

    /// A description of the astronomical photo.
    var description: String

    /// The date the given entry was added to the catalog.
    var date: Date

    /// A link to the image contained within the entry.
    var url: URL
}


extension SpacePhoto: Codable {
    enum CodingKeys: String, CodingKey {
        case title
        case description = "explanation"
        case date
        case url
    }

    init(data: Data) throws {
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy =
            .formatted(SpacePhoto.dateFormatter)

        self = try JSONDecoder()
            .decode(SpacePhoto.self, from: data)
    }
}

extension SpacePhoto: Identifiable {
    var id: Date { date }
}

extension SpacePhoto {
    static let urlTemplate = "https://example.com/photos"
    static let dateFormat = "yyyy-MM-dd"

    static var dateFormatter: DateFormatter {
        let formatter = DateFormatter()
        formatter.dateFormat = Self.dateFormat
        return formatter
    }

    static func requestFor(date: Date) -> URL {
        let dateString = SpacePhoto.dateFormatter.string(from: date)
        return URL(string: "\(SpacePhoto.urlTemplate)&date=\(dateString)")!
    }

    private static func parseDate(
        fromContainer container: KeyedDecodingContainer<CodingKeys>
    ) throws -> Date {
        let dateString = try container.decode(String.self, forKey: .date)
        guard let result = dateFormatter.date(from: dateString) else {
            throw DecodingError.dataCorruptedError(
                forKey: .date,
                in: container,
                debugDescription: "Invalid date format")
        }
        return result
    }

    private var dateString: String {
        Self.dateFormatter.string(from: date)
    }
}

extension SpacePhoto {
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        title = try container.decode(String.self, forKey: .title)
        description = try container.decode(String.self, forKey: .description)
        date = try Self.parseDate(fromContainer: container)
        url = try container.decode(URL.self, forKey: .url)
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(title, forKey: .title)
        try container.encode(description, forKey: .description)
        try container.encode(dateString, forKey: .date)
    }
}
```

### 2. Observable Object with @MainActor
```swift
/// An observable object representing a random list of space photos.
@MainActor
class Photos: ObservableObject {
    @Published private(set) var items: [SpacePhoto] = []

    /// Updates `items` to a new, random list of `SpacePhoto`.
    func updateItems() async {
        let fetched = await fetchPhotos()
        items = fetched
    }

    /// Fetches a new, random list of `SpacePhoto`.
    func fetchPhotos() async -> [SpacePhoto] {
        var downloaded: [SpacePhoto] = []
        for date in randomPhotoDates() {
            let url = SpacePhoto.requestFor(date: date)
            if let photo = await fetchPhoto(from: url) {
                downloaded.append(photo)
            }
        }
        return downloaded
    }

    /// Fetches a `SpacePhoto` from the given `URL`.
    func fetchPhoto(from url: URL) async -> SpacePhoto? {
        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            return try SpacePhoto(data: data)
        } catch {
            return nil
        }
    }
}
```

**Key Points:**
- `@MainActor` annotation ensures all properties and methods are accessed from the main actor
- `await` yields the main actor during network calls
- Updates to `@Published` properties happen safely on the main actor

### 3. SwiftUI Views with New Concurrency APIs

#### CatalogView with task modifier
```swift
struct CatalogView: View {
    @StateObject private var photos = Photos()

    var body: some View {
        NavigationView {
            List {
                ForEach(photos.items) { item in
                    PhotoView(photo: item)
                        .listRowSeparator(.hidden)
                }
            }
            .navigationTitle("Catalog")
            .listStyle(.plain)
            .refreshable {
                await photos.updateItems()
            }
        }
        .task {
            await photos.updateItems()
        }
    }
}
```

#### PhotoView with AsyncImage
```swift
struct PhotoView: View {
    var photo: SpacePhoto

    var body: some View {
        ZStack(alignment: .bottom) {
            AsyncImage(url: photo.url) { image in
                image
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } placeholder: {
                ProgressView()
            }
            .frame(minWidth: 0, minHeight: 400)

            HStack {
                Text(photo.title)
                Spacer()
                SavePhotoButton(photo: photo)
            }
            .padding()
            .background(.thinMaterial)
        }
        .background(.thickMaterial)
        .mask(RoundedRectangle(cornerRadius: 16))
        .padding(.bottom, 8)
    }
}
```

#### SavePhotoButton with Task
```swift
struct SavePhotoButton: View {
    var photo: SpacePhoto
    @State private var isSaving = false

    var body: some View {
        Button {
            Task {
                isSaving = true
                await photo.save()
                isSaving = false
            }
        } label: {
            Text("Save")
                .opacity(isSaving ? 0 : 1)
                .overlay {
                    if isSaving {
                        ProgressView()
                    }
                }
        }
        .disabled(isSaving)
        .buttonStyle(.bordered)
    }
}
```

## New SwiftUI APIs

### 1. `.task` modifier
- Replaces `onAppear` for async operations
- Automatically ties task lifetime to view lifetime
- Task is canceled when view disappears
- Supports async sequences and long-running operations

### 2. `AsyncImage`
- Loads images from remote URLs concurrently
- Built-in placeholder and error handling
- Intelligent defaults with customizable behavior

### 3. `.refreshable` modifier
- Enables pull-to-refresh functionality
- Takes an async closure for refresh operations
- Automatically dismisses indicator when async work completes

## Best Practices

1. **Use `@MainActor` on ObservableObjects** for robust state management
2. **Replace `onAppear` with `.task`** for async operations
3. **Use `await` instead of dispatching to background queues** - it yields the main actor safely
4. **Leverage SwiftUI's new concurrent APIs** like AsyncImage and refreshable
5. **Use `Task` for async operations in synchronous contexts** like button actions

## Benefits
- **Safer concurrency**: Main actor guarantees prevent race conditions
- **Better performance**: Async/await yields the main actor instead of blocking
- **Simpler code**: Built-in SwiftUI APIs handle common concurrent patterns
- **Automatic lifecycle management**: Tasks are tied to view lifetimes
