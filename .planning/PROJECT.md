# bitchat PTT — Push-to-Talk Walkie-Talkie

## What This Is

A half-duplex, real-time voice communication mode added to bitchat — the existing iOS/macOS BLE mesh messaging app. Users hold a button to transmit audio; the mesh delivers it to the channel or a specific peer with no internet required. Think walkie-talkie semantics layered on top of the existing BLE mesh protocol.

## Core Value

A user holds the PTT button and their voice reaches nearby peers in under 200ms — no internet, no server, no latency from buffering.

## Requirements

### Validated

- ✓ BLE mesh networking with 469-byte MTU, ~140 Kbps effective throughput — existing
- ✓ Fragment reassembly and multi-hop TTL flooding for broadcast and directed messages — existing
- ✓ Noise Protocol (XX pattern, Curve25519 + ChaCha20-Poly1305) for directed encrypted channels — existing
- ✓ fileTransfer (0x22) packet type with TLV encoding — existing
- ✓ VoiceRecorder.swift: 16 Kbps AAC, 16kHz mono, mic capture pipeline — existing
- ✓ MVVM architecture with ChatViewModel as central coordinator — existing
- ✓ iOS 16+ / macOS 13+ dual-platform support with #if os(iOS)/#if os(macOS) guards — existing

### Active

- [ ] New audio stream packet type (0x23 or next available) with session ID, sequence number, and payload
- [ ] PTT_START and PTT_END control signals broadcast as protocol lock/unlock events
- [ ] Half-duplex channel lock: first transmitter wins; peers reject audio from secondary transmitters while channel is locked
- [ ] Hold-to-talk capture: encode AAC in real-time and stream 469-byte BLE fragments continuously while button is held — don't wait for recording to complete
- [ ] Jitter buffer on receiving side: reorder out-of-order fragments by sequence number, flush on PTT_END or timeout
- [ ] Broadcast PTT: transmit to entire channel (unencrypted, same as text broadcasts)
- [ ] Directed PTT: transmit to a single peer, audio fragments encrypted via existing Noise session
- [ ] Skip zlib compression on audio fragments (AAC is already compressed)
- [ ] Graceful congestion degradation: drop stale audio frames rather than buffering indefinitely
- [ ] PTT button in chat view (hold-to-talk, prominent placement)
- [ ] "On air" indicator: visual signal when another peer is transmitting on the channel
- [ ] Level meter / waveform while transmitting so user knows mic is active
- [ ] No audio data in logs; no stream content persisted to disk
- [ ] Battery: BLE scanning must not be suppressed during PTT receive mode
- [ ] Text messaging performance must not degrade while PTT is idle

### Out of Scope

- Full-duplex two-way conversation — BLE mesh half-duplex architecture; latency budget doesn't allow it
- Audio recording/playback history — voice notes already cover async audio; PTT is ephemeral
- Multiple simultaneous PTT speakers — first-transmitter wins; no mixing, no queue, no preemption
- PTT over Nostr/internet relay — BLE-only for v1; latency constraints don't translate to internet relay

## Context

bitchat is a decentralized BLE mesh messaging app (iOS + macOS) with no central server. The existing protocol handles text, file transfer, and announcements via a custom binary packet format. The audio pipeline foundation (VoiceRecorder.swift) and BLE fragmentation/reassembly (BLEService.swift) already exist — this feature wires them together with a new packet type and stream coordination layer.

**Relevant existing files:**
- `bitchat/Services/BLE/BLEService.swift` — fragment reassembly, MTU negotiation, characteristic write
- `bitchat/Protocols/BitchatProtocol.swift` — packet type definitions, BitchatDelegate
- `bitchat/Protocols/BinaryProtocol.swift` — binary encode/decode for wire format
- `bitchat/Services/NoiseEncryptionService.swift` — Noise session for directed encryption
- `bitchat/Services/VoiceRecorder.swift` — AAC capture pipeline
- `bitchat/ViewModels/ChatViewModel.swift` — central coordinator to wire PTT into

**BLE throughput budget:** ~140 Kbps effective; 16 Kbps AAC leaves ~124 Kbps headroom for text. Single-hop target: <200ms capture-to-playback.

## Constraints

- **Performance**: <200ms end-to-end latency on single hop — drives jitter buffer depth and fragment pipelining approach
- **Platform**: iOS 16+ and macOS 13+; AVFoundation APIs differ between platforms — use #if guards
- **Architecture**: Must not increase BLE scan interval during PTT receive (audio is timing-sensitive; suppressing scans breaks delivery)
- **Security**: No audio content in logs or on disk at any point in the pipeline
- **Protocol**: New packet type must coexist with existing types; no breaking changes to existing packet parsing
- **Compression**: Audio fragments must bypass the zlib compression path in BLEService
- **Congestion**: Must drop frames, not buffer them — stale audio is worse than silence

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Protocol-enforced channel lock via PTT_START/PTT_END | Prevents collision races across mesh hops; receivers can authoritatively reject second transmitters | — Pending |
| Directed PTT encrypted via existing Noise session | Consistent with text chat security model; reuses established session without key renegotiation | — Pending |
| Skip zlib on audio fragments | AAC is pre-compressed; double-compression adds latency with zero gain | — Pending |
| Stream fragments continuously (don't wait for recording end) | Necessary to hit <200ms latency target; batch-and-send would add full-recording delay | — Pending |
| First-transmitter wins (no queue/preemption) | Simplest correct half-duplex semantics; avoids coordination complexity | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-01 after initialization*
