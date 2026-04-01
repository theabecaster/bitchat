# Requirements: bitchat PTT — Push-to-Talk Walkie-Talkie

**Defined:** 2026-04-01
**Core Value:** A user holds the PTT button and their voice reaches nearby peers in under 200ms — no internet, no server, no latency from buffering.

## v1 Requirements

### Protocol

- [ ] **PROTO-01**: New audio stream packet type (0x23) defined in BitchatProtocol.swift with session ID (UUID), sequence number (UInt16), and payload fields
- [ ] **PROTO-02**: PTT_START control signal broadcast when transmitter begins — carries session ID and sender peer ID
- [ ] **PROTO-03**: PTT_END control signal broadcast when transmitter stops — carries session ID
- [ ] **PROTO-04**: BinaryProtocol encodes and decodes audio stream packets (encode to wire format, decode from incoming fragments)
- [ ] **PROTO-05**: Audio fragments bypass the zlib compression path in BLEService (compression flag not set for 0x23 packets)

### Channel Lock

- [ ] **LOCK-01**: PTTChannelLock state machine: idle → locked (on PTT_START) → idle (on PTT_END or timeout)
- [ ] **LOCK-02**: First-transmitter wins: peers receiving PTT_START while channel is idle lock the channel to that sender
- [ ] **LOCK-03**: Secondary PTT_START signals from other peers rejected while channel is locked
- [ ] **LOCK-04**: Auto-unlock timeout: if PTT_END is not received within 10 seconds of last audio fragment, channel unlocks automatically (prevents deadlock on mesh packet loss)
- [ ] **LOCK-05**: Transmitter who sent PTT_START can cancel early via PTT_END before timeout

### Audio Capture & Transmit

- [ ] **CAPT-01**: AVAudioEngine with installTap on input node — streams audio buffers in real-time as they are captured (does not wait for recording to complete)
- [ ] **CAPT-02**: Real-time AAC encoding at 16 Kbps, 16kHz mono using persistent AVAudioConverter (not recreated per-buffer — avoids 130ms silence penalty)
- [ ] **CAPT-03**: Encoded AAC frames fragmented into ≤469-byte chunks and written to BLE characteristic continuously while button is held
- [ ] **CAPT-04**: Audio tap callback enqueues fragments via lock-free queue to BLE write layer (audio runs on real-time thread; direct BLEService calls would deadlock)
- [ ] **CAPT-05**: AVAudioSession configured for .playAndRecord with .measurement mode and 5ms preferredIOBufferDuration (iOS only, via #if os(iOS))
- [ ] **CAPT-06**: Stream session ID (UUID) generated at PTT_START; sequence number increments per fragment within a session
- [ ] **CAPT-07**: Broadcast PTT transmits to the current channel (unencrypted, same routing as broadcast text)
- [ ] **CAPT-08**: Directed PTT transmits to a specific peer using the existing Noise-encrypted session for that peer

### Jitter Buffer & Receive

- [ ] **RECV-01**: PTTAudioBuffer accumulates incoming fragments keyed by (sessionID, sequenceNumber) and reorders out-of-order arrivals
- [ ] **RECV-02**: Jitter buffer target depth 20-50ms; flushes immediately on PTT_END signal rather than waiting for timeout
- [ ] **RECV-03**: Stale frames dropped rather than buffered indefinitely — if a fragment arrives after the stream has been flushed, it is discarded
- [ ] **RECV-04**: Decoded AAC frames scheduled for playback via AVAudioEngine output node as they arrive and are ordered
- [ ] **RECV-05**: BLE scanning interval not suppressed during PTT receive mode (audio delivery is timing-sensitive)
- [ ] **RECV-06**: Incoming audio from locked transmitter accepted; audio from non-transmitter peers rejected while channel is locked

### BLE Integration

- [ ] **BLE-01**: BLEService routes incoming 0x23 packets to PTTService delegate callback (separate from text/file packet handling)
- [ ] **BLE-02**: BLEService write queue monitors congestion; drops oldest audio fragment (not newest) when queue depth exceeds threshold
- [ ] **BLE-03**: Text messaging characteristic writes not starved by audio fragment writes when PTT is active

### UI

- [ ] **UI-01**: PTT button visible in chat view; hold-to-talk interaction (press and hold to transmit, release to stop)
- [ ] **UI-02**: Button visually disabled (and PTT_START rejected) when channel is locked by another peer
- [ ] **UI-03**: "On air" indicator displayed when a remote peer is transmitting — shows who is transmitting (peer display name or ID)
- [ ] **UI-04**: Level meter or waveform animation shown during active transmission so the user knows the mic is hot
- [ ] **UI-05**: "On air" indicator clears when PTT_END received or channel lock timeout expires

### Safety & Privacy

- [ ] **SAFE-01**: No audio data written to logs at any point in the transmit or receive pipeline
- [ ] **SAFE-02**: No audio stream content persisted to disk (ephemeral in-memory only)
- [ ] **SAFE-03**: AVAudioSession interruption (phone call, Siri, other audio app) immediately sends PTT_END and releases channel lock

### Platform

- [ ] **PLAT-01**: All AVFoundation capture APIs that differ between iOS and macOS gated with #if os(iOS) / #if os(macOS) guards
- [ ] **PLAT-02**: Feature functional on both iOS 16+ (physical device) and macOS 13+

## v2 Requirements

### Polish & Resilience

- **POLSH-01**: Audio beep/haptic confirmation on PTT button press (transmit start) and release
- **POLSH-02**: Adaptive jitter buffer depth — grows when underflows are detected, shrinks when network stabilizes
- **POLSH-03**: Packet Loss Concealment (PLC) — brief silence insertion rather than dropout on lost fragment
- **POLSH-04**: User-configurable channel lock timeout (default 10s; range 5-30s)

### Advanced Features

- **ADV-01**: Voice Activity Detection (VAD) — auto-release PTT button if no voice detected for >2s (accidental holds)
- **ADV-02**: Multi-hop latency awareness — display estimated hop count to transmitting peer
- **ADV-03**: PTT volume control independent of system volume

## Out of Scope

| Feature | Reason |
|---------|--------|
| Full-duplex two-way voice conversation | BLE mesh half-duplex; latency budget doesn't allow mixing |
| Audio recording / playback history | Voice notes already cover async audio; PTT is ephemeral by design |
| Multiple simultaneous PTT speakers | First-transmitter wins; no mixing, no queue, no preemption |
| PTT over Nostr / internet relay | BLE-only for v1; internet relay latency violates the <200ms target |
| Background PTT receive (app backgrounded) | CoreBluetooth BLE background modes add complexity; defer |
| PTT over non-mesh (point-to-point BT classic) | Out of scope for bitchat's architecture |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| PROTO-01 | Phase 1 | Pending |
| PROTO-02 | Phase 1 | Pending |
| PROTO-03 | Phase 1 | Pending |
| PROTO-04 | Phase 1 | Pending |
| PROTO-05 | Phase 1 | Pending |
| LOCK-01 | Phase 1 | Pending |
| LOCK-02 | Phase 1 | Pending |
| LOCK-03 | Phase 1 | Pending |
| LOCK-04 | Phase 1 | Pending |
| LOCK-05 | Phase 1 | Pending |
| CAPT-01 | Phase 2 | Pending |
| CAPT-02 | Phase 2 | Pending |
| CAPT-03 | Phase 2 | Pending |
| CAPT-04 | Phase 2 | Pending |
| CAPT-05 | Phase 2 | Pending |
| CAPT-06 | Phase 2 | Pending |
| CAPT-07 | Phase 2 | Pending |
| CAPT-08 | Phase 3 | Pending |
| RECV-01 | Phase 2 | Pending |
| RECV-02 | Phase 2 | Pending |
| RECV-03 | Phase 2 | Pending |
| RECV-04 | Phase 2 | Pending |
| RECV-05 | Phase 2 | Pending |
| RECV-06 | Phase 2 | Pending |
| BLE-01 | Phase 1 | Pending |
| BLE-02 | Phase 2 | Pending |
| BLE-03 | Phase 2 | Pending |
| UI-01 | Phase 3 | Pending |
| UI-02 | Phase 3 | Pending |
| UI-03 | Phase 3 | Pending |
| UI-04 | Phase 3 | Pending |
| UI-05 | Phase 3 | Pending |
| SAFE-01 | Phase 2 | Pending |
| SAFE-02 | Phase 2 | Pending |
| SAFE-03 | Phase 2 | Pending |
| PLAT-01 | Phase 2 | Pending |
| PLAT-02 | Phase 3 | Pending |

**Coverage:**
- v1 requirements: 35 total
- Mapped to phases: 35
- Unmapped: 0 ✓

---
*Requirements defined: 2026-04-01*
*Last updated: 2026-04-01 after initial definition*
