# <a href="https://developer.apple.com/videos/play/wwdc2021/10132/">Meet async/await in Swift</a>

## 1. Problems with Asynchronous Programming and Solutions

### Synchronous vs Asynchronous Functions
- **Synchronous functions**: Block the thread until the function completes
- **Asynchronous functions**: Quickly release the thread, allowing other work to be performed
- **Traditional async approaches**: Using completion handlers and delegate callbacks
- **New approach**: Write async code as easily as regular code with async/await

## 2. Evolution from Completion Handlers to Async/Await

### Traditional Completion Handler Approach (Many Problems)

```swift
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
    let request = thumbnailURLRequest(for: id)
    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            completion(nil, error)
        } else if (response as? HTTPURLResponse)?.statusCode != 200 {
            completion(nil, FetchError.badID)
        } else {
            guard let image = UIImage(data: data!) else {
                completion(nil, FetchError.badImage)
                return
            }
            image.prepareThumbnail(of: CGSize(width: 40, height: 40)) { thumbnail in
                guard let thumbnail = thumbnail else {
                    completion(nil, FetchError.badImage)
                    return
                }
                completion(thumbnail, nil)
            }
        }
    }
    task.resume()
}
```

**Problems:**
- 20 lines of complex code
- Easy to forget calling completion handlers (5 opportunities for subtle bugs)
- Cannot use Swift's usual error handling mechanism
- Nested structure reduces readability

### Using Result Type (Slight Improvement)

```swift
func fetchThumbnail(for id: String, completion: @escaping (Result<UIImage, Error>) -> Void) {
    let request = thumbnailURLRequest(for: id)
    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            completion(.failure(error))
        } else if (response as? HTTPURLResponse)?.statusCode != 200 {
            completion(.failure(FetchError.badID))
        } else {
            guard let image = UIImage(data: data!) else {
                completion(.failure(FetchError.badImage))
                return
            }
            image.prepareThumbnail(of: CGSize(width: 40, height: 40)) { thumbnail in
                guard let thumbnail = thumbnail else {
                    completion(.failure(FetchError.badImage))
                    return
                }
                completion(.success(thumbnail))
            }
        }
    }
    task.resume()
}
```

### Async/Await Approach (Revolutionary Improvement)

```swift
func fetchThumbnail(for id: String) async throws -> UIImage {
    let request = thumbnailURLRequest(for: id)  
    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else { throw FetchError.badID }
    let maybeImage = UIImage(data: data)
    guard let thumbnail = await maybeImage?.thumbnail else { throw FetchError.badImage }
    return thumbnail
}
```

**Improvements:**
- Reduced from 20 lines to 6 lines
- Straight-line code structure
- Can use Swift's standard error handling
- Always guarantees notification to caller (return or throw)

## 3. Async Properties and Sequences

### Defining Async Properties

```swift
extension UIImage {
    var thumbnail: UIImage? {
        get async {
            let size = CGSize(width: 40, height: 40)
            return await self.byPreparingThumbnail(ofSize: size)
        }
    }
}
```

**Important Rules:**
- Explicit getter required
- Read-only properties only
- When using async and throws together: `async throws` order

### Async Sequences

```swift
for await id in staticImageIDsURL.lines {
    let thumbnail = await fetchThumbnail(for: id)
    collage.add(thumbnail)
}
let result = await collage.draw()
```

## 4. Writing Test Code

### Traditional XCTestExpectation Approach

```swift
class MockViewModelSpec: XCTestCase {
    func testFetchThumbnails() throws {
        let expectation = XCTestExpectation(description: "mock thumbnails completion")
        self.mockViewModel.fetchThumbnail(for: mockID) { result, error in
            XCTAssertNil(error)
            expectation.fulfill()
        }
        wait(for: [expectation], timeout: 5.0)
    }
}
```

### Async/Await Testing Approach

```swift
class MockViewModelSpec: XCTestCase {
    func testFetchThumbnails() async throws {
        XCTAssertNoThrow(try await self.mockViewModel.fetchThumbnail(for: mockID))
    }
}
```

## 5. Bridging from Sync to Async

### Using Task in SwiftUI

```swift
struct ThumbnailView: View {
    @ObservedObject var viewModel: ViewModel
    var post: Post
    @State private var image: UIImage?

    var body: some View {
        Image(uiImage: self.image ?? placeholder)
            .onAppear {
                Task {
                    self.image = try? await self.viewModel.fetchThumbnail(for: post.id)
                }
            }
    }
}
```

## 6. Async APIs in the SDK

### ClockKit Example

```swift
import ClockKit

extension ComplicationController: CLKComplicationDataSource {
    func currentTimelineEntry(for complication: CLKComplication) async -> CLKComplicationTimelineEntry? {
        let date = Date()
        let thumbnail = try? await self.viewModel.fetchThumbnail(for: post.id)
        guard let thumbnail = thumbnail else {
            return nil
        }

        let entry = self.createTimelineEntry(for: thumbnail, date: date)
        return entry
    }
}
```

## 7. Converting Legacy Code with Continuations

### Existing Core Data Function

```swift
// Existing function
func getPersistentPosts(completion: @escaping ([Post], Error?) -> Void) {       
    do {
        let req = Post.fetchRequest()
        req.sortDescriptors = [NSSortDescriptor(key: "date", ascending: true)]
        let asyncRequest = NSAsynchronousFetchRequest<Post>(fetchRequest: req) { result in
            completion(result.finalResult ?? [], nil)
        }
        try self.managedObjectContext.execute(asyncRequest)
    } catch {
        completion([], error)
    }
}
```

### Async Alternative Function

```swift
// Async alternative
func persistentPosts() async throws -> [Post] {       
    typealias PostContinuation = CheckedContinuation<[Post], Error>
    return try await withCheckedThrowingContinuation { (continuation: PostContinuation) in
        self.getPersistentPosts { posts, error in
            if let error = error { 
                continuation.resume(throwing: error) 
            } else {
                continuation.resume(returning: posts)
            }
        }
    }
}
```

## 8. Using Continuations with Delegate Pattern

```swift
class ViewController: UIViewController {
    private var activeContinuation: CheckedContinuation<[Post], Error>?
    func sharedPostsFromPeer() async throws -> [Post] {
        try await withCheckedThrowingContinuation { continuation in
            self.activeContinuation = continuation
            self.peerManager.syncSharedPosts()
        }
    }
}

extension ViewController: PeerSyncDelegate {
    func peerManager(_ manager: PeerManager, received posts: [Post]) {
        self.activeContinuation?.resume(returning: posts)
        self.activeContinuation = nil // guard against multiple calls to resume
    }

    func peerManager(_ manager: PeerManager, hadError error: Error) {
        self.activeContinuation?.resume(throwing: error)
        self.activeContinuation = nil // guard against multiple calls to resume
    }
}
```

## 9. Key Concepts Summary

### Suspend and Resume Behavior
1. **Suspend**: Async function yields thread control to the system
2. **Other work execution**: System performs other work based on priority
3. **Resume**: System resumes the original function at an appropriate time

### Important Rules
- **Function signature**: `async throws` order (async comes before throws)
- **await keyword**: Indicates where function might suspend
- **try await**: When calling async functions that can throw errors
- **Continuation rules**: resume() must be called exactly once

### Safety Guarantees
- Swift automatically ensures completion notification (return or throw)
- Runtime detects duplicate continuation calls and triggers fatal errors
- await keyword marks points where state changes are possible

### Key Takeaways
- **Cooperative execution**: Async functions cooperatively yield control to allow other work
- **Thread efficiency**: Threads are not blocked while functions are suspended
- **State changes**: App state can change dramatically while functions are suspended
- **Resumption**: Functions may resume on entirely different threads

Async/await is a core Swift feature that makes complex, error-prone asynchronous code safe, concise, and intuitive to write and understand.
