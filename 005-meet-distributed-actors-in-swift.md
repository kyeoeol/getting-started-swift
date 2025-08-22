# <a href="https://developer.apple.com/videos/play/wwdc2022/110356/?time=106">Meet Distributed Actors in Swift - Complete Guide</a>

## Introduction

This session introduces distributed actors in Swift. The session demonstrates how to extend Swift concurrency-based apps beyond a single process using the example of "Tic Tac Fish" - a tic-tac-toe-style game where players select teams (represented by emojis) and play against each other.

**Prerequisites:** Familiarity with Swift actors is recommended. Watch "Protect mutable state with Swift actors" from WWDC 2021 if needed.

## The Tic Tac Fish Game

The demo app is a tic-tac-toe game with the following features:
- Players select teams (fish vs rodents) represented by different emojis
- Currently has offline mode (player vs bot)
- Goal: Add multiplayer modes using distributed actors
- Uses actors to manage concurrency and model players

## Conceptual Foundation: "Sea of Concurrency"

### Local Actors Model
- Each actor is an "island in the sea of concurrency"
- Actors exchange messages instead of accessing each other's state directly
- Swift implements message passing as asynchronous method calls with async/await
- Actor state isolation + compiler checks = guaranteed freedom from low-level data races

### Distributed Systems Model
- Each device/node/process is an independent "sea of concurrency"
- Same message-passing concept applies but with additional restrictions
- Distributed actors establish channels between processes
- **Location Transparency**: Interact with distributed actors the same way regardless of their location

## Converting Local Actors to Distributed Actors

### Original Local Actor Implementation

The session starts with two existing actors:

**OfflinePlayer Actor:**
```swift
public actor OfflinePlayer: Identifiable {
    nonisolated public let id: ActorIdentity = .random

    let team: CharacterTeam
    let model: GameViewModel
    var movesMade: Int = 0

    public init(team: CharacterTeam, model: GameViewModel) {
        self.team = team
        self.model = model
    }

    public func makeMove(at position: Int) async throws -> GameMove {
        let move = GameMove(
            playerID: id,
            position: position,
            team: team,
            teamCharacterID: team.characterID(for: movesMade))
        await model.userMadeMove(move: move)

        movesMade += 1 
        return move
    }

    public func opponentMoved(_ move: GameMove) async throws {
        do {
            try await model.markOpponentMove(move)
        } catch {
            log("player", "Opponent made illegal move! \(move)")
        }
    }

}
```

**BotPlayer Actor (simpler implementation):**
```swift
public actor BotPlayer: Identifiable {
    nonisolated public let id: ActorIdentity = .random
    
    var ai: RandomPlayerBotAI
    var gameState: GameState
    
    public init(team: CharacterTeam) {
        self.gameState = .init()
        self.ai = RandomPlayerBotAI(playerID: self.id, team: team)
    }
    
    public func makeMove() throws -> GameMove {
        return try ai.decideNextMove(given: &gameState)
    }
    
    public func opponentMoved(_ move: GameMove) async throws {
        try gameState.mark(move)
    }
}
```

### Step-by-Step Conversion Process

The session demonstrates converting BotPlayer to a distributed actor:

**Step 1: Import and Add Keywords**
- Import the `Distributed` module (introduced in Swift 5.7)
- Add `distributed` keyword before `actor`
- This automatically conforms the actor to `DistributedActor` protocol

**Step 2: Configure Actor System**
This example shows how to convert a regular actor to a distributed actor by adding the `distributed` keyword and configuring the actor system:

```swift
import Distributed 

public distributed actor BotPlayer: Identifiable {
    typealias ActorSystem = LocalTestingDistributedActorSystem
  
    var ai: RandomPlayerBotAI
    var gameState: GameState
    
    public init(team: CharacterTeam, actorSystem: ActorSystem) {
        self.actorSystem = actorSystem // first, initialize the implicitly synthesized actor system property
        self.gameState = .init()
        self.ai = RandomPlayerBotAI(playerID: self.id, team: team) // use the synthesized `id` property
    }
    
    public distributed func makeMove() throws -> GameMove {
        return try ai.decideNextMove(given: &gameState)
    }
    
    public distributed func opponentMoved(_ move: GameMove) async throws {
        try gameState.mark(move)
    }
}
```

### Key Changes Required

1. **Actor System Declaration**: Define which actor system to use (handles serialization and networking)
2. **Remove Manual ID**: The `id` property is automatically synthesized by the distributed actor system
3. **Initialize Actor System**: Accept and initialize the `actorSystem` property in the initializer
4. **Mark Distributed Methods**: Add `distributed` keyword to methods that can be called remotely
5. **Serialization Requirements**: Ensure all parameters and return types conform to `Codable`

## Client/Server Architecture

### Remote Actor Resolution

This demonstrates how to obtain a reference to a potentially remote actor using its ID and actor system:

```swift
let sampleSystem: SampleWebSocketActorSystem

let opponentID: BotPlayer.ID = .randomID(opponentFor: self.id)
let bot = try BotPlayer.resolve(id: opponentID, using: sampleSystem) // resolve potentially remote bot player
```

**Important Notes:**
- `resolve()` is synchronous and doesn't perform network operations
- Actor systems assume valid IDs exist or can be created on-demand
- The actual remote actor instance may not exist until the first message arrives

### Server-Side Implementation

This server-side implementation creates actors dynamically when messages arrive for non-existent actor IDs:

```swift
import Distributed
import TicTacFishShared

/// Stand alone server-side swift application, running our SampleWebSocketActorSystem in server mode.
@main
struct Boot {
    
    static func main() {
        let system = try! SampleWebSocketActorSystem(mode: .serverOnly(host: "localhost", port: 8888))
        
        system.registerOnDemandResolveHandler { id in
            // We create new BotPlayers "ad-hoc" as they are requested for.
            // Subsequent resolves are able to resolve the same instance.
            if system.isBotID(id) {
                return system.makeActorWithID(id) {
                    OnlineBotPlayer(team: .rodents, actorSystem: system)
                }
            }
            
            return nil // unable to create-on-demand for given id
        }
        
        print("========================================================")
        print("=== TicTacFish Server Running on: ws://\(system.host):\(system.port) ==")
        print("========================================================")
        
        try await server.terminated // waits effectively forever (until we shut down the system)
    }
}
```

**On-Demand Actor Creation Pattern:**
- Server creates actors dynamically when messages arrive for unknown IDs
- Client can "make up" actor IDs knowing the server will create them
- Subsequent calls resolve to the same existing instance
- This pattern is specific to the sample implementation, not built into distributed actors

## Peer-to-Peer Networking

### Use Case: Local Network Gaming
- Scenario: Poor internet but good local Wi-Fi while traveling
- Want to play with friends on the same network
- Implementation uses Network framework for local networking
- Privacy consideration: Local network access can expose sensitive information

### Service Discovery with Receptionist Pattern

The receptionist pattern is commonly used for actor discovery in distributed systems:

**Concept:**
- Actors "check in" with a receptionist to become discoverable
- Other actors can query the receptionist to find available actors
- Similar to a hotel receptionist helping guests find each other
- Each actor system provides its own receptionist implementation

This code shows how to discover available actors on the network using async sequences and filtering by team affiliation:

```swift
/// As we are playing for our `model.team` team, we try to find a player of the opposing team
let opponentTeam = model.team == .fish ? CharacterTeam.rodents : CharacterTeam.fish

/// The local network actor system provides a receptionist implementation that provides us an async sequence
/// of discovered actors (past and new)
let listing = await localNetworkSystem.receptionist.listing(of: OpponentPlayer.self, tag: opponentTeam.tag)
for try await opponent in listing where opponent.id != self.player.id {
    log("matchmaking", "Found opponent: \(opponent)")
    model.foundOpponent(opponent, myself: self.player, informOpponent: true)
    // inside foundOpponent:
    // if informOpponent {
    //     Task {
    //         try await opponent.startGameWith(opponent: myself, startTurn: false)
    //     }
    // }

    return // make sure to return here, we only need to discover a single opponent
}
```

### LocalNetworkPlayer Implementation

This distributed actor handles both local human input and remote method calls, demonstrating bidirectional peer-to-peer communication:

```swift
public distributed actor LocalNetworkPlayer: GamePlayer {
    public typealias ActorSystem = SampleLocalNetworkActorSystem

    let team: CharacterTeam
    let model: GameViewModel

    var movesMade: Int = 0

    public init(team: CharacterTeam, model: GameViewModel, actorSystem: ActorSystem) {
        self.team = team
        self.model = model
        self.actorSystem = actorSystem
    }

    public distributed func makeMove() async -> GameMove {
        let field = await model.humanSelectedField()

        movesMade += 1
        let move = GameMove(
            playerID: self.id,
            position: field,
            team: team,
            teamCharacterID: movesMade % 2)

        return move
    }

    public distributed func makeMove(at position: Int) async -> GameMove {
        let move = GameMove(
            playerID: id,
            position: position,
            team: team,
            teamCharacterID: movesMade % 2)

        log("player", "Player makes move: \(move)")
        _ = await model.userMadeMove(move: move)

        movesMade += 1
        return move
    }

    public distributed func opponentMoved(_ move: GameMove) async throws {
        do {
            try await model.markOpponentMove(move)
        } catch {
            log("player", "Opponent made illegal move! \(move)")
        }
    }

    public distributed func startGameWith(opponent: OpponentPlayer, startTurn: Bool) async {
        log("local-network-player", "Start game with \(opponent.id), startTurn:\(startTurn)")
        await model.foundOpponent(opponent, myself: self, informOpponent: false)

        await model.waitForOpponentMove(shouldWaitForOpponentMove(myselfID: self.id, opponentID: opponent.id))
    }
}
```

**Key Challenge:** The `makeMove()` method can be called remotely, but actual move selection requires human input. The solution uses `humanSelectedField()` which waits for user interaction via a `@Published` property.

## Advanced Scenarios

### Hybrid Actor Systems

The session describes combining different actor systems for more complex architectures:

**Example: WebSocket + Server Lobby System**
- Device-hosted player actors register with a server-side GameLobby
- GameLobby pairs players and acts as a proxy for distributed calls
- Game sessions can be distributed across multiple servers for load balancing
- Horizontal scaling: Create game session actors on different nodes in a cluster

## Available Actor Systems

### Built-in Systems
- **LocalTestingDistributedActorSystem**: Ships with Swift for local testing and development

### Sample Systems (Reference Implementations)
- **SampleWebSocketActorSystem**: Client/server WebSocket-based system
- **SampleLocalNetworkActorSystem**: Peer-to-peer local networking system
- These are incomplete implementations meant for inspiration, available in session sample code

### Production Systems
- **Swift Distributed Actors Cluster**: Open-source, feature-rich clustering system
  - Implemented using SwiftNIO
  - Specialized for server-side data-center clustering
  - Advanced failure detection techniques
  - Cluster-wide receptionist implementation
  - Available as beta package, maturing alongside Swift 5.7

## Session Recap

### What We Learned
1. **Distributed Actors**: Extend actor isolation and serialization checking across processes
2. **Location Transparency**: Freedom from process-specific actor placement
3. **Actor System Implementations**: Various approaches for different use cases
4. **Practical Implementation**: Step-by-step conversion and real-world scenarios

### Key Principle
Distributed actors are only as powerful as the actor systems they're used with. The actor system handles all the complex networking, serialization, and discovery mechanisms.

## Resources for Further Learning

- **Sample Code**: Complete Tic Tac Fish implementation with all steps
- **Swift Evolution Proposals**: Detailed documentation of distributed actors mechanisms
- **Swift Forums**: Dedicated distributed actors category for developers and users
- **WWDC 2019 "Advances in Networking, Part 2"**: For implementing custom network protocols

## Best Practices

1. **Start Local**: Convert actors to distributed actors while keeping them local initially
2. **Explicit API Surface**: Only mark methods as `distributed` that need remote access
3. **Serialization**: Ensure all distributed method parameters and return types are `Codable`
4. **Actor System Selection**: Choose appropriate actor system for your use case
5. **Error Handling**: Account for network failures and remote actor unavailability
6. **Privacy Considerations**: Be mindful when using local network features

## Mental Model

Think of distributed actors as "islands in the sea of distributed systems":
- Each process is its own "sea of concurrency"
- Distributed actors can communicate across process boundaries
- Same message-passing model applies locally and remotely
- Location transparency enables seamless testing and deployment flexibility
