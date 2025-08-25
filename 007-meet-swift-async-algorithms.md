# <a href="https://developer.apple.com/videos/play/wwdc2022/110355/">Swift Async Algorithms Package Summary</a>

## AsyncSequence Recap

AsyncSequence is a protocol for describing values produced asynchronously. It's similar to Sequence but with two key differences:
- The `next` function from its iterator is asynchronous, using Swift concurrency
- It handles potential failures using Swift's throw effect
- Uses `for-await-in` syntax for iteration

The package augments Swift concurrency with advanced AsyncSequence algorithms that interoperate with clocks.

## Messaging App Example

The presentation uses a messaging app migration to Swift concurrency to demonstrate the package's capabilities, highlighting several key algorithms:

### Combining Multiple AsyncSequences

These algorithms take multiple input AsyncSequences and produce one output AsyncSequence.

#### Zip Algorithm
**Purpose**: Takes multiple inputs and produces tuples of results from each base
**Key Features**:
- Iterates each input concurrently
- Rethrows errors if any base fails
- Awaits both sides to construct complete tuples

**Code Example**:
```swift
// Upload attachments of videos and previews concurrently
for try await (vid, preview) in zip(videos, previews) {
  try await upload(vid, preview)
}
```

**Use Case**: Coordinating video transcoding and preview generation, ensuring each video gets a preview when uploaded to the server.

#### Merge Algorithm
**Purpose**: Combines multiple AsyncSequences with the same element type into one singular AsyncSequence
**Key Features**:
- Concurrently iterates multiple AsyncSequences
- Takes the first element produced by any side
- Continues until all base AsyncSequences return nil
- Cancels other iterations if any base produces an error

**Code Example**:
```swift
struct Account {
  var messages: AsyncStream<Message>
}  

actor AccountManager {
  var primaryAccount: Account
  var secondaryAccount: Account? 
}

// Display previews of messages from either account
for try await message in merge(primaryAccount.messages, secondaryAccount.messages) {
  displayPreview(message)
}
```

**Use Case**: Supporting multiple accounts by merging their message streams into one AsyncSequence.

### Time-Based Algorithms

The package leverages Swift's new Clock API (Clock, Instant, and Duration) to work with time safely and consistently.

#### Clock Types
- **ContinuousClock**: Measures time like a stopwatch, progresses regardless of machine state
- **SuspendingClock**: Suspends when the machine sleeps

**Code Examples**:
```swift
// Sleep until a deadline
let clock = SuspendingClock()
var deadline = clock.now + .seconds(3)
try await clock.sleep(until: deadline)

// Measuring elapsed time - different results based on clock type
let clock = SuspendingClock()
let elapsed = await clock.measure {
  await someLongRunningWork()
}
//Elapsed time reads 00:05.40

let clock = ContinuousClock()
let elapsed = await clock.measure {
  await someLongRunningWork()
}
//Elapsed time reads 00:19.54
```

**Usage Guidelines**:
- Use SuspendingClock for animations and machine-relative timing
- Use ContinuousClock for human-relative timing and absolute duration delays

#### Debounce Algorithm
**Purpose**: Awaits a quiescence period before emitting values
**Key Features**:
- Waits for quiet periods when rapid events occur
- Uses ContinuousClock by default
- Rate limits interactions

**Code Example**:
```swift
class SearchController {
  let searchResults = AsyncChannel<SearchResult>()

  func search<SearchValues: AsyncSequence>(_ searchValues: SearchValues) 
    where SearchValues.Element == String 
}

let queries = searchValues
   .debounce(for: .milliseconds(300))

for await query in queries {
  let results = try await performSearch(query)
  await channel.send(results)
}
```

**Use Case**: Search functionality that waits for users to stop typing before executing searches.

#### Chunking Algorithm
**Purpose**: Groups values by count, time, or content
**Key Features**:
- Interoperates with clocks and durations
- Rethrows errors for safe failure handling
- Controls chunk processing by various criteria

**Code Example**:
```swift
let batches = outboundMessages.chunked(
  by: .repeating(every: .milliseconds(500))
)

let encoder = JSONEncoder() 
for await batch in batches {
  let data = try encoder.encode(batch)
  try await postToServer(data) 
}
```

**Use Case**: Efficiently batching messages to the server every 500 milliseconds, grouping rapid user input.

### Collection Conversion

The package provides initializers for constructing collections from finite AsyncSequences.

**Code Example**:
```swift
// Create a message with awaiting attachments to be encoded
init<Attachments: AsyncSequence>(_ attachments: Attachments) async rethrows {
  self.attachments = try await Array(attachments)
}
```

**Benefits**: Enables incremental migration to Swift concurrency while maintaining existing data structures.

## Additional Algorithms

The package includes many more algorithms beyond those highlighted:
- Buffering
- Reducing  
- Joining
- Injecting values intermittently
- Rate limiting
- And more

## Conclusion

The Swift Async Algorithms package expands the toolkit for handling time-based processing with AsyncSequence, providing advanced functionality for building robust concurrent applications. The package is developed openly with community participation.
