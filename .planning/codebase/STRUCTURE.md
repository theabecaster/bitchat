---
generated: 2026-04-01
focus: arch
---

# Codebase Structure

**Analysis Date:** 2026-04-01

## Directory Layout

```
bitchat/                              # Repository root
├── bitchat/                          # Shared app source (iOS + macOS)
│   ├── BitchatApp.swift              # App entry point
│   ├── Models/                       # Data models and value types
│   ├── Protocols/                    # Binary wire protocol and packet definitions
│   ├── Services/                     # Business logic services
│   │   └── BLE/                      # Bluetooth LE transport layer
│   ├── ViewModels/                   # MVVM view models
│   │   └── Extensions/               # ChatViewModel feature partitions
│   ├── Views/                        # SwiftUI views
│   │   ├── Components/               # Reusable UI components
│   │   └── Media/                    # Media-specific views (images, voice)
│   ├── Features/                     # Self-contained feature modules
│   │   ├── media/                    # Media capture/display utilities
│   │   └── voice/                    # Voice recording and playback
│   ├── Noise/                        # Noise Protocol cryptographic session layer
│   ├── Nostr/                        # Nostr protocol transport and identity
│   ├── Identity/                     # Secure identity state management
│   ├── Sync/                         # Gossip-based message sync
│   ├── Utils/                        # Shared utilities and extensions
│   ├── _PreviewHelpers/              # SwiftUI preview stubs (not shipped)
│   └── Assets.xcassets/              # App icons and color assets
├── bitchatTests/                     # All unit and integration tests
│   ├── EndToEnd/                     # Full message flow E2E tests
│   ├── Features/                     # Feature-level tests (image utils, etc.)
│   ├── Fragmentation/                # BLE packet fragmentation tests
│   ├── Integration/                  # Cross-service integration tests
│   ├── Localization/                 # Localization key coverage tests
│   ├── Mocks/                        # Shared mock objects
│   ├── Noise/                        # Noise Protocol crypto tests
│   ├── Nostr/                        # Nostr relay and transport tests
│   ├── Protocol/                     # Binary protocol encode/decode tests
│   ├── Protocols/                    # Packet structure tests
│   ├── Services/                     # Per-service unit tests
│   ├── Sync/                         # Gossip sync tests
│   ├── TestUtilities/                # Shared test constants and helpers
│   └── Utils/                        # Utility unit tests
├── bitchatShareExtension/            # iOS Share Extension target
│   └── Localization/                 # Extension localization strings
├── localPackages/
│   └── Arti/                         # Bundled Tor client (arti) XCFramework + Swift shim
│       ├── Sources/C/                # C shim (arti_shim.c)
│       └── Frameworks/arti.xcframework/   # Pre-built arti binary for arm64
├── Configs/                          # Xcode build configuration files
│   ├── Debug.xcconfig
│   ├── Release.xcconfig
│   ├── Local.xcconfig.example
│   └── Local.xcconfig                # Git-ignored local overrides
├── docs/                             # Developer documentation
├── relays/                           # Nostr relay data
│   └── online_relays_gps.csv         # GPS-tagged relay list (auto-updated)
├── project.yml                       # XcodeGen project definition
├── Package.swift                     # Swift Package Manager manifest
├── Justfile                          # macOS build/run convenience targets
└── AGENTS.md / CLAUDE.md             # AI agent instructions (CLAUDE.md symlinks AGENTS.md)
```

## Directory Purposes

**`bitchat/Models/`:**
- Purpose: Plain data types passed throughout the app; no business logic
- Contains: `BitchatMessage.swift`, `BitchatPacket.swift`, `BitchatPeer.swift`, `PeerID.swift`, `ReadReceipt.swift`, `CommandInfo.swift`, `MessagePadding.swift`, `NoisePayload.swift`, `RequestSyncPacket.swift`
- Key files: `BitchatMessage.swift` (core message type), `BitchatPeer.swift` (peer representation)

**`bitchat/Protocols/`:**
- Purpose: Wire-format definitions and binary codec; single source of truth for packet structure
- Contains: `BitchatProtocol.swift` (packet types, constants), `BinaryProtocol.swift` (encode/decode), `BitchatFilePacket.swift`, `Packets.swift`, `Geohash.swift`, `LocationChannel.swift`, `BinaryEncodingUtils.swift`
- Key files: `BitchatProtocol.swift` — update this when adding new packet types

**`bitchat/Services/`:**
- Purpose: All non-UI business logic; each file is a focused, injectable service
- Contains: `BLE/BLEService.swift`, `NoiseEncryptionService.swift`, `MessageRouter.swift`, `PrivateChatManager.swift`, `KeychainManager.swift`, `NotificationService.swift`, `RelayController.swift`, `UnifiedPeerService.swift`, `VerificationService.swift`, `GeohashPresenceService.swift`, `TransferProgressManager.swift`, and more
- Key files: `BLE/BLEService.swift` (core mesh networking), `NoiseEncryptionService.swift` (E2E crypto)

**`bitchat/ViewModels/`:**
- Purpose: MVVM view models; `ChatViewModel` is the central coordinator
- Contains: `ChatViewModel.swift`, `GeoChannelCoordinator.swift`, `MessageRateLimiter.swift`, `PublicMessagePipeline.swift`, `PublicTimelineStore.swift`, `MinimalDistancePalette.swift`
- Key files: `ChatViewModel.swift` — primary business logic wiring point
- `Extensions/`: Feature partitions — `ChatViewModel+Nostr.swift`, `ChatViewModel+PrivateChat.swift`, `ChatViewModel+Tor.swift`

**`bitchat/Views/`:**
- Purpose: SwiftUI view layer; no business logic
- Contains: `ContentView.swift` (root view), `AppInfoView.swift`, `FingerprintView.swift`, `MeshPeerList.swift`, `GeohashPeopleList.swift`, `LocationChannelsSheet.swift`, `LocationNotesView.swift`, `VerificationViews.swift`, `MessageTextHelpers.swift`
- `Components/`: Small reusable views — `CommandSuggestionsView.swift`, `DeliveryStatusView.swift`, `PaymentChipView.swift`, `TextMessageView.swift`
- `Media/`: `BlockRevealImageView.swift`, `VoiceNoteView.swift`, `WaveformView.swift`

**`bitchat/Noise/`:**
- Purpose: Complete Noise Protocol XX handshake and session management, isolated from BLE/transport
- Contains: `NoiseProtocol.swift`, `NoiseSession.swift`, `NoiseSessionManager.swift`, `SecureNoiseSession.swift`, `NoiseSessionState.swift`, `NoiseSessionError.swift`, `NoiseRateLimiter.swift`, `NoiseSecurityConstants.swift`, `NoiseSecurityError.swift`, `NoiseSecurityValidator.swift`

**`bitchat/Nostr/`:**
- Purpose: Nostr protocol integration for relay-based transport and identity bridging
- Contains: `NostrProtocol.swift`, `NostrIdentity.swift`, `NostrIdentityBridge.swift`, `NostrRelayManager.swift`, `NostrTransport.swift`, `NostrEmbeddedBitChat.swift`, `GeoRelayDirectory.swift`, `Bech32.swift`, `XChaCha20Poly1305Compat.swift`

**`bitchat/Identity/`:**
- Purpose: Cryptographic identity lifecycle (key generation, Keychain persistence, state machine)
- Contains: `IdentityModels.swift`, `SecureIdentityStateManager.swift`

**`bitchat/Sync/`:**
- Purpose: Gossip-based sync protocol for message history reconciliation across peers
- Contains: `GossipSyncManager.swift`, `GCSFilter.swift`, `PacketIdUtil.swift`, `RequestSyncManager.swift`, `SyncTypeFlags.swift`

**`bitchat/Utils/`:**
- Purpose: Pure utility functions and Swift extensions; no dependencies on app types
- Contains: `CompressionUtil.swift`, `InputValidator.swift`, `MessageDeduplicator.swift`, `PeerDisplayNameResolver.swift`, `Font+Bitchat.swift`, `Color+Peer.swift`, `Data+SHA256.swift`, `String+DJB2.swift`, `String+Nickname.swift`, `FileTransferLimits.swift`

**`bitchat/Features/`:**
- Purpose: Self-contained feature bundles; `Features/media/` and `Features/voice/` hold the logic-side counterparts (`ImageUtils.swift`, `VoiceRecorder.swift`, `VoiceNotePlaybackController.swift`, `Waveform.swift`)

**`bitchat/_PreviewHelpers/`:**
- Purpose: SwiftUI preview support stubs not included in production builds
- Contains: `BitchatMessage+Preview.swift`, `PreviewKeychainManager.swift`

**`bitchatTests/Mocks/`:**
- Purpose: Shared mock objects injected across all test suites
- Contains: `MockBLEBus.swift`, `MockBLEService.swift`, `MockIdentityManager.swift`, `MockKeychain.swift`, `MockTransport.swift`, `TestNetworkHelper.swift`

**`bitchatShareExtension/`:**
- Purpose: iOS Share Extension; allows sharing text, URLs, and images into bitchat from other apps

**`localPackages/Arti/`:**
- Purpose: Bundled Tor client as a local Swift Package; wraps a pre-built `arti.xcframework` with a C shim and Swift API

**`Configs/`:**
- Purpose: Xcode build configuration `.xcconfig` files for Debug, Release, and local developer overrides
- Key files: `Local.xcconfig.example` — copy to `Local.xcconfig` for local secrets; `Local.xcconfig` is git-ignored

**`relays/`:**
- Purpose: Machine-updated CSV of live Nostr relay coordinates used by `GeoRelayDirectory`

## Key File Locations

**Entry Points:**
- `bitchat/BitchatApp.swift`: `@main` struct; initializes `KeychainManager`, `NostrIdentityBridge`, `SecureIdentityStateManager`, and `ChatViewModel`; handles platform delegates for iOS (`AppDelegate`) and macOS (`MacAppDelegate`)

**Project Definition:**
- `project.yml`: XcodeGen spec; edit this (not `.xcodeproj`) to add targets, sources, or dependencies

**Build Automation:**
- `Justfile`: `just run`, `just build`, `just clean` for macOS

**Core Business Logic:**
- `bitchat/ViewModels/ChatViewModel.swift`: Primary coordinator; wires all services together
- `bitchat/ViewModels/Extensions/ChatViewModel+Nostr.swift`: Nostr-specific logic
- `bitchat/ViewModels/Extensions/ChatViewModel+PrivateChat.swift`: DM routing logic
- `bitchat/ViewModels/Extensions/ChatViewModel+Tor.swift`: Tor transport lifecycle

**Protocol Definitions:**
- `bitchat/Protocols/BitchatProtocol.swift`: All packet type constants; update when adding packet types
- `bitchat/Protocols/BinaryProtocol.swift`: Codec for over-the-air binary format

**Networking:**
- `bitchat/Services/BLE/BLEService.swift`: Bluetooth LE mesh; peer discovery, multi-hop routing, fragmentation
- `bitchat/Services/Transport.swift`: Abstraction over BLE and Nostr transports
- `bitchat/Services/TransportConfig.swift`: Transport configuration

**Cryptography:**
- `bitchat/Noise/NoiseSessionManager.swift`: Noise XX handshake orchestrator
- `bitchat/Services/NoiseEncryptionService.swift`: AES-256-GCM message encryption
- `bitchat/Services/KeychainManager.swift`: Key storage via Keychain

**Root View:**
- `bitchat/Views/ContentView.swift`: Root SwiftUI view

**Test Helpers:**
- `bitchatTests/TestUtilities/TestConstants.swift`: Shared test constants
- `bitchatTests/TestUtilities/TestHelpers.swift`: Shared test utilities
- `bitchatTests/Mocks/MockBLEService.swift`: Primary BLE mock for unit tests

## Naming Conventions

**Files:**
- Services: `[Noun]Service.swift` or `[Noun]Manager.swift`
- View models: `[Name]ViewModel.swift` plus extensions as `[ViewModel]+[Feature].swift`
- Views: Descriptive noun phrases ending in `View.swift`, `Sheet.swift`, or `List.swift`
- Protocols/wire: `Bitchat[Thing].swift` for shared protocol types
- Tests: Mirror the source file name with `Tests` suffix

**Directories:**
- PascalCase for all source subdirectories (`Services/`, `ViewModels/`, `Protocols/`)
- Lowercase only for `Features/` subdirectories (`media/`, `voice/`)

## Where to Add New Code

**New service (business logic):**
- Implementation: `bitchat/Services/[Name]Service.swift` or `bitchat/Services/[Name]Manager.swift`
- Tests: `bitchatTests/Services/[Name]ServiceTests.swift`
- Wire into: `bitchat/ViewModels/ChatViewModel.swift` (inject via initializer or property)

**New packet type:**
- Add constant to: `bitchat/Protocols/BitchatProtocol.swift`
- Add encode/decode to: `bitchat/Protocols/BinaryProtocol.swift`
- Tests: `bitchatTests/Protocol/` or `bitchatTests/Protocols/`

**New SwiftUI view:**
- Full-screen view: `bitchat/Views/[Name]View.swift`
- Reusable component: `bitchat/Views/Components/[Name]View.swift`
- Media-specific: `bitchat/Views/Media/[Name]View.swift`

**New feature extension on ChatViewModel:**
- Add file: `bitchat/ViewModels/Extensions/ChatViewModel+[FeatureName].swift`

**Shared utility / extension:**
- `bitchat/Utils/[Type]+[Feature].swift` or `bitchat/Utils/[Name]Util.swift`

**New crypto/session logic:**
- Noise-layer: `bitchat/Noise/[Name].swift`
- Nostr-layer: `bitchat/Nostr/[Name].swift`
- Identity: `bitchat/Identity/[Name].swift`

**New gossip/sync logic:**
- `bitchat/Sync/[Name].swift` with tests at `bitchatTests/Sync/[Name]Tests.swift`

**Platform-specific code:**
- Keep in shared `bitchat/` source but guard with `#if os(iOS)` / `#if os(macOS)` — do not create separate target source directories

**Preview helpers (development only):**
- `bitchat/_PreviewHelpers/[Type]+Preview.swift` — used only in SwiftUI `#Preview` blocks

## Special Directories

**`bitchat.xcodeproj/`:**
- Generated: Yes — via `xcodegen generate` from `project.yml`; do not edit manually
- Committed: Yes (for CI compatibility)

**`localPackages/Arti/Frameworks/arti.xcframework/`:**
- Pre-built Tor client binary for arm64 iOS device, arm64 iOS simulator, and arm64 macOS
- Committed: Yes (vendor-supplied binary)

**`.planning/codebase/`:**
- Generated: Yes — by `/gsd:map-codebase`
- Committed: Yes

**`relays/`:**
- Generated: Yes — via `.github/workflows` on a schedule
- Committed: Yes

---

*Structure analysis: 2026-04-01*
