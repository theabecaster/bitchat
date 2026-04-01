# Stack Research: PTT Audio Streaming

**Research Date:** 2026-04-01  
**Milestone Context:** Subsequent — adding PTT real-time audio streaming to existing bitchat  
**Platform Support:** iOS 16+ / macOS 13+  
**Overall Confidence:** HIGH

## Executive Summary

Real-time audio streaming over BLE requires a fundamentally different approach than the existing VoiceRecorder (which is file-based). The standard 2025-2026 stack for iOS/macOS PTT is:

1. **Audio Capture:** AVAudioEngine with AVAudioInputNode tap + AVAudioSession (iOS only)
2. **Real-Time AAC Encoding:** AVAudioConverter (PCM → AAC) with stateful encoder persistence
3. **Jitter Buffer:** Ring buffer (circular buffer) with sequence-number-based reordering
4. **Latency Profile:** 10-20ms capture → encode → fragment → BLE TX pipeline (within 200ms end-to-end goal)

The key insight: **Don't use AVAudioRecorder for PTT** — it buffers the entire recording before access. Instead, use AVAudioEngine's real-time tap to get audio buffers as they arrive, encode incrementally, and stream fragments immediately.

---

## Audio Capture API

### Recommendation

**Use AVAudioEngine with AVAudioInputNode tap** for real-time audio capture.

```swift
import AVFoundation

let audioEngine = AVAudioEngine()
let inputNode = audioEngine.inputNode
let format = inputNode.outputFormat(forBus: 0)!

inputNode.installTap(onBus: 0, bufferSize: 1024, format: format) { buffer, _ in
    // Audio buffer received immediately; process/encode here
    // No waiting for recording to finish
}

try audioEngine.start()
```

### Rationale

- **Real-time access:** Audio buffers arrive as they're captured (callback-driven), not after recording ends
- **No artificial buffering:** Unlike AVAudioRecorder, which waits until `stopRecording()` to release audio
- **Tap flexibility:** Can tap at any node in the audio graph; input node gives raw mic data
- **Consistent API:** Available on both iOS and macOS (though setup differs)
- **Industry standard:** Used by Zello, WebRTC implementations, and other PTT/VoIP apps

### iOS vs macOS Differences

#### iOS: AVAudioSession required

```swift
#if os(iOS)
let session = AVAudioSession.sharedInstance()
// Category must include record
try session.setCategory(
    .playAndRecord,  // Required to record while app might also play
    mode: .measurement,  // Minimizes system signal processing for clean signal
    options: [
        .defaultToSpeaker,
        .allowBluetoothA2DP,
        .allowBluetoothHFP  // Allow BLE headset audio routing (not on simulator)
    ]
)
// Set low buffer duration for PTT latency
try session.setPreferredIOBufferDuration(0.005)  // 5ms (range: 0.002–0.1)
try session.setActive(true, options: .notifyOthersOnDeactivation)
#endif
```

- **AVAudioSession required:** iOS 16+ uses AVAudioSession to coordinate audio between apps
- **Buffer duration control:** iOS allows `setPreferredIOBufferDuration()` to reduce latency from default ~20ms to ~5ms
- **Bluetooth mode:** `.allowBluetoothHFP` enables HFP routing; not available in simulator

#### macOS: No AVAudioSession

```swift
#if os(macOS)
// macOS has no AVAudioSession; only AVAudioEngine
// Set up engine and input node directly; routing happens via System Preferences
#endif
```

- **No AVAudioSession:** macOS doesn't have session categories/modes; audio context is application-level
- **Simpler setup:** No need to manage `.recording` permissions or session categories
- **Routing via system:** Audio input/output routing is configured in System Settings, not by the app

---

## Real-Time AAC Encoding

### Recommendation

**Use AVAudioConverter (PCM → AAC) with persistent state** for real-time frame-by-frame encoding.

```swift
import AVFoundation

// Create once at PTT start; reuse across all buffers
let pcmFormat = inputNode.outputFormat(forBus: 0)!
let aacFormat = AVAudioFormat(
    commonFormat: .packedInt16,  // AAC I/O format
    sampleRate: 16000,
    channels: 1,
    interleaved: true
)!

var converter = try AVAudioConverter(from: pcmFormat, to: aacFormat)

// Per audio buffer
var encodedBuffer: AVAudioBuffer?
converter.convert(to: &encodedBuffer, from: pcmBuffer) { inNumPackets, outStatus in
    outStatus.pointee = .haveData
    return pcmBuffer
}

if let encoded = encodedBuffer {
    // encoded.byteLength contains AAC frames
    // Send to BLE immediately
}
```

### Rationale

**Why AVAudioConverter over AVAssetWriter:**
- **Immediate output:** AVAudioConverter returns encoded bytes on each call; no file context
- **Stateful:** Maintains encoder state across calls (critical for AAC quality consistency)
- **No 2112-frame priming delay on each conversion:** If you recreate the converter per buffer, you get 2112 frames of silence at each creation — **that's 130ms of silence at 16kHz**. Reuse the same converter instance.
- **Lower CPU:** No file I/O overhead; encoder only
- **Streaming-native:** Designed for incremental encoding

**Why not AVAssetWriter:**
- File-context overhead: expects media file headers, sample timing metadata
- Priming complexity: requires attaching `kCMSampleBufferAttachmentKey_TrimDurationAtStart` to handle AAC's 2112 priming frames
- Not designed for fragment-level streaming: better suited for final file output

**Why not raw Audio Toolbox (AudioConverter C API):**
- More boilerplate; same underlying functionality
- AVAudioConverter is the Swift-idiomatic choice (iOS 16+ maturity)

### Buffer Size and Latency Trade-off

- **Tap buffer size:** 1024 samples = 64ms at 16kHz
  - Larger reduces CPU overhead but increases latency
  - Smaller (512) = 32ms, higher CPU, meets PTT latency budget
- **Encode once per tap:** Process each buffer as it arrives; don't accumulate
- **Frames per BLE fragment:** A 469-byte BLE MTU holds ~8 AAC frames (60 bytes/frame avg); goal is <5 fragments per capture block

### Confidence Notes

- **HIGH confidence:** AVAudioConverter is the current (2025-2026) standard; WWDC 2025 "Enhance your app's audio recording capabilities" explicitly covers this pattern
- **Caveat:** AAC encoder state is opaque; don't recreate converters mid-session or you lose encoder state and introduce silence

---

## Jitter Buffer Implementation

### Recommendation

**Ring buffer (circular buffer) with sequence-number-based reordering** on the receiving side.

```swift
// Pseudo-code structure
class JitterBuffer {
    private var buffer: [UInt16: Data] = [:]  // sequence -> encoded frame
    private var nextSequence: UInt16 = 0
    private let reorderWindow = 16  // Accept up to 16 out-of-order frames

    func insert(sequence: UInt16, data: Data) -> [Data]? {
        // Only accept if within reorder window
        let distance = UInt16(wrapping: sequence &- nextSequence)
        guard distance < UInt16(reorderWindow) else { return nil }

        buffer[sequence] = data

        // Flush contiguous frames starting from nextSequence
        var toPlay: [Data] = []
        while let frame = buffer.removeValue(forKey: nextSequence) {
            toPlay.append(frame)
            nextSequence = nextSequence &+ 1
        }

        return toPlay.isEmpty ? nil : toPlay
    }

    func flush() -> [Data] {
        let frames = Array(buffer.keys).sorted()
        buffer.removeAll()
        return frames.map { buffer[$0]! }
    }
}
```

### Parameters

| Parameter | Recommended Value | Rationale |
|-----------|-------------------|-----------|
| **Buffer depth** | 16–32 frames | 16 frames = ~256ms at 60fps fragment rate; handles BLE out-of-order without over-buffering |
| **Reorder window** | UInt16 (0–65535 wrapping) | Sequence numbers naturally wrap; reorder window size limits how far "behind" we accept frames |
| **Flush trigger** | PTT_END signal OR timeout | On PTT_END, immediately play remaining buffer; on 500ms silence timeout, drop stale frames (avoid hanging audio) |
| **Frame size** | 60 bytes (avg AAC frame) | At 469-byte MTU, fits ~7–8 frames per BLE packet |

### Why Ring Buffer (vs. linked list, heap, etc.)

- **Fixed memory:** Circular buffer pre-allocates; no dynamic allocation during real-time playback
- **O(1) insert/retrieve:** Ring buffer with wrapping arithmetic is O(1) for both
- **Bounded latency:** No GC pauses; predictable behavior under load
- **Industry standard:** WebRTC's NetEQ uses this pattern; proven for VoIP

### Confidence Notes

- **HIGH:** Ring buffer/jitter buffer patterns are well-established in VoIP (WebRTC, Opus codec)
- **Caveat:** BLE mesh may deliver frames via different paths; sequence numbers must be assigned by the transmitter (not hop count or receipt time)

---

## AVAudioSession Configuration (iOS only)

### Category/Mode Recommendation

```swift
#if os(iOS)
let session = AVAudioSession.sharedInstance()

try session.setCategory(
    .playAndRecord,  // Allow simultaneous capture and playback
    mode: .measurement,  // Removes system echo cancellation, auto-gain control
    options: [
        .defaultToSpeaker,      // Route audio to speaker (not receiver)
        .allowBluetoothA2DP,    // Allow A2DP Bluetooth routing
        .allowBluetoothHFP      // Allow HFP Bluetooth routing (PTT headsets)
    ]
)

try session.setPreferredIOBufferDuration(0.005)  // 5ms for PTT latency
try session.setActive(true, options: .notifyOthersOnDeactivation)
#endif
```

### Rationale

| Setting | Value | Why |
|---------|-------|-----|
| **Category** | `.playAndRecord` | PTT may need to play incoming audio while user transmits; also covers monitoring own mic for level meter |
| **Mode** | `.measurement` | Disables automatic echo cancellation, noise suppression, and AGC — these add latency (50–100ms) and aren't useful for one-way PTT |
| **Option: defaultToSpeaker** | Yes | Default routes audio to receiver (ear speaker) on iPhone; PTT is walkie-talkie style (speaker output) |
| **Option: allowBluetoothA2DP** | Yes | Support BLE stereo headphones (fallback if HFP unavailable) |
| **Option: allowBluetoothHFP** | Yes | Support BLE headsets with mic input (PTT + listen) |
| **preferredIOBufferDuration** | 0.005 (5ms) | Default is ~20ms; 5ms reduces latency by 15ms. Lower than 2ms is risky (CPU overhead). |

### Why NOT .voiceChat or .videoChat mode

- **.voiceChat:** Optimizes for two-way conversation; adds echo cancellation (latency penalty)
- **.videoChat:** Similar to voiceChat; adds video sync overhead
- **.measurement:** Strips all processing for clean signal capture (what we want for one-way PTT)

---

## What NOT to Use

| Approach | Why to Avoid |
|----------|-------------|
| **AVAudioRecorder** | Buffers entire recording until `stopRecording()` called. For PTT, you need real-time buffers. Results in full-recording-duration delay (unacceptable for <200ms latency target). |
| **AVAssetWriter** | File-oriented API; requires media container metadata, priming frame handling, CMSampleBuffer attachment keys. Overkill for streaming fragments; adds complexity and latency. |
| **Recreating AVAudioConverter per buffer** | AAC encoder needs persistent state across buffers. Each new converter instance adds 2112 frames (~130ms at 16kHz) of silence priming. Massively inflates audio output and breaks real-time. |
| **Lowering preferredIOBufferDuration below 2ms** | iOS can't guarantee sub-2ms buffer delivery; attempting to set it causes system to ignore request or add CPU overhead that causes audio drops. |
| **.playback or .record category alone** | `.playback` doesn't allow recording; `.record` doesn't allow simultaneous playback (needed for incoming audio in two-way scenarios, level meter feedback, etc.). Must use `.playAndRecord`. |
| **Echo cancellation in PTT context** | Half-duplex walkie-talkie (one speaker at a time) has no echo. EC adds 50–100ms latency; waste. Use `.measurement` mode to disable. |
| **FLAC or Opus codec on iOS** | Existing app uses 16 Kbps AAC; stick with it. FLAC is lossless but huge (8x bitrate). Opus is excellent but requires additional encoding library (libopus); AAC is hardware-accelerated on iOS. |
| **Trying to use microphone in simulator** | Bluetooth not available in iOS Simulator. Must test on physical device. |

---

## Integration with Existing Codebase

### Changes to VoiceRecorder.swift

The existing VoiceRecorder is file-based (for voice notes). PTT needs a separate capture pipeline:

```swift
// New: PTTAudioStreamer.swift (not VoiceRecorder modifications)
final class PTTAudioStreamer {
    private let audioEngine = AVAudioEngine()
    private var converter: AVAudioConverter?
    private var tapBlock: ((AVAudioPCMBuffer, AVAudioTime) -> Void)?

    func startStreaming(onBuffer: @escaping (Data) -> Void) throws {
        #if os(iOS)
        try setupAudioSession()
        #endif

        let inputNode = audioEngine.inputNode
        let format = inputNode.outputFormat(forBus: 0)!

        setupConverter(format: format)

        inputNode.installTap(onBus: 0, bufferSize: 512, format: format) { [weak self] buffer, _ in
            guard let self = self else { return }
            do {
                let aacData = try self.encodeBuffer(buffer)
                onBuffer(aacData)
            } catch {
                // Log error, skip frame
            }
        }

        try audioEngine.start()
    }

    func stopStreaming() {
        audioEngine.inputNode.removeTap(onBus: 0)
        audioEngine.stop()
        converter = nil
    }
}
```

Keep VoiceRecorder.swift as-is for voice notes; introduce PTTAudioStreamer for real-time streaming.

### Integration with BLEService

BLEService already handles 469-byte fragmentation (existing code paths). PTT adds:

1. **New packet type:** 0x23 (or next available) for audio fragments
2. **Fragment headers:** Session ID (4 bytes) + Sequence # (2 bytes) + AAC frame data
3. **No compression:** Bypass zlib path (AAC already compressed)
4. **Jitter buffer:** Receiver-side only; on incoming audio fragments

---

## Confidence Levels

| Recommendation | Confidence | Caveat |
|---|---|---|
| AVAudioEngine + tap for capture | **HIGH** | WWDC 2025 standard; proven pattern for real-time streaming |
| AVAudioConverter for AAC | **HIGH** | Mature API (iOS 9+); hardware acceleration on modern devices |
| Ring buffer for jitter | **HIGH** | Standard VoIP pattern (WebRTC, SIP implementations) |
| AVAudioSession (.playAndRecord, .measurement, 5ms buffer) | **HIGH** | Apple documentation and community practice align |
| Avoid AVAudioRecorder for PTT | **HIGH** | Fundamental architectural mismatch; file-oriented design |
| iOS/macOS #if guards for AVAudioSession | **HIGH** | macOS has no AVAudioSession; iOS 16+ requires it |
| 16 kHz AAC bitrate choice | **MEDIUM** | App already uses it; for PTT, consider stepping down to 8 Kbps if BLE bandwidth is tight (still acceptable quality). See FEATURES research for codec trade-offs. |

---

## Sources

- [Apple Developer Documentation: AVAudioEngine](https://developer.apple.com/documentation/avfaudio/avaudioengine)
- [Apple Developer Documentation: AVAudioConverter](https://developer.apple.com/documentation/avfaudio/avaudioconverter)
- [Apple Developer Documentation: AVAudioSession](https://developer.apple.com/documentation/avfaudio/avaudiosession)
- [WWDC 2025: Enhance your app's audio recording capabilities](https://developer.apple.com/videos/play/wwdc2025/251/)
- [Using AVAudioEngine to Record, Compress and Stream Audio on iOS - Tarka Labs](https://arvindhsukumar.medium.com/using-avaudioengine-to-record-compress-and-stream-audio-on-ios-48dfee09fde4)
- [AVAudioEngine Tutorial for iOS: Getting Started - Kodeco](https://www.kodeco.com/21672160-avaudioengine-tutorial-for-ios-getting-started)
- [How WebRTC's NetEQ Jitter Buffer Provides Smooth Audio](https://webrtchacks.com/how-webrtcs-neteq-jitter-buffer-provides-smooth-audio/)
- [Audio priming - handling encoder delay in AAC - Apple Developer Documentation](https://developer.apple.com/documentation/quicktime-file-format/appendix_g_audio_priming_handling_encoder_delay_in_aac)
- [Technical Note TN2258: AAC Audio - Encoder Delay and Synchronization](https://developer.apple.com/library/archive/technotes/tn2258/_index.html)
- [Opus vs AAC: Audio Quality, Latency & Best Uses (2026) - Vibbit](https://vibbit.ai/blog/aac-vs-opus-audio)
- [Improving latency and broadening audio horizons with Bluetooth LE Audio - Bluetooth Technology Website](https://www.bluetooth.com/blog/improving-latency-and-broadening-audio-horizons-with-le-audio/)
- [Audio API Overview - objc.io](https://www.objc.io/issues/24-audio/audio-api-overview/)
- [How to reduce audio latency in iOS - Overloud](https://www.overloud.com/node/325)
- [WebRTC and Buffers - getStream](https://getstream.io/resources/projects/webrtc/advanced/buffers/)

---

*Stack research complete: 2026-04-01*
