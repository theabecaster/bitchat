# Features Research: PTT Walkie-Talkie

**Research Date:** 2026-04-01
**Milestone Context:** Subsequent — adding PTT to existing bitchat text messaging app
**Domain:** Half-duplex, real-time voice over constrained BLE mesh (140 Kbps, <200ms latency target)

---

## Table Stakes

Features PTT must have or it feels broken/incomplete as a walkie-talkie experience.

### Core PTT Capture & Transmission

- **Hold-to-Talk Button UX** — Button held down = transmitting, released = listening. No mode switching required. Complexity: **Low**
  - Users expect instant transmission when they press (no "start recording" delay). Discord and Telegram patterns show button press → immediate audio flow.
  - Release delay configurable (typically 100-300ms) to tolerate unintentional releases mid-sentence.

- **Visual "Transmitting" Feedback** — User sees they are actually transmitting (not just assuming). Complexity: **Low**
  - Discord uses green border around chat panel during transmission. Telegram shows mic active indicator.
  - Without this, users feel uncertain ("Is my mic on?") and repeatedly press the button.
  - Minimally: color change or icon on the button itself. Optionally: animated waveform or level meter.

- **"On Air" Indicator for Others** — Receiving users see *who* is transmitting right now. Complexity: **Low**
  - Prevents talking over each other in half-duplex. Users need to know "Alice is speaking, I should listen."
  - Priority speaker patterns (Discord) show visual distinction when someone priority speaks.
  - Minimally: speaker name + "live" badge or mic icon. Can highlight the peer's row in chat view.

### Half-Duplex Channel Lock

- **First Transmitter Wins (Channel Lock)** — Once Alice presses PTT, Bob's PTT button should be visually blocked (or gracefully ignored). Complexity: **Medium**
  - Prevents audio collision/overlapping. PROJECT.md specifies "first transmitter wins; no mixing."
  - Protocol enforces via PTT_START signal broadcast (or control packet) that locks the channel.
  - Receiving side: silently drop packets from secondary transmitters while channel is locked.
  - User expectation (from traditional walkie-talkie): only one person can speak at a time, period.

- **Lock Timeout / Dead Transmitter Recovery** — If Alice crashes mid-transmission, channel doesn't stay locked forever. Complexity: **Medium**
  - Typical timeout: 5-10 seconds (radio industry standard). If no PTT_END signal arrives, timeout auto-unlocks.
  - Without this, network glitch = stuck channel = app feels broken.
  - Detection: PTT_END signal from transmitter (clean release) *or* timeout after last audio fragment received.

- **Graceful Rejection of Secondary Speakers** — Bob tries to press PTT while channel locked; what happens? Complexity: **Low-Medium**
  - Best UX: Button becomes un-pressable / disabled with visual hint ("Alice is speaking").
  - Fallback: Button works but audio is silently dropped by mesh (confusing but works).
  - Worst UX: Button works but audio is sent then discarded on receive (feels like transmission failed).

### Audio Stream Delivery

- **Real-Time Fragment Streaming (Not Batch)** — Audio fragments flow as soon as user speaks; don't wait for recording to finish. Complexity: **Medium**
  - PROJECT.md specifies: "Stream fragments continuously while button held — don't wait for recording to complete."
  - Needed to hit <200ms latency target. Batch-and-send model adds full-recording delay (unacceptable for real-time feel).
  - Codec: 16 Kbps AAC (existing in VoiceRecorder.swift) leaves ~124 Kbps headroom for text.

- **Sequence Number on Fragments** — Out-of-order network delivery is reordered on receive. Complexity: **Low**
  - BLE mesh multi-hop may deliver fragments out of order; sequence number allows jitter buffer to reorder.
  - Also enables duplicate detection (same fragment arrives twice from different hops).

- **Jitter Buffer with Timeout** — Receiver buffers out-of-order fragments, flushes on PTT_END or timeout. Complexity: **Medium**
  - Latency research shows: 20-60ms buffer is typical for VoIP; larger buffer = more delay. With <200ms target, buffer should be small (~30-50ms).
  - Tradeoff: Smaller buffer = more silence/dropouts if packet is slightly late; larger buffer = more latency.
  - User expectation: smooth audio with occasional dropouts preferred over delayed audio (perceived as lag).
  - Timeout: if no new fragment received for 200-300ms, assume stream ended and flush buffer immediately.

### Security & Privacy

- **Broadcast PTT Unencrypted** — Channel-wide PTT transmitted in the clear (like text broadcasts). Complexity: **Low**
  - Matches existing broadcast security model. All channel members hear it.

- **Directed PTT Encrypted** — 1:1 PTT encrypted via existing Noise session. Complexity: **Low**
  - PROJECT.md specifies: "Directed PTT encrypted via existing Noise session."
  - Reuses established session; no key renegotiation needed.

- **No Audio in Logs / No Disk Persistence** — Audio fragments never written to disk; never logged. Complexity: **Low**
  - PROJECT.md constraint: "No audio data in logs; no stream content persisted to disk."
  - Only in-memory buffers during playback.

### Playback & User Feedback

- **Automatic Playback on Receive** — User receives PTT audio; app plays it automatically (not a download-then-play model). Complexity: **Low**
  - Expected in messaging apps: audio arrives → immediately heard (like Telegram voice messages, but real-time).

- **Silence Handling** — What if packet is dropped? Playback continues or brief silence. Complexity: **Low**
  - Walkie-talkie apps typically prefer brief silence over playback stutter/glitches. This is baked into jitter buffer design.

---

## Differentiators

Features that would make this stand out (nice-to-have; not blocking).

### Audio Quality & Level Meters

- **Visual Level Meter During Transmission** — Waveform or dB meter shows user their own voice level. Complexity: **Low-Medium**
  - Reassures user: "My mic is working, I'm being heard." Telegram shows this.
  - Not critical, but significantly improves confidence in the feature.
  - Can be animated bars (e.g., AVAudioRecorder provides real-time levels).

- **Adaptive Audio Bitrate** — If network congestion detected, reduce codec bitrate on-the-fly. Complexity: **High**
  - BLE mesh throughput is variable; if text traffic spikes, could starve PTT.
  - 16 Kbps AAC is already quite aggressive; hard to reduce further without major quality loss.
  - Mitigation: congestion detection + graceful dropout (drop stale frames) rather than bitrate scaling.

### Mesh-Specific Optimizations

- **Multi-Hop Awareness** — Receiver knows if speaker is direct (single hop) or distant (multi-hop); UI hint possible. Complexity: **Medium**
  - No concrete UX need, but could inform user: "Alice (2 hops away) is speaking" → expected latency higher.
  - Low priority; existing text delivery already handles multi-hop.

- **Per-Hop Latency Display** — Show end-to-end latency to user. Complexity: **Medium**
  - Military/emergency radio standard; nice-to-have for niche users.
  - Requires timestamping packets; receiver measures RTT or one-way delay.
  - Deferred if <200ms target consistently met.

### Channel & Conversation Features

- **Multiple Channels with Separate PTT Locks** — PTT lock is per-channel (not global). Complexity: **Low-Medium**
  - If bitchat supports multiple channels (existing feature?), each channel needs independent lock state.
  - Allows user to listen to Channel A while someone transmits on Channel B.

- **Voice Call History / Transcript** — Record *that* a PTT session happened, but not the audio itself. Complexity: **Medium**
  - Metadata only: "Alice transmitted 12 seconds at 15:30 UTC on #logistics" → visible in chat log.
  - Useful for compliance/audit; not core to UX.
  - Conflicts with "no disk persistence" constraint; would need optional flag.

### Accessibility

- **Voice Activity Detection (VAD) Option** — Auto-transmit when speech detected (no button hold required). Complexity: **High**
  - Useful for hands-free scenarios (driving, emergency).
  - Requires robust VAD logic; risk of ambient noise triggering transmit.
  - Second-phase feature; core PTT (button-hold) is simpler and proven.

- **Haptic Feedback** — Vibration when PTT button pressed/released. Complexity: **Low**
  - Standard in messaging apps; improves tactile confirmation.
  - Platform-specific (iOS has haptic engine; macOS minimal haptics).

- **Audio Cues (Beep on TX/RX)** — Traditional radio "beep" at start/end of transmission. Complexity: **Low**
  - Expected in walkie-talkie UX; trains user: "I'm transmitting now." / "They're done talking."
  - Can be optional toggle (some users find it annoying).

---

## Anti-Features

Features to explicitly NOT build (especially given BLE constraints).

| Feature | Why to Avoid |
|---------|-------------|
| **Full-Duplex Two-Way Simultaneous Audio** | BLE mesh is half-duplex only. Mixing two audio streams requires >2x bandwidth. Latency would exceed 200ms target. Protocol doesn't support concurrent speakers. |
| **Audio Recording/Playback History** | PROJECT.md: PTT is ephemeral. Voice notes already exist as async alternative. Persistent audio storage = disk space, privacy liability, compliance burden. |
| **Multiple Simultaneous Speakers (Audio Mixing)** | No mixing logic. First speaker locks channel; others are silently dropped. Coordination across mesh hops is hard (no central arbiter). Users expect walkie-talkie semantics (one speaker at a time). |
| **PTT Over Internet/Nostr Relay** | PROJECT.md: BLE-only for v1. Latency budget (<200ms) doesn't translate to internet relay (300-500ms typical). Feature creep; separate product. |
| **Echo Cancellation** | BLE half-duplex means speaker never receives their own audio; echo is impossible. Don't add complexity. |
| **Noise Suppression (Real-Time)** | 16 Kbps AAC codec is already lossy; aggressive noise suppression would overprocess and sound worse. User can apply it client-side in VoiceRecorder.swift settings if needed. |
| **Automatic Gain Control (AGC)** | Risk: AGC pumps audio (volume surges/drops), or triggers on low ambient noise. Better to let user control mic gain manually. Walkie-talkie UX expects consistent input. |
| **Priority Queue for Secondary Speakers** | "If Alice is speaking, queue Bob's PTT button and auto-transmit when she releases." Tempting but breaks walkie-talkie semantics. Users expect to "fight for the channel" like a real radio. |
| **Codec Negotiation** | Force 16 Kbps AAC for all. No fallback to lower bitrate codecs. Simplifies implementation; bandwidth is non-negotiable on 140 Kbps mesh. |
| **Audio Compression Above Codec** | PROJECT.md: Skip zlib on audio fragments. AAC is pre-compressed; double-compression adds latency with no gain. |
| **Playback Speed Control** | Walkie-talkie is synchronous real-time. Allowing "1.5x playback" breaks the metaphor (feels like replay, not real-time). Defer to async voice notes if user wants that. |
| **Cross-Channel PTT Broadcast** | One transmission to multiple channels simultaneously. Adds complexity; users expect to "join" a channel and listen/speak there, not broadcast omni-directionally. |

---

## UX Patterns

What makes PTT feel right.

### Button UX

**Hold-to-Talk (PTT) Button Behavior:**

1. **Placement:** Prominent, always visible in chat view (not buried in menu). Large enough for thumb on mobile.
2. **Visual State:**
   - **Inactive (idle):** Button gray, slightly dimmed. Label: "Hold to Talk" or mic icon.
   - **Pressed (transmitting):** Button bright (green or app primary color). Visual pulse or "live" badge. Label becomes "Release to Send" or shows a timer (seconds transmitted).
   - **Locked (someone else transmitting):** Button disabled/grayed out with lock icon or "Alice is speaking" overlay. Button is not pressable.
   - **Timeout/Error:** Button shows warning icon if previous transmission timed out or failed to send.

3. **Interaction:**
   - On press: Immediately start recording & streaming (no delay).
   - While held: Level meter or waveform animates (shows user they're being heard).
   - On release: Stop recording, send PTT_END signal, playback of remote audio resumes.
   - Release delay: 100-300ms configurable tolerance (so brief pauses in speech don't cancel transmission).

4. **Fallback for One-Handed Use:** Consider a toggle mode option (press once to start, press again to stop) for accessibility, but primary UX is hold-to-talk (proven in walkie-talkies and Telegram).

### "On Air" Indicators

**Peer List / Channel Indicator:**

- **Active Speaker Row Highlight:** If Alice is transmitting on #logistics, her row in the chat list is highlighted (e.g., green bar, badge with "🎙️ Live").
- **Temporal Feedback:** "Alice is speaking (5 seconds)..." updates in real-time.
- **Indicator Removal:** Disappears immediately on PTT_END signal (or after timeout).
- **For Directed PTT:** Separate indicator in 1:1 chat header (e.g., "Alice is speaking to you").

**Global / Persistent Indicator:**

- **Tab/Channel Header Badge:** If in a multi-channel view, the #logistics channel tab shows "(Alice speaking)" until transmission ends.
- **Sound-Only Apps (macOS):** No visual indicator possible; rely on audio cues only.

### Lock/Timeout Handling

**Channel Lock (First Speaker Wins):**

1. **Alice Presses PTT:**
   - PTT_START signal broadcast to channel (or as first audio fragment header).
   - All other users' PTT buttons disabled immediately (visual: grayed out + "Alice is speaking" hint).
   - Alice's button shows "Live" badge and animates.

2. **Bob Tries PTT While Locked:**
   - Button doesn't respond (grayed out).
   - Optional: Show toast/hint: "Wait for Alice to finish." (Not required; graying out is enough.)

3. **Alice Releases PTT (Clean Path):**
   - Alice releases button.
   - VoiceRecorder stops. PTT_END signal sent (or jitter buffer timeout on receive side).
   - Bob's PTT button re-enabled immediately.
   - "On air" indicator disappears.

4. **Network Glitch (Alice Disappears Mid-Stream):**
   - Last audio fragment received at 15:30:02 UTC.
   - Receiver's jitter buffer timeout fires at 15:30:02 + 200-300ms.
   - Jitter buffer flushes; playback ends.
   - Lock automatically expires (no explicit PTT_END signal required).
   - Other users' PTT buttons re-enabled.

5. **Timeout Duration:**
   - Recommend: 5-10 seconds (radio industry standard).
   - Rationale: Covers network jitter + brief disconnects; long enough that speaker doesn't feel rushed, short enough that dead channel unlocks quickly.
   - Configurable in settings (advanced users may want 15-20s for degraded network).

**User Perception:**
- Should be **invisible if everything works** (Alice speaks, releases, Bob speaks immediately).
- Should **gracefully recover** if network glitches (not require manual unlock or app restart).
- Should **never surprise users** (no phantom "channel locked" states for no reason).

### Dependencies Between Features

```
PTT_START/PTT_END Signals
  └─→ Channel Lock
  └─→ "On Air" Indicator
  └─→ Button State Management (enable/disable)

Hold-to-Talk Button
  └─→ VoiceRecorder (existing)
  └─→ Real-Time Fragment Streaming
  └─→ Level Meter (display feedback)

Fragment Streaming
  └─→ Sequence Numbers
  └─→ Jitter Buffer (reorder + timeout)
  └─→ Noise Encryption (for directed PTT)

Jitter Buffer
  └─→ Playback on Receive
  └─→ Automatic Silence Handling

Lock Timeout
  └─→ Automatic Recovery (no manual unlock)
  └─→ Graceful Degradation (if PTT_END missed)
```

---

## Complexity & Phase Mapping

**Low Complexity (Phase 1-2):**
- Hold-to-talk button UX
- Visual transmitting feedback (color change on button)
- "On air" indicator (peer name + badge)
- Channel lock (first transmitter wins via protocol signal)
- Basic jitter buffer (sequence number reorder + timeout)
- PTT_END signal handling
- Broadcast PTT (unencrypted)

**Medium Complexity (Phase 2-3):**
- Directed PTT (encrypted via Noise session)
- Lock timeout & automatic recovery
- Level meter / waveform animation
- Graceful rejection of secondary speakers (button disabled)
- Fragment streaming with backpressure handling
- Congestion detection (drop stale frames)

**High Complexity (Defer / Phase 4+):**
- Adaptive audio bitrate
- Voice activity detection (VAD)
- Multi-hop latency awareness
- Voice call history / transcript
- Cross-channel coordination

---

## Summary: What Makes PTT Feel Real vs. Broken

**Feels Real (Table Stakes):**
1. Button pressed → Audio flows in <100ms (no startup delay)
2. Only one person talks at a time (channel lock prevents collision)
3. User can see they're transmitting (button visual feedback)
4. Others know who's speaking (on-air indicator)
5. Network glitch doesn't permanently break the channel (timeout recovery)

**Feels Broken (Red Flags):**
1. Button press has laggy response (users tap repeatedly, audio distorts)
2. Two people's audio mixes or overlaps (no channel lock)
3. User doesn't know if they're being heard (no visual feedback)
4. "On air" indicator missing or delayed (doesn't match reality)
5. Channel locks and doesn't unlock after speaker finishes (timeout missing or too long)
6. PTT audio quality worse than text delivery (wrong codec choice or congestion handling)

---

## Sources

- [Discord Voice Input Modes 101 - Push-to-Talk & Voice Activated](https://support.discord.com/hc/en-us/articles/211376518-Voice-Input-Modes-101-Push-to-Talk-Voice-Activated)
- [Push-to-Talk - Wikipedia](https://en.wikipedia.org/wiki/Push-to-talk)
- [Telegram Walkie Talkie Feature Overview](https://technewspedia.com/walkie-talkie-on-telegram-what-is-it-how-to-activate-it-%E2%96%B7-2021/)
- [Jitter Buffer - What is it and how does it work? - GetStream](https://getstream.io/glossary/jitter-buffer/)
- [Jitter Buffer and Latency Optimization - TRTC](https://trtc.io/blog/details/Jitter-and-Jitter-Buffer)
- [BLE Bluetooth Latency Characteristics](https://www.blueiot.com/blog/what-is-the-latency-of-Bluetooth-4.0.html)
- [Audio Codecs Explained - VoIP Call Quality](https://telnyx.com/resources/codecs-affect-voip-sound-quality)
- [Comparative Study of Low-Latency Audio Codecs](https://www.researchgate.net/publication/341607597_Comparative_Study_of_Low-Latency_Audio_Codecs)
- [Real-Time Communication Latency Requirements - GetStream](https://getstream.io/glossary/low-latency/)
- [Voice Activity Detection vs. Push-to-Talk - TeamSpeak Support](https://support.teamspeak.com/hc/en-us/articles/360002745898-What-is-the-difference-between-Push-To-Talk-and-Voice-Activity-Detection)
- [Walkie-Talkie App Features - Talker Network](https://talker.network/push-to-talk-ptt-technology-explained/)
