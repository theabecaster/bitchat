# Architecture Research: PTT Integration

**Research Date:** 2026-04-01  
**Milestone Context:** Subsequent — adding PTT to existing bitchat MVVM stack  
**Overall Confidence:** HIGH (based on existing codebase analysis + verified patterns)

---

## Executive Summary

PTT audio streaming integrates into bitchat's architecture as a **new service layer component** alongside existing Services (BLEService, NoiseEncryptionService). The architecture follows MVVM principles: PTTService owns all audio state and transport, ChatViewModel serves as the state coordinator exposing PTT UI properties, and Views consume via `@EnvironmentObject` bindings.

**Key architectural insight:** PTT is a **streaming transport protocol** (like fileTransfer but real-time), not a message type. It requires:
1. A new packet type (0x23, audio stream fragment)
2. A new Service to manage capture → fragment → send and receive → jitter buffer → playback
3. Minimal ChatViewModel additions (mostly property exposure, not business logic)
4. Protocol-enforced channel lock semantics to prevent collisions in a mesh

The critical difference from fileTransfer: **streaming is live and continuous** (fragments sent while button is held), not batch-and-send after recording. This drives the architecture choice of separating capture timing from fragment pipelining.

---

## Component Map

### New Components Needed

| Component | Layer | Responsibility | Depends On | Created in Phase |
|-----------|-------|----------------|------------|------------------|
| **PTTService** | Services | Audio capture (via VoiceRecorder), fragment generation, transmission, channel lock state | BLEService, NoiseEncryptionService, VoiceRecorder | 1 |
| **PTTAudioBuffer** | Services/Audio | Receive-side jitter buffer: reorder fragments by seq#, enforce PTT_END timeout, expose decoded audio frames | — | 1 |
| **PTTChannelLock** | Services/Audio | Broadcast/enforce half-duplex channel lock: owns PTT_START/PTT_END signaling, tracks activeTransmitter, rejects secondary senders | — | 1 |
| **PTTAudioCodec** (optional) | Services/Audio | Encode AAC fragments in real-time, decode incoming fragments. Wraps AVAudioEncoder/Decoder | AVFoundation | 2 |
| **AudioStreamPlaybackController** | Features/audio | Scheduled playback of jitter-buffered audio frames via AVAudioEngine | AVFoundation | 2 |

### Existing Components to Modify

| Component | Current Role | Change Needed | Rationale |
|-----------|-------------|---------------|-----------|
| **BitchatProtocol.swift** | Packet type enum | Add `case audioStream = 0x23` and packet constants (seq#, session ID, flags) | New packet type for audio fragments |
| **BinaryProtocol.swift** | Encode/decode packets | Add `encodeAudioStreamPacket()` and `decodeAudioStreamPacket()` functions | Binary wire format for audio fragments |
| **BLEService.swift** | Transport (send/receive) | Skip compression on audio packets; add `audioStreamDelegate` callback | Audio bypass compression path; notify PTTService of incoming streams |
| **ChatViewModel.swift** | State coordinator | Add `@Published var isTransmittingPTT`, `channelLockedBy`, `pttSessions` | Expose PTT state to Views |
| **VoiceRecorder.swift** | Voice note recording | Add `realTimeFrameBuffer` option alongside batch mode | Support streaming capture for PTT (existing batching still used for voice notes) |

---

## Data Flow

### Transmit Path: Button Press → BLE Write

```
1. User holds PTT button in View
   ↓
2. ChatViewModel.startPTTTransmit() called
   ↓
3. PTTService.startTransmit()
   ├─ Broadcasts PTT_START (lock signal, channel lock type)
   ├─ Starts VoiceRecorder in streaming mode (not batch)
   └─ Stores session ID (ephemeral, unique per transmission)
   ↓
4. VoiceRecorder emits audio frames (continuous callback)
   ↓
5. PTTService.onAudioFrame(bytes)
   ├─ Encodes AAC chunk (if needed, or pass-through from VoiceRecorder)
   ├─ Wraps in BitchatPacket(type: .audioStream, sessionID, seqNum++, payload)
   ├─ Route: Noise encrypted (if directed) or plain (if broadcast)
   └─ Sends via BLEService (no compression, high priority)
   ↓
6. BLEService.send(packet, skipCompression: true)
   ├─ Fragments if >469 bytes (audio is continuous, so fragmentation expected)
   ├─ Queues with OutboundPriority.high (audio > text, but maybe fileTransfer higher)
   └─ Writes via CBCharacteristic to connected centrals
   ↓
7. User releases button
   ↓
8. ChatViewModel.stopPTTTransmit()
   ├─ Stops VoiceRecorder
   ├─ PTTService.stopTransmit()
   │  ├─ Broadcasts PTT_END (unlock signal)
   │  └─ Clears session state
   └─ ChatViewModel updates isTransmittingPTT = false
```

**Key constraint:** Fragments sent **while button is held**, not after recording. VoiceRecorder must provide a frame callback (not just URL-based batch).

---

### Receive Path: BLE Write → Jitter Buffer → Playback

```
1. BLEService receives characteristic write with audioStream packet
   ↓
2. BLEService.didReceiveAudioStreamPacket(packet)
   ├─ Calls delegate: BitchatDelegate.didReceiveAudioStream(...)
   └─ ChatViewModel consumes event
   ↓
3. ChatViewModel → PTTService.onIncomingAudioFragment(packet)
   ↓
4. PTTService checks channel lock
   ├─ If not locked: accept, store in PTTAudioBuffer
   ├─ If locked by other sender: drop silently (log at .debug)
   └─ If locked by this sender: accept (echo suppression)
   ↓
5. PTTAudioBuffer.insertFragment(sessionID, seqNum, payload)
   ├─ Store by session ID (allows interleaved sessions, though we reject secondary)
   ├─ Reorder by sequence number
   └─ On receive PTT_END or timeout (5s): flush ordered frames
   ↓
6. PTTAudioBuffer emits readyFrames (or frameReadyPublisher)
   ↓
7. AudioStreamPlaybackController.enqueue(frames)
   ├─ Decompress AAC (or pass if already decoded)
   ├─ Schedule playback via AVAudioEngine sink
   └─ Continues until PTT_END or timeout
   ↓
8. Speaker output: audio plays as it arrives (subject to jitter buffer latency)
```

**Critical detail:** Jitter buffer must flush on PTT_END signal, not wait for timeout. Timeout is fallback only.

---

### Channel Lock Flow: PTT_START/PTT_END

```
1. Local user holds PTT button
   PTTService broadcasts: PTT_START { channelID, senderID, sessionID, timestamp }
   ↓
2. All peers receive PTT_START
   ├─ PTTChannelLock.onPTTStart(sender, sessionID)
   ├─ If channel is free: lock it to this sender, store sessionID
   ├─ If already locked: ignore (sender loses floor)
   └─ If locked by self: accept (allow own echo)
   ↓
3. During transmission
   ├─ Incoming audio from locked sender: accept and buffer
   ├─ Incoming audio from other peer: reject silently
   ├─ Views show "On air: {senderName}" indicator
   └─ Peers cannot transmit (button sends audio but BLEService filters locally)
   ↓
4. Local user releases button
   PTTService broadcasts: PTT_END { channelID, sessionID }
   ↓
5. All peers receive PTT_END
   ├─ PTTChannelLock.onPTTEnd(sessionID)
   ├─ Verify sessionID matches locked sender
   ├─ Flush jitter buffer immediately (don't wait for timeout)
   ├─ Unlock channel
   └─ Clear "On air" indicator
   ↓
6. Channel now free for next transmission
```

**Implementation detail:** PTT_START and PTT_END are **control signals broadcast unencrypted** (like announcements), separate from audio fragments. They carry just IDs and timestamps — no audio data.

---

## State Ownership

| State | Owner | Consumers | Thread Safety |
|-------|-------|-----------|---------------|
| **isTransmitting** | PTTService | ChatViewModel → View | @Published (main thread) |
| **channelLocked** | PTTChannelLock | ChatViewModel, PTTService | DispatchQueue.concurrent with barrier |
| **activeTransmitter** | PTTChannelLock | ChatViewModel (shows "On air: name") | Main thread via ChatViewModel |
| **pttSessions** | PTTService | PTTAudioBuffer, ChatViewModel (session tracking) | Service-local queue |
| **jitterBuffer** | PTTAudioBuffer | AudioStreamPlaybackController | Concurrent read, barrier write |
| **audioFrames** (decoded) | AudioStreamPlaybackController | AVAudioEngine | Main thread + render callbacks |
| **currentSessionID** | PTTService | BLEService (in packet headers) | Atomic (value type) |
| **receiverState** | PTTAudioBuffer | View (shows waveform while listening) | Published via ChatViewModel |

**Key principle:** PTTService is the single owner of PTT-specific state. ChatViewModel exposes read-only views (via `@Published`) for the UI layer. All mutations happen in PTTService on its own queue, then propagated to ChatViewModel on main thread for UI updates.

---

## Interface Contracts

### PTTService Public Interface

```swift
final class PTTService {
    // Configuration
    var pttSessionTimeout: TimeInterval = 5.0  // Flush jitter buffer if no frames for 5s
    var audioFrameEncoding: AudioEncoding = .aac  // .aac or .pcm
    
    // Transmit API
    func startTransmit(channelID: ChannelID) async throws
    func stopTransmit() async throws
    func onAudioFrame(_ pcmBuffer: AVAudioPCMBuffer) // Called by VoiceRecorder
    
    // Receive API
    func onIncomingAudioFragment(_ packet: BitchatPacket) async
    func onPTTControl(_ signal: PTTControlSignal)  // PTT_START, PTT_END
    
    // State queries (safe to call from any thread)
    var isTransmitting: Bool { get }
    var channelLockedBy: PeerID? { get }
    var activeSessionID: String? { get }
    
    // Published properties for ChatViewModel binding
    @Published var isTransmittingPTT: Bool
    @Published var channelIsLocked: Bool
    @Published var lockedByName: String?  // "Alice is on air"
}

// Delegate for audio events
protocol PTTServiceDelegate: AnyObject {
    func pttService(_ service: PTTService, didStartReceivingFrom peer: PeerID)
    func pttService(_ service: PTTService, didEndReceiving from peer: PeerID)
    func pttService(_ service: PTTService, frameReadyForPlayback: Data, sessionID: String)
}
```

### ChatViewModel Additions

```swift
// In ChatViewModel
@Published var isTransmittingPTT: Bool = false
@Published var channelLockedBy: PeerID? = nil
@Published var lockedByPeerName: String? = nil
@Published var pttReceiverWaveform: [Float] = []  // For visual feedback

// New delegate method
func didReceiveAudioStream(_ packet: BitchatPacket, from sender: PeerID) {
    pttService.onIncomingAudioFragment(packet)
}

func didReceivePTTControl(_ signal: PTTControlSignal) {
    pttService.onPTTControl(signal)
    // Update UI state
    lockedByName = chatViewModel.displayName(for: signal.senderID)
}
```

### BLEService Delegate Extension

```swift
// Existing BitchatDelegate protocol (extend in BitchatProtocol.swift)
extension BitchatDelegate {
    optional func didReceiveAudioStream(_ packet: BitchatPacket, from sender: PeerID)
    optional func didReceivePTTControl(_ signal: PTTControlSignal, from sender: PeerID)
}
```

### VoiceRecorder Streaming Mode

```swift
extension VoiceRecorder {
    // Add to existing API
    typealias AudioFrameCallback = (AVAudioPCMBuffer) -> Void
    
    func startStreamingRecord(callback: @escaping AudioFrameCallback) throws -> URL
    // Called continuously while button is held, not just at end
    
    func stopStreamingRecord() async throws -> URL
    // Finalize, return URL (optional, for backup/fallback)
}
```

---

## Build Order & Dependencies

### Phase 1: Protocol & Core Infrastructure

**1.1 Update BitchatProtocol.swift**
- Add `case audioStream = 0x23` to `MessageType`
- Define audio stream packet structure: `sessionID (8 bytes), seqNum (4 bytes), flags (1 byte), payload`
- Add `PTTControlSignal` enum: `.start(channelID, senderID)`, `.end(sessionID)`
- Define control packet constants

**1.2 Update BinaryProtocol.swift**
- Add `encodeAudioStreamPacket(sessionID, seqNum, payload) -> Data`
- Add `decodeAudioStreamPacket(data) -> (sessionID, seqNum, payload)?`
- Add control signal encode/decode

**1.3 PTTChannelLock (new service, no BLE dependency)**
- Pure state machine: `onPTTStart`, `onPTTEnd`, `canTransmit(peerID)` checks
- Thread-safe (barrier queue)
- **No external dependencies** — can be tested in isolation

**1.4 Update BitchatDelegate in BitchatProtocol**
- Add optional methods: `didReceiveAudioStream`, `didReceivePTTControl`
- ChatViewModel adopts these optional methods

**Why first:** Protocol changes must exist before any service can encode/decode. PTTChannelLock has zero runtime dependencies, making it safe to add before audio systems.

---

### Phase 2: Audio Capture & Streaming

**2.1 Update VoiceRecorder.swift**
- Add streaming mode alongside batch mode
- Add `startStreamingRecord(callback:)` that calls callback for each audio frame
- Reuse existing AVAudioRecorder setup (session, settings, permissions)
- **Backward compatible:** Voice notes (batch) unchanged

**2.2 PTTAudioBuffer (new service)**
- Receives fragments by (sessionID, seqNum)
- Reorders and stores
- Exposes `readyFrames` publisher or callback
- Flushes on PTT_END signal or timeout
- **Dependency:** BitchatProtocol (for packet structure)

**2.3 PTTService (new, main service)**
- Orchestrates transmit: capture → fragment → BLE send
- Orchestrates receive: incoming fragments → jitter buffer → playback
- Owns channel lock state (delegates to PTTChannelLock)
- **Dependencies:** BLEService, NoiseEncryptionService, PTTChannelLock, PTTAudioBuffer, VoiceRecorder

**Why second:** Audio systems depend on protocol being defined and PTTChannelLock logic being available. VoiceRecorder modifications are minimal (add callback option).

---

### Phase 3: Playback & UI Integration

**3.1 AudioStreamPlaybackController (new feature)**
- Receives ready frames from PTTAudioBuffer
- Schedules playback via AVAudioEngine or AVAudioPlayer
- Handles real-time scheduling (don't buffer all at once)
- **Dependency:** AVFoundation, PTTAudioBuffer

**3.2 ChatViewModel Extensions**
- Bind `PTTService.isTransmitting` → `@Published var isTransmittingPTT`
- Bind `PTTChannelLock.activeTransmitter` → `@Published var channelLockedBy`
- Implement delegate methods for audio events
- **Dependency:** PTTService

**3.3 Views: PTT Button & Indicators**
- PTT button in ContentView (prominent, hold-to-talk)
- "On air: {name}" indicator
- Optional: waveform visualization during capture
- **Dependency:** ChatViewModel (read `isTransmittingPTT`, `channelLockedBy`)

**Why third:** UI depends on services being complete. Playback can be deferred further if necessary (silent mode first, playback second).

---

### Phase 4: Optimization & Polish

**4.1 Audio Codec Optimization**
- If direct AAC encoding from VoiceRecorder is slow, introduce PTTAudioCodec service
- Pre-encode frames on encoder queue, reduce real-time latency

**4.2 Level Meter / Waveform**
- Integrate `metering` from AVAudioRecorder into VoiceRecorder
- Publish levels to ChatViewModel
- Render in real-time during transmit

**4.3 Adaptive Jitter Buffer**
- Monitor RTT and packet loss
- Adjust buffer depth dynamically (currently fixed at ~100ms)

**4.4 Battery & Scanning Optimization**
- Ensure BLE scan interval is not suppressed during PTT receive
- Test battery drain under load

**Why last:** Polish improves UX but doesn't block core functionality. Core PTT works without optimizations.

---

## Integration Risks

### Risk 1: BLE Fragment Collisions (Audio + Text)

**Problem:** If text message fragments and audio fragments interleave on the BLE characteristic, reassembly breaks.

**Mitigation:**
- Use separate packet type constants (0x02 for text, 0x20 for fragment, 0x23 for audio)
- BLEService already demultiplexes by packet type before reassembly
- Audio fragments are self-identifying (sessionID + seqNum), so reordering is idempotent

**Test:** Send text + audio simultaneously, verify both reconstruct correctly.

---

### Risk 2: Jitter Buffer Latency

**Problem:** Jitter buffer adds latency. If buffer is too large, >200ms latency violates PTT UX expectation.

**Mitigation:**
- Default buffer size: 100ms (roughly 2-4 audio frames at 16kHz)
- Flush on PTT_END immediately (don't wait for timeout)
- Use sequence numbers, not arrival time, for reordering (tolerates bursty fragments)
- Monitor RTT via control signals; skip frames if >300ms behind

**Test:** Measure end-to-end latency (capture → BLE write → local decode → speaker) in single-hop and multi-hop scenarios.

---

### Risk 3: Channel Lock Race (Simultaneous PTT_START)

**Problem:** Two peers broadcast PTT_START simultaneously. Who wins?

**Mitigation:**
- Add `timestamp` to PTT_START packet
- All peers apply consistent tiebreak: lower (peerID XOR timestamp) wins
- Losing peer's audio is silently rejected
- Allows clear "floor control" semantics

**Test:** Synthetic simultaneous start from two peers; verify one is rejected consistently.

---

### Risk 4: VoiceRecorder Callback Timing

**Problem:** AVAudioRecorder callback timing is unpredictable. May fire too slowly, causing gaps in transmission, or too fast, overwhelming BLE write queue.

**Mitigation:**
- Buffer frames in PTTService (queue size = 1-2 frames, drop oldest if queue full)
- Don't backpressure VoiceRecorder; silently drop frames under congestion
- Real-time streaming is better with gaps than with queuing

**Test:** Capture high-frequency events while text is being sent; verify audio is still sent without stalling text.

---

### Risk 5: NoiseEncryption Handshake for Directed PTT

**Problem:** If there's no Noise session with peer yet, starting directed PTT will trigger handshake. User might start talking before session is ready.

**Mitigation:**
- Broadcast PTT only (unencrypted) in v1; defer directed PTT to v2
- OR: Start lazy handshake immediately when user holds button (don't wait for first audio)
- OR: Fail gracefully with "Handshake in progress, please wait" message

**Decision:** Phase 1 uses broadcast only. Directed PTT deferred to Phase 2.

---

### Risk 6: Echo Suppression

**Problem:** In half-duplex, if a peer is transmitting, they might hear their own audio echoed back from the mesh.

**Mitigation:**
- Add `echoSuppression: Bool` flag to PTTService
- If transmitting, mute receive locally
- OR: PTTChannelLock prevents self from being "secondary transmitter" (see lock flow)

**Decision:** Allow peers to hear their own echo (standard in walkie-talkie), but suppress if UX testing shows it's confusing.

---

## Layering Diagram

```
View Layer
├─ ContentView (PTT button, hold-to-talk)
├─ OnAirIndicator (shows "Alice is transmitting")
└─ AudioWaveform (visual feedback during capture)
         ↓
ViewModel Layer
├─ ChatViewModel (@Published isTransmittingPTT, channelLockedBy)
└─ Delegate: onIncomingAudioFragment, onPTTControl
         ↓
Services Layer
├─ PTTService (main orchestrator)
│  ├─ Depends: BLEService, NoiseEncryptionService, PTTChannelLock
│  └─ Publishes: isTransmitting, lockedByPeer
├─ PTTChannelLock (state machine, half-duplex enforcer)
├─ PTTAudioBuffer (jitter buffer, fragment reorder)
└─ AudioStreamPlaybackController (playback scheduling)
         ↓
Protocol/Encoding Layer
├─ BitchatProtocol (audioStream packet type 0x23, PTT_START/END)
└─ BinaryProtocol (encode/decode audio packets)
         ↓
Crypto Layer
└─ NoiseEncryptionService (for directed PTT, Phase 2)
         ↓
Transport Layer
└─ BLEService (send/receive audio fragments, skip compression)
         ↓
Audio Capture Layer
├─ VoiceRecorder (streaming mode + batch mode)
└─ AVAudioSession (hardware configuration)
```

---

## Key Design Principles

1. **Real-time over reliability:** Drop stale frames, don't queue them. Jitter buffer is small and bounded.

2. **Protocol-enforced semantics:** PTT_START/PTT_END broadcast signals ensure all peers agree on channel lock state, not just sender's expectation.

3. **Half-duplex first:** Don't attempt mixing or simultaneous transmission. Simplest correct behavior: first transmitter wins.

4. **Service isolation:** PTTService owns all PTT state. ChatViewModel is a consumer of PTTService state, not a participant in it.

5. **No persistence:** Audio is never logged, never stored, never backed up. Ephemeral streaming only.

6. **Backward compatible:** Existing text and file transfer packet types unchanged. New 0x23 type is additive.

---

## Test Strategy

### Unit Tests
- **PTTChannelLock:** Simulate concurrent PTT_START from two peers, verify tiebreak
- **PTTAudioBuffer:** Insert out-of-order fragments, verify reorder and flush logic
- **BinaryProtocol:** Encode/decode audio packets, verify round-trip

### Integration Tests
- **Single-hop:** Capture audio, send, receive, play on same device pair
- **Multi-hop:** Three devices, one transmits, one relays, one receives; measure latency
- **Simultaneous text + audio:** Send text while PTT is transmitting, verify no interference
- **Handshake:** Start PTT with peer lacking Noise session; verify broadcast fallback works

### E2E Tests
- **Multi-user scenario:** Four devices, two attempting PTT simultaneously, verify lock fairness
- **Battery:** Monitor BLE scan interval and power during 5-minute PTT session
- **Congestion:** Saturate BLE with text, ensure PTT still transmits without queueing

---

## Sources

- [How WebRTC's NetEQ Jitter Buffer Provides Smooth Audio](https://webrtchacks.com/how-webrtcs-neteq-jitter-buffer-provides-smooth-audio/)
- [Jitter Buffer Implementation in PJSIP](https://docs.pjsip.org/en/latest/specific-guides/audio/jitter_buffer.html)
- [iOS AVAudioSession for Real-Time Audio](https://developer.apple.com/forums/thread/25197)
- [Audio API Overview (objc.io)](https://www.objc.io/issues/24-audio/audio-api-overview/)
- [Push-to-Talk Protocol Design](https://www.intercomsonline.com/2-way-radio-push-to-talk-ptt)
- [Reactive MVVM Pattern](https://betterprogramming.pub/reactive-mvvm-and-the-coordinator-pattern-done-right-88248baf8ca5)

---

**Last Updated:** 2026-04-01  
**Next Steps:** Initiate Phase 1 to implement protocol changes and PTTChannelLock service.
