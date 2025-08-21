# <a href="https://developer.apple.com/videos/play/wwdc2021/10095/">URLSession with Swift Concurrency</a>

## Introduction

Swift concurrency makes networking code linear, concise, and supports native Swift error handling. In iOS 15 and macOS Monterey, URLSession introduced new APIs to take advantage of Swift concurrency features.

## Problems with Completion Handler Approach

The traditional completion handler approach has several issues:

- **Complex control flow**: Jumping back and forth between different execution contexts
- **Threading complexity**: Three different execution contexts (caller thread, delegate queue, main queue)
- **Error-prone**: Missing early returns, inconsistent dispatching, and unhandled nil cases
- **Compiler can't help**: No compile-time protection against threading issues or data races

## Benefits of async/await

- **Linear control flow**: Code runs from top to bottom
- **Single concurrency context**: No threading issues to worry about
- **Native error handling**: Use Swift's throw/catch mechanisms
- **Compiler enforcement**: Forces proper handling of optionals and errors

## Basic Data Fetching

### Fetch Photo with async/await

```swift
// Fetch photo with async/await

func fetchPhoto(url: URL) async throws -> UIImage
{
    let (data, response) = try await URLSession.shared.data(from: url)

    guard let httpResponse = response as? HTTPURLResponse,
          httpResponse.statusCode == 200 else {
        throw WoofError.invalidServerResponse
    }

    guard let image = UIImage(data: data) else {
        throw WoofError.unsupportedImage
    }

    return image
}
```

## Core URLSession Async Methods

### Data Methods

URLSession.data methods accept either a URL or URLRequest and are equivalent to existing data task convenience methods.

```swift
let (data, response) = try await URLSession.shared.data(from: url)
guard let httpResponse = response as? HTTPURLResponse,
      httpResponse.statusCode == 200 /* OK */ else {
    throw MyNetworkingError.invalidServerResponse
}
```

### Upload Methods

Upload methods allow uploading data or files. Be sure to set the correct HTTP method since GET (default) doesn't support upload.

```swift
var request = URLRequest(url: url)
request.httpMethod = "POST"

let (data, response) = try await URLSession.shared.upload(for: request, fromFile: fileURL)
guard let httpResponse = response as? HTTPURLResponse,
      httpResponse.statusCode == 201 /* Created */ else {
    throw MyNetworkingError.invalidServerResponse
}
```

### Download Methods

Download methods store the response body as a file rather than in memory. Unlike download task convenience methods, these new methods don't automatically delete the file.

```swift
let (location, response) = try await URLSession.shared.download(from: url)
guard let httpResponse = response as? HTTPURLResponse,
      httpResponse.statusCode == 200 /* OK */ else {
    throw MyNetworkingError.invalidServerResponse
}

try FileManager.default.moveItem(at: location, to: newLocation)
```

## Cancellation

Swift concurrency's cancellation works with URLSession async methods using Task.Handle. Note that concurrency Task is unrelated to URLSessionTask.

```swift
let task = Task {
    let (data1, response1) = try await URLSession.shared.data(from: url1)

    let (data2, response2) = try await URLSession.shared.data(from: url2)

}

task.cancel()
```

## Streaming Data with AsyncSequence

### URLSession.bytes Methods

For incremental response body processing, URLSession.bytes methods return when response headers are received and deliver the response body as an AsyncSequence of bytes.

```swift
let (bytes, response) = try await URLSession.shared.bytes(from: Self.eventStreamURL)
guard let httpResponse = response as? HTTPURLResponse, httpResponse.statusCode == 200 else {
    throw WoofError.invalidServerResponse
}
for try await line in bytes.lines {
    let photoMetadata = try JSONDecoder().decode(PhotoMetadata.self, from: Data(line.utf8))
    await updateFavoriteCount(with: photoMetadata)
}
```

### Real-time Processing

The bytes method returns `URLSession.AsyncBytes` which provides incremental consumption of response body. Use the `lines` method to consume response line by line as data is received. UI updates must happen on the main actor using await syntax.

## Task-Specific Delegates

### Authentication Handling

Since async methods don't expose the underlying task, URLSession provides task-specific delegates for handling authentication challenges, metrics, and other events specific to individual requests.

```swift
class AuthenticationDelegate: NSObject, URLSessionTaskDelegate {
    private let signInController: SignInController
    
    init(signInController: SignInController) {
        self.signInController = signInController
    }
    
    func urlSession(_ session: URLSession,
                    task: URLSessionTask,
                    didReceive challenge: URLAuthenticationChallenge) async
    -> (URLSession.AuthChallengeDisposition, URLCredential?) {
        if challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodHTTPBasic {
            do {
                let (username, password) = try await signInController.promptForCredential()
                return (.useCredential,
                        URLCredential(user: username, password: password, persistence: .forSession))
            } catch {
                return (.cancelAuthenticationChallenge, nil)
            }
        } else {
            return (.performDefaultHandling, nil)
        }
    }
}
```

### Usage

Pass the delegate as an additional parameter to URLSession methods:

```swift
let authDelegate = AuthenticationDelegate(signInController: signInController)
let (data, response) = try await URLSession.shared.data(from: url, delegate: authDelegate)
```

### Key Points

- The delegate is strongly held by the task until it completes or fails
- Task-specific delegate is not supported by background URLSession  
- If a method is implemented on both session delegate and task delegate, the task delegate takes precedence
- Also available in Objective-C via the delegate property on NSURLSessionTask

## Migration Guidelines

To adopt async/await with URLSession:

1. Change functions taking completion handlers to async functions
2. Change repeated event handlers to AsyncSequences
3. Use task-specific delegates for request-specific logic
4. Leverage Swift's native error handling with throw/catch
