# <a href="https://developer.apple.com/videos/play/wwdc2021/10058/">Swift AsyncSequence Guide</a>

## Introduction

AsyncSequence is a powerful new feature in Swift that allows you to work with sequences of values that are delivered asynchronously over time. It's designed to be familiar to developers who already understand regular Sequence, but with the added capability of handling asynchronous operations.

## What is AsyncSequence?

AsyncSequence is essentially like a regular sequence, but with key differences:
- Each element is delivered asynchronously
- Elements are produced over time, not all at once
- They can handle failure scenarios
- They participate in Swift's concurrency system

AsyncSequence represents zero or more values delivered over time, completing when the iterator returns nil or when an error occurs.

## Basic Usage

### Simple Earthquake Data Tool

Here's a practical example that demonstrates AsyncSequence fundamentals:

```swift
@main
struct QuakesTool {
    static func main() async throws {
        let endpointURL = URL(string: "https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_month.csv")!

        // skip the header line and iterate each one 
        // to extract the magnitude, time, latitude and longitude
        for try await event in endpointURL.lines.dropFirst() {
            let values = event.split(separator: ",")
            let time = values[0]
            let latitude = values[1]
            let longitude = values[2]
            let magnitude = values[4]
            print("Magnitude \(magnitude) on \(time) at \(latitude) \(longitude)")
        }
    }
}
```

This tool downloads earthquake data from a USGS endpoint and processes each line as it arrives. The `for try await` syntax allows processing CSV data line-by-line without waiting for the entire file to download, making the application feel responsive even with large datasets.

## How Iteration Works

### Regular Sequence Iteration

```swift
for quake in quakes {
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

This is the familiar synchronous for-in loop that processes all earthquakes in a sequence. Each iteration happens immediately one after another without any waiting or suspension.

### Behind the Scenes - Regular Iterator

```swift
var iterator = quakes.makeIterator()
while let quake = iterator.next() {
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

This shows what the compiler actually generates for a for-in loop. It creates an iterator and repeatedly calls `next()` until no more elements are available (nil is returned).

### Behind the Scenes - Async Iterator

```swift
var iterator = quakes.makeAsyncIterator()
while let quake = await iterator.next() {
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

The async version creates an async iterator and uses `await` to suspend execution while waiting for each element. The function can yield control to other tasks between iterations.

### AsyncSequence Iteration

```swift
for await quake in quakes {
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

The `for await` syntax provides a clean, familiar way to iterate over async sequences. It handles the async iterator creation and awaiting automatically while maintaining the same readable loop structure.

## Control Flow

### Breaking Early

```swift
for await quake in quakes {
    if quake.location == nil {
        break
    }
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

Standard control flow works with async sequences - here we terminate the loop early when encountering a quake without location data. The `break` statement works exactly like in regular loops.

### Skipping Values

```swift
for await quake in quakes {
    if quake.depth > 5 {
        continue
    }
    if quake.magnitude > 3 {
        displaySignificantEarthquake(quake)
    }
}
```

The `continue` statement skips to the next iteration when certain conditions are met. This example skips processing earthquakes with depth greater than 5, moving immediately to await the next quake.

### Error Handling

```swift
do {
    for try await quake in quakeDownload {
        ...
    }
} catch {
    ...
}
```

When an AsyncSequence can throw errors, you must use `try await` and wrap it in a do-catch block. The compiler enforces this safety, ensuring you handle potential failures during iteration.

## Concurrent Iteration

You can run iterations concurrently using Tasks:

```swift
let iteration1 = Task {
    for await quake in quakes {
        ...
    }
}

let iteration2 = Task {
    do {
        for try await quake in quakeDownload {
            ...
        }
    } catch {
        ...
    }
}

//... later on  
iteration1.cancel()
iteration2.cancel()
```

Tasks allow multiple async sequences to be processed concurrently rather than sequentially. This is especially useful for long-running or potentially infinite sequences, and provides cancellation capabilities for cleanup.

## Built-in AsyncSequence APIs

### File Operations

Reading bytes from a FileHandle:

```swift
for try await line in FileHandle.standardInput.bytes.lines {
    ...
}
```

This reads from standard input byte-by-byte and converts the bytes to lines asynchronously. It's perfect for processing large files or streaming input without loading everything into memory at once.

Reading lines from a URL:

```swift
let url = URL(fileURLWithPath: "/tmp/somefile.txt")
for try await line in url.lines {
    ...
}
```

URLs now have a convenient `.lines` property that returns an AsyncSequence of lines. This works for both local files and network resources, handling the streaming automatically.

### Network Operations

Reading bytes from URLSession:

```swift
let (bytes, response) = try await URLSession.shared.bytes(from: url)

guard let httpResponse = response as? HTTPURLResponse,
      httpResponse.statusCode == 200 /* OK */
else {
    throw MyNetworkingError.invalidServerResponse
}

for try await byte in bytes {
    ...
}
```

URLSession's new `bytes` method returns an AsyncSequence of bytes along with the response metadata. This allows you to validate the HTTP response before processing and handle large downloads efficiently by processing data as it arrives.

### Notifications

Working with NotificationCenter:

```swift
let center = NotificationCenter.default
let notification = await center.notifications(named: .NSPersistentStoreRemoteChange).first {
    $0.userInfo[NSStoreUUIDKey] == storeUUID
}
```

NotificationCenter now provides an AsyncSequence of notifications, allowing you to await specific notifications with conditions. The `first` method with a closure lets you wait for the first notification that matches your criteria.

## Creating Your Own AsyncSequence

### Using AsyncStream

AsyncStream is perfect for adapting existing callback-based APIs:

```swift
class QuakeMonitor {
    var quakeHandler: (Quake) -> Void
    func startMonitoring()
    func stopMonitoring()
}

let quakes = AsyncStream(Quake.self) { continuation in
    let monitor = QuakeMonitor()
    monitor.quakeHandler = { quake in
        continuation.yield(quake)
    }
    continuation.onTermination = { @Sendable _ in
        monitor.stopMonitoring()
    }
    monitor.startMonitoring()
}

let significantQuakes = quakes.filter { quake in
    quake.magnitude > 3
}

for await quake in significantQuakes {
    ...
}
```

This example transforms a callback-based monitoring class into an AsyncSequence using AsyncStream. The continuation's `yield` method sends values to the sequence, while `onTermination` handles cleanup when the sequence is cancelled or completes.

## Key Features

### Familiar Syntax
If you know how to use Sequence, you already know how to use AsyncSequence. The same methods like `map`, `filter`, `reduce`, and `dropFirst` are available.

### Safety
The compiler ensures proper error handling when working with throwing AsyncSequences, just like with throwing functions.

### Concurrency Integration
AsyncSequence integrates seamlessly with Swift's concurrency system, supporting cancellation and Task-based execution.

### Buffering and Resource Management
AsyncStream handles buffering and resource cleanup automatically, making it safe and efficient to use.

## When to Use AsyncSequence

AsyncSequence is ideal for:
- Processing data that arrives over time
- Handling network streams
- File I/O operations
- Event-driven architectures
- Converting callback-based APIs to async/await style
- Any scenario where you have multiple values delivered asynchronously

## Availability

AsyncSequence is available in:
- macOS Monterey
- iOS 15
- tvOS 15
- watchOS 8
