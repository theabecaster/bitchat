# Research Summary: bitchat PTT (Push-to-Talk)

**Synthesized:** 2026-04-01  
**Milestone:** Subsequent — adding real-time audio PTT to existing bitchat mesh messaging platform

---

## Executive Summary

Building PTT (walkie-talkie) over BLE mesh requires a fundamentally different architecture than bitchat's existing text and file-transfer services. Real-time audio streaming demands low latency (<200 ms end-to-end), bounded memory, and protocol-enforced half-duplex semantics. The recommended approach is:

1. **Capture & Encode:** Use AVAudioEngine with real-time tap + persistent AVAudioConverter (not batch AVAudioRecorder) to stream AAC fragments immediately as user speaks
2. **Transport:** Add new packet type (0x23, audio stream) with sequence numbering for reordering; skip compression (AAC is pre-compressed); use high BLE priority
3. **Receive & Buffer:** Small jitter buffer (20-50 ms, not 100+ ms) with aggressive timeout to keep latency under 200 ms; reorder by sequence number
4. **Half-Duplex Control:** Broadcast PTT_START/PTT_END signals to enforce channel lock (first transmitter wins); timeout-based recovery prevents deadlock on network glitches
5. **UI/UX:** Visual transmitting feedback + "on air" indicators are table stakes (without them, users think feature is broken)

**Key architectural insight:** PTT is a **streaming transport layer** (like a new message type), not a feature bolted onto VoiceRecorder. It requires five new components (PTTService, PTTChannelLock, PTTAudioBuffer, AudioStreamPlaybackController, updated VoiceRecorder) wired into ChatViewModel's state coordinator.

**Risk posture:** HIGH confidence on audio APIs (AVAudioEngine, AVAudioConverter are mature 2025 standards). MEDIUM confidence on BLE constraints (140 Kbps mesh, multi-hop jitter, no congestion feedback) — these will require physical device testing early. Critical pitfalls are well-documented in research; avoiding them demands disciplined phase 1 execution.

---

## Stack Recommendation

### Audio Capture (AVAudioEngine + Real-Time Tap)
- **Why:** AVAudioRecorder buffers entire recording until `stopRecording()` called (unacceptable for <200 ms latency). AVAudioEngine tap gives audio buffers as they arrive, callback-driven.
- **iOS setup:** AVAudioSession with `.playAndRecord` category, `.measurement` mode (strips echo cancellation), 5ms preferred buffer duration to reduce latency from default 20ms
- **macOS setup:** No AVAudioSession; direct AVAudioEngine configuration
- **Tap configuration:** 512-sample buffer (32 ms at 16 kHz), one encode per callback

### Real-Time AAC Encoding (AVAudioConverter with Persistent State)
- **Critical:** Reuse same AVAudioConverter instance across all frames. Recreating per frame adds 2112 frames (~130 ms) of silence priming, breaking latency.
- **Input:** PCM from tap
- **Output:** 16 Kbps AAC (existing VoiceRecorder.swift codec, hardware-accelerated, leaves 124 Kbps headroom)
- **Why NOT:** AVAssetWriter (file-oriented, overhead), Audio Toolbox C API (boilerplate), Opus (no hardware acceleration)

### Jitter Buffer (Ring Buffer with Sequence Reordering)
- **Structure:** O(1) insert/retrieve via sequence number mapping; no linked list or heap (memory allocation latency)
- **Size:** 20-50 ms target (see Pitfall 1: oversized buffers kill latency). Accept occasional silence over lag.
- **Flush logic:** PTT_END signal triggers immediate flush (primary); timeout (5-10 s) is fallback recovery on network glitch
- **Sequence handling:** Transmitter assigns sequence numbers (not mesh hop count or arrival order) to handle multi-hop delivery

### Why This Stack

| Component | Choice | Alternative | Why Not |
|-----------|--------|-------------|---------|
| Capture | AVAudioEngine tap | AVAudioRecorder | Batch mode has 500ms+ delay before playback |
| Encoding | AVAudioConverter | AVAssetWriter | File context overhead; requires CMSampleBuffer attachment keys |
| Codec | 16 Kbps AAC | Opus, FLAC | AAC hardware-accelerated; Opus needs libopus; FLAC is 8x bitrate |
| Jitter | Ring buffer | Linked list, heap | Fixed memory, O(1) operations, no GC pauses |

---

## Table Stakes Features

Features that cannot launch without (confirmed by FEATURES.md research):

1. **Hold-to-Talk Button UX**
   - Button held = transmitting, released = listening
   - Instant response (no startup delay) expected by users familiar with Telegram/Discord
   - Complexity: Low

2. **Visual Transmitting Feedback**
   - Button color change, "Live" badge, or animated icon while transmitting
   - Without this, users feel uncertain ("Is my mic on?") and repeatedly press button
   - Complexity: Low

3. **"On Air" Indicator for Peers**
   - Receiving users see who is transmitting right now (peer name + badge)
   - Prevents talking over each other in half-duplex
   - Complexity: Low

4. **Half-Duplex Channel Lock (First Transmitter Wins)**
   - Once Alice presses PTT, Bob's button is visually disabled or audio is silently dropped
   - Protocol enforces via PTT_START signal; secondary speakers are rejected
   - Complexity: Medium

5. **Lock Timeout / Dead Transmitter Recovery**
   - If PTT_END signal is lost (network glitch), channel auto-unlocks after 5-10s
   - Without this, network glitch = stuck channel = feature feels broken
   - Complexity: Medium

6. **Real-Time Fragment Streaming (Not Batch)**
   - Audio flows as user speaks; don't wait for recording to finish
   - Needed to hit <200 ms latency target
   - Complexity: Medium

7. **Sequence Numbers on Fragments**
   - Enable jitter buffer reordering (BLE mesh may deliver fragments out of order)
   - Also detects duplicates from different mesh hops
   - Complexity: Low

8. **Broadcast PTT (Unencrypted)**
   - Channel-wide PTT transmitted in clear, matching existing broadcast model
   - Directed PTT (encrypted via Noise session) deferred to v2
   - Complexity: Low

9. **No Audio Persistence**
   - Never log audio; never store to disk
   - Ephemeral in-memory only during playback
   - Complexity: Low

**Differentiators (nice-to-have, not blocking):**
- Level meter (waveform display during capture)
- Adaptive bitrate (detect congestion, reduce codec bitrate)
- Haptic feedback on transmit/receive
- Audio cues (beep on TX/RX)

**Anti-Features (explicitly NOT building):**
- Full-duplex simultaneous audio (BLE is half-duplex; mixing requires 2x bandwidth)
- Audio recording history (ephemeral by design; voice notes exist for async)
- Multiple simultaneous speakers (no mixing logic; first speaker locks channel)
- PTT over internet (latency budget doesn't translate; feature creep)
- Real-time noise suppression (16 Kbps AAC already lossy; over-processing degrades quality)

---

## Architecture Decisions

### Component Ownership

| Component | Layer | Responsibility | New/Modified | Phase |
|-----------|-------|-----------------|--------------|-------|
| **PTTService** | Services | Capture → encode → fragment → send; receive → jitter buffer → playback | New | 1 |
| **PTTChannelLock** | Services/State | Half-duplex state machine: track active transmitter, enforce "first wins", timeout recovery | New | 1 |
| **PTTAudioBuffer** | Services/Audio | Jitter buffer: reorder fragments by seq#, flush on PTT_END or timeout | New | 1 |
| **AudioStreamPlaybackController** | Features | Scheduled playback of jitter-buffered frames via AVAudioEngine | New | 2 |
| **BitchatProtocol** | Protocol | Add `case audioStream = 0x23` and PTT control signal enums | Modified | 1 |
| **BinaryProtocol** | Protocol | Add `encodeAudioStreamPacket()` and `decodeAudioStreamPacket()` | Modified | 1 |
| **BLEService** | Transport | Skip compression on audio packets; add audioStreamDelegate callback | Modified | 1 |
| **VoiceRecorder** | Audio | Add streaming mode (callback-based) alongside batch mode | Modified | 1 |
| **ChatViewModel** | ViewModel | Expose `@Published` PTT state: isTransmittingPTT, channelLockedBy | Modified | 1 |

### Packet Types & Protocol

- **New packet type:** 0x23 (audioStream) with structure: sessionID (8 bytes) + seqNum (4 bytes) + flags (1 byte) + payload
- **Control signals:** PTT_START (broadcast, includes timestamp + sender ID) and PTT_END (broadcast, includes sessionID) — separate from audio fragments
- **Channel lock resolution:** Simultaneous PTT_START resolved by (peerID XOR timestamp) tiebreak; losing peer's audio silently rejected
- **No compression:** Audio packets bypass zlib path (AAC pre-compressed)

### Data Flow

**Transmit Path:**
```
User holds PTT button
  ↓
ChatViewModel.startPTTTransmit()
  ↓
PTTService.startTransmit()
  ├─ Broadcast PTT_START (lock signal)
  ├─ Start VoiceRecorder in streaming mode
  └─ Store ephemeral sessionID
  ↓
VoiceRecorder.tap emits audio frame (continuously)
  ↓
PTTService.onAudioFrame(buffer)
  ├─ Encode PCM → AAC using persistent AVAudioConverter
  ├─ Wrap in audioStream packet (sessionID, seqNum++, payload)
  ├─ Route via Noise (directed) or plain (broadcast)
  └─ Send via BLEService (high priority, no compression)
  ↓
User releases button
  ↓
ChatViewModel.stopPTTTransmit()
  ├─ Stop VoiceRecorder
  └─ PTTService broadcasts PTT_END
```

**Receive Path:**
```
BLEService receives audioStream packet
  ↓
PTTService.onIncomingAudioFragment(packet)
  ├─ Check channel lock (accept if locked by sender, drop if locked by other)
  └─ Insert into PTTAudioBuffer
  ↓
PTTAudioBuffer.insertFragment(sessionID, seqNum, payload)
  ├─ Reorder by sequence number
  └─ On PTT_END signal or timeout: flush ordered frames
  ↓
AudioStreamPlaybackController.enqueue(decodedFrames)
  ├─ Decode AAC → PCM
  └─ Schedule playback via AVAudioEngine
  ↓
Speaker output
```

### Thread Safety Critical

- **Audio I/O thread isolation:** AVAudioEngine tap runs on high-priority audio I/O thread (NOT main, NOT BLE queues). Use lock-free queue to hand frames off to BLEService. Never call BLE methods from tap callback (risk of deadlock).
- **Pre-allocation:** Allocate audio buffers before PTT starts. Never allocate in real-time callback (blocking the audio thread = dropout).
- **State propagation:** PTTService state updates on its own queue. Propagate to ChatViewModel on main thread for UI binding.

### Build Order (4 Phases)

**Phase 1: Protocol & Core Infrastructure** (3-4 weeks)
- BitchatProtocol: Add 0x23 audioStream type, PTT_START/END signal enums
- BinaryProtocol: Encode/decode audio packets and control signals
- PTTChannelLock: State machine for half-duplex (zero external dependencies)
- VoiceRecorder: Add streaming mode callback alongside batch
- PTTAudioBuffer: Jitter buffer with sequence reordering + timeout
- PTTService: Main orchestrator (capture, encode, fragment, channel lock, receive, buffer, playback)
- ChatViewModel: Expose PTT state (@Published properties)
- Views: PTT button (hold-to-talk), on-air indicator, button lock state
- **Critical pitfalls to prevent:** #1 (buffer sizing), #2 (overflow/underflow), #3 (BLE congestion), #4 (channel lock deadlock), #10 (thread safety)

**Phase 2: Directed PTT & Recovery** (2-3 weeks)
- Directed PTT: Lazy Noise handshake before first audio
- AVAudioSession interruption handling (iOS): send PTT_END on interrupt, re-initialize on resume
- Adaptive jitter buffer: grow depth if underflows observed
- Level meter: waveform display during capture
- Memory monitoring: cap session size, log usage
- **Critical pitfalls:** #8 (interruption), #9 (memory pressure)

**Phase 3: Network Optimization** (2-3 weeks)
- Per-peer congestion metrics: backpressure tracking
- Packet Loss Concealment (PLC): synthesize missing frames
- Multi-hop latency awareness (optional)
- **Critical pitfalls:** #11 (graceful degradation)

**Phase 4: Advanced Features** (v2+)
- Voice Activity Detection (VAD)
- Voice call history / transcript
- Cross-channel coordination

---

## Watch Out For: Critical Pitfalls & Prevention

### Pitfall 1: Oversized Jitter Buffer (Latency Creep)
**Problem:** Copying VoIP buffers (100-200 ms) into <200 ms PTT target creates unacceptable lag. Conversation feels sluggish.  
**Root cause:** BLE jitter is smaller than cellular/internet; VoIP designs assume higher latency budget.  
**Prevention:** Target 20-50 ms buffer. Measure end-to-end latency (button press → speaker output) in Phase 1. Accept occasional silence over lag.  
**Detection:** User reports "PTT feels delayed like a delayed echo" or measurement shows >200 ms end-to-end on single hop.

### Pitfall 2: Jitter Buffer Overflow vs. Underflow (Wrong Tuning Kills Quality)
**Problem:** Too small → audio gaps on out-of-order delivery; too large → latency explodes. Mesh jitter varies; fixed sizing fails.  
**Prevention:** Implement adaptive sizing (grow buffer if underflows detected). Separate buffers per sessionID to prevent collision. Flush on timeout, don't wait forever.  
**Detection:** Audio plays smooth initially, then choppy; frequent gaps; receiver CPU climbs during PTT.

### Pitfall 3: BLE Write Queue Congestion (Silent Packet Loss)
**Problem:** Audio fragments queue faster than radio can send (poor RF, multiple peers). Writes succeed but packets drop silently (no error callback).  
**Root cause:** BLE link scheduler queues writes. When full, drops subsequent writes. Writes-without-response provide no feedback.  
**Prevention:** Monitor pending write latency before queuing. If >50 ms pending, drop frames. Backpressure tracking is mandatory.  
**Detection:** Audio cuts out at receiver; transmitter sees no errors. Problem correlates with high BLE traffic.

### Pitfall 4: Channel Lock Deadlock (Stuck After Network Glitch)
**Problem:** PTT_END lost → channel locked forever → no one can transmit.  
**Prevention:** Timeout is mandatory (5-10 s per radio industry standard). Retransmit PTT_END 3 times. Each peer's timeout is independent (no consensus).  
**Detection:** "Channel locked, nobody can transmit, waiting..." Measurement: drop PTT_END packets, verify auto-unlock within timeout.

### Pitfall 5: No Visual Feedback (User Confusion)
**Problem:** Button exists but users don't know if transmitting, who's on air, or why button is disabled.  
**Prevention:** Button color change (Live badge), peer name + "on air" indicator, disabled button with hint text ("Alice is speaking"). Not polish; core to viability.  
**Detection:** User feedback: "I don't know if it's working". Low feature adoption despite working implementation.

### Pitfall 6: No Silence Detection / Timeout Misalignment
**Problem:** Silence frames keep lock alive, or timeout during pauses mid-sentence.  
**Prevention:** PTT_END on button release is authoritative (not silence detection). Timeout is fallback only.  
**Detection:** Audio cuts mid-sentence or channel stays locked after release.

### Pitfall 7: Backpressure / Unbounded Transmit Buffer
**Problem:** Encoder output (50 frames/sec) vs. BLE send rate (200 bytes/sec) mismatch → app queue grows → latency explodes beyond target.  
**Prevention:** Cap transmit queue at 5 frames (~50 ms). Drop oldest on overflow. Measure app-side latency; must stay <50 ms.  
**Detection:** Latency far exceeds <200 ms target. Users hear significant delay.

### Pitfall 8: AVAudioSession Interruption (iOS)
**Problem:** Phone call during PTT → session suspended → PTT_END never sent → channel lock outlives transmission.  
**Prevention:** Handle `AVAudioSessionInterruptionNotification`. Send PTT_END on interrupt. Track `isAudioInterrupted` flag; disable button. Re-initialize audio on resume with error handling.  
**Detection:** Incoming call during PTT leaves channel locked. Siri activation breaks audio capture.

### Pitfall 9: Memory Pressure from Continuous Buffers
**Problem:** Jitter buffers + audio queues accumulate; triggers swapping, compression, or app termination.  
**Prevention:** Pre-allocate buffer pool. Cap jitter buffer at 20 frames max (not 100+ ms). Monitor memory; alert if session exceeds 2 MB.  
**Detection:** Memory usage grows continuously during long PTT. App becomes sluggish. Crash logs show SIGKILL.

### Pitfall 10: Thread Safety (Audio Callback Deadlock)
**Problem:** AVAudioEngine tap callback on audio I/O thread. Calling BLEService with locks → deadlock risk. Allocation in callback blocks audio thread.  
**Prevention:** Use lock-free queue between audio callback and BLE writer. Pre-allocate buffers. Never acquire locks in audio callback.  
**Detection:** Audio stutters while encoding. Occasional deadlock mid-PTT. Race condition crashes in BLEService.

### Pitfall 11: Graceful Congestion Degradation
**Problem:** BLE congestion → jitter buffer grows → latency exceeds 200 ms or audio gaps appear.  
**Prevention:** Implement adaptive frame dropping at 150 ms threshold. Packet Loss Concealment (interpolation) instead of abrupt silence. Log jitter metrics.  
**Detection:** Audio choppy initially then smooth; frequent gaps; receiver CPU climbs.

### Pitfall 12: Audio Data Leakage
**Problem:** Audio samples logged or persisted accidentally (crash logs, temp files, UserDefaults).  
**Prevention:** Never log audio buffers (log metrics only). No persistence to disk. Code review for leaks.  
**Detection:** Security audit finds audio in logs or UserDefaults. Crash reports contain audio samples.

---

## Recommended Phase Structure

### Phase 1: Core Protocol & Half-Duplex (Broadcast Only)

**Goal:** Minimal viable PTT — button pressed → audio heard by all peers on channel within <200 ms, one speaker at a time.

**Deliverables:**
- Protocol changes (BitchatProtocol, BinaryProtocol, audio packet types)
- PTTChannelLock service (state machine, no dependencies)
- VoiceRecorder streaming mode (callback-based capture)
- PTTAudioBuffer (jitter buffer with sequence reordering, timeout flush)
- PTTService (orchestrate capture → encode → fragment → send; receive → buffer → playback)
- ChatViewModel extensions (expose isTransmittingPTT, channelLockedBy)
- Views: PTT button, on-air indicator

**Critical pitfalls to prevent:** #1 (buffer sizing), #2 (overflow/underflow), #3 (BLE congestion), #4 (channel lock deadlock), #10 (thread safety)

**Testing:** Single-hop latency <200 ms, multi-hop jitter measurement, simultaneous text + audio, synthetic channel lock recovery

**Why first:** Protocol must exist before services. Half-duplex + broadcast is simplest correct behavior. Channel lock timeout mandatory (prevents stuck channel). Buffer sizing and backpressure essential for latency. Thread safety prevents crashes.

---

### Phase 2: Directed PTT & Recovery Resilience

**Goal:** Encrypt PTT for 1:1 sessions; handle real-world interruptions.

**Deliverables:**
- Directed PTT (lazy Noise handshake or fail gracefully)
- AVAudioSession interruption handling (iOS)
- Adaptive jitter buffer sizing
- Level meter (waveform during capture)
- Memory monitoring
- Audio cues (optional)

**Critical pitfalls:** #8 (interruption), #9 (memory pressure)

**Why second:** Broadcast PTT must work first. Encryption adds complexity. Interruption handling iOS-specific. Level meter is UX polish (high impact but not blocking).

---

### Phase 3: Network Optimization & Multi-Hop Awareness

**Goal:** Improve latency and quality on degraded networks.

**Deliverables:**
- Per-peer congestion metrics (backpressure tracking)
- Packet Loss Concealment (PLC)
- Multi-hop latency awareness (optional)

**Critical pitfall:** #11 (graceful degradation)

**Why third:** Optimization; deferred post-launch if Phase 1 + 2 hit latency targets.

---

### Phase 4: Advanced Features (Defer to v2+)

- Voice Activity Detection (VAD)
- Voice call history / transcript
- Priority speaker mode
- Cross-channel broadcasts

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| **Stack (Audio APIs)** | **HIGH** | AVAudioEngine, AVAudioConverter are mature 2025 standards (WWDC 2025). Hardware acceleration on modern iOS/macOS. Proven patterns in WebRTC, SIP implementations. |
| **Architecture (MVVM integration)** | **HIGH** | Follows bitchat's existing patterns. Component separation is clear. BLE integration mirrors text/file transfer. Service isolation prevents state coupling. |
| **Features (Table stakes)** | **HIGH** | Patterns from Discord, Telegram, traditional walkie-talkies well-established. Half-duplex model is simpler than attempted full-duplex. User expectations clear from field research. |
| **Pitfalls (12 identified)** | **HIGH** | Well-documented in VoIP/mobile audio literature. Prevention strategies are disciplined execution, not research unknowns. Most pitfalls discovered by community (WebRTC, mobile VoIP). |
| **BLE Constraints (140 Kbps, multi-hop jitter)** | **MEDIUM** | bitchat's mesh production-tested, but PTT's tight latency budget (<200 ms) is new. Physical device testing Phase 1 mandatory. Jitter buffer tuning likely needs iteration. |
| **Latency & Memory (Phase 1 delivery)** | **MEDIUM** | Confident hitting <200 ms single hop. Multi-hop latency and memory pressure under sustained load require measurement. Backpressure logic and buffer sizing tunable. |

**Gaps to Address During Roadmap:**

1. **Exact jitter buffer sizing:** Research recommends 20-50 ms; Phase 1 testing will refine per network condition
2. **Fragment size optimization:** Research uses 469-byte MTU; may need adjustment for multi-hop (smaller = more headers; larger = fewer retries)
3. **Real-world memory budget:** Phase 1 must include extended (5+ min) PTT sessions on physical iOS devices; simulator won't reveal memory pressure
4. **BLE write queue behavior:** Differs by iOS version, device (iPhone vs. iPad), role (central vs. peripheral); Phase 1 testing on target devices critical
5. **macOS audio latency:** No AVAudioSession; latency profile may differ from iOS; separate testing stream recommended

---

## Sources & Attribution

**Stack Research (STACK.md):**
- Apple WWDC 2025: "Enhance your app's audio recording capabilities"
- Apple Developer Documentation: AVAudioEngine, AVAudioConverter, AVAudioSession
- WebRTC NetEQ jitter buffer implementation
- Technical Note TN2258: AAC Audio — Encoder Delay and Synchronization

**Features Research (FEATURES.md):**
- Discord Voice Input Modes documentation
- Telegram Walkie-Talkie feature overview
- Push-to-Talk Wikipedia and radio industry standards (5-10 s timeout)
- Jitter buffer and latency optimization (GetStream, TRTC)

**Architecture Research (ARCHITECTURE.md):**
- bitchat codebase: MVVM patterns, BLEService design, ChatViewModel state coordination
- WebRTC architecture for jitter buffer and half-duplex channel control
- Reactive MVVM and Coordinator patterns

**Pitfalls Research (PITFALLS.md):**
- Real-time audio thread safety (Apple audio documentation, objc.io)
- BLE mesh constraints (Bluetooth latency characteristics, bitchat mesh topology)
- Memory pressure and jetsam on iOS
- AVAudioSession interruption handling (Apple documentation, community forums)

---

**Synthesis complete: 2026-04-01**  
**Ready for requirements definition and Phase 1 roadmap.**
