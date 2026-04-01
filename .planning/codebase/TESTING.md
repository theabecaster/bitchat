---
generated: 2026-04-01
focus: quality
---

# Testing Patterns

**Analysis Date:** 2026-04-01

## Test Framework

**Primary Runner:**
- Swift Testing (Apple's first-party framework, introduced Xcode 16)
- Import: `import Testing`
- ~54 test files use Swift Testing

**Legacy Runner:**
- XCTest — still used in ~14 files that have not been migrated
- Import: `import XCTest`
- Files using XCTestCase inherit from it and use `XCTAssert*` macros

**Assertion Libraries:**
- Swift Testing: `#expect(...)`, `#require(...)`, `Issue.record(...)`, `confirmation(...)`
- XCTest (legacy): `XCTAssertTrue`, `XCTAssertFalse`, `XCTAssertEqual`, `XCTAssertNil`

**No separate test config file** — tests are registered via the `bitchatTests` Xcode target in `project.yml`.

**Run Commands:**
```bash
# Xcode keyboard shortcut
Cmd+U                   # Run all tests

# CLI (requires physical device for BLE-dependent tests)
xcodebuild test -scheme bitchat -destination 'platform=macOS'
```

> **Important:** Any test exercising real `BLEService` or `CoreBluetooth` requires a physical device — the iOS Simulator does not support Bluetooth.

---

## Test File Organization

**Location:** All tests live in `bitchatTests/` alongside the main target.

**Directory Structure:**
```
bitchatTests/
├── TestUtilities/
│   ├── TestHelpers.swift        # Factory methods, async helpers
│   └── TestConstants.swift      # Shared constants (timeouts, fixtures)
├── Mocks/
│   ├── MockBLEBus.swift         # In-memory mesh adjacency registry
│   ├── MockBLEService.swift     # Full BLE service test double
│   ├── MockTransport.swift      # Transport protocol test double
│   ├── MockIdentityManager.swift
│   ├── MockKeychain.swift
│   └── MockTransport.swift
├── Protocol/                    # BinaryProtocol encode/decode
├── Protocols/                   # Packet type tests (FilePacket, etc.)
├── Noise/                       # Noise protocol session and rate-limit tests
├── Nostr/                       # Nostr relay directory tests
├── Services/                    # Per-service unit tests
├── EndToEnd/                    # E2E message flow tests
├── Integration/                 # Multi-peer integration tests
├── Fragmentation/               # Packet fragmentation/reassembly
├── Sync/                        # Gossip sync / request-sync tests
├── Utils/                       # PeerID, hex string utils
├── Features/                    # Feature-specific tests (image utils, etc.)
├── Localization/                # Localization key coverage
└── README.md                    # Test harness guide
```

**Naming Convention:**
- Test files: `<Subject>Tests.swift` (e.g., `BinaryProtocolTests.swift`, `MessageRouterTests.swift`)
- Test types: `struct` for Swift Testing, `final class` inheriting `XCTestCase` for legacy tests
- Test functions: Swift Testing uses `@Test func camelCase()`, XCTest uses `func test_camelCase_description()`

---

## Test Structure

**Swift Testing suite (preferred pattern):**
```swift
import Testing
@testable import bitchat

@Suite("NoiseEncryptionService Tests")
struct NoiseEncryptionServiceTests {

    @Test("Encryption status accessors cover all cases")
    func encryptionStatusAccessorsCoverAllCases() {
        #expect(EncryptionStatus.none.icon == "lock.slash")
        #expect(!EncryptionStatus.none.description.isEmpty)
    }

    @Test("Announce and packet signatures round-trip")
    func announceAndPacketSignaturesRoundTrip() throws {
        let service = NoiseEncryptionService(keychain: MockKeychain())
        let signature = try #require(
            service.buildAnnounceSignature(...),
            "Expected announce signature"
        )
        #expect(service.verifyAnnounceSignature(...))
    }
}
```

**XCTest legacy pattern (still in use):**
```swift
import XCTest
@testable import bitchat

final class NoiseRateLimiterTests: XCTestCase {
    func test_allowHandshake_blocksAfterPerPeerLimit() {
        let limiter = NoiseRateLimiter()
        for _ in 0..<NoiseSecurityConstants.maxHandshakesPerMinute {
            XCTAssertTrue(limiter.allowHandshake(from: makePeerID(1)))
        }
        XCTAssertFalse(limiter.allowHandshake(from: makePeerID(1)))
    }
}
```

**ViewModel test pattern (factory function + dependency injection):**
```swift
@MainActor
private func makeTestableViewModel() -> (viewModel: ChatViewModel, transport: MockTransport) {
    let keychain = MockKeychain()
    let identityManager = MockIdentityManager(keychain)
    let transport = MockTransport()
    let viewModel = ChatViewModel(
        keychain: keychain,
        idBridge: NostrIdentityBridge(keychain: MockKeychainHelper()),
        identityManager: identityManager,
        transport: transport
    )
    return (viewModel, transport)
}

struct ChatViewModelInitializationTests {
    @Test @MainActor
    func initialization_setsDelegate() async {
        let (viewModel, transport) = makeTestableViewModel()
        #expect(transport.delegate === viewModel)
    }
}
```

**@Suite annotation** is optional but used for grouping when a file covers a focused area. Most test files omit `@Suite` and rely on the struct name for identification.

---

## Mocking

**Framework:** Hand-written Swift mock types — no third-party mocking library.

**Mock types location:** `bitchatTests/Mocks/`

**MockTransport** — primary isolation mock for `ChatViewModel` unit tests. Conforms to the `Transport` protocol. Records all method calls in typed arrays for assertion:
```swift
final class MockTransport: Transport {
    private(set) var sentMessages: [(content: String, mentions: [String], messageID: String?, timestamp: Date?)] = []
    private(set) var sentPrivateMessages: [(content: String, peerID: PeerID, recipientNickname: String, messageID: String)] = []
    private(set) var startServicesCallCount = 0
    var connectedPeers: Set<PeerID> = []
    var reachablePeers: Set<PeerID> = []
}

// Usage in test:
#expect(transport.sentPrivateMessages.count == 1)
#expect(transport.startServicesCallCount == 1)
```

**MockBLEService** — in-memory BLE service for E2E and integration tests. Uses `MockBLEBus` as a shared adjacency registry. Supports topology simulation:
```swift
let bus = MockBLEBus()
let alice = MockBLEService(peerID: PeerID(str: UUID().uuidString), nickname: "Alice", bus: bus)
let bob = MockBLEService(peerID: PeerID(str: UUID().uuidString), nickname: "Bob", bus: bus)

alice.simulateConnection(with: bob)   // bidirectional

bob.messageDeliveryHandler = { message in ... }
bob.packetDeliveryHandler = { packet in ... }

alice.simulateIncomingPacket(packet)
alice.simulateIncomingMessage(message)
```

**MockBLEBus** — registry + adjacency map shared across `MockBLEService` instances. Created per-test; supports `autoFloodEnabled` for integration broadcast tests.

**MockKeychain / MockIdentityManager / MockIdentityBridge** — lightweight in-memory implementations of their respective protocol interfaces.

**What to mock:**
- BLE transport layer (`MockBLEService` or `MockTransport`) — always mock in unit tests
- Keychain — always mock (`MockKeychain`)
- Identity manager — mock in ViewModel and service unit tests

**What NOT to mock:**
- `BinaryProtocol` encode/decode — tested directly against real implementation
- `NoiseSession` — tested with real crypto keys via `CryptoKit`
- `BitchatMessage` serialization — tested against real encoder

---

## Fixtures and Factories

**Shared test constants** (`bitchatTests/TestUtilities/TestConstants.swift`):
```swift
struct TestConstants {
    static let defaultTimeout: TimeInterval = 5.0
    static let shortTimeout:   TimeInterval = 1.0
    static let longTimeout:    TimeInterval = 10.0

    static let testNickname1 = "Alice"
    static let testNickname2 = "Bob"
    static let testMessage1 = "Hello, World!"
    static let testMessage2 = "How are you?"
    static let testLongMessage = String(repeating: "This is a long message. ", count: 100)
    static let testSignature = Data(repeating: 0xAB, count: 64)
}
```

**Factory methods** (`bitchatTests/TestUtilities/TestHelpers.swift`):
- `TestHelpers.createTestMessage(...)` → `BitchatMessage`
- `TestHelpers.createTestPacket(...)` → `BitchatPacket`
- `TestHelpers.generateRandomData(length:)` → `Data`
- `TestHelpers.generateTestPeerID()` → `String`
- `TestHelpers.generateTestKeyPair()` → `(privateKey, publicKey)`

---

## Async Testing

**Swift Testing `confirmation` API** — preferred for async event observation:
```swift
@Test func simplePublicMessage() async {
    alice.simulateConnection(with: bob)

    await confirmation("Bob receives message") { bobReceivesMessage in
        bob.messageDeliveryHandler = { message in
            if message.content == TestConstants.testMessage1 {
                bobReceivesMessage()
            }
        }
        alice.sendMessage(TestConstants.testMessage1)
    }
}

// expectedCount for multi-event assertions:
await confirmation("Both receive message", expectedCount: 2) { receiveMessage in
    bob.messageDeliveryHandler = { _ in receiveMessage() }
    charlie.messageDeliveryHandler = { _ in receiveMessage() }
    alice.sendMessage(TestConstants.testMessage1)
}
```

**`TestHelpers.waitUntil`** — polling-based async wait for state changes.
**`TestHelpers.waitFor`** — throwing async wait.

**Default timeout:** `5.0s`. Short: `1.0s`. Long: `10.0s`.

---

## Error Testing

**`#require` for unwrapping optionals** (fails test immediately on nil):
```swift
let encodedData = try #require(BinaryProtocol.encode(packet), "Failed to encode packet")
let decodedPacket = try #require(BinaryProtocol.decode(encodedData), "Failed to decode packet")
```

**Nil result assertions** for graceful failure on invalid input:
```swift
#expect(BinaryProtocol.decode(tooSmall) == nil)
#expect(BinaryProtocol.decode(malformedData) == nil)
```

**`Issue.record`** to flag unexpected events without failing the full test:
```swift
if messageCount > 1 {
    Issue.record("Duplicate message was not filtered")
}
```

---

## Test Types

### Unit Tests (majority)
- **Location:** `bitchatTests/Services/`, `bitchatTests/Protocol/`, `bitchatTests/Utils/`, `bitchatTests/Noise/`, `bitchatTests/Nostr/`
- **Key files:** `BinaryProtocolTests.swift`, `MessageRouterTests.swift`, `NoiseEncryptionServiceTests.swift`

### ViewModel Tests
- **Location:** `bitchatTests/ChatViewModelTests.swift`, `ChatViewModelExtensionsTests.swift`, `ChatViewModelDeliveryStatusTests.swift`, `ChatViewModelRefactoringTests.swift`, `ChatViewModelTorTests.swift`
- **Pattern:** `@MainActor` factory returns `(ChatViewModel, MockTransport)` tuple; assertions on `transport.sent*` arrays

### View Smoke Tests
- **Location:** `bitchatTests/ViewSmokeTests.swift`
- **Pattern:** `mount(_:)` helper wraps views in hosting controller, calls layout — verifies no crash

### End-to-End (E2E) Tests
- **Location:** `bitchatTests/EndToEnd/PublicChatE2ETests.swift`, `PrivateChatE2ETests.swift`
- **Pattern:** 2–4 `MockBLEService` nodes on shared `MockBLEBus`; `autoFloodEnabled = false`; `confirmation(...)` for event observation
- **Covers:** broadcast delivery, TTL enforcement, duplicate prevention, relay chaining, mesh topology

### Integration Tests
- **Location:** `bitchatTests/Integration/IntegrationTests.swift`
- **Pattern:** `MockBLEBus(autoFloodEnabled: true)` propagates broadcasts across full connected component
- **Key distinction:** TTL not enforced; verifies all nodes receive in large simulated mesh

### Protocol Contract Tests
- **Location:** `bitchatTests/ProtocolContractTests.swift`
- **Scope:** Invariants that must hold across the encode/decode boundary

### Fragmentation Tests
- **Location:** `bitchatTests/Fragmentation/FragmentationTests.swift`
- **Pattern:** Injects real `BLEService` with `initializeBluetoothManagers: false`; uses `_test_handlePacket` hook to feed shuffled fragments

---

## Coverage

**Requirements:** No enforced coverage threshold.

**Physical device constraint:** Tests touching `BLEService` directly must run on device. Simulator-only runs will skip or fail those cases.

**Notable gaps:**
- Real Bluetooth mesh behavior — untestable without physical hardware
- `BluetoothMeshService` itself is not unit-tested; covered indirectly through E2E harness

---

## Localization Coverage

- `bitchatTests/Localization/PrimaryLocalizationKeys.json` — list of expected localization keys
- Used to detect missing keys at test time (plain JSON comparison, no framework)

---

*Testing analysis: 2026-04-01*
