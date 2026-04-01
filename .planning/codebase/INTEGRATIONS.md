---
generated: 2026-04-01
focus: tech
---

# External Integrations

**Analysis Date:** 2026-04-01

## Bluetooth LE (CoreBluetooth)

**Role:** Primary transport layer — the app operates entirely over BLE mesh without any internet requirement.

**Implementation:** `bitchat/Services/BLE/BLEService.swift`

**Pattern:**
- `BLEService` subclasses `NSObject` and acts as both `CBCentralManagerDelegate` (scanning/connecting) and `CBPeripheralManagerDelegate` (advertising)
- Dual-role: the device simultaneously acts as a BLE central (connects to peers) and a BLE peripheral (advertises to peers)
- Service UUID: `F47B5E2D-4A9E-4C5A-9B3F-8E1D2C3A4B5A` (debug/testnet) or `…4B5C` (release/mainnet)
- Characteristic UUID: `A1B2C3D4-E5F6-4A5B-8C9D-0E1F2A3B4C5D`
- State restoration identifiers: `chat.bitchat.ble.central` and `chat.bitchat.ble.peripheral`
- Max MTU: 512 bytes; messages larger than MTU are fragmented and reassembled
- Fragment reassembly keyed on `(sender: UInt64, id: UInt64)` pairs

**iOS Background Modes** (declared in `project.yml` Info.plist):
- `bluetooth-central` — scanning continues in background
- `bluetooth-peripheral` — advertising continues in background

**iOS Usage Strings:**
- `NSBluetoothAlwaysUsageDescription` — present in both iOS and macOS Info.plist
- `NSBluetoothPeripheralUsageDescription` — present in both

**Rate limiting:** Subscription-triggered announce rate limiting per central UUID to prevent enumeration attacks (`SubscriptionRateLimitState`).

**Adaptive TTL:** Default message TTL configurable via `TransportConfig.messageTTLDefault`; `RelayController.swift` applies probabilistic relay suppression in high-degree (dense) mesh topologies.

## Tor Network (Arti / local package)

**Role:** All internet-facing connections (Nostr relay WebSockets, geo-relay CSV fetch) are routed through a local Tor SOCKS5 proxy.

**Implementation:** `localPackages/Arti/` (local SPM package)

**Pattern:**
- `TorManager` (singleton at `TorManager.shared`) bootstraps an embedded Arti Rust client
- SOCKS5 proxy binds to `127.0.0.1:39050`
- Calls into the Rust static library via Swift FFI (`@_silgen_name`): `arti_start`, `arti_stop`, `arti_is_running`, `arti_bootstrap_progress`, `arti_go_dormant`, `arti_wake`
- `TorURLSession` wraps `URLSession` to route requests through the SOCKS proxy
- **Fail-closed by default:** production builds refuse all network requests until Tor bootstraps (`torEnforced = true` unless `BITCHAT_DEV_ALLOW_CLEARNET` compile flag is set)
- `NetworkActivationService.swift` gates Nostr relay connections on both Tor readiness and the presence of mutual favorites
- Dormant/wake lifecycle tied to app foreground/background transitions (iOS `scenePhase`, macOS `NSApplication` notifications)

**Published state:** `@Published` properties `isReady`, `isStarting`, `bootstrapProgress`, `bootstrapSummary`

## Nostr Protocol

**Role:** Secondary transport for private DMs to favorited/mutual peers when they are not reachable over BLE; also used for location-based presence (geohash channels).

**Implementation files:**
- `bitchat/Nostr/NostrProtocol.swift` — NIP-17 private DM construction (rumor → seal → gift-wrap)
- `bitchat/Nostr/NostrRelayManager.swift` — WebSocket connections to Nostr relays, managed via `URLSessionWebSocketTask` routed through `TorURLSession`
- `bitchat/Nostr/NostrIdentity.swift`, `NostrIdentityBridge.swift` — secp256k1 identity management
- `bitchat/Nostr/NostrEmbeddedBitChat.swift` — BitChat message embedding inside Nostr events
- `bitchat/Nostr/GeoRelayDirectory.swift` — geo-located relay directory for proximity routing
- `bitchat/Services/NostrTransport.swift` — `Transport` protocol conformance wrapping Nostr

**Nostr event kinds used:**
| Kind | Purpose |
|------|---------|
| 0 | Metadata |
| 1 | Text note |
| 13 | NIP-17 sealed event |
| 14 | NIP-17 DM rumor |
| 1059 | NIP-59 gift wrap |
| 20000 | Ephemeral event |
| 20001 | Geohash presence |

**Default relays** (hardcoded in `NostrRelayManager.swift`):
- `wss://relay.damus.io`
- `wss://nos.lol`
- `wss://relay.primal.net`
- `wss://offchain.pub`
- `wss://nostr21.com`

**Cryptography for Nostr:** secp256k1 Schnorr signatures via `P256K` (swift-secp256k1 0.21.1); XChaCha20-Poly1305 via custom HChaCha20 subkey derivation in `bitchat/Nostr/XChaCha20Poly1305Compat.swift` (built on top of CryptoKit's `ChaChaPoly`).

**Geo-relay directory:** Remote CSV fetched from `https://raw.githubusercontent.com/permissionlesstech/georelays/refs/heads/main/nostr_relays.csv` (via Tor); bundled fallback at `relays/online_relays_gps.csv`. Refresh interval controlled by `TransportConfig.geoRelayFetchIntervalSeconds`.

## Keychain

**Implementation:** `bitchat/Services/KeychainManager.swift`

**Service name:** `Bundle.main.bundleIdentifier` (e.g., `dev.abrahamgonzalez.chat.bitchat`)
**Access group:** `group.dev.abrahamgonzalez.chat.bitchat`

**What is stored:**
- Noise static Curve25519 key pair (`identity_noiseStaticKey`)
- Ed25519 signing key pair for peer authentication
- Nostr secp256k1 private key
- Identity cache encrypted with AES-GCM (in `SecureIdentityStateManager`)

**Accessibility:** `kSecAttrAccessibleWhenUnlocked` for all items

**Platform differences:**
- iOS: attempts access group first; falls back to no access group if entitlement missing (error -34018)
- macOS: always uses no access group; `kSecAttrSynchronizable = false` explicitly set

**Panic mode:** `deleteAllKeychainData()` performs comprehensive sweep across all known service names and app group to support emergency data wipe ("triple-tap logo" feature).

**Error classification:** `KeychainReadResult` and `KeychainSaveResult` enums distinguish `itemNotFound` (expected) from `accessDenied`, `deviceLocked`, `authenticationFailed`, and `otherError` states. Transient errors retry with exponential backoff (100ms, 200ms).

## Noise Protocol

**Implementation:** `bitchat/Noise/NoiseProtocol.swift`, `NoiseSession.swift`, `NoiseSessionManager.swift`, `NoiseEncryptionService.swift`

**Pattern:** Noise XX (mutual authentication, identity hiding)
- **DH:** Curve25519 (`Curve25519.KeyAgreement` from CryptoKit)
- **Cipher:** ChaCha20-Poly1305 (`ChaChaPoly` from CryptoKit)
- **Hash:** SHA-256 (`SHA256` from CryptoKit), HKDF via `HMAC<SHA256>`
- **Keys:** Static Curve25519 pair + Ed25519 signing pair, persisted in Keychain
- **Replay protection:** 1024-message sliding window nonce tracker

## CoreLocation

**Implementation:** `bitchat/Services/LocationStateManager.swift`, `GeohashPresenceService.swift`, `GeohashParticipantTracker.swift`

**Usage:**
- `CLLocationManager` wrapped behind `LocationStateManaging` protocol for testability
- Accuracy: `kCLLocationAccuracyHundredMeters` (coarse, privacy-preserving)
- Location encoded to geohash strings for proximity channel routing
- Permission state published via `@Published var permissionState` on `LocationChannelManager`
- iOS entitlement: `com.apple.security.personal-information.location` (macOS only; iOS handled via `NSLocationWhenInUseUsageDescription`)

## AVFoundation (Voice Notes)

**Implementation:** `bitchat/Features/voice/VoiceRecorder.swift`, `VoiceNotePlaybackController.swift`

**Usage:**
- `AVAudioRecorder` for voice note capture, runs on internal serial queue to avoid `AVAudioSession` contention
- `AVAudioPlayer` for playback
- `AVAudioSession.sharedInstance().requestRecordPermission` for microphone access (iOS)
- macOS entitlement: `com.apple.security.device.microphone`

## Image Handling

**Implementation:** `bitchat/Features/media/ImageUtils.swift`

**Frameworks used:** `CoreImage`, `CoreImage.CIFilterBuiltins`, `ImageIO`

**Pattern:**
- JPEG/PNG compression using `CGImageDestination` + `kCGImageDestinationLossyCompressionQuality`
- Default compression quality: 82%
- QR code generation via `CIFilter.qrCodeGenerator()` (CoreImage)

## Share Extension

**Target:** `bitchatShareExtension` (iOS only)
**Implementation:** `bitchatShareExtension/ShareViewController.swift`
**Framework:** UIKit (avoids deprecated `SLComposeServiceViewController`)

**Activation rules** (from `project.yml`):
- `NSExtensionActivationSupportsText: true`
- `NSExtensionActivationSupportsWebURLWithMaxCount: 1`
- `NSExtensionActivationSupportsImageWithMaxCount: 1`

**Data handoff:** Uses shared `UserDefaults` suite keyed on App Group ID (`group.dev.abrahamgonzalez.chat.bitchat`) to pass content from the extension to the main app. The group ID is read from `AppGroupID` in the extension's Info.plist.

**Entitlements:** App sandbox + App Group only (no Bluetooth, no network).

## Entitlements Summary

### iOS App (`bitchat/bitchat.entitlements`)
- `com.apple.security.app-sandbox`: true
- `com.apple.security.application-groups`: `group.dev.abrahamgonzalez.chat.bitchat`

### macOS App (`bitchat/bitchat-macOS.entitlements`)
- `com.apple.security.app-sandbox`: true
- `com.apple.security.application-groups`: `$(APP_GROUP_ID)`
- `com.apple.security.device.bluetooth`: true
- `com.apple.security.device.microphone`: true
- `com.apple.security.personal-information.location`: true
- `com.apple.security.network.client`: true
- `com.apple.security.network.server`: true
- `com.apple.security.files.user-selected.read-only`: true
- `com.apple.security.files.user-selected.read-write`: true
- `com.apple.security.assets.pictures.read-only`: true

### Share Extension (`bitchatShareExtension/bitchatShareExtension.entitlements`)
- `com.apple.security.app-sandbox`: true
- `com.apple.security.application-groups`: `group.dev.abrahamgonzalez.chat.bitchat`

## Local Storage

**UserDefaults:**
- Sent read receipts set (suite: standard)
- Tor enable/disable preference (`NetworkActivationService`)
- Location permission state
- Geo-relay cache last-fetched timestamp

**No remote database.** No CloudKit, no Firebase, no CoreData. All state is local and ephemeral or stored in Keychain/UserDefaults.

## URL Scheme

**Registered scheme:** `bitchat://` (declared in both iOS and macOS `CFBundleURLTypes`)

Used for deep-linking into specific contacts or channels.

---

*Integration audit: 2026-04-01*
