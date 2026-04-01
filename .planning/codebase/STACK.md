---
generated: 2026-04-01
focus: tech
---

# Technology Stack

**Analysis Date:** 2026-04-01

## Languages

**Primary:**
- Swift 5.0 — all application code across iOS, macOS, and the share extension
  - `SWIFT_VERSION: 5.0` set in all targets in `project.yml`
  - Swift concurrency used extensively: `async/await`, `Task`, `@MainActor`, `@Sendable`, `actor`-isolation annotations

**Secondary:**
- Rust (compiled as `arti.xcframework`) — Tor client via the Arti library, shipped as a pre-built binary XCFramework in `localPackages/Arti/Frameworks/`
- C/C++ — minimal FFI bridge between Swift and the Rust static library; C headers are in `localPackages/Arti/Sources/C/`

## Runtime

**Environment:**
- iOS 16.0+ (physical device required; Bluetooth unavailable in Simulator)
- macOS 13.0+ (Ventura)

**Swift Tools Version:**
- SPM: swift-tools-version 5.9 (in `Package.swift` and both local packages)

## Package Manager

**Primary:** XcodeGen + SPM hybrid
- `project.yml` is the canonical project definition. Never edit `.xcodeproj` directly.
- `xcodegen generate` regenerates `bitchat.xcodeproj` from `project.yml`
- `Package.swift` at repo root declares SPM dependencies consumed by both targets
- Lockfile: `Package.resolved` — present and committed (pins `swift-secp256k1` to revision `8c62aba`)

## Build Tooling

**XcodeGen:**
- Config: `project.yml`
- Generates `bitchat.xcodeproj` from YAML spec
- Must be run after any `project.yml` change (`xcodegen generate`)

**Justfile (macOS only):**
- `just run` — build + launch the macOS app
- `just build` — build only (calls `xcodebuild -scheme "bitchat (macOS)"`)
- `just clean` — clean DerivedData and restore git-tracked files
- `just nuke` — nuclear clean: remove all artifacts and regenerate
- Signing disabled during local macOS builds: `CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO`

**Xcode:**
- Schemes: `bitchat (iOS)` and `bitchat (macOS)`
- iOS builds embed the `bitchatShareExtension` target
- Development team: `R2PVQ496X7`

## Frameworks (Apple)

| Framework | Purpose |
|-----------|---------|
| `SwiftUI` | Primary UI framework; used across all views and view models |
| `Combine` | Reactive state propagation; `@Published` properties, `AnyPublisher` pipelines |
| `CoreBluetooth` | BLE mesh networking (central + peripheral roles) |
| `CryptoKit` | `Curve25519`, `ChaChaPoly`, `AES.GCM`, `SHA256`, `HMAC<SHA256>` |
| `Security` | Keychain access (`SecItemAdd`, `SecItemCopyMatching`, `SecItemDelete`) |
| `CoreLocation` | Location for geohash proximity channels (`CLLocationManager`) |
| `AVFoundation` | Voice note recording (`AVAudioRecorder`) and playback (`AVAudioPlayer`) |
| `Network` | Path monitoring via `NWPathMonitor` in `TorManager.swift` |
| `UserNotifications` | Push/local notification delivery |
| `Compression` | ZLIB payload compression via `COMPRESSION_ZLIB` algorithm |
| `UniformTypeIdentifiers` | MIME type identification for media (`UTType`) |
| `UIKit` | iOS-only UI interop (`UIImagePickerController`, `UIApplicationDelegate`) |
| `AppKit` | macOS-only UI interop (`NSApplicationDelegate`) |
| `CommonCrypto` | Supplemental hashing in `ChatViewModel.swift` |
| `CoreImage` / `CoreImage.CIFilterBuiltins` | Image processing and QR code generation |
| `ImageIO` | Low-level image encoding (JPEG/PNG compression in `ImageUtils.swift`) |

## SPM Dependencies

**External (remote):**

| Package | Version | Source | Purpose |
|---------|---------|--------|---------|
| `swift-secp256k1` (`P256K`) | `0.21.1` (exact) | `https://github.com/21-DOT-DEV/swift-secp256k1` | secp256k1 elliptic curve; Schnorr signing for Nostr NIP-17 |

**Local packages (`localPackages/`):**

| Package | Products | Purpose |
|---------|---------|---------|
| `Arti` | `Tor` library | Swift wrapper around the Arti Rust binary (`arti.xcframework`); exposes `TorManager`, `TorURLSession`, `TorNotifications`; links `libresolv`, `libz`, `libsqlite3` |
| `BitLogger` | `BitLogger` library | Structured `os.log`-backed logger (`SecureLogger`); defines OSLog subsystem `chat.bitchat` with categories: `noise`, `encryption`, `keychain`, `session`, `security`, `handshake`, `sync` |

## Deployment Targets

| Platform | Minimum Version |
|----------|----------------|
| iOS | 16.0 |
| macOS | 13.0 (Ventura) |

Both targets are built from a shared `bitchat/` source tree using `#if os(iOS)` / `#if os(macOS)` guards for platform-specific code.

## Configuration

**Build configurations:** Debug / Release (Xcode standard)

**Debug-only differentiation:**
- BLE service UUID differs between debug (`…4B5A`) and release (`…4B5C`) to separate testnet from mainnet
- `BITCHAT_DEV_ALLOW_CLEARNET` compile flag disables Tor enforcement in development builds

**App identifiers:**
- Bundle ID: `dev.abrahamgonzalez.chat.bitchat`
- App Group: `group.dev.abrahamgonzalez.chat.bitchat`
- Share Extension Bundle ID: `dev.abrahamgonzalez.chat.bitchat.ShareExtension`

**Localization:**
- `.xcstrings` catalog at `bitchat/Localizable.xcstrings`
- `String(localized:comment:)` API used throughout (Swift 5.7+ style)
- Share extension has its own `Localization/` subfolder

---

*Stack analysis: 2026-04-01*
