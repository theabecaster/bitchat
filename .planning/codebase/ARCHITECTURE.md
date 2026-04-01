---
generated: 2026-04-01
focus: arch
---

# Architecture

## Pattern Overview

**Overall:** MVVM with a layered protocol stack

The app is organized as a Model-View-ViewModel system where `ChatViewModel` is the single business-logic coordinator. Below the ViewModel sits a transport abstraction layer (`Transport` protocol), currently satisfied by two concrete implementations: `BLEService` (primary, Bluetooth mesh) and `NostrTransport` (secondary, internet relay via Tor). The protocol stack itself is documented in `WHITEPAPER.md` as four explicit layers:

```
Application Layer   →  BitchatMessage, DeliveryAck, commands
Session Layer       →  BitchatPacket, TTL routing, fragmentation, binary framing
Encryption Layer    →  Noise Protocol (XX pattern, Curve25519 + ChaCha20-Poly1305)
Transport Layer     →  BLE (primary), Nostr WebSocket over Tor (fallback)
```

**Key Characteristics:**
- No central server; all peers are equal participants
- No persistent message storage; message history is ephemeral (capped at 1337 items)
- No persistent peer identifiers; peers are addressed by ephemeral short IDs derived from their Noise static public keys
- Two transport channels coexist: BLE mesh for nearby peers and Nostr relays (proxied through Arti/Tor) for mutual-favorite peers when internet is available

---

## Layers

### View Layer
- **Purpose:** SwiftUI views that render UI and dispatch user actions to the ViewModel
- **Location:** `bitchat/Views/`
- **Contains:** `ContentView.swift` (main screen), `LocationChannelsSheet.swift`, `VerificationViews.swift`, `MeshPeerList.swift`, `FingerprintView.swift`, component views, and media views
- **Depends on:** `ChatViewModel` (via `@EnvironmentObject`), `LocationChannelManager`, `GeohashBookmarksStore`
- **Used by:** `BitchatApp.swift` (root `WindowGroup`)

### ViewModel Layer
- **Purpose:** All business logic, state mutation, and cross-service coordination. Acts as the `BitchatDelegate` receiving network events and updating `@Published` properties.
- **Location:** `bitchat/ViewModels/ChatViewModel.swift`, `bitchat/ViewModels/Extensions/`
- **Contains:** `ChatViewModel` (170 KB — the central coordinator), `ChatViewModel+Nostr.swift`, `ChatViewModel+PrivateChat.swift`, `ChatViewModel+Tor.swift`, `PublicMessagePipeline.swift`, `PublicTimelineStore.swift`, `GeoChannelCoordinator.swift`, `MessageRateLimiter.swift`
- **Depends on:** All services, both transports, `NostrRelayManager`, `TorManager`
- **Used by:** View layer (via `@EnvironmentObject`)

### Services Layer
- **Purpose:** Focused single-responsibility services wired together by `ChatViewModel`
- **Location:** `bitchat/Services/`, `bitchat/Services/BLE/`
- **Contains:** `BLEService` (primary transport), `NostrTransport`, `MessageRouter`, `UnifiedPeerService`, `PrivateChatManager`, `CommandProcessor`, `NoiseEncryptionService`, `KeychainManager`, `FavoritesPersistenceService`, location services, and others
- **Depends on:** Protocol and Model layers, `Noise/`, `Nostr/`, `Identity/`, `Sync/`
- **Used by:** ViewModel layer

### Protocol / Encoding Layer
- **Purpose:** Wire formats, packet types, and binary encoding/decoding
- **Location:** `bitchat/Protocols/`
- **Contains:** `BitchatProtocol.swift` (message types, `BitchatDelegate`), `BinaryProtocol.swift` (wire format), `BinaryEncodingUtils.swift`, `Packets.swift` (TLV-encoded `AnnouncementPacket`), `BitchatFilePacket.swift`, `LocationChannel.swift`, `Geohash.swift`
- **Depends on:** `Models/`
- **Used by:** `BLEService`, `GossipSyncManager`

### Cryptography Layer
- **Purpose:** Noise Protocol implementation and session management
- **Location:** `bitchat/Noise/`
- **Contains:** `NoiseProtocol.swift` (Noise XX/IK/NK handshake primitives), `NoiseSession.swift`, `NoiseSessionManager.swift`, `SecureNoiseSession.swift`, `NoiseSessionState.swift`, `NoiseSessionError.swift`, `NoiseRateLimiter.swift`, `NoiseSecurityConstants.swift`, `NoiseSecurityValidator.swift`
- **Depends on:** `CryptoKit`
- **Used by:** `NoiseEncryptionService` (in Services)

### Identity Layer
- **Purpose:** Persistent identity and key mapping with encryption at rest
- **Location:** `bitchat/Identity/`
- **Contains:** `SecureIdentityStateManager.swift`, `IdentityModels.swift`
- **Depends on:** `KeychainManager`, `CryptoKit`
- **Used by:** `BLEService`, `ChatViewModel`, `NoiseEncryptionService`

### Nostr Layer
- **Purpose:** Nostr protocol support for location channels and internet DMs
- **Location:** `bitchat/Nostr/`
- **Contains:** `NostrRelayManager.swift` (WebSocket relay pool), `NostrProtocol.swift` (event types, signing), `NostrIdentity.swift`, `NostrIdentityBridge.swift`, `GeoRelayDirectory.swift`, `NostrEmbeddedBitChat.swift`, `Bech32.swift`, `XChaCha20Poly1305Compat.swift`
- **Depends on:** Arti (`localPackages/Arti/`) for Tor proxying
- **Used by:** `ChatViewModel+Nostr.swift`, `NostrTransport`, `GeohashPresenceService`

### Sync Layer
- **Purpose:** Gossip-based mesh history synchronization
- **Location:** `bitchat/Sync/`
- **Contains:** `GossipSyncManager.swift`, `GCSFilter.swift` (Golomb-Coded Set), `RequestSyncManager.swift`, `SyncTypeFlags.swift`, `PacketIdUtil.swift`
- **Depends on:** Protocol layer
- **Used by:** `BLEService`

### Local Packages
- **`localPackages/BitLogger/`** — secure structured logging via `SecureLogger` and `OSLog`. Used throughout every service; never logs keys or message content.
- **`localPackages/Arti/`** — Arti (Rust-based Tor client) as a pre-built `.xcframework`, wrapped by `TorManager.swift` and `TorURLSession.swift`. Exposes a local SOCKS5 proxy on `127.0.0.1:39050`.

---

## Data Flow

### Outbound Public Message
1. User types in `ContentView` and taps send
2. `ChatViewModel.sendMessage(_:)` validates and rate-limits via `MessageRateLimiter`
3. Delegates to `meshService.sendMessage(_:mentions:)` on the `Transport` protocol
4. `BLEService` encodes into `BitchatPacket` using `BinaryProtocol`
5. If payload > 469 bytes, fragments into `fragment` packets (MTU = 512 bytes, overhead ~43 bytes)
6. Sends via CoreBluetooth characteristic notifications to all subscribed centrals
7. Connected peripherals relay with TTL decrement (default TTL = 7)

### Outbound Private Message
1. `ChatViewModel.sendPrivateMessage(_:to:)` checks if peer is on BLE or Nostr
2. Routes via `MessageRouter.sendPrivate(_:to:...)` which selects the first reachable `Transport`
3. **BLE path:** `BLEService` calls `NoiseEncryptionService.encrypt(_:for:)`, wraps in `noiseEncrypted` packet type, sends to recipient's peripheral
4. **Nostr path:** `NostrTransport.sendPrivateMessage` creates an NIP-17 gift-wrap event, signs with the per-geohash Ed25519 identity, sends via `NostrRelayManager` over Tor-proxied WebSocket
5. If no transport is reachable, `MessageRouter` queues in its outbox (max 100 per peer, 24-hour TTL)

### Inbound Message
1. `BLEService` receives characteristic write or notification
2. Assembles fragments via `NotificationStreamAssembler` (or direct if small)
3. Decodes `BitchatPacket` using `BinaryProtocol`
4. Deduplicates via `MessageDeduplicator` (bloom filter)
5. Relays to other connected peers (with TTL check and jitter to reduce floods)
6. Dispatches to `BitchatDelegate` callbacks on `ChatViewModel`
7. `ChatViewModel.didReceiveMessage` / `didReceiveNoisePayload` updates `@Published` arrays on main thread
8. SwiftUI re-renders affected views

### Noise Handshake Flow
1. When a private message is needed for a peer with no existing session, `BLEService` checks `LazyHandshakeState`
2. If `.none`, transitions to `.handshakeQueued`, sends Noise XX initiator message (ephemeral key only)
3. Responder replies with `noiseHandshake` packet (ephemeral + encrypted static + DH)
4. Initiator sends final handshake message; both sides transition to `.established`
5. `NoiseEncryptionService` caches the `SecureNoiseSession` per peer
6. Pending messages queued in `pendingMessagesAfterHandshake` are flushed

### Location / Geohash Channel Flow
1. `LocationStateManager` tracks device location via `CoreLocation`
2. `LocationChannelManager` derives geohash channels at multiple precision levels
3. `NetworkActivationService` gates Tor start based on location permission OR mutual favorites
4. Once Tor is ready, `NostrRelayManager` connects to WebSockets for nearby geo-relays from `GeoRelayDirectory`
5. `GeohashPresenceService` periodically broadcasts Kind 20001 heartbeats (randomized 40–80s intervals)
6. `ChatViewModel+Nostr.swift` subscribes to Nostr filters and ingests events via `subscribeNostrEvent`

### State Management
- All `@Published` properties live on `ChatViewModel` and its sub-services (`UnifiedPeerService`, `PrivateChatManager`, `FavoritesPersistenceService`)
- `@EnvironmentObject` propagates `ChatViewModel` to all views
- Combine publishers are used by services that need reactive updates without UI coupling
- BLE operations run on a dedicated `bleQueue` (serial); message processing on `messageQueue` (concurrent); UI updates dispatched to `DispatchQueue.main`

---

## Key Abstractions

### Transport Protocol
- **Purpose:** Decouples the ViewModel from specific networking implementations
- **Location:** `bitchat/Services/Transport.swift`
- **Conformers:** `BLEService` (via `extension BLEService: Transport {}`), `NostrTransport`
- **Pattern:** Delegate-based event delivery (`BitchatDelegate`, `TransportPeerEventsDelegate`) plus a `peerSnapshotPublisher: AnyPublisher<[TransportPeerSnapshot], Never>` for Combine consumers

### BitchatDelegate
- **Purpose:** Protocol that `ChatViewModel` conforms to for receiving all network events
- **Location:** `bitchat/Protocols/BitchatProtocol.swift`
- **Pattern:** Callbacks for `didReceiveMessage`, `didConnectToPeer`, `didDisconnectFromPeer`, `didUpdatePeerList`, `didReceiveNoisePayload`, `didReceivePublicMessage`, `didUpdateMessageDeliveryStatus`

### PeerID
- **Purpose:** Typed peer identity value supporting multiple namespaces
- **Location:** `bitchat/Models/PeerID.swift`
- **Pattern:** Struct with a `Prefix` enum (`mesh:`, `noise:`, `nostr:`, `nostr_`, `name:`, empty). Short routing IDs are 16-hex (8-byte SHA-256 prefix of Noise public key). Full Noise key IDs are 64-hex. Geo-chat peers use `nostr:` prefix + 8-char pubkey; geo-DM peers use `nostr_` prefix + 16-char pubkey.

### MessageRouter
- **Purpose:** Transport selection for private messages, with outbox queuing when no transport is reachable
- **Location:** `bitchat/Services/MessageRouter.swift`
- **Pattern:** Iterates registered transports, picks first reachable, otherwise queues. Flushes queued messages on `favoriteStatusChanged` notification.

### UnifiedPeerService
- **Purpose:** Single source of truth for peer state across BLE and favorites
- **Location:** `bitchat/Services/UnifiedPeerService.swift`
- **Pattern:** Conforms to `TransportPeerEventsDelegate`, combines mesh snapshots and favorites data into `[BitchatPeer]` with `@Published` properties

---

## Entry Points

### iOS Application Entry
- **Location:** `bitchat/BitchatApp.swift` — `@main struct BitchatApp: App`
- **Triggers:** System app launch
- **Responsibilities:** Creates `ChatViewModel` with `KeychainManager`, `NostrIdentityBridge`, `SecureIdentityStateManager`; wires `NotificationDelegate`; starts `NetworkActivationService` and `GeohashPresenceService`; handles scene phase transitions for Tor lifecycle management

### macOS Application Entry
- **Location:** Same `BitchatApp.swift` with `#if os(macOS)` delegate
- **Triggers:** System app launch; uses `NSApplicationDelegate` via `MacAppDelegate`

### Share Extension Entry
- **Location:** `bitchatShareExtension/ShareViewController.swift`
- **Triggers:** System share sheet (iOS only)
- **Responsibilities:** Writes shared content (text/URL/image) to `UserDefaults(suiteName:)` app group, then posts a URL with scheme `bitchat://share` to wake the main app

### BLE Service Start
- **Location:** `bitchat/Services/BLE/BLEService.swift`
- **Triggers:** `ChatViewModel` init (via `Transport.startServices()`), and on `UIApplication.didBecomeActiveNotification`
- **Responsibilities:** Initializes `CBCentralManager` and `CBPeripheralManager`, starts scanning/advertising, fires maintenance timer every 5 seconds

---

## Error Handling

**Strategy:** Graceful degradation — services fail softly and log via `SecureLogger`; no crashes from network errors

**Patterns:**
- Keychain operations return typed enums (`KeychainReadResult`, `KeychainSaveResult`) rather than throwing, allowing callers to handle `.itemNotFound` vs `.accessDenied` vs `.deviceLocked` distinctly
- Noise session failures transition to `LazyHandshakeState.failed(Error)` and are retried on next send
- BLE write failures go to `pendingPeripheralWrites` for retry; notification queue overflow is bounded at 128 entries (`bleMaxInFlightAssemblies`)
- Missing Nostr transport is handled by `MessageRouter` outbox with 24-hour TTL and per-peer 100-message cap
- Fragment reassembly timeouts are cleaned by the 5-second maintenance timer; maximum fragment lifetime is 30 seconds (`bleFragmentLifetimeSeconds`)

---

## Cross-Cutting Concerns

**Logging:** `BitLogger` local package (`SecureLogger`) — wraps `OSLog` with categories (`.session`, `.noise`, `.sync`, etc.), scrubs sensitive strings via `String+Sanitization.swift`. Never logs key material or message content.

**Validation:** `InputValidator` in `bitchat/Utils/` enforces message length (`Limits.maxMessageLength`) and peer ID format. `BLEService` rate-limits subscription attempts per central (`centralSubscriptionRateLimits`).

**Authentication:** QR-based out-of-band verification via `VerificationService`. User scans a QR code encoding Noise key hex + Ed25519 signing key hex + optional Nostr npub + nickname + timestamp + nonce + signature. On scan, a Noise-encrypted challenge/response verifies both parties share the same keys, promoting the peer to `.noiseVerified` status.

**Thread Safety:** BLE callbacks on `bleQueue`; message processing on concurrent `messageQueue` with barrier writes; all `@Published` / UI mutations on `DispatchQueue.main` or with `@MainActor`; per-service `DispatchQueue` with `attributes: .concurrent` for reader-writer patterns.

**Privacy:** No local name in BLE advertising. No persistent message storage. `SecureIdentityStateManager` encrypts identity cache at rest (AES-GCM, key in Keychain). Tor routing for Nostr connections. Location is never broadcast at high precision (block/building geohash levels are excluded from presence heartbeats).

---

*Architecture analysis: 2026-04-01*
