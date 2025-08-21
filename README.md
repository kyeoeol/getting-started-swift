```swift
let hello = "Hello, world!"
let multilineString = """
                  @@@
      @@          @@@@
    @@  @@@         @@@@@
    @@@@@@@@@       @@@@@
      @@@@@@@@@@    @@@@@@
        @@@@@@@@@@  @@@@@@
          @@@@@@@@@@@@@@@@@
@           @@@@@@@@@@@@@@@
@@@@@@        @@@@@@@@@@@@@
  @@@@@@@@@@@@@@@@@@@@@@@@@@
    @@@@@@@@@@@@@@@@@@@@@@@@
        @@@@@@@@@@@@@     @
                        \(hello)
"""
```

# Swift Programming Language
This repository serves as personal learning space for Swift, where I document knowledge gained through study, store code examples used in real projects, and organize implementation tips and pitfalls encountered along the way. I also collect helpful articles and save various practice codes to build Swift learning archive.

1. <a href="https://github.com/kyeoeol/getting-started-swift-concurrency/blob/main/001-meet-async-await-in-swift.md">Meet async/await in Swift</a>
2. <a href="https://github.com/kyeoeol/getting-started-swift-concurrency/blob/main/002-discover-concurrency-in-swiftui.md">Discover concurrency in SwiftUI</a>
3. <a href="https://github.com/kyeoeol/swift-programming-language/blob/main/003-use-async-await-with-urlsession.md">Use async/await with URLSession</a>

<br>

## 📝 Articles
*Collection of helpful articles and resources related to Swift.*

- <a href="https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/">The Swift Programming Language: Concurrency</a> - Concurrency, perform asynchronous operations.
- <a href="Get started with Swift concurrency">Get started with Swift concurrency</a> - Discover how you can started with Swift concurrency, debug your tasks and actors, explore the latest tools, and much more.

<br>

## 💻 Codes
*Swift code examples that can be directly used in projects.*

#### 1. Defining Async Properties
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

#### 2. Using Continuations with Delegate Pattern
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
