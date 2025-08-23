# <a href="https://developer.apple.com/videos/play/wwdc2022/110350/">Visualize and optimize Swift concurrency</a>

## Overview
The focus is on understanding Swift Concurrency code and improving performance using the new concurrency instrument in Instruments 14.

## Swift Concurrency Components Recap

### Core Features
- **Async/await:** Basic syntactic building blocks that allow functions to suspend and resume without blocking threads
- **Tasks:** Basic units of work that execute concurrent code, manage state, handle cancellation, and control async code execution
- **Structured Concurrency:** Enables spawning child tasks in parallel with automatic waiting/cancellation
- **Actors:** Coordinate multiple tasks accessing shared data by isolating data and allowing only one task at a time to manipulate internal state

## New Instruments 14 Features

### Swift Concurrency Template
New instruments for capturing and visualizing concurrency activity:

#### Swift Tasks Instrument
**Top-level Statistics:**
- **Running Tasks:** Shows simultaneous task execution
- **Alive Tasks:** Number of tasks present at a given time
- **Total Tasks:** Cumulative task creation count

**Detail Views:**
- **Task Forest:** Graphical representation of parent-child task relationships
- **Task Summary:** Time spent in different task states
- **Pin to Timeline:** Right-click to pin tasks for detailed tracking

**Pinned Task Features:**
1. State tracking on timeline
2. Task creation backtrace in extended detail
3. Narrative view with context about task state
4. Related element pinning (child tasks, threads, actors)

#### Swift Actors Instrument
- Visualizes actor queues and contention
- Shows task waiting times for actor access

## Common Performance Problems

### 1. Main Actor Blocking
**Problem:** Long-running tasks on Main Actor cause UI freezes
- Main Actor executes all work on main thread
- UI work must be on main thread
- Long tasks make app unresponsive

**Solution:** Move computation to background actors or detached tasks

### 2. Actor Contention
**Problem:** Multiple tasks waiting for same actor serializes execution
- Actors serialize access to shared state
- Only one task at a time can occupy an actor
- Loses parallel computation benefits

**Solution:** Minimize time spent on actors
- Run only actor-isolated code on actors
- Move non-isolated work to detached tasks
- Divide tasks into actor and non-actor chunks

### 3. Thread Pool Exhaustion
**Problem:** Blocked tasks prevent full CPU utilization
- Tasks must make forward progress
- Blocking calls (file/network IO, locks) occupy threads without using CPU
- Can cause deadlock if entire thread pool is blocked

**Solutions:**
- Avoid blocking calls in tasks
- Use async APIs for file/network IO
- Avoid condition variables/semaphores
- Use fine-grained, briefly-held locks only
- Move blocking code to Dispatch queues with continuation bridges

### 4. Continuation Misuse
**Problem:** Incorrect continuation usage causes leaks or crashes
- Continuations bridge Swift concurrency with callback-based APIs
- **Critical Rule:** Must be called exactly once

**Common Bugs:**
- Never calling continuation = task leak
- Calling twice = crash or misbehavior

**Best Practice:** Always use `withCheckedContinuation` instead of the unsafe version
- Automatically detects misuse
- Traps on double-call
- Prints warning on leak

## Demo Case Study: File Squeezer App

### Initial Problem: UI Hang
**Issue:** Large file compression freezes UI

**Investigation:**
- Running Tasks shows only 1 task executing (serialization)
- Task runs on Main Thread for extended time
- `CompressionState` class marked with `@MainActor`

### Code Evolution

#### Stage 1: Original Problem Code
**Problem:** The entire `CompressionState` class is marked with `@MainActor`, forcing all compression work to run on the main thread. This blocks the UI during heavy file compression operations.

```swift
@MainActor
class CompressionState: ObservableObject {
    @Published var files: [FileStatus] = []
    var logs: [String] = []
    
    func compressAllFiles() {
        for file in files {
            Task {
                let compressedData = compressFile(url: file.url)
                await save(compressedData, to: file.url)
            }
        }
    }
    
    func compressFile(url: URL) -> Data {
        // Heavy compression work on Main Actor
    }
}
```

#### Stage 2: Move to Separate Actor
**Improvement:** Created `ParallelCompressor` actor to move compression work off the main thread. However, the entire `compressFile` function still runs on the actor, causing all compression tasks to serialize as they wait for exclusive actor access.

Created `ParallelCompressor` actor to handle compression:
- Separated UI state (needs MainActor) from logs (needs protection but not MainActor)
- Tasks hop between actors as needed
- UI no longer hangs but still serialized

```swift
actor ParallelCompressor {
    var logs: [String] = []
    
    func compressFile(url: URL) -> Data {
        // Compression work now on separate actor
        // But entire function runs on actor, serializing work
    }
}
```

#### Stage 3: Minimize Actor Isolation
**Solution:** Mark `compressFile` as `nonisolated` and use detached tasks. The function only accesses the actor when updating logs, allowing multiple compression operations to run in parallel on the thread pool.

Final optimization with `nonisolated` and detached tasks:

```swift
actor ParallelCompressor {
    var logs: [String] = []
    
    nonisolated func compressFile(url: URL) async -> Data {
        await log(update: "Starting")  // Only on actor when needed
        let compressedData = CompressionUtils.compressDataInFile(...)
        await log(update: "Ending")     // Only on actor when needed
        return compressedData
    }
}

@MainActor
class CompressionState: ObservableObject {
    func compressAllFiles() {
        for file in files {
            Task.detached {  // Detached to avoid inheriting actor context
                let compressedData = await self.compressor.compressFile(url: file.url)
                await save(compressedData, to: file.url)
            }
        }
    }
}
```

**Result:** 
- Compression runs in parallel on thread pool
- Actors accessed only when updating protected state
- Full CPU utilization achieved
- UI remains responsive

## Key Takeaways

1. **Use Instruments** to visualize and diagnose concurrency issues
2. **Keep Main Actor work minimal** - move heavy computation to background
3. **Minimize actor occupation time** - use `nonisolated` and detached tasks
4. **Avoid blocking operations** in tasks - use async APIs
5. **Use checked continuations** and ensure exactly-once calling
6. **Structure for parallelism** - let tasks run freely when actor isolation isn't needed

## Related Topics Mentioned
- "Swift Concurrency: Behind the Scenes" - Understanding runtime internals
- "Eliminate data races using Swift Concurrency" - Data race prevention
