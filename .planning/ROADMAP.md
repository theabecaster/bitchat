# Roadmap: bitchat PTT v1.0

**Milestone:** v1.0 PTT  
**Goal:** Add push-to-talk (walkie-talkie) functionality to bitchat. Users hold a button to transmit voice over the BLE mesh to peers in under 200ms, with half-duplex semantics (first transmitter wins, others hear but cannot transmit).

**Overall Strategy:** Protocol layer first (foundation), then audio capture and jitter buffering (services), then UI and broadcast PTT end-to-end, then directed PTT (encrypted). Each phase completes one verifiable capability.

---

## Phases

- [ ] **Phase 1: Protocol & Core Infrastructure** - Define audio stream packet types, channel lock state machine, and jitter buffer foundation
- [ ] **Phase 2: Audio Capture & Streaming Pipeline** - Real-time AAC encoding and fragment transmission; receive-side jitter buffering and playback
- [ ] **Phase 3: Broadcast PTT UI & End-to-End** - PTT button, on-air indicators, and complete broadcast PTT flow (MVP)
- [ ] **Phase 4: Directed PTT & Recovery** - Encrypted PTT for 1:1 sessions, interruption handling, and resilience

---

## Phase Details

### Phase 1: Protocol & Core Infrastructure

**Goal:** Establish the protocol foundation and half-duplex channel lock mechanism so services can coordinate transmit/receive semantics.

**Depends on:** Nothing (foundation phase)

**Requirements covered:** PROTO-01, PROTO-02, PROTO-03, PROTO-04, PROTO-05, LOCK-01, LOCK-02, LOCK-03, LOCK-04, LOCK-05, BLE-01

**Deliverables:**

- New audio stream packet type (0x23) in BitchatProtocol.swift with sessionID, sequence number, and payload fields
- PTT_START and PTT_END control signals broadcast when transmission begins/ends
- BinaryProtocol encode/decode functions for audio stream packets and control signals
- PTTChannelLock state machine: idle → locked → idle with timeout recovery (5-10s auto-unlock on deadlock)
- VoiceRecorder streaming mode callback alongside existing batch recording
- PTTAudioBuffer jitter buffer: stores fragments by (sessionID, seqNum), reorders by sequence, flushes on PTT_END or timeout
- BLEService integration: route incoming 0x23 packets to PTTService, skip compression on audio
- ChatViewModel extensions: expose `@Published` properties for PTT state (isTransmittingPTT, channelLockedBy)

### Plans

1. **Update BitchatProtocol.swift** — Add `case audioStream = 0x23` to MessageType enum; define audio packet structure (sessionID: 8 bytes, seqNum: 4 bytes, flags: 1 byte, payload); add PTTControlSignal enum for PTT_START/PTT_END
2. **Implement BinaryProtocol encode/decode** — Add `encodeAudioStreamPacket()` and `decodeAudioStreamPacket()` functions; add control signal serialization
3. **Build PTTChannelLock state machine** — Implement first-transmitter-wins semantics, timeout recovery, and tiebreak resolution for simultaneous PTT_START from two peers
4. **Update VoiceRecorder** — Add `startStreamingRecord(callback:)` that emits frames continuously (callback-driven, not batch); reuse existing AVAudioRecorder session setup
5. **Implement PTTAudioBuffer** — Ring buffer for jitter buffer: O(1) insert/retrieve by sequence number, flush on PTT_END signal or timeout, separate buffers per sessionID
6. **Extend BLEService** — Route incoming audioStream packets to PTTService delegate; set compression flag to false for audio (AAC is pre-compressed)
7. **Wire ChatViewModel** — Add `@Published var isTransmittingPTT`, `@Published var channelLockedBy` properties; implement BitchatDelegate methods for audio events

**Success Criteria:**

- [ ] New audio stream packet type (0x23) encodes/decodes without error; round-trip test via BinaryProtocol succeeds
- [ ] PTT_START and PTT_END control signals broadcast and are decoded correctly by all peers
- [ ] PTTChannelLock state machine tested: simultaneous PTT_START from two peers resolves consistently (one wins, one rejected)
- [ ] Channel lock auto-unlocks after 5-10s if PTT_END is not received (timeout recovery tested)
- [ ] VoiceRecorder streaming mode calls callback continuously; does not batch-wait for recording to complete
- [ ] PTTAudioBuffer inserts fragments out-of-order and reorders correctly by sequence number
- [ ] BLEService routes audioStream packets separately from text/file packets; compression is skipped for 0x23
- [ ] ChatViewModel state changes trigger UI updates (verified with logging or manual testing)

**Dependencies:** None

---

### Phase 2: Audio Capture & Streaming Pipeline

**Goal:** Implement real-time audio capture, encoding, and transmission; receive-side buffering and playback scheduling.

**Depends on:** Phase 1 (protocol and lock state machine must exist)

**Requirements covered:** CAPT-01, CAPT-02, CAPT-03, CAPT-04, CAPT-05, CAPT-06, CAPT-07, RECV-01, RECV-02, RECV-03, RECV-04, RECV-05, RECV-06, BLE-02, BLE-03, SAFE-01, SAFE-02, SAFE-03, PLAT-01

**Deliverables:**

- PTTService main orchestrator: manages capture start/stop, encodes AAC in real-time, fragments into 469-byte chunks, sends via BLEService with no compression
- AVAudioEngine with real-time tap on input node: 512-sample buffer (~32ms at 16kHz), callback-driven
- Persistent AVAudioConverter for 16 Kbps AAC encoding (reuse same instance to avoid silence penalty on recreation)
- Lock-free queue from audio tap callback to BLE write layer (thread safety critical: audio runs on real-time thread)
- Transmit path: User holds button → broadcast PTT_START → capture frames → encode → fragment → send continuously
- Receive path: Incoming audioStream packet → check channel lock → insert into jitter buffer → on PTT_END flush → schedule playback
- AudioStreamPlaybackController: decode AAC frames and schedule via AVAudioEngine
- AVAudioSession configuration iOS-only: `.playAndRecord` category, `.measurement` mode, 5ms preferredIOBufferDuration (via #if os(iOS))
- Incoming audio from locked transmitter accepted; audio from secondary transmitters rejected while channel is locked
- Text messaging queue not starved by audio (separate priorities or backpressure handling in BLEService)
- No audio data in logs; no stream content persisted to disk
- BLE scanning interval not suppressed during PTT receive

### Plans

1. **Create PTTService** — Coordinate capture (start/stop), encode, fragment, send; own channel lock state; implement PTTServiceDelegate callbacks for audio lifecycle
2. **Integrate AVAudioEngine with real-time tap** — Install tap on input node; configure 512-sample buffer; emit frames to callback continuously
3. **Implement real-time AAC encoding** — Create persistent AVAudioConverter (reuse across frames); encode PCM to 16 Kbps AAC without silence penalty
4. **Build fragment pipeline** — Wrap encoded AAC in audioStream packets (sessionID, seqNum++), apply no compression, queue to BLEService
5. **Implement thread-safe audio callback** — Use lock-free queue from audio I/O thread to BLEService write layer; never call BLE methods from tap callback (deadlock risk)
6. **Wire receive path** — On incoming audioStream, check PTTChannelLock; if locked by sender, insert into PTTAudioBuffer; if locked by other, reject silently
7. **Create AudioStreamPlaybackController** — Receive ordered frames from jitter buffer, decode AAC to PCM, schedule playback via AVAudioEngine
8. **Implement backpressure & congestion handling** — Drop oldest audio frame (not newest) when BLEService write queue depth exceeds threshold; accept stale frames
9. **Add AVAudioSession interruption handling** — iOS only: listen for `AVAudioSessionInterruptionNotification`; on interrupt, send PTT_END and disable button; re-initialize on resume
10. **Verify no audio leakage** — Code review for audio buffers in logs, temp files, or UserDefaults; confirm only metrics are logged

**Success Criteria:**

- [ ] User holds PTT button; audio captured at 16 Kbps AAC, 16kHz mono, 512-sample buffer (~32ms)
- [ ] Encoded frames fragmented into ≤469-byte chunks and sent via BLE without stalling
- [ ] Sequence number increments per fragment within a session; fragments can be reordered by sequence number
- [ ] Receiving peer sees incoming audioStream packets routed separately (not mixed with text/file)
- [ ] Jitter buffer reorders out-of-order fragments and flushes on PTT_END or timeout (5-10s)
- [ ] Audio decoded and scheduled for playback; no gaps in output during single-hop transmission
- [ ] Incoming audio from secondary transmitter rejected while channel is locked (verified with two peers)
- [ ] Text messaging does not stall while audio is transmitting (e.g., send text while holding PTT button)
- [ ] BLE scan interval measured; confirmed not suppressed during receive
- [ ] Incoming call or Siri interrupt during PTT sends PTT_END and unlocks channel (iOS)
- [ ] No audio buffers found in logs via grep; no stream content written to disk (verified by file audit)
- [ ] Single-hop latency measured: capture → BLE write → local decode → playback under 200ms (physical device test)

**Dependencies:** Phase 1

---

### Phase 3: Broadcast PTT UI & End-to-End

**Goal:** Complete the broadcast PTT feature end-to-end: button UI, on-air indicators, and full MVP workflow where users hold button, hear peers, and see who is transmitting.

**Depends on:** Phase 2 (audio services must work)

**Requirements covered:** UI-01, UI-02, UI-03, UI-04, UI-05, PLAT-02

**Deliverables:**

- PTT button in ChatView: prominent placement, hold-to-talk interaction (press and hold to transmit, release to stop)
- Visual transmitting feedback: button color change or "Live" badge while user is holding button
- "On air" indicator: displays when a remote peer is transmitting (shows peer display name or ID)
- Level meter or waveform animation during active transmission: user sees visual feedback that mic is hot
- Disabled button state: when channel is locked by another peer, button is visually disabled with hint text
- "On air" indicator clears on PTT_END signal or timeout expiration
- Feature functional on both iOS 16+ (physical device) and macOS 13+
- Platform-specific AVFoundation APIs gated with #if os(iOS) / #if os(macOS)

### Plans

1. **Design PTT button in ContentView** — Add button with hold-to-talk gesture (onLongPressGesture or similar); style for prominence
2. **Implement transmit feedback** — Button color changes to "Live" or animated badge while isTransmittingPTT is true
3. **Add on-air indicator** — Display peer name/ID when channelLockedBy is not nil; animated icon
4. **Build level meter UI** — Optional: animated waveform or bar graph reflecting audio levels during capture
5. **Implement button disabling** — When channelIsLocked and lockedByName != self, disable button and show hint ("Alice is on air, wait your turn")
6. **Clear on-air on PTT_END** — Subscribe to PTTChannelLock state; clear indicator when unlock event fires
7. **Test iOS + macOS** — Verify button interaction works on both platforms; audio capture permissions differ

**Success Criteria:**

- [ ] PTT button present in chat view and visually prominent
- [ ] Hold-to-talk interaction: pressing button transmits, releasing stops (tested on physical device)
- [ ] Button color changes to "Live" or animated while transmitting (isTransmittingPTT = true)
- [ ] "On air" indicator displays peer name when remote peer is locked transmitter
- [ ] Indicator clears when PTT_END received or timeout expires
- [ ] Button disabled/hint shown when channel locked by another peer (tested with two peers)
- [ ] Level meter or waveform visible during transmit (visual feedback that mic is active)
- [ ] User holds button → remote peer hears audio within 200ms (end-to-end test, physical device)
- [ ] Second peer cannot transmit while first peer holds button (tested on physical device)
- [ ] Button and indicators work on both iOS and macOS without crashes

**Dependencies:** Phase 2

---

### Phase 4: Directed PTT & Recovery

**Goal:** Add encrypted PTT for 1:1 sessions, improve resilience to network interruptions, and fine-tune latency and audio quality.

**Depends on:** Phase 3 (broadcast PTT MVP must work first)

**Requirements covered:** CAPT-08

**Deliverables:**

- Directed PTT: transmit to a specific peer using existing Noise-encrypted session for that peer
- Lazy Noise handshake before first audio: if no Noise session exists, initiate immediately on button press (don't wait for first fragment)
- Graceful fallback: if Noise session unavailable, fallback to broadcast (not blocking)
- Adaptive jitter buffer: grow depth if underflows detected, shrink when network stabilizes
- Packet Loss Concealment (PLC): interpolate missing frames rather than abrupt silence
- Memory monitoring: cap session size and log usage; clear buffers aggressively
- Level meter refinement: smooth waveform animation
- Optional: audio cues (beep) on transmit/receive start
- Latency and memory profiling under sustained load (5+ minute PTT sessions)

### Plans

1. **Implement directed PTT** — Check if Noise session exists for target peer; if yes, encrypt audio fragments; if no, initiate lazy handshake or fallback to broadcast
2. **Add lazy Noise handshake** — On user selecting "directed PTT" mode, start handshake immediately (don't wait for first audio)
3. **Implement adaptive jitter buffer** — Monitor underflows; grow buffer depth on underflow detection, shrink on stability; cap at max 200ms
4. **Add Packet Loss Concealment** — Detect missing sequence numbers; interpolate silence or repeat last frame rather than dropout
5. **Memory monitoring** — Track session memory usage; log warnings if >2MB; implement aggressive cleanup on memory pressure
6. **Profile latency & memory** — Sustained load test (5+ min PTT); measure end-to-end latency, memory, and CPU on physical devices
7. **Optional: Audio cues** — Add beep or haptic feedback on transmit start (accessibility + confirmation)

**Success Criteria:**

- [ ] Directed PTT transmits to specific peer with Noise encryption (tested with pre-established Noise session)
- [ ] Lazy Noise handshake initiated on button press if session missing; audio transmitted after handshake completes
- [ ] Graceful fallback to broadcast if Noise session fails
- [ ] Jitter buffer grows on underflow detection; shrinks on stability (measured via buffer depth metric)
- [ ] Packet Loss Concealment: missing frames filled with interpolated silence or repeated frame (not abrupt dropout)
- [ ] Memory usage stays under 2MB during sustained 5-minute PTT session (physical device test)
- [ ] Latency under 200ms maintained during sustained load
- [ ] Audio quality acceptable during packet loss (PLC prevents robotic/choppy playback)
- [ ] Beep/haptic feedback on button press (optional, accessibility feature)

**Dependencies:** Phase 3

---

## Milestone Gate: v1.0 PTT

**Complete when:**

- [ ] Broadcast PTT works end-to-end: user holds button, remote peer hears audio within 200ms (single-hop, physical device)
- [ ] Channel lock enforced: second transmitter cannot interrupt active PTT (tested on two devices simultaneously)
- [ ] Latency target achieved: <200ms on single-hop BLE connection (measured via capture → playback)
- [ ] No audio in logs; no audio persisted to disk (verified by grep/file audit)
- [ ] Feature functional on both iOS 16+ and macOS 13+ (tested on physical devices)
- [ ] Text messaging unaffected when PTT idle (simultaneous text + audio sends without stalling)
- [ ] Multi-hop delivery confirmed (three devices: A transmits, B relays, C receives)
- [ ] Channel lock timeout recovery tested: PTT_END loss triggers auto-unlock within 10s
- [ ] AVAudioSession interruption handled: incoming call during PTT sends PTT_END and unlocks
- [ ] User confidence high: feature is intuitive (hold button, hear peers, see who is on air)

**Then:** `/gsd:complete-milestone`

---

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Protocol & Core Infrastructure | 0/7 | Not started | — |
| 2. Audio Capture & Streaming Pipeline | 0/10 | Not started | — |
| 3. Broadcast PTT UI & End-to-End | 0/7 | Not started | — |
| 4. Directed PTT & Recovery | 0/7 | Not started | — |

---

**Roadmap created:** 2026-04-01  
**Granularity:** Standard (4 phases, 7-10 plans each)  
**Requirement coverage:** 35/35 v1 requirements mapped ✓
