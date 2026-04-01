---
generated: 2026-04-01
focus: quality
---

# Coding Conventions

**Analysis Date:** 2026-04-01

## File Header Pattern

Every source file begins with a standard block comment:

```swift
//
// FileName.swift
// bitchat
//
// This is free and unencumbered software released into the public domain.
// For more information, see <https://unlicense.org>
//
```

Test files use `bitchatTests` as the target name. This header appears in every `.swift` file including mocks, test helpers, and source files.

## Naming Conventions

**Files:**
- Source files: `PascalCase.swift` matching the primary type — `BinaryProtocol.swift`, `ChatViewModel.swift`, `MessageRouter.swift`
- ViewModel extensions: `ChatViewModel+FeatureName.swift` — `ChatViewModel+Nostr.swift`, `ChatViewModel+PrivateChat.swift`, `ChatViewModel+Tor.swift`
- Test files: `TypeNameTests.swift` — `BinaryProtocolTests.swift`, `ChatViewModelTests.swift`, `NoiseEncryptionServiceTests.swift`
- Mocks: `MockTypeName.swift` — `MockBLEService.swift`, `MockTransport.swift`, `MockKeychain.swift`
- Extensions on types: `TypeName+Concept.swift` — `Color+Peer.swift`, `Data+SHA256.swift`, `String+Nickname.swift`
- Protocol/test utilities: `TestHelpers.swift`, `TestConstants.swift`

**Types (classes, structs, enums, protocols):**
- `PascalCase` throughout: `BitchatMessage`, `BinaryProtocol`, `EncryptionStatus`, `KeychainManagerProtocol`
- Protocol names end in `Protocol` when defining service contracts: `KeychainManagerProtocol`, `SecureIdentityStateManagerProtocol`
- Protocol names end in `Delegate` for callback interfaces: `BitchatDelegate`, `TransportPeerEventsDelegate`
- Protocol names end in `Context` for read-only dependency injection: `CommandContextProvider`, `MessageFormattingContext`

**Functions and methods:**
- `camelCase` for all functions and methods: `sendMessage(_:mentions:)`, `simulateConnectedPeer(_:)`, `validateNickname(_:)`
- Verb-first for actions: `sendPrivateMessage`, `buildAnnounceSignature`, `validateTimestamp`
- Predicate-style for booleans: `isPeerConnected`, `isPeerReachable`, `isRecoverableError`

**Variables and properties:**
- `camelCase` for all: `myPeerID`, `connectedPeers`, `sentPrivateMessages`
- Private backing stores use underscore prefix: `_cachedFormattedText`
- Constants: `camelCase` static lets inside `struct Limits {}` or `struct Constants {}`
- Published properties use `@Published` directly, no backing store convention

**Enum cases:**
- `camelCase` for cases: `.success`, `.itemNotFound`, `.accessDenied`, `.noiseHandshaking`

## MARK Organization

Files consistently use `// MARK: -` sections to organize code. Standard sections in services and view models:

```swift
// MARK: - Constants
// MARK: - Properties (or Core State)
// MARK: - Initialization
// MARK: - Protocol Conformance (or Transport Protocol Conformance)
// MARK: - Public Methods
// MARK: - Private Helpers
// MARK: - Delegate Methods
```

Extension files at file scope use `// MARK: - CBCentralManagerDelegate` style to mark protocol extension blocks.

## Platform-Specific Code

All platform divergence uses compiler directives, never runtime checks. The dominant pattern is `#if os(iOS)` / `#elseif os(macOS)`:

```swift
#if os(iOS)
import UIKit
#elseif os(macOS)
import AppKit
#endif
```

```swift
#if os(iOS)
isAppActive = UIApplication.shared.applicationState == .active
NotificationCenter.default.addObserver(self, selector: #selector(appDidBecomeActive),
    name: UIApplication.didBecomeActiveNotification, object: nil)
#endif
```

In views, the `#if os(macOS)` block typically handles hosting differences:

```swift
#if os(iOS)
let host = UIHostingController(rootView: view)
#else
let host = NSHostingView(rootView: view)
#endif
```

Key files with extensive platform branching:
- `bitchat/Services/BLE/BLEService.swift` — iOS background mode, UIApplication state, app lifecycle
- `bitchat/Services/KeychainManager.swift` — Keychain attribute differences (kSecAttrAccessibleWhenUnlocked etc.)
- `bitchat/BitchatApp.swift` — scene phases, window styles
- `bitchat/Views/ContentView.swift` — focus modifiers, keyboard handling

**Rule from CLAUDE.md:** Platform-specific code uses `#if os(iOS)` / `#if os(macOS)`. Shared code stays in `bitchat/`.

## Class vs Struct vs Protocol

- **`final class`** for services, view models, and anything that needs reference semantics or delegates: `BLEService`, `ChatViewModel`, `MockTransport`, `BitchatMessage`
- **`struct`** for value types, models without reference needs, and protocol definitions: `BitchatPacket`, `TransportPeerSnapshot`, `InputValidator`, test structs
- **`enum`** for type-safe constants and state: `MessageType`, `EncryptionStatus`, `KeychainReadResult`, `LazyHandshakeState`
- **`protocol`** for dependency injection seams: `Transport`, `KeychainManagerProtocol`, `BitchatDelegate`

## Access Control

- `private` used extensively for implementation details, especially in services
- `private(set)` for read-only public properties in mocks and services: `private(set) var sentMessages`, `private(set) var startServicesCallCount`
- Internal (no modifier) for cross-module-within-target access
- `@testable import bitchat` in all test files to access internal types

## Async Patterns

The codebase mixes `async/await` with `DispatchQueue`-based concurrency. The preferred modern pattern is `Task { @MainActor in }`:

```swift
Task { @MainActor in
    try? await Task.sleep(nanoseconds: UInt64(...))
    // UI updates
}
```

Legacy patterns still present in `BLEService.swift` and `ChatViewModel.swift`:

```swift
DispatchQueue.main.asyncAfter(deadline: .now() + delay) { [weak self] in
    guard let self = self else { return }
    // work
}
```

Combine publishers are used for reactive state with `.receive(on: DispatchQueue.main)`:

```swift
publisher
    .receive(on: DispatchQueue.main)
    .sink { [weak self] value in
        // handle
    }
    .store(in: &cancellables)
```

**`@MainActor`** annotation is applied broadly:
- All `@Published` property owners are on the main actor
- `ChatViewModel` itself is accessed via `@MainActor` functions
- Services annotated `@MainActor` at class level: `MessageRouter`, `PrivateChatManager`

**`[weak self]` in closures** is mandatory in all async closures that capture `self` to prevent retain cycles. No exceptions observed.

## Error Handling

**No thrown errors in service APIs.** Services return optionals, enum result types, or `Bool` success flags rather than throwing.

Keychain operations use a custom result enum:
```swift
enum KeychainReadResult {
    case success(Data)
    case itemNotFound
    case accessDenied
    case deviceLocked
    case authenticationFailed
    case otherError(OSStatus)
}
```

Functions that can fail return `Optional`:
```swift
func encode(_ packet: BitchatPacket, padding: Bool = true) -> Data? { ... }
func decode(_ data: Data) -> BitchatPacket? { ... }
static func validateNickname(_ nickname: String) -> String? { ... }
```

Guard-let is the dominant error-handling form:
```swift
guard let encoded = BinaryProtocol.encode(packet) else { return }
guard !trimmed.isEmpty else { return nil }
guard trimmed.count <= maxLength else { return nil }
```

No `try/catch` in service code except where CryptoKit or Foundation requires it, and those are typically wrapped as `try?`.

## Logging

All logging goes through `SecureLogger` from the `BitLogger` local package (`localPackages/BitLogger/Sources/SecureLogger.swift`). Never use `print()`, `NSLog`, or `os_log` directly.

```swift
import BitLogger

SecureLogger.debug("Some debug info", category: .noise)
SecureLogger.info("Peer connected: \(peerID)", category: .bluetooth)
SecureLogger.warning("Decode failed for packet", category: .protocol)
SecureLogger.error("Keychain access denied", category: .security)
```

The `category` parameter is an `OSLog` value defined in `localPackages/BitLogger/Sources/OSLog+Categories.swift`. Common categories: `.noise`, `.bluetooth`, `.security`, `.protocol`.

**Security rule (from CLAUDE.md):** Never log keys or message content.

Log level is controlled by the `BITCHAT_LOG_LEVEL` environment variable. Default is `.info`.

## Services Pattern

Services are injected via `init` parameters using protocol types, enabling testability:

```swift
final class NoiseEncryptionService {
    init(keychain: KeychainManagerProtocol) { ... }
}

final class ChatViewModel: ObservableObject {
    init(keychain: KeychainManagerProtocol,
         idBridge: NostrIdentityBridge,
         identityManager: SecureIdentityStateManagerProtocol,
         transport: Transport) { ... }
}
```

New services should:
1. Live in `bitchat/Services/`
2. Be `final class` with a protocol counterpart when testability is needed
3. Accept dependencies via `init`, not singletons
4. Expose a minimal public API; use `private` for everything else
5. Log via `SecureLogger`, never `print`
6. Be wired into `ChatViewModel` in `bitchat/ViewModels/ChatViewModel.swift`

## ViewModel Pattern (MVVM)

`ChatViewModel` is the sole ViewModel and is an `ObservableObject` with `@Published` properties. It implements multiple protocols:

```swift
final class ChatViewModel: ObservableObject, BitchatDelegate, CommandContextProvider,
                           GeohashParticipantContext, MessageFormattingContext { ... }
```

Large feature areas are split into `ChatViewModel+Feature.swift` extension files in `bitchat/ViewModels/Extensions/`.

UI state lives in `@Published` properties. Services are `let` constants initialized in `init`. Never access services directly from views — always via the ViewModel.

## Protocol Design

Protocol types serve as dependency inversion seams. The `Transport` protocol (`bitchat/Services/Transport.swift`) is the primary example — `BLEService` conforms to it, `MockTransport` conforms to it for tests, and `ChatViewModel` only knows `Transport`.

Protocol extensions provide default no-op implementations for optional methods:

```swift
extension Transport {
    func sendVerifyChallenge(to peerID: PeerID, noiseKeyHex: String, nonceA: Data) {}
    func acceptPendingFile(id: String) -> URL? { nil }
}
```

New packet types require updating `bitchat/Protocols/BitchatProtocol.swift` per CLAUDE.md.

## Comments and Documentation

Key types and files carry extensive doc-comment headers with `///` triple-slash comments explaining purpose, architecture, data flows, and design rationale. See `BinaryProtocol.swift`, `ChatViewModel.swift`, `NoiseEncryptionService.swift`.

Inline comments explain non-obvious decisions and reference audit/issue identifiers: `// BCH-01-004:`, `// BCH-01-009:`, `// BCH-01-011:` track security audit findings.

`TODO:` comments are used for known issues, e.g.:
```swift
// TODO: Check if this is intended that the decoding only gets the first 8
```

---

*Convention analysis: 2026-04-01*
