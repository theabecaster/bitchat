# PTT Build Order Rationale

**Research Date:** 2026-04-01  
**Purpose:** Justify the phased build order for PTT integration

---

## The Build Order (Revisited)

```
Phase 1: Protocol & Lock Mechanism (weeks 1-2)
├─ 1.1: BitchatProtocol + BinaryProtocol changes
├─ 1.2: PTTChannelLock service
├─ 1.3: Update BLEService to demux audio packets
└─ 1.4: Update BitchatDelegate

Phase 2: Streaming & Fragment Pipeline (weeks 3-4)
├─ 2.1: VoiceRecorder streaming mode
├─ 2.2: PTTAudioBuffer (jitter buffer)
├─ 2.3: PTTService (main orchestrator)
└─ 2.4: Update ChatViewModel

Phase 3: Playback & UI (weeks 5-6)
├─ 3.1: AudioStreamPlaybackController
└─ 3.2: Views (PTT button, indicators)

Phase 4: Polish & Optimization (weeks 7+)
├─ 4.1: Level meter / waveform
├─ 4.2: Adaptive jitter buffer
├─ 4.3: Directed PTT (Noise encryption)
└─ 4.4: Battery & performance tuning
```

---

## Why This Order?

### Phase 1: Protocol First

#### 1.1 Why BitchatProtocol Before Everything Else

**Dependency depth:** Protocol is at the foundation. Cannot write encode/decode, cannot fragment, cannot send/receive without protocol definition.

```
View ← ViewModel ← Service ← Protocol ← Code can compile
```

**What gets unblocked:**
- All services can be written (once they compile, even if not functional)
- Tests can verify encode/decode round-trips
- BLE transport layer can route audio packets

**Risk of skipping:** If protocol isn't finalized, all downstream work becomes rework.

**Example:**
```swift
// Can't write this until MessageType.audioStream exists
func onIncomingPacket(_ type: MessageType, _ data: Data) {
    switch type {
    case .audioStream:  // ← Must exist in enum
        handleAudioStream(data)
    // ...
    }
}
```

**Confidence:** HIGH — existing project uses clear protocol layering. We're following the existing pattern.

---

#### 1.2 Why PTTChannelLock Before Services

**Isolation:** PTTChannelLock has **zero dependencies** on audio hardware, networking, or other services.

```
PTTChannelLock:
  ├─ Input: PTT_START, PTT_END signals
  ├─ State: isLockedBy, sessionID, tiebreakTimestamp
  ├─ Output: canTransmit(peer), lockedBy()
  └─ Dependencies: none ← Can be tested immediately
```

**Why early:** 
- It's the simplest, most isolated component
- Can be tested with synthetic inputs (no hardware)
- Unblocks other services to depend on it
- Gets highest confidence (well-tested before anything uses it)

**Test example:**
```swift
// Pure unit test, no mocks needed
func testSimultaneousPTTStart() {
    let lock = PTTChannelLock()
    
    // Alice and Bob both start at same time, Alice has lower ID
    lock.onPTTStart(sender: alice, timestamp: now, sessionID: "ABC")
    lock.onPTTStart(sender: bob, timestamp: now, sessionID: "XYZ")
    
    XCTAssertEqual(lock.lockedBy, alice)  // Tiebreak wins
    XCTAssertEqual(lock.canTransmit(bob), false)  // Bob can't transmit
}
```

**Risk of skipping:** If logic is buggy, every downstream service that depends on it is buggy. Early testing = high confidence.

---

#### 1.3 Why BLEService Changes Now, Not Later

**Current behavior:** BLEService compresses all packets (zlib).  
**Problem:** Audio fragments don't compress well and lose time to compression.  
**Solution:** Add flag to skip compression for `audioStream` type.

**Why now (Phase 1):**
- Change is minimal (1-2 lines: `if type == .audioStream { skipCompress = true }`)
- Unblocks Phase 2 (PTTService needs to send audio without compression)
- Can be tested immediately: send one audio packet, verify it doesn't compress

**Why not later:** If deferred, Phase 2 PTTService will be written assuming compression bypass exists. Then we find it doesn't work and have to rework PTTService. Better to fix transport early.

**Integration risk:** LOW — BLEService already handles multiple packet types. Adding one more is straightforward.

---

### Phase 2: Streaming & Fragment Pipeline

#### 2.1 Why VoiceRecorder Streaming Mode Now

**Current state:** VoiceRecorder exists, records to URL, batch-mode only.  
**Needed:** Streaming mode with per-frame callback.

**Why after protocol (but why not later):**
- Depends: BitchatProtocol (audio packet structure)
- Unblocks: PTTService (needs audio frames to fragment)
- Simple: Add one new public method `startStreamingRecord(callback:)`
- Backward compatible: Existing batch mode unchanged

**Implementation risk:** LOW — AVAudioRecorder already fires callbacks internally. We're just exposing them.

```swift
// Pseudo-code, realistic implementation
extension VoiceRecorder {
    func startStreamingRecord(
        onFrame: @escaping (AVAudioPCMBuffer) -> Void
    ) throws -> URL {
        // Reuse existing setup
        let session = AVAudioSession.sharedInstance()
        // ...
        
        // New: pass callback to recorder
        let recorder = ...
        recorder.onFrameCallback = onFrame  // ← Expose internal callback
        
        return outputURL
    }
}
```

**Why not part of Phase 1:** Protocol doesn't depend on audio capture. But Phase 2 does. So it belongs in Phase 2.

---

#### 2.2 Why PTTAudioBuffer (Jitter) Before PTTService

**Reason:** PTTAudioBuffer is simpler and more isolated than PTTService.

```
PTTAudioBuffer:
  ├─ Input: audio fragments (sessionID, seqNum, payload)
  ├─ State: buffer of fragments, ordered by seqNum
  ├─ Output: ordered audio frames (when complete or timeout)
  └─ Dependencies: BitchatProtocol only

PTTService:
  ├─ Input: user action (start/stop), audio frames, incoming packets
  ├─ State: isTransmitting, currentSessionID, lockedBy
  ├─ Output: send audio packets, playback frames, notifications
  └─ Dependencies: BLEService, NoiseEncryptionService, PTTChannelLock, PTTAudioBuffer, VoiceRecorder
```

**Why build smaller first:**
- Smaller components are easier to test
- Once tested, can be depended on by larger components
- PTTService depends on PTTAudioBuffer; not vice versa

**Test example:**
```swift
func testJitterBufferReorder() {
    let buffer = PTTAudioBuffer()
    
    // Fragments arrive out of order
    buffer.insert(seqNum: 3, data: frame3)
    buffer.insert(seqNum: 1, data: frame1)
    buffer.insert(seqNum: 2, data: frame2)
    
    // Flush with PTT_END
    let ordered = buffer.flush()
    XCTAssertEqual(ordered, [frame1, frame2, frame3])
}
```

**Confidence:** HIGH — straightforward reorder logic, no external dependencies.

---

#### 2.3 Why PTTService (Main Orchestrator) Here

**Role:** Wires everything together — captures audio, fragments, sends, receives, buffers, plays.

**Dependencies (now available):**
- ✅ BitchatProtocol (Phase 1)
- ✅ BLEService (Phase 1 mods)
- ✅ PTTChannelLock (Phase 1)
- ✅ VoiceRecorder streaming (Phase 2.1)
- ✅ PTTAudioBuffer (Phase 2.2)

**Why not earlier:** Cannot be built until all dependencies exist.

**Integration risk:** MEDIUM — many dependencies. Requires clear separation of concerns:
- Transmit path: capture → fragment → send
- Receive path: incoming packets → buffer → notify
- State machine: lock transitions

**Test example:**
```swift
func testTransmitFlow() {
    let pttService = PTTService(
        bleService: mockBLE,
        noiseService: mockNoise,
        recorder: mockRecorder,
        buffer: mockBuffer
    )
    
    pttService.startTransmit(channel: .mesh)
    
    // Simulate audio frame from recorder
    let frame = AVAudioPCMBuffer(...)
    pttService.onAudioFrame(frame)
    
    // Verify BLEService was called to send packet
    XCTAssertTrue(mockBLE.sendWasCalled)
}
```

---

#### 2.4 Why ChatViewModel Mods Here (Not Phase 3)

**Current:** ChatViewModel is the state coordinator. It already has many `@Published` properties.

**Addition:** Three new properties:
```swift
@Published var isTransmittingPTT: Bool = false
@Published var channelLockedBy: PeerID? = nil
@Published var lockedByPeerName: String? = nil
```

**Plus:** New delegate methods:
```swift
func didReceiveAudioStream(_ packet: BitchatPacket, from sender: PeerID) {
    pttService.onIncomingAudioFragment(packet)
}
```

**Why here (Phase 2) not Phase 3:** 
- UI layer (Phase 3) depends on these properties existing
- Better to add them alongside the service that populates them
- Easier to debug state flow if ViewModel setup is complete before Views are written

**Confidence:** HIGH — no new patterns; follows existing `@Published` style.

---

### Phase 3: Playback & UI

#### 3.1 Why AudioStreamPlaybackController Now

**Role:** Receives buffered frames from PTTAudioBuffer, schedules real-time playback.

**Dependencies (now available):**
- ✅ PTTAudioBuffer (Phase 2)
- ✅ AVFoundation (system framework, always available)
- ✅ ChatViewModel with audio state (Phase 2)

**Why not earlier:** Depends on jitter buffer being complete.

**Why not later:** UI (Phase 3.2) needs playback to work. Better to have it ready.

**Architecture:** Keep separate from PTTService to avoid mixing capture and playback logic. Single-responsibility principle.

---

#### 3.2 Why Views/UI Last

**Current state:** ContentView exists, renders messages.

**Additions:**
- PTT button (prominent, hold-to-talk)
- "On air: {peer name}" indicator
- Optional: waveform during capture

**Dependencies:**
- ✅ ChatViewModel with `@Published isTransmittingPTT`, `channelLockedBy` (Phase 2)
- ✅ Playback working (Phase 3.1)

**Why last:** UI is a consumer, not a provider. It reads state from ViewModel and calls methods on it. All state and logic must exist before we write UI.

**Risk of doing earlier:** If we write UI first, we'd have to mock all the state. Then when real services are built, we'd find the UI doesn't match their actual behavior. Better to build bottom-up.

---

### Phase 4: Polish & Optimization

#### Why Not in Earlier Phases

**These features are nice-to-have, not need-to-have:**
- Level meter (users can use system volume indicators)
- Adaptive jitter buffer (fixed 100ms works for MVP)
- Directed PTT with encryption (broadcast works first)
- Battery optimization (acceptable in initial release)

**Deferred because:**
- Core feature works without them
- They add complexity (each one is a separate mini-project)
- Real user feedback might reveal we're optimizing the wrong thing
- Time-boxed: if not in Phase 4, defer to v2.1

---

## Dependency Verification

### Phase 1 → Phase 2 (Can build Phase 2 once Phase 1 done?)

**Phase 1 deliverables:**
- ✅ `MessageType.audioStream = 0x23`
- ✅ `encodeAudioStreamPacket()`, `decodeAudioStreamPacket()`
- ✅ `PTTChannelLock` service
- ✅ `BLEService.sendAudio(skipCompress: true)`

**Phase 2 requirements:**
- ✅ Protocol definition ← Phase 1 provides
- ✅ Lock state machine ← Phase 1 provides
- ✅ BLE compression bypass ← Phase 1 provides
- ✅ Audio frame callback in VoiceRecorder ← Phase 2 provides

**Verdict:** YES, can start Phase 2 once Phase 1 completes.

---

### Phase 2 → Phase 3 (Can build Phase 3 once Phase 2 done?)

**Phase 2 deliverables:**
- ✅ `PTTService` (transmit/receive orchestrator)
- ✅ `PTTAudioBuffer` (jitter buffer)
- ✅ `ChatViewModel` with `@Published isTransmittingPTT`, etc.
- ✅ Audio frames flowing from capture to buffer

**Phase 3 requirements:**
- ✅ PTTAudioBuffer ready for playback consumption ← Phase 2 provides
- ✅ ChatViewModel state accessible ← Phase 2 provides

**Verdict:** YES, can start Phase 3 once Phase 2 completes.

---

## Critical Path Analysis

**Day 1-5: Protocol & State Machine**
- Define audio packet structure (30 min)
- Implement BitchatProtocol constants (1 hour)
- Implement BinaryProtocol encode/decode (2 hours)
- Build PTTChannelLock (4 hours)
- Write unit tests (4 hours)
- Integrate with BLEService (1 hour)

**Day 6-10: Streaming Pipeline**
- Add streaming callback to VoiceRecorder (2 hours)
- Build PTTAudioBuffer (4 hours)
- Build PTTService (8 hours)
- Integration tests (4 hours)
- ChatViewModel bindings (2 hours)

**Day 11-15: Playback & UI**
- AudioStreamPlaybackController (4 hours)
- PTT button & UI (6 hours)
- E2E tests (multi-hop, latency) (4 hours)
- Bug fixes & polish (2 hours)

**Day 16+: Optional Polish**
- Level meter (2-4 hours)
- Waveform (2-4 hours)
- Battery tuning (2-4 hours)

**Total (MVP):** 2-3 weeks  
**Total (with polish):** 4-5 weeks

---

## Why Not Parallel?

**Could we build PTTService and AudioStreamPlaybackController in parallel?**

Theoretically yes, but:
- AudioStreamPlaybackController depends on PTTAudioBuffer (Phase 2 output)
- PTTService produces the data that playback consumes
- If they're not done in the right order, one will stall waiting for the other
- Serial schedule is lower-risk and clearer

**Could we start UI (Phase 3) before Playback (3.1)?**

Technically yes with mocks, but:
- UI button needs to work end-to-end
- Mocked audio doesn't catch real bugs (latency, callback timing)
- Better to have real playback before shipping UI

---

## Risk Register

| Risk | Mitigation | Phase |
|------|-----------|-------|
| Protocol definition incomplete | Finalize by end of Phase 1 sprint | 1 |
| PTTChannelLock tiebreak logic has race | Write tests first, do code review | 1 |
| BLEService compression bypass breaks text | Add regression tests for text packets | 1 |
| VoiceRecorder callback fires too slowly | Monitor frame rate in testing, add buffering | 2 |
| PTTService becomes too complex | Split transmit and receive paths, separate files | 2 |
| Jitter buffer doesn't flush fast enough | Flush on PTT_END signal, not timeout | 2 |
| Playback has latency artifacts | Monitor AVAudioEngine timing, use sample-accurate playback | 3 |
| UI button feels unresponsive | Ensure state propagates to main thread immediately | 3 |

---

## Conclusion

**The phased build order is optimal because:**

1. **Each phase builds on fully-complete prior phases** — no rework due to missing dependencies
2. **Simpler components first** — highest confidence before building on top
3. **Integration complexity increases gradually** — Phase 1 is pure protocol, Phase 2 adds services, Phase 3 adds UI
4. **Testing at each phase** — unit tests → integration tests → E2E tests
5. **Allows parallelization of QA** — Phase 1 testing starts while Phase 2 is being coded

**Estimated timeline:** MVP in 2-3 weeks, polished version in 4-5 weeks.

---

*Build order rationale finalized: 2026-04-01*
