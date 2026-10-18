# System Design — LAN Voice Call App

**Version:** 1.0
**Status:** Draft
**Last updated:** Oct 19, 2026
**Companion docs:** PRD.md, TRD.md

---

## 1. Overview

This document describes the **architecture, components, protocols, and data flows** of the LAN Voice Call App. It assumes the reader has read `PRD.md` and `TRD.md`.

**Core design principle:** every device on the LAN is equal. There is no host, no server, no coordinator. Any device can call any other device at any time. All state is either local (per-device) or distributed (via broadcast UDP packets).

---

## 2. High-level architecture

```text
┌──────────────────────── Any Device (identical) ────────────────────────┐
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     Presentation Layer                           │  │
│  │  HomeScreen · IncomingCallScreen · InCallScreen · Settings       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                              ▲                                         │
│                              │ (ChangeNotifier)                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    CallController                                │  │
│  │  State machine · Orchestration · Event routing                   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│         │                 │                │                │          │
│         ▼                 ▼                ▼                ▼          │
│  ┌───────────┐    ┌─────────────┐   ┌─────────────┐   ┌───────────┐    │
│  │Discovery  │    │CallSignaling│   │AudioEngine  │   │ Tombstone │    │
│  │Service    │    │             │   │             │   │  Store    │    │
│  └───────────┘    └─────────────┘   └─────────────┘   └───────────┘    │
│         │                 │                │                           │
│         └────────┬────────┴────────────────┘                           │
│                  ▼                                                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    NetworkTransport (UDP)                        │  │
│  │  RawDatagramSocket · Packet routing · Broadcast + unicast        │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                              ▲                                         │
└──────────────────────────────┼─────────────────────────────────────────┘
                               │
                        Wi-Fi LAN (UDP)
```

**Key architectural invariants:**

1. **Every device runs identical code.** Roles (caller/callee, inviter/invitee) are **per-call**, not per-device.
2. **The UI never touches sockets or audio directly.** All interaction goes through `CallController`.
3. **Engines are unaware of each other.** `DiscoveryService` doesn't know `AudioEngine` exists; the controller wires them.
4. **All state changes are observable.** The UI subscribes to `CallController` (which is a `ChangeNotifier`) and re-renders on change.
5. **All network I/O is non-blocking.** `RawDatagramSocket` is event-driven; no polling.

---

## 3. Component breakdown

### 3.1 `NetworkTransport`

**Responsibility:** Own the UDP socket. Encode/decode packets. Route incoming packets to the right handler.

```dart
class NetworkTransport {
  final RawDatagramSocket _socket;
  final StreamController<IncomingPacket> _incoming = ...;

  Stream<IncomingPacket> get incoming => _incoming.stream;

  static Future<NetworkTransport> bind({int port = 5000}) async {
    final socket = await RawDatagramSocket.bind(
      InternetAddress.anyIPv4, port,
    );
    socket.broadcastEnabled = true;
    socket.listen(_onRawData);
    return NetworkTransport._(socket);
  }

  void send(Uint8List bytes, InternetAddress dest, int port) {
    _socket.send(bytes, dest, port);
  }

  void broadcast(Uint8List bytes, {int port = 5000}) {
    _socket.send(bytes, InternetAddress('255.255.255.255'), port);
  }

  void _onRawData(RawSocketEvent event) {
    if (event != RawSocketEvent.read) return;
    final dg = _socket.receive();
    if (dg == null) return;
    final packet = decodePacket(dg.data);
    if (packet != null) {
      _incoming.add(IncomingPacket(packet, dg.address, dg.port));
    }
  }
}
```

**Rules:**
- If port 5000 is in use, retry 5001–5010, report actual port to controller
- Malformed packets (decode returns null) are silently dropped
- Unknown packet types are silently dropped

### 3.2 `DiscoveryService`

**Responsibility:** Broadcast our PRESENCE. Listen for others' PRESENCE and DISCOVERY. Update `PeerRegistry`.

```dart
class DiscoveryService {
  final NetworkTransport transport;
  final PeerRegistry registry;
  final DeviceIdentity identity;
  final TombstoneStore tombstones;
  Timer? _heartbeat;

  void start() {
    // Send DISCOVERY immediately on start for instant peer population
    _broadcastDiscovery();
    Future.delayed(200.ms, _broadcastDiscovery);
    Future.delayed(400.ms, _broadcastDiscovery);

    // Then heartbeat every 3s
    _heartbeat = Timer.periodic(3.seconds, (_) => _broadcastPresence());

    // Listen for incoming
    transport.incoming.listen((p) {
      if (p.packet is PresencePacket) _onPresence(p);
      else if (p.packet is DiscoveryPacket) _onDiscovery(p);
    });
  }

  void _onPresence(IncomingPacket p) {
    if (tombstones.isDeviceDeleted(p.packet.deviceId)) return; // §7 of design
    registry.upsert(p.packet, p.sourceIp);
  }

  void _onDiscovery(IncomingPacket p) {
    // Reply immediately with unicast PRESENCE
    _sendPresenceUnicast(p.sourceIp, p.sourcePort);
  }
}
```

### 3.3 `PeerRegistry`

**Responsibility:** In-memory source of truth for "who's on the network right now."

```dart
class Peer {
  final int deviceId;         // uint32
  String username;            // may change
  String ipAddress;           // updated on each presence
  DateTime lastSeen;
  bool inCall;
  int? callPeerId;
  DateTime? claimedAt;

  String get deviceIdHex => deviceId.toRadixString(16).padLeft(8, '0');
  PeerStatus get status { /* §4.4 of identity design */ }
}

class PeerRegistry extends ChangeNotifier {
  final Map<int, Peer> _peers = {};

  void upsert(PresencePacket p, String srcIp) {
    // update or insert
    notifyListeners();
  }

  List<Peer> search(String q) { /* substring + prefix match */ }
  Peer? byId(int id) => _peers[id];
  Peer? byUsername(String u) { /* null if ambiguous */ }
  void remove(int id) { /* user-triggered or tombstone */ }
  void clearOnlinePeers() { /* on network change */ }
  void pruneStale({Duration threshold = const Duration(days: 7)}) { /* auto-tombstone */ }
}
```

### 3.4 `CallController`

**Responsibility:** The state machine and orchestration hub. Every user action and every incoming packet funnels through here.

See §5 for state machine. Key methods:

```dart
class CallController extends ChangeNotifier {
  CallState state = CallState.idle;
  Peer? currentPeer;
  String? currentCallId;
  Duration callDuration = Duration.zero;

  Future<void> startCall(Peer peer);
  Future<void> acceptIncoming();
  Future<void> rejectIncoming(String reason);
  Future<void> hangUp();
  Future<void> setMuted(bool muted);

  // Internal handlers
  void _onInvite(IncomingPacket p);
  void _onAccept(IncomingPacket p);
  void _onReject(IncomingPacket p);
  void _onBye(IncomingPacket p);
  void _onAudio(IncomingPacket p);
  void _onTimeout();
}
```

### 3.5 `AudioEngine`

**Responsibility:** Own the capture → reframe → transport → jitter → playback pipeline.

```dart
class AudioEngine {
  final NetworkTransport transport;
  final String? Function() getCurrentCallId;
  final int Function() getPeerDeviceId;

  Future<void> start();  // begin capture + playback
  Future<void> stop();
  void setMuted(bool muted);
}
```

**Internally:**

```text
record.startStream() ──► PcmReframer ──► AudioSender ──► transport.send
                                              │
transport.incoming ──► AudioReceiver ──► JitterBuffer ──► flutter_pcm_sound
```

### 3.6 `TombstoneStore`

**Responsibility:** Persistent record of deleted devices and freed usernames. See §7.

### 3.7 `DeviceIdentity`

**Responsibility:** Load/save `device_id`, `username`, `claimed_at`. Handle first-launch setup.

```dart
class DeviceIdentity {
  final int deviceId;
  final String? username;
  final DateTime? claimedAt;

  static Future<DeviceIdentity> load() async { /* shared_preferences */ }
  Future<void> setUsername(String name) async { /* persists */ }
  Future<void> delete() async { /* §8 deletion flow */ }
}
```

---

## 4. Protocol specification

### 4.1 Packet header (all packets)

```text
Offset  Size  Field
0       1     type
1       1     version  (= 0x01)
2       ...   payload (type-specific)
```

- Unknown `type` → drop
- Unknown `version` → drop (allows v2 rollout)
- Total packet size must be ≤ 1500 bytes

### 4.2 Packet types

| Value | Type | Payload |
|---|---|---|
| 0x01 | PRESENCE | `[device_id: 4][uname_len: 1][username: ≤20][timestamp: 8][flags: 1][call_id: 8 or 0]` |
| 0x02 | DISCOVERY | `[device_id: 4]` |
| 0x03 | CALL_INVITE | `[device_id: 4][call_id: 8][call_ts: 8]` |
| 0x04 | CALL_ACCEPT | `[device_id: 4][call_id: 8]` |
| 0x05 | CALL_REJECT | `[device_id: 4][call_id: 8][reason: 1]` |
| 0x06 | CALL_BYE | `[device_id: 4][call_id: 8]` |
| 0x07 | AUDIO | `[call_id: 8][seq: 2][pcm: 640]` |
| 0x08 | CLAIM | `[device_id: 4][uname_len: 1][username: ≤20][claimed_at: 8]` |
| 0x09 | CONFLICT | `[device_id: 4][uname_len: 1][username: ≤20][conflicting_claimed_at: 8][reason: 1]` |
| 0x0A | USER_DELETE | `[device_id: 4][uname_len: 1][username: ≤20][deleted_at: 8]` |
| 0x0B | DISCOVERY_REPLY | (future: tombstone sharing) |

### 4.3 Flags byte in PRESENCE

```text
bit 0: in_call
bit 1: do_not_disturb (reserved)
bit 2–7: reserved, must be 0
```

### 4.4 RECOMMENDED vs REQUIRED handling

- Receivers **MUST** process: PRESENCE, DISCOVERY, CALL_INVITE, CALL_ACCEPT, CALL_REJECT, CALL_BYE, AUDIO, USER_DELETE
- Receivers **SHOULD** process: CLAIM, CONFLICT
- Receivers **MAY** ignore: DISCOVERY_REPLY (future)

---

## 5. State machines

### 5.1 Call state machine (per device)

```text
                     ┌───────────────────────────────────────┐
                     │                                       │
                     ▼                                       │
                   IDLE ◄──────────── ENDING ◄──── IN_CALL   │
                     │                    ▲          ▲       │
     ┌───────────────┼────────────────────┘          │       │
     │               │                               │       │
     │               │ CALL_INVITE recv              │       │
     │               ▼                               │       │
     │            RINGING ────── CALL_ACCEPT ────────┘       │
     │               │  (send ACCEPT)                        │
     │               │                                       │
     │ user cancels  │                                       │
     │ or times out  │                                       │
     │               ▼                                       │
     │             IDLE                                      │
     │                                                       │
     │ user taps peer                                        │
     ▼                                                       │
   CALLING ────── CALL_ACCEPT recv ──────────────────────────┘
     │
     │ CALL_REJECT recv  OR  timeout (10 s)
     ▼
   IDLE
```

**Transition triggers:**

| From | To | Trigger | Actions |
|---|---|---|---|
| IDLE | CALLING | User taps peer | Send CALL_INVITE, start 10s timer |
| IDLE | RINGING | CALL_INVITE received | Show incoming UI, start 30s timer |
| CALLING | IN_CALL | CALL_ACCEPT received | Start AudioEngine |
| CALLING | IDLE | CALL_REJECT received | Show error toast |
| CALLING | IDLE | 10s timeout | Show "unreachable" |
| RINGING | IN_CALL | User taps Accept | Send CALL_ACCEPT, start AudioEngine |
| RINGING | IDLE | User taps Decline | Send CALL_REJECT |
| RINGING | IDLE | 30s timeout | Send CALL_REJECT (reason: no_answer) |
| IN_CALL | ENDING | User taps Hang up | Send CALL_BYE, stop audio |
| IN_CALL | ENDING | CALL_BYE received | Stop audio |
| IN_CALL | ENDING | 2s no-audio timeout | Stop audio, show "connection lost" |
| ENDING | IDLE | After 500 ms | Reset state |

### 5.2 Discovery state (internal to DiscoveryService)

```text
STOPPED ──start()──► RUNNING ──stop()──► STOPPED
                       │
                       │ network change
                       ▼
                    RESTARTING ──► RUNNING
```

On `RESTARTING`: `registry.clearOnlinePeers()`, rebroadcast DISCOVERY ×3, resume heartbeat.

---

## 6. Data flows

### 6.1 App launch → peer list populated

```text
1. main() → load DeviceIdentity, TombstoneStore, PeerRegistry
2. If no username → show setup screen (block until set)
3. NetworkTransport.bind(5000) → success
4. DiscoveryService.start()
   a. Broadcast DISCOVERY ×3 (200ms apart)
   b. Start PRESENCE heartbeat (every 3s)
5. Peers receive DISCOVERY → reply with PRESENCE
6. DiscoveryService._onPresence → registry.upsert → notifyListeners
7. UI rebuilds with populated peer list
8. (typical latency: ~500ms from launch)
```

### 6.2 Outgoing call (A calls B)

```text
A (caller)                          B (callee)
──────────                          ──────────
User taps B
  │
  ▼
CallController.startCall(B)
  ├─ state = CALLING
  ├─ generate callId
  ├─ CALL_INVITE sent ─────────────►
  ├─ start 10s timeout              B's CallController._onInvite
                                      ├─ state = RINGING
                                      ├─ show incoming UI
                                      ├─ start 30s timeout
User accepts on B's screen ◄────────
                                      │
                          CALL_ACCEPT │
  ◄───────────────────────────────────┘
CallController._onAccept
  ├─ cancel timeout
  ├─ state = IN_CALL
  ├─ AudioEngine.start()             CallController._onAccept (self-path)
  ├─ start AUDIO send loop           ├─ state = IN_CALL
  ├─ start AUDIO receive             ├─ AudioEngine.start()
                                     ├─ start AUDIO send
                                     ├─ start AUDIO receive
```

### 6.3 Audio packet flow

**Outgoing:**

```text
Mic → record.startStream() → chunks (variable size, ~2000-8000 bytes)
  ↓
PcmReframer (buffer, emit 640-byte frames)
  ↓
AudioSender (prepend call_id + seq)
  ↓
NetworkTransport.send(peer.ip, 5000)
```

**Incoming:**

```text
NetworkTransport.incoming → AUDIO packet
  ↓
AudioReceiver
  ├─ if packet.callId != currentCallId → drop
  ├─ if seq < expectedSeq - 5 → drop (too late)
  └─ else → jitter buffer
  ↓
JitterBuffer (prebuffer 3 slots, then playout timer every 20ms)
  ├─ expected seq present → emit
  └─ missing → emit silence (640 zero bytes)
  ↓
flutter_pcm_sound.feed(640 bytes)
```

### 6.4 Deletion flow

```text
User taps "Delete account"
  │
  ▼
CallController:
  ├─ if in call: send CALL_BYE, stop AudioEngine
  ├─ persist pending_delete = { old_id, old_username, requested_at }
  ├─ broadcast USER_DELETE ×3 (200ms apart)
  ├─ clear local identity (username = null, deviceId = null)
  ├─ clear PeerRegistry, TombstoneStore
  ├─ generate new device_id (excluding any locally-tombstoned IDs)
  └─ transition UI to setup screen
  │
  ▼
Peers receive USER_DELETE:
  ├─ TombstoneStore.add({deviceId, username, deletedAt, expiresAt: +30d})
  ├─ PeerRegistry.remove(deviceId)
  └─ If in call with deleted peer → end call, show "peer deleted account"
```

**Crash-safe recovery:**

```text
On app launch, before anything else:
  if pending_delete exists in storage:
    ├─ re-send USER_DELETE (using stored old_id, old_username)
    ├─ clear pending_delete
    └─ continue with new identity flow
```

---

## 7. Identity & tombstone design

### 7.1 Identity composition

```text
DeviceIdentity = {
  deviceId: uint32,    // persists forever (until deletion)
  username: String?,   // null after deletion
  claimedAt: DateTime? // last successful claim
}
```

### 7.2 Tombstone composition

```text
Tombstone = {
  deviceId: uint32,
  freedUsername: String,
  deletedAt: DateTime,
  expiresAt: DateTime   // deletedAt + 30 days
}
```

### 7.3 State transitions

```text
                ┌──────────────────────────────────┐
                │                                  │
  ┌─────────────┴──────────┐  user deletes  ┌──────┴─────┐  30 days   ┌───────────┐
  │ ACTIVE                 │ ──────────────►│ DELETED    │ ─────────► │ FORGOTTEN │
  │ (peer in registry,     │                │ (tombstone │            │ (pruned)  │
  │  presence broadcasts)  │                │  exists)   │            │           │
  └────────────────────────┘                └────────────┘            └───────────┘
           ▲                                       │
           │                                       │
           │  auto-tombstone (7 days no presence)  │
           │  if device was never seen again       │
           └────────── (path for ungraceful) ──────┘
```

### 7.4 Tombstone enforcement

Three places where tombstones are checked:

1. **PRESENCE handler** — reject any PRESENCE from a tombstoned `deviceId` (never add to registry)
2. **CLAIM handler** — reject any CLAIM from a tombstoned `deviceId` (reason: `device_deleted`)
3. **Claim comparison** — when processing a claim for a username, if the newest tombstone for that username has `deletedAt > claimedAt`, reject (reason: `username_tombstoned`)

---

## 8. Component interactions

### 8.1 Dependency graph

```text
       ┌───────────────┐
       │ CallController│
       └──────┬────────┘
       ┌──────┴──────┬─────────┬──────────┐
       ▼             ▼         ▼          ▼
 ┌───────────┐ ┌──────────┐ ┌─────────┐ ┌─────────┐
 │ Discovery │ │  Audio   │ │Tombstone│ │Identity │
 │  Service  │ │  Engine  │ │  Store  │ │         │
 └─────┬─────┘ └────┬─────┘ └────┬────┘ └────┬────┘
       │            │            │            │
       └────────────┴────────────┴────────────┘
                        │
                        ▼
                ┌────────────────┐
                │NetworkTransport│
                └────────────────┘
```

**No cycles.** `CallController` orchestrates; all others are leaf modules.

### 8.2 Event routing

Incoming packets go through this pipeline:

```text
RawDatagramSocket
  ↓
NetworkTransport (decode)
  ↓
stream of IncomingPacket
  ├─→ DiscoveryService (PRESENCE, DISCOVERY)
  ├─→ CallController (CALL_*, AUDIO)
  └─→ IdentityService (CLAIM, CONFLICT)
```

Each service filters for the types it cares about. Unknown types are ignored by all — silent drop.

---

## 9. Failure modes & recovery

| Failure | Detection | Recovery |
|---|---|---|
| Port 5000 in use | bind throws | Retry 5001..5010; report actual port |
| Mic permission denied | `record` returns error | Show error, prompt settings |
| Speaker unavailable | `flutter_pcm_sound` throws | Show error, mute audio |
| Peer stops sending presence | 10s timeout in `PeerRegistry` | Mark offline |
| Peer stops sending audio | 2s timeout in AudioEngine | End call ("connection lost") |
| Call invite not answered | 10s timer in CallController | Cancel, return to IDLE |
| Incoming call not answered | 30s timer in CallController | Send CALL_REJECT, return to IDLE |
| Wi-Fi disconnect | `connectivity_plus` event | Clear peers, restart discovery on reconnect |
| App crash mid-call | N/A (crash) | On restart, no state to recover; user starts fresh |
| App crash mid-deletion | `pending_delete` persisted | Retry USER_DELETE on next launch |
| Two simultaneous invites to same peer | Peer rejects second with "busy" | Caller sees "user is in another call" |

---

## 10. Storage schemas

### 10.1 `shared_preferences` keys

| Key | Value | Example |
|---|---|---|
| `device_id` | String (decimal) | `"2947561821"` |
| `username` | String | `"alice"` |
| `claimed_at` | String (ISO 8601) | `"2026-10-19T10:30:00Z"` |
| `pending_delete` | String (JSON) | `{"old_id":123,"old_username":"alice","ts":...}` |
| `tombstones` | String (JSON array) | `[{"device_id":456,"username":"bob","deleted_at":...,"expires_at":...}]` |
| `contacts` | String (JSON array) | `[{"username":"priya","device_id":789}]` |

### 10.2 JSON schemas

**Tombstone:**

```json
{
  "device_id": 456,
  "username": "bob",
  "deleted_at": 1729344600000,
  "expires_at": 1731936600000
}
```

**Pending delete:**

```json
{
  "old_id": 123,
  "old_username": "alice",
  "requested_at": 1729344500000
}
```

**Contact:**

```json
{
  "username": "priya",
  "device_id": 789,
  "added_at": 1729344500000
}
```

---

## 11. Constants (`lib/core/constants.dart`)

```dart
// Audio
const int kSampleRate = 16000;
const int kChannels = 1;
const int kBitsPerSample = 16;
const int kFrameDurationMs = 20;
const int kSamplesPerFrame = 320;         // 16000 * 0.02
const int kBytesPerFrame = 640;           // 320 * 2

// Network
const int kDefaultPort = 5000;
const int kPortRetryMax = 10;             // 5000..5010
const int kMaxPacketSize = 1500;

// Timing
const Duration kPresenceInterval = Duration(seconds: 3);
const Duration kPresenceTimeout = Duration(seconds: 10);
const Duration kDiscoveryBurst = Duration(milliseconds: 200);
const int kDiscoveryBurstCount = 3;
const Duration kClaimWait = Duration(milliseconds: 1500);
const Duration kCallInviteTimeout = Duration(seconds: 10);
const Duration kRingoutTimeout = Duration(seconds: 30);
const Duration kAudioLossTimeout = Duration(seconds: 2);

// Jitter buffer
const int kJitterBufferSlots = 5;
const int kPrebufferCount = 3;

// Tombstones
const Duration kAutoTombstoneAfter = Duration(days: 7);
const Duration kTombstoneLifetime = Duration(days: 30);
const int kMaxTombstones = 100;

// Registry
const int kMaxPeers = 100;

// Username
const int kUsernameMinLen = 3;
const int kUsernameMaxLen = 20;
final RegExp kUsernamePattern = RegExp(r'^[a-z][a-z0-9_]{2,19}$');
const Set<String> kReservedUsernames = {'me', 'all', 'system', 'admin'};

// Packet versions
const int kProtocolVersion = 0x01;

// Packet types
const int kTypePresence = 0x01;
const int kTypeDiscovery = 0x02;
const int kTypeCallInvite = 0x03;
const int kTypeCallAccept = 0x04;
const int kTypeCallReject = 0x05;
const int kTypeCallBye = 0x06;
const int kTypeAudio = 0x07;
const int kTypeClaim = 0x08;
const int kTypeConflict = 0x09;
const int kTypeUserDelete = 0x0A;

// Conflict reasons
const int kConflictUsernameTaken = 0x01;
const int kConflictDeviceDeleted = 0x02;
const int kConflictUsernameTombstoned = 0x03;
```

---

## 12. Directory structure

```text
lib/
├── main.dart
├── app.dart
│
├── core/
│   ├── constants.dart
│   ├── app_error.dart
│   └── logging.dart
│
├── identity/
│   ├── device_identity.dart
│   ├── username_validator.dart
│   └── claim_coordinator.dart
│
├── discovery/
│   ├── discovery_service.dart
│   ├── peer_registry.dart
│   ├── peer.dart
│   └── tombstone_store.dart
│
├── call/
│   ├── call_controller.dart
│   ├── call_state.dart
│   ├── call_id.dart
│   └── signaling.dart
│
├── audio/
│   ├── audio_engine.dart
│   ├── audio_capture.dart
│   ├── pcm_reframer.dart
│   ├── audio_sender.dart
│   ├── audio_receiver.dart
│   ├── jitter_buffer.dart
│   └── audio_playback.dart
│
├── network/
│   ├── network_transport.dart
│   ├── packets.dart              # encode/decode all types
│   ├── packet_types.dart         # sealed class hierarchy
│   └── network_watcher.dart
│
├── screens/
│   ├── setup_screen.dart
│   ├── home_screen.dart
│   ├── incoming_call_screen.dart
│   ├── outgoing_call_screen.dart
│   ├── in_call_screen.dart
│   └── settings_screen.dart
│
└── widgets/
    ├── peer_card.dart
    ├── search_bar.dart
    ├── call_timer.dart
    └── status_indicator.dart
```

---

## 13. Build & test sequence

Recommended implementation order (each step independently testable):

| Step | Build | Test |
|---|---|---|
| 1 | `NetworkTransport` + broadcast echo | Two phones see each other's echo logs |
| 2 | `packets.dart` codec (all types) | Unit tests round-trip all packets |
| 3 | `DeviceIdentity` + persistence | Relaunch → same device ID |
| 4 | `PeerRegistry` + presence heartbeat | Peer list populates within 1 s |
| 5 | `TombstoneStore` (empty stub) | Persistence works |
| 6 | `DiscoveryService` full | Peer status (online/offline) correct |
| 7 | `CallController` state machine (no audio) | Tap peer → rings → accept → "connected" state |
| 8 | `AudioEngine` capture only | Logs show 640-byte frames at 50/sec |
| 9 | `AudioEngine` playback (one direction) | One-way voice works |
| 10 | `AudioEngine` both directions | Two-way voice call |
| 11 | `JitterBuffer` | Audio smooths out |
| 12 | `claim_coordinator` | Two devices claim same username → conflict resolution |
| 13 | `USER_DELETE` flow | Peer vanishes, username freed |
| 14 | `pending_delete` crash-safe recovery | Kill app mid-deletion → retries on relaunch |
| 15 | Network watcher | Wi-Fi switch clears peers |
| 16 | Full UI polish | Complete MVP |

---

## 14. Design decisions & rationale

| Decision | Rationale | Alternatives rejected |
|---|---|---|
| **Raw PCM over UDP** | Simplest pipeline; LAN bandwidth is free; teaches fundamentals | Opus (adds codec complexity); MP3 (bad for real-time) |
| **Single port 5000** | Simpler than negotiated ports; all traffic on one channel | Separate ports for discovery/media (adds coordination) |
| **Broadcast discovery** | Zero config; works on all home routers | mDNS/Bonjour (complex, intermittent issues); manual IP (bad UX) |
| **Presence every 3 s** | Balances freshness with battery | 1 s (too chatty); 10 s (slow updates) |
| **10 s offline threshold** | Tolerates 3 missed heartbeats | 5 s (too aggressive); 30 s (stale UI) |
| **20 ms audio frames** | Sweet spot for latency vs overhead; matches Opus | 10 ms (more overhead); 40 ms (more latency) |
| **3-slot prebuffer** | 60 ms hides typical jitter | 1 slot (crackly); 5+ slots (more latency) |
| **uint32 device ID** | Small (4 bytes), negligible collision risk | UUID (16 bytes, unnecessary); uint64 (8 bytes) |
| **Tombstones for deletion** | Serverless, distributed, self-healing | Server-side registry (needs server) |
| **7-day auto-tombstone** | Handles ungraceful deletions | Never free (unfair); instant free (user hostile) |
| **30-day tombstone lifetime** | Long enough to prevent resurrection abuse | Permanent (storage grows); 7 days (too short) |
| **Claim protocol with 1.5 s wait** | Best-effort uniqueness without server | No protocol (frequent collisions); server (defeats purpose) |
| **Foreground-only calls** | Mobile OS suspends UDP in background | Background audio permission (unreliable, app store scrutiny) |

---

## 15. Open design questions

| # | Question | Impact |
|---|---|---|
| DQ1 | Should we implement WSOLA time-stretch in the jitter buffer for smoother underrun handling? | Audio quality (post-v1) |
| DQ2 | Should presence use unicast instead of broadcast after discovery (to reduce network load)? | Scalability on large LANs |
| DQ3 | Should tombstone sharing (DISCOVERY_REPLY with recent tombstones) be in v1? | Faster claim conflict detection for new devices |
| DQ4 | Should audio capture run on a separate isolate? | CPU/latency on low-end devices |
| DQ5 | How should we handle a peer that changes IP mid-call (DHCP renewal)? | Rare but possible call drop |
| DQ6 | Should the app support Bluetooth tethering as a "LAN" in v1? | Extends reach; adds platform complexity |

---

## 16. Summary

The system is **16 components**, **11 packet types**, **2 state machines** (call + discovery), and **~30 constants**. Everything else is standard Flutter.

**What makes it work:**
- Uniform peer-to-peer architecture — no privileged node
- Broadcast discovery + unicast media
- Heartbeat-driven presence with bounded timeouts
- Serverless identity via claim protocol + tombstones
- Raw PCM over UDP for the simplest possible audio path
- Jitter buffer for smooth playback over lossy UDP

**What it deliberately doesn't do (v1):**
- Encryption, authentication, authorization
- Group calls, video, recording
- Internet calling, NAT traversal
- Echo cancellation, noise suppression

**Ready to build.** Start with step 1 (broadcast echo). Ship when all 16 build steps pass their tests.

---