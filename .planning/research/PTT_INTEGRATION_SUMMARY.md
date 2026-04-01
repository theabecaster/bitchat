# PTT Integration Research Summary

**Research Date:** 2026-04-01  
**Project:** bitchat PTT (Push-to-Talk) Audio Streaming  
**Status:** Architecture research complete

---

## Quick Reference: Where PTT Fits

| Layer | Component | Role |
|-------|-----------|------|
| **View** | ContentView | PTT button (hold-to-talk), "On air" indicator |
| **ViewModel** | ChatViewModel | Expose `@Published isTransmittingPTT`, `channelLockedBy` |
| **Services** | PTTService | Main orchestrator for capture, fragment, transmit, receive, playback |
| **Services** | PTTChannelLock | Half-duplex enforcer: owns channel lock state machine |
| **Services** | PTTAudioBuffer | Jitter buffer: reorder fragments, flush on PTT_END |
| **Services** | BLEService | Transport (existing), add audio stream packet type (0x23) |
| **Services** | NoiseEncryptionService | Encrypt audio for directed PTT (Phase 2) |
| **Protocol** | BitchatProtocol | Add `audioStream` packet type, PTT_START/END control signals |
| **Protocol** | BinaryProtocol | Add audio packet encode/decode |
| **Audio** | VoiceRecorder | Streaming mode (new callback) + existing batch mode |
| **Audio** | AudioStreamPlaybackController | Play jitter-buffered audio frames in real-time |

---

## Critical Architectural Decisions

### 1. Service Ownership: PTTService is NOT Middleware

**Bad approach (avoid):**
- BLEService directly owns jitter buffer (violates SoC)
- ChatViewModel orchestrates buffer flushing (view model shouldn't own audio state)
- VoiceRecorder manages fragmentation (it's just an audio capturer)

**Good approach (what we're doing):**
- **PTTService** owns all PTT-specific state: transmit/receive coordination, channel lock, buffer lifecycle
- **BLEService** is transport only: send/receive packets, skip compression for audio
- **ChatViewModel** is a consumer: exposes state via `@Published`, handles delegate callbacks
- **VoiceRecorder** is a capturer: streams frames via callback, doesn't know about fragmentation

**Why:** Keeps audio logic isolated. If we later add SIP or other audio transport, we can plug it in without changing ChatViewModel.

---

### 2. Half-Duplex Enforcement: Protocol Level + Service Level

**Protocol level (PTT_START/END broadcasts):**
- All peers receive PTT_START and agree on channel lock
- Prevents simultaneous transmission across mesh (collision avoidance)
- Control signals are unencrypted (broadcast semantics)

**Service level (PTTService + PTTChannelLock):**
- PTTChannelLock maintains state: `isMeTransmitting`, `lockedByPeer`, `sessionID`
- PTTService rejects local send if channel is locked by another peer
- PTTService rejects incoming audio if sender is not the lock holder

**Why both:** Protocol-level avoids collision races (all peers see START simultaneously). Service-level avoids implementation bugs (local state check catches edge cases).

---

### 3. Streaming, Not Batch: VoiceRecorder Callback Model

**Don't do this (wrong for PTT):**
```swift
// ❌ Wait for recording to complete before sending
VoiceRecorder.startRecording()
// User holds button for 5 seconds
VoiceRecorder.stopRecording()
let audioURL = await VoiceRecorder.finish()
// NOW we can encode and send
let fragments = encodeAudioToFragments(audioURL)
fragments.forEach { sendFragment($0) }
```

**Do this instead (correct for PTT):**
```swift
// ✅ Stream fragments while button is held
VoiceRecorder.startStreamingRecord { audioFrame in
    let fragment = encodeFrameToFragment(audioFrame)
    PTTService.sendFragment(fragment)
}
// User holds button for 5 seconds
// During those 5 seconds, fragments stream continuously
VoiceRecorder.stopStreamingRecord()
```

**Why:** <200ms latency target requires continuous pipelining. Waiting for recording to finish adds 5s+ delay.

---

### 4. Jitter Buffer: Minimal, Immediate Flush

**Size:** 100ms buffer (roughly 2-4 audio frames at 16kHz mono)
- Tolerates network jitter, not massive gaps
- If packets are lost, audio stops (better than stale buffering)

**Flush trigger:** PTT_END signal, not timeout
- When sender broadcasts PTT_END, all receivers flush immediately
- Timeout (5s) is fallback only, for robustness if PTT_END is lost

**Reorder key:** Sequence number, not arrival time
- Fragment arrives out of order → lookup by (sessionID, seqNum), insert in correct position
- When PTT_END arrives, emit ordered frames in seqNum order

**Why:** Audio latency must stay low. No waiting for timeout; signal-driven cleanup.

---

### 5. Encryption: Phase 1 Broadcast Only, Phase 2 Directed

**Phase 1 (MVP):**
- PTT is broadcast only (unencrypted, like text chat)
- All peers in channel receive audio
- Simpler to reason about (no Noise session complexity)
- Mirrors text chat model

**Phase 2 (enhancement):**
- Directed PTT: encrypt via existing Noise session
- Reuse Noise XX handshake (no new crypto needed)
- Audio packets wrapped in `noiseEncrypted` type, like private messages

**Why separate:** Noise handshakes add latency. Phase 1 unblocks core feature. Phase 2 adds privacy for group/1-on-1 directed audio.

---

## Data Structure Examples

### Audio Stream Packet (BitchatProtocol)

```
MessageType: 0x23 (audioStream)
Payload structure:
  [0:8]   sessionID (64-bit, unique per transmission)
  [8:12]  seqNum (32-bit, increments per frame)
  [12:13] flags (0x01 = final frame, others reserved)
  [13:]   audioData (encoded AAC or PCM, payload)
```

### PTT Control Signal (BitchatProtocol)

```
MessageType: 0x02 (message) with special content:
Payload structure:
  [0:1]   controlType (0x50 = PTT_START, 0x51 = PTT_END)
  [1:9]   senderID (short peer ID)
  [9:17]  sessionID (must match audio packets)
  [17:25] timestamp (for tiebreaking simultaneous START)
```

### Jitter Buffer State (PTTAudioBuffer)

```swift
struct PTTAudioBuffer {
    var sessionID: String  // Current active session
    var fragments: [UInt32: Data] = [:]  // seqNum -> decoded audio data
    var isComplete: Bool = false  // PTT_END received
    var createdAt: Date = Date()
    var lastFrameAt: Date = Date()
}
```

---

## Dependency Graph (Build Order)

```
1. BitchatProtocol  (add audioStream 0x23)
        ↓
2. BinaryProtocol   (encode/decode audioStream)
        ↓
3. PTTChannelLock   (state machine, zero external deps)
        ↓
4. PTTAudioBuffer   (depends on protocol)
        ↓
5. BLEService       (mod: add audioStream callback)
        ↓
6. VoiceRecorder    (mod: add streaming mode)
        ↓
7. PTTService       (depends: BLE, Noise, ChannelLock, AudioBuffer, Recorder)
        ↓
8. ChatViewModel    (mod: bind PTTService state)
        ↓
9. AudioStreamPlaybackController  (depends: PTTAudioBuffer)
        ↓
10. Views           (PTT button, indicators)
```

**No circular dependencies.** PTTService depends on lower layers; View only depends on ViewModel.

---

## Thread Safety Model

| Component | Owner Queue | Access Pattern |
|-----------|-------------|-----------------|
| PTTService | `pttQueue` (serial) | Enqueue audio frames, control signals |
| PTTChannelLock | `lockQueue` (concurrent, barrier writes) | Check `canTransmit()` (read), `lock/unlock()` (write) |
| PTTAudioBuffer | `bufferQueue` (concurrent, barrier writes) | Insert fragments (write), emit ready frames (read) |
| ChatViewModel | `DispatchQueue.main` | `@Published` properties updated on main thread only |
| BLEService | `bleQueue` (serial) | Already has established queue discipline |
| VoiceRecorder | internal `queue` (serial) | Callback dispatched to PTTService's queue |

**Rule:** Each service has its own queue. Cross-service communication happens via callbacks or publishers, serialized to correct thread before mutation.

---

## Risk Mitigation Checklist

- [ ] Protocol constants finalized (0x23 for audio, 0x50/0x51 for control)
- [ ] PTTChannelLock tiebreak logic written and tested (simultaneous PTT_START)
- [ ] Jitter buffer reorder logic handles out-of-order fragments
- [ ] BLE fragmentation works with audio (test text + audio simultaneously)
- [ ] VoiceRecorder callback timing is predictable (monitor frame rate)
- [ ] Echo suppression policy documented (allow or mute own audio)
- [ ] Latency measured: <200ms single-hop, <500ms multi-hop
- [ ] Battery drain acceptable during 5-minute PTT session
- [ ] Fallback for missing Noise session (broadcast only, Phase 1)

---

## Success Criteria (Acceptance)

1. **Functional:** User holds PTT button, audio reaches peer in <200ms, peer hears clear audio
2. **Reliable:** Multi-hop relay works (3+ device chain)
3. **Fair:** Simultaneous PTT from two peers results in one clear winner (tiebreak)
4. **Graceful:** Congestion (saturated BLE) doesn't stall text messaging
5. **Tested:** Unit tests for state machines, E2E tests for latency

---

## Next: Roadmap Implications

This research informs the roadmap as follows:

**Phase 1: Protocol + Core**
- Implement BitchatProtocol and BinaryProtocol changes
- Build PTTChannelLock and PTTAudioBuffer (low-risk, high isolation)
- Integrate into BLEService (add audioStream type handling)
- Bonus: implement broadcast PTT (unencrypted)

**Phase 2: Capture + Playback**
- Streaming mode for VoiceRecorder
- PTTService orchestration
- AudioStreamPlaybackController
- UI button and indicators

**Phase 3: Directed + Polish**
- Noise encryption for directed PTT
- Level meter and waveform visualization
- Adaptive jitter buffer (if needed)

---

*Research completed by architecture team, 2026-04-01*  
*Ready for roadmap construction.*
