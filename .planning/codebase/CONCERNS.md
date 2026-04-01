# Codebase Concerns

**Analysis Date:** 2026-04-01

## Tech Debt

**God-class ChatViewModel:**
- Issue: `ChatViewModel` is a 3,889-line class conforming to `ObservableObject`, `BitchatDelegate`, `CommandContextProvider`, `GeohashParticipantContext`, and `MessageFormattingContext` simultaneously. It directly references `FavoritesPersistenceService.shared`, `LocationChannelManager.shared`, `GeoRelayDirectory.shared`, `VerificationService.shared`, and `NostrRelayManager.shared` — at least eight singleton dependencies called inline throughout its body.
- Files: `bitchat/ViewModels/ChatViewModel.swift`, `bitchat/ViewModels/Extensions/ChatViewModel+PrivateChat.swift` (1,060 lines), `bitchat/ViewModels/Extensions/ChatViewModel+Nostr.swift` (849 lines), `bitchat/ViewModels/Extensions/ChatViewModel+Tor.swift`
- Impact: Extremely difficult to unit-test any single responsibility in isolation. Extensions share mutable state freely with the core class because Swift extensions on the same type have no encapsulation boundary. Adding features consistently inflates this single object.
- Fix approach: Extract domain-specific coordinators (e.g., `GeoChannelCoordinator` already exists as a pattern); inject singleton dependencies via protocols at init time; split `BitchatDelegate` conformance into a dedicated adapter type.

**BLEService monolith (4,604 lines):**
- Issue: `BLEService` handles peer discovery, peripheral state, central state, fragment assembly, message deduplication, probabilistic relay, adaptive TTL, gossip-sync integration, and Noise handshake coordination in a single class.
- Files: `bitchat/Services/BLE/BLEService.swift`
- Impact: Any change to BLE behaviour requires navigating ~4,600 lines with three concurrent queues. The threading model (see below) becomes increasingly fragile as the file grows.
- Fix approach: Extract fragment assembly into `FragmentAssembler`, relay scheduling into `RelayScheduler`, and peer tracking into `PeerTracker` — each owning its own queue access.

**ContentView monolith (2,151 lines):**
- Issue: `ContentView.swift` is a single SwiftUI `View` of 2,151 lines. It directly binds to `ChatViewModel` via `@EnvironmentObject` without intermediate view models.
- Files: `bitchat/Views/ContentView.swift`
- Impact: Every `@Published` property change on `ChatViewModel` (29 published properties) invalidates the entire view hierarchy unless explicit `equatable` diffing is applied. Slow to compile, hard to preview individual sections.
- Fix approach: Decompose into focused child views each receiving only the data they need. Add `@Binding` pass-throughs or dedicated child view models.

**Singleton proliferation:**
- Issue: At least seven singletons called as `.shared` from inside `ChatViewModel`: `NostrRelayManager.shared`, `FavoritesPersistenceService.shared`, `LocationChannelManager.shared`, `GeoRelayDirectory.shared`, `VerificationService.shared`, `TorManager.shared`, `UIApplication.shared`.
- Files: `bitchat/ViewModels/ChatViewModel.swift`
- Impact: Makes dependency injection and unit testing impossible without method swizzling or global state mutation. Test isolation requires careful setUp/tearDown to reset shared state.
- Fix approach: Define protocol interfaces for each singleton and inject at `ChatViewModel` init; singletons remain but are hidden behind the protocol boundary.

---

## Known Bugs

**Incomplete private-chat message trim:**
- Symptoms: `trimMessagesIfNeeded()` is called in `ChatViewModel` but operates only on the public `messages` array. `PrivateChatManager.privateChatCap` (1,337) is defined but the trim call inside `ChatViewModel+PrivateChat.swift` is manually invoked, and at least one extension call path appends to `privateChats[peerID]` without triggering a cap check.
- Files: `bitchat/ViewModels/ChatViewModel.swift`, `bitchat/ViewModels/Extensions/ChatViewModel+PrivateChat.swift`, `bitchat/Services/PrivateChatManager.swift`
- Trigger: High-volume private chat sessions; the cap may silently not fire on paths that bypass `trimMessagesIfNeeded()`.
- Workaround: None; memory growth is bounded by app process lifetime.

---

## Security Considerations

**`DispatchQueue.main.sync` called from non-main BLE callbacks:**
- Risk: Two sites in `BLEService` call `DispatchQueue.main.sync { ... }` from within BLE callback contexts. If the main thread is itself blocked waiting on a BLE queue (e.g., via `collectionsQueue.sync`), this produces a deadlock, silently hanging the app.
- Files: `bitchat/Services/BLE/BLEService.swift` lines 275 and 2773
- Current mitigation: Both call sites guard with `Thread.isMainThread` before the sync, so the immediate risk requires a re-entrant path that bypasses the guard.
- Recommendations: Replace with `DispatchQueue.main.async` and handle any ordering requirements through completion closures or `Task { @MainActor in }` consistently.

**Fingerprint prefix logged at `.info` level:**
- Risk: `ChatViewModel` logs the first 8 characters of every loaded verified fingerprint in a space-separated string at `.info` severity. While truncated, this associates user identities with log output that may flow to crash reporters or system logs.
- Files: `bitchat/ViewModels/ChatViewModel.swift` line 2904–2905
- Current mitigation: `SecureLogger` category is `.security`; system log visibility depends on deployment configuration.
- Recommendations: Remove fingerprint prefix sampling from production log paths; restrict to `#if DEBUG` only.

**`try!` on static `NSRegularExpression` in production path:**
- Risk: Six `try!` calls on regex literals in `MessageFormattingEngine` and one in `MessageDeduplicationService` will crash the app with a fatal error if any pattern ever fails to compile (e.g., after a Swift regex engine change or copy-paste corruption).
- Files: `bitchat/Services/MessageFormattingEngine.swift` lines 42–74, `bitchat/Services/MessageDeduplicationService.swift` line 108
- Current mitigation: Patterns are all literals unlikely to change at runtime; risk is low in practice but non-zero under future refactors.
- Recommendations: Use `try?` with a fallback no-op regex, or assert in `#if DEBUG` and fall back gracefully in release.

**`as!` force cast in AVCaptureVideoPreviewLayer subclass:**
- Risk: `VerificationViews.swift` force-casts `layer` to `AVCaptureVideoPreviewLayer`. If the view is instantiated in a context where the layer class differs (e.g., SwiftUI preview, test host), the app crashes at the cast.
- Files: `bitchat/Views/VerificationViews.swift` line 273
- Current mitigation: View is only used inside the verification flow on physical devices.
- Recommendations: Add a `guard let` with a graceful failure path.

**Read receipts stored in UserDefaults:**
- Risk: `sentReadReceipts` (a set of message IDs) is persisted to `UserDefaults.standard` under the key `"sentReadReceipts"`. UserDefaults is not encrypted on-device and is included in unencrypted iCloud backups unless explicitly excluded.
- Files: `bitchat/ViewModels/ChatViewModel.swift` lines 367–369, 422
- Current mitigation: Message IDs alone do not expose content; timing metadata is exposed.
- Recommendations: Migrate to Keychain storage or add `NSFileProtectionComplete` attribute to the UserDefaults file; explicitly exclude from iCloud backup.

---

## Performance Bottlenecks

**29 `@Published` properties on a single ObservableObject:**
- Problem: `ChatViewModel` publishes 29 properties. SwiftUI re-renders any view observing the object whenever any property changes, even unrelated ones. High-frequency BLE events (peer updates, message arrivals) trigger broad re-renders of `ContentView` (2,151 lines).
- Files: `bitchat/ViewModels/ChatViewModel.swift`, `bitchat/Views/ContentView.swift`
- Cause: All state is co-located in one `ObservableObject` rather than split into focused observable objects.
- Improvement path: Introduce child `ObservableObject` instances (e.g., `PeerListViewModel`, `ComposeViewModel`) owned by `ChatViewModel`; bind child views to those instead of the root object.

**`collectionsQueue.sync(flags: .barrier)` called 61 times:**
- Problem: `BLEService` uses `collectionsQueue.sync(flags: .barrier)` in 61 locations. Barrier writes block all concurrent readers on a shared concurrent queue. Frequent barriers under active BLE mesh activity can serialise work that was intended to run concurrently.
- Files: `bitchat/Services/BLE/BLEService.swift`
- Cause: Shared mutable `peripherals`, `peerToPeripheralUUID`, and `peers` dictionaries all contend on the same `collectionsQueue`.
- Improvement path: Split collections into separate queues with narrower scope, or migrate to Swift actors (`actor BLEPeerTracker`).

**Fragment oldest-eviction is O(n):**
- Problem: When `incomingFragments` reaches capacity (128 entries), the oldest is found with `fragmentMetadata.min(by:)` — an O(n) linear scan over all in-flight assemblies, called from inside a barrier block.
- Files: `bitchat/Services/BLE/BLEService.swift` line 3633
- Cause: No sorted index maintained alongside the dictionary.
- Improvement path: Maintain a separate insertion-ordered list of keys alongside `fragmentMetadata` to make eviction O(1).

---

## Fragile Areas

**Multi-queue threading model in BLEService:**
- Files: `bitchat/Services/BLE/BLEService.swift`
- Why fragile: Three distinct queues (`bleQueue`, `collectionsQueue`, `messageQueue`) interlock through conditional `DispatchQueue.getSpecific` checks at call sites (lines 917–918, 943–944, 1580, 2894, 2957, 2964, 3577, 3716). The correctness of these paths depends on which queue a calling context is on, which is not enforced by the type system.
- Safe modification: Any new method that touches shared collections must explicitly confirm queue context via `getSpecific` or be always called from a known queue. Prefer `actor` isolation for new subsystems.
- Test coverage: `BLEServiceTests.swift` uses `MockBLEService` which does not exercise real queue interleaving; concurrency bugs require physical multi-peer testing.

**Noise session renegotiation path:**
- Files: `bitchat/Noise/SecureNoiseSession.swift`, `bitchat/Services/NoiseEncryptionService.swift`
- Why fragile: `SecureNoiseSession.needsRenegotiation()` is checked by a timer in `NoiseEncryptionService.checkSessionsForRekey()`. When the check fires, `onHandshakeRequired?` is called. However, if the remote peer is not reachable at that moment, the session simply enters `sessionExpired` or `sessionExhausted` error states on the next encrypt/decrypt call with no automatic retry or UI notification to the user.
- Safe modification: Always test renegotiation paths with simulated peer disconnection before the session reaches the threshold.
- Test coverage: `NoiseSecurityConstants.maxMessagesPerSession` is set to 1 billion — the timer-based renegotiation is the only practical path; it is not covered by automated tests for the unreachable-peer case.

**Private chat peer ID consolidation:**
- Files: `bitchat/Services/PrivateChatManager.swift`, `bitchat/ViewModels/Extensions/ChatViewModel+PrivateChat.swift`
- Why fragile: When a peer reconnects with a new ephemeral Noise key, `ChatViewModel` migrates `privateChats` from `oldPeerID` to `newPeerID` in multiple places (lines 940, 1485, 1512). If the same migration fires more than once (e.g., rapid reconnect), messages can be duplicated or lost. Consolidation logic walks `allPeers` by nickname match as a fallback.
- Safe modification: Any change to peer ID lifecycle or key rotation must audit all migration call sites.
- Test coverage: `ChatViewModelExtensionsTests.swift` covers happy-path consolidation; no tests for rapid reconnect races.

---

## Scaling Limits

**BLE mesh peer capacity:**
- Current capacity: `highDegreeThreshold` in `TransportConfig` governs adaptive TTL. The `subscribedCentrals` array and `peripherals` dictionary have no hard upper bound beyond OS-imposed BLE connection limits (typically 8–20 simultaneous connections on iOS).
- Limit: BLE throughput and latency degrade significantly beyond ~8 simultaneous connections; gossip sync flooding grows as O(peers × messages).
- Scaling path: Probabilistic relay logic already exists; tighten `messageTTL` and consider hierarchical clustering for large gatherings.

**Nostr relay reconnection:**
- Current capacity: `NostrRelayManager` maintains persistent WebSocket connections to all configured relays. Relay count is bounded by `GeoRelayDirectory` entries.
- Limit: Battery and network impact grows linearly with relay count; no circuit-breaker or adaptive relay selection based on latency is implemented.
- Scaling path: Add latency-based relay selection and a back-off circuit breaker for consistently failing relays.

---

## Dependencies at Risk

**`Tor` framework (iOS only):**
- Risk: The Tor framework is a heavyweight dependency (~20 MB binary) that significantly increases app size and startup time. It is conditionally started but always linked.
- Impact: App Store review sensitivity; entitlements required; background process constraints on iOS.
- Migration plan: No migration planned; evaluate whether Tor support frequency justifies always-linked inclusion vs. on-demand dynamic framework loading.

---

## Test Coverage Gaps

**Real BLEService concurrency:**
- What's not tested: The actual `BLEService` class under concurrent queue load. All BLE-layer tests use `MockBLEService` which is single-threaded.
- Files: `bitchatTests/BLEServiceTests.swift`, `bitchat/Services/BLE/BLEService.swift`
- Risk: Race conditions, deadlocks, and stale-state bugs in the three-queue model are invisible to the test suite.
- Priority: High

**Noise session renegotiation failure paths:**
- What's not tested: Behaviour when `onHandshakeRequired` fires but the peer is unreachable; session expiry during active send; concurrent rekey from both peers.
- Files: `bitchat/Noise/SecureNoiseSession.swift`, `bitchat/Services/NoiseEncryptionService.swift`
- Risk: Encrypted sessions may silently fail to renew, causing persistent send errors with no recovery path surfaced to the user.
- Priority: High

**ChatViewModel integration (UI layer):**
- What's not tested: The full delegate callback chain from `BLEService` through `ChatViewModel` to `@Published` state updates. Existing `ChatViewModelTests.swift` uses `MockTransport` and tests individual methods, not end-to-end delegate flows.
- Files: `bitchatTests/ChatViewModelTests.swift`, `bitchatTests/Integration/IntegrationTests.swift`
- Risk: Regressions in message routing, delivery status updates, and read receipt propagation can ship undetected.
- Priority: Medium

**Tor integration:**
- What's not tested: `ChatViewModel+Tor.swift` has corresponding `ChatViewModelTorTests.swift` but Tor startup/shutdown notifications are tested with mocks that do not exercise actual `TorManager` state transitions.
- Files: `bitchatTests/ChatViewModelTorTests.swift`, `bitchat/ViewModels/Extensions/ChatViewModel+Tor.swift`
- Risk: Tor enforcement mode (`torEnforced`) may incorrectly allow non-Tor traffic if lifecycle transitions are not correctly serialised.
- Priority: Medium

**ContentView smoke coverage only:**
- What's not tested: `ViewSmokeTests.swift` only instantiates views and checks they don't crash. No interaction tests, no state-driven rendering tests.
- Files: `bitchatTests/ViewSmokeTests.swift`, `bitchat/Views/ContentView.swift`
- Risk: UI regressions in message rendering, private chat switching, and peer list updates are not caught before release.
- Priority: Low

---

*Concerns audit: 2026-04-01*
