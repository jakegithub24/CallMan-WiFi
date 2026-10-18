# Technical Requirements Document — LAN Voice Call App

**Version:** 1.0
**Status:** Draft
**Last updated:** Oct 19, 2026
**Companion docs:** PRD.md, SystemDesign.md

---

## 1. Overview

This document specifies the technical stack, constraints, and implementation requirements for the LAN Voice Call App described in `PRD.md`. It focuses on **what technology to use and why**, not how to architect the components (see `SystemDesign.md`).

---

## 2. Platform & runtime requirements

### 2.1 Target platforms

| Platform | Minimum version | Why |
|---|---|---|
| **iOS** | 14.0 | Requires `NSLocalNetworkUsageDescription`; iOS 14 introduces stricter local network permission prompts |
| **Android** | API 26 (8.0) | `NetworkInterface` APIs and UDP broadcast behavior are stable from API 26 |

### 2.2 Framework

| Item | Version | Reason |
|---|---|---|
| Flutter | **3.19+** | Latest stable with modern Dart 3 features |
| Dart | **3.3+** | Records, patterns, sealed classes |
| **Target:** | Both iOS and Android | Web not supported in v1 (no raw UDP sockets on browsers) |

### 2.3 Platform permissions

**iOS (`Info.plist`):**
```xml
<key>NSLocalNetworkUsageDescription</key>
<string>This app communicates with other devices on your local network for voice calls.</string>
<key>NSMicrophoneUsageDescription</key>
<string>Required for voice calls.</string>
<key>NSBonjourServices</key>
<array>
  <!-- Not using Bonjour in v1, but reserve for future -->
</array>
```

**Android (`AndroidManifest.xml`):**
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.CHANGE_WIFI_MULTICAST_STATE" />
<!-- Required for broadcast reception on some OEMs -->
```

---

## 3. Technology stack

### 3.1 Package selections

| Concern | Package | Version | Reason |
|---|---|---|---|
| **Microphone capture** | `record` | ^5.1.0 | Mature, supports `startStream()` with raw PCM output. Cross-platform (iOS/Android). |
| **Audio playback** | `flutter_pcm_sound` | ^3.0.0 | Purpose-built for low-latency raw PCM chunk playback. Alternatives (`just_audio`, `audioplayers`) are file-oriented and unsuitable. |
| **Persistent storage** | `shared_preferences` | ^2.2.0 | Simple key-value storage for identity, settings, tombstones. Sufficient for our data sizes. |
| **UUID/random** | `dart:math` `Random.secure()` | built-in | Native secure randomness for `device_id` generation. No package needed. |
| **State management** | `provider` + `ChangeNotifier` | ^6.1.0 | Simple, well-supported, testable. Alternatives (`riverpod`, `bloc`) are heavier for this scope. |
| **Connectivity monitoring** | `connectivity_plus` | ^5.0.0 | Detect Wi-Fi network changes to trigger peer list refresh. |
| **Permissions** | `permission_handler` | ^11.0.0 | Unified permission flow across iOS/Android. |

**Total third-party packages:** 6. Deliberately minimal to reduce supply-chain risk and build complexity.

### 3.2 Platform APIs (no package needed)

| API | Source | Use |
|---|---|---|
| `RawDatagramSocket` | `dart:io` | UDP unicast + broadcast |
| `NetworkInterface.list()` | `dart:io` | Discover local IP / subnet |
| `Timer` / `Stream` | `dart:async` | Heartbeats, timeouts, state machines |

### 3.3 Explicitly NOT used

| Item | Why not |
|---|---|
| **WebRTC** | Deferred to v2 branch. v1 uses raw UDP + PCM. |
| **Opus / any codec** | LAN bandwidth is free; PCM is simpler. Codec adds complexity we don't need in v1. |
| **Firebase** | No cloud dependency per PRD. |
| **`socket_io`** | Adds TCP/WebSocket layer; UDP is directly better for real-time audio. |
| **`just_audio` / `audioplayers`** | File-oriented; unsuitable for sub-100ms playout latency. |
| **`ffi` / native audio** | Adds complexity. `flutter_pcm_sound` covers the need. |

---

## 4. Audio requirements

### 4.1 Audio format (fixed, no negotiation)

```text
Sample rate:    16,000 Hz
Channels:       1 (mono)
Bit depth:      16-bit signed little-endian
Frame duration: 20 ms
Samples/frame:  320
Bytes/frame:    640
```

**Rationale:** 16 kHz mono is the industry standard for voice (Opus, AMR-WB, G.722 all use or support it). 20 ms frames balance latency vs. packet overhead. This choice is locked and must not be changed without a PRD revision.

### 4.2 Capture pipeline requirements

- Must use `record` package with **`AudioEncoder.pcm16bits`** (not WAV, not AAC, not MP3)
- Must handle `startStream()` output chunks of arbitrary size
- Must run on a dedicated Dart isolate if CPU profiling shows main-isolate contention (deferred until measured)
- Must respect the OS mic permission model; on denial, emit a typed error to the controller

### 4.3 Playback pipeline requirements

- Must use `flutter_pcm_sound` with `sampleRate: 16000`, `channels: 1`
- Must not buffer more than **100 ms** of audio in the package's internal queue (measure via `remainingFrames` if exposed)
- Must gracefully handle underruns (play silence, don't crash)
- Must handle device speaker-to-headphones switch mid-call without dropping audio

### 4.4 Audio processing (v1)

**None.** No AEC, no noise suppression, no AGC, no VAD. These are out of scope. Document as known limitation.

---

## 5. Networking requirements

### 5.1 Socket requirements

- **Protocol:** UDP (`RawDatagramSocket`)
- **Port:** **5000** (single port for all traffic)
- **Bind address:** `0.0.0.0` (all interfaces)
- **Broadcast:** enabled via `socket.broadcastEnabled = true`
- **Receive buffer size:** 32 KB minimum (handle bursts of 20 ms frames)

### 5.2 Packet requirements

- **All packets must have a 2-byte header:** `[type: 1B][version: 1B]`
- **Version** must be `0x01` in v1. Unknown versions must be dropped silently (forward compatibility).
- **Unknown types** must be dropped silently (allows adding new types without breaking old clients).
- **Max packet size:** 1500 bytes (fit in one Ethernet frame without IP fragmentation)

### 5.3 Timing requirements

| Event | Interval | Tolerance |
|---|---|---|
| Presence heartbeat | 3 s | ± 500 ms |
| Peer offline threshold | 10 s since last presence | ± 1 s |
| Discovery broadcast (on app start) | 3 rapid sends, 200 ms apart | — |
| Username claim wait | 1.5 s | ± 100 ms |
| Call invite timeout | 10 s | ± 1 s |
| Ringout timeout | 30 s | ± 2 s |
| Audio loss timeout | 2 s | ± 200 ms |
| Claim retry interval (on conflict) | User-driven | — |

---

## 6. Data storage requirements

### 6.1 Persistent storage (must survive app restart)

| Key | Type | Purpose |
|---|---|---|
| `device_id` | uint32 (hex string) | Stable identity |
| `username` | string (or null if deleted) | User handle |
| `claimed_at` | int64 (ms) | Timestamp of last successful claim |
| `pending_delete` | JSON | Crash-safe record during deletion (see SystemDesign §Deletion) |
| `tombstones` | JSON array | Known deletions (max 100 entries) |
| `contacts` | JSON array | (P1) favorited peers |
| `settings` | JSON | App preferences |

**Storage limits:**
- Total persistent storage: **< 100 KB** (all tombstones + contacts fit easily)
- No file system writes; `shared_preferences` only

### 6.2 In-memory state (lost on restart)

| Store | Size | Contents |
|---|---|---|
| `PeerRegistry` | ~50 peers max | Active peers known to this device |
| Current call state | single | State machine + peer reference |
| Jitter buffer | 5 × 640 B = 3.2 KB | Playback queue |
| Audio playback buffer | ~100 ms | `flutter_pcm_sound` internal |

---

## 7. Security requirements (v1)

**Explicitly NOT required in v1** — documented as known limitations:

- Encryption (DTLS, SRTP, Noise, TLS)
- Authentication (device certificates, challenge-response)
- Integrity checks (packet checksums beyond UDP's built-in)
- Anti-replay protection
- Rate limiting

**Must-have security in v1:**

| Requirement | Reason |
|---|---|
| Never store mic audio to disk | Privacy |
| Never send data outside the LAN | Privacy |
| Never include telemetry / analytics SDKs | Privacy |
| Permission handling must be correct | Security |
| Username validation must be strict | Prevent injection / weird states |

**Threat model (v1):** Trusted LAN. An attacker on the same LAN can impersonate peers, forge deletions, or intercept audio. This is **accepted risk** for a v1 designed for home networks. Document prominently in user-facing docs.

---

## 8. Performance requirements

### 8.1 Measured on mid-range device (e.g., Pixel 6a, iPhone SE 2022)

| Metric | Target | Measurement method |
|---|---|---|
| Peer discovery (cold start, 3 peers) | < 500 ms | Time from `initState` to first non-empty peer list |
| Peer discovery (warm, 3 peers) | < 100 ms | Time from foreground to peer list populated |
| Call setup (tap → audio flowing) | < 2 s | End-to-end stopwatch |
| Audio latency (speaker-to-mic, LAN) | < 200 ms | Loopback test with tone bursts |
| CPU during call (both cores) | < 10% | Android Studio profiler |
| Memory during call (RSS) | < 30 MB | Android Studio profiler |
| Battery drain during 10 min call | < 3% | OS battery stats |
| Packet loss (healthy LAN) | < 0.1% | `getStats()`-style counter |

### 8.2 Bottlenecks to avoid

- **Main-isolate blocking:** all audio processing must be non-blocking. If `record.startStream()` or `flutter_pcm_sound.feed()` blocks the main isolate, move to an isolate.
- **Synchronous I/O:** `shared_preferences` writes must be async; never block the UI thread.
- **Unbounded buffers:** jitter buffer capped at 5 slots. Peer registry capped at 100 (drop oldest offline). Tombstones capped at 100.

### 8.3 Load requirements

- Must support up to **20 concurrent peers** on the LAN (peer registry size)
- Must support 1 active call + presence with 20 peers simultaneously
- Must recover from Wi-Fi drop within **10 s** of reconnect

---

## 9. Testing requirements

### 9.1 Test categories

| Category | Coverage target | Tool |
|---|---|---|
| **Unit tests** | Packet encode/decode, jitter buffer, tombstone logic, state machine | `flutter_test` |
| **Widget tests** | Peer list rendering, search, call screens | `flutter_test` |
| **Integration tests** | End-to-end call flow with mock UDP | `flutter_test` + mock socket |
| **Manual tests** | Multi-device discovery, real calls, network edge cases | Device labs |
| **Network tests** | Lossy LAN simulation, high-jitter, AP isolation | `tc netem` on Linux |

### 9.2 Required test scenarios

| # | Scenario | Expected |
|---|---|---|
| T1 | Two devices, both open app | Both see each other within 1 s |
| T2 | Peer goes offline | Status → offline after 10 s |
| T3 | Peer comes back online | Status → online within 3 s |
| T4 | Tap peer → accept → call | Audio flows both directions |
| T5 | Hang up from either side | Both return to IDLE |
| T6 | Delete account | Peer vanishes within 100 ms; username claimable |
| T7 | Two devices claim same username simultaneously | Older claim wins; younger rejected |
| T8 | Wi-Fi switch | Peer list clears; new peers discovered |
| T9 | Kill peer app mid-call | Other side detects loss within 2 s |
| T10 | AP isolation network | Peer list empty; hint shown |

### 9.3 Device matrix

| Device | iOS/Android | Priority |
|---|---|---|
| iPhone 12+ | iOS 15+ | High |
| Samsung Galaxy A-series | Android 12+ | High |
| Pixel 6+ | Android 13+ | High |
| Older low-end Android (2020) | Android 8 | Low (perf test) |
| iPad | iPadOS 15+ | Medium |

---

## 10. Build & deployment

### 10.1 Build configuration

- **Flavor:** `debug`, `profile`, `release`
- **Release mode required** for meaningful perf tests (Dart AOT)
- **No ProGuard/R8** configuration needed (no reflection)
- **Tree shaking:** enabled by default
- **APK split per ABI** for size reduction

### 10.2 CI/CD requirements

- **Pre-merge:** `flutter analyze` + `flutter test` must pass
- **On merge to main:** build APK + IPA (signed) + run integration tests
- **Release:** manual trigger with version bump

### 10.3 App size targets

| Platform | Max size (release) |
|---|---|
| Android APK (arm64) | < 25 MB |
| iOS IPA | < 50 MB |

---

## 11. Coding standards

- **Dart style:** `dart format` enforced
- **Lint:** `flutter_lints` + custom rules from `.analysis_options.yaml`
- **Naming:** `camelCase` for variables/methods, `PascalCase` for classes, `kCamelCase` for constants
- **File size:** soft limit 300 lines, hard limit 500
- **Public APIs:** all public methods documented with `///`
- **State management:** `ChangeNotifier` only; no `setState` in screens (use controller)
- **Error handling:** all `Future` calls wrapped in try/catch; errors surface as typed `AppError`

---

## 12. Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Some Wi-Fi networks block P2P traffic | High | Call doesn't work | Detect + show hint; document in onboarding |
| iOS local network permission denial | Medium | No discovery | Clear prompt + settings link |
| Android OEM backgrounding kills sockets | Medium | Call drops | Foreground-only calls; document |
| Username collisions | Low | UX confusion | Device ID shown alongside; claim protocol |
| Clock skew breaks tombstone comparisons | Low | Deletion edge cases | Use LAN-relative timing or accept |
| `flutter_pcm_sound` breaks on new Android | Low | No audio | Pin version; test on Android 14+ early |
| App store rejection (background audio? unclear) | Low | Release delay | Explicitly foreground-only in review notes |
| User expects internet calling | Medium | Support burden | Clear "LAN only" messaging in app store listing |

---

## 13. Dependencies & assumptions

### 13.1 Assumptions

- All users are on the **same subnet** (single Wi-Fi router)
- Port **5000/UDP is not blocked** by the router
- **AP isolation is disabled** (most home routers)
- Devices have **functioning microphones and speakers**
- Users can **manually coordinate** (e.g., "are you on the same Wi-Fi?")

### 13.2 External dependencies

- **Package updates:** monthly dependency audit
- **iOS SDK / Android SDK updates:** follow stable channels
- **Flutter SDK:** stay on latest stable; upgrade quarterly

---

## 14. Open technical questions

| # | Question | Owner | Blocking |
|---|---|---|---|
| TQ1 | Is `flutter_pcm_sound` still maintained and compatible with Flutter 3.19+? | Eng | M3 |
| TQ2 | Does Android's Doze mode kill our UDP socket mid-call? | Eng | M3 |
| TQ3 | What's the actual UDP broadcast range on common home routers? | Eng | M1 |
| TQ4 | Should we isolate audio processing into a separate isolate from the start? | Eng | M3 |
| TQ5 | Can we implement WSOLA or similar pitch-preserving time-stretch for jitter buffer? | Eng | post-v1 |

---