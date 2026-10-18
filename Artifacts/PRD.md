# Product Requirements Document — LAN Voice Call App

**Version:** 1.0
**Status:** Draft
**Last updated:** Oct 19, 2026
**Owner:** jakegithub24

---

## 1. Overview

### 1.1 What this is

A Flutter application that enables **live voice calls between devices on the same local Wi-Fi network**, with **no server, no cloud, no account system, and no internet dependency**. Every device on the LAN discovers every other device, and any two devices can call each other by tapping a peer or searching by `@username`.

### 1.2 Why

Existing voice-call solutions (WhatsApp, Signal, Zoom, Google Meet) all require internet connectivity, server infrastructure, and phone-number or email-based accounts. They fail when:

- The internet is down but local Wi-Fi works (offices, schools, homes with ISP outages)
- The user wants private, non-trail-leaving voice communication
- The environment is air-gapped (labs, secure facilities, ships, remote sites)
- The user wants to talk to someone physically nearby without going through a third party

This app solves those problems by treating the **local Wi-Fi network as the entire communication universe**.

### 1.3 Target users

| User | Context | Why they'd use it |
|---|---|---|
| Home users | House with multiple devices | Talk room-to-room without internet |
| Office teams | Shared office Wi-Fi | Quick voice chats without meeting software |
| Event venues | Conference hall, hackathon | Ad-hoc communication |
| Schools / classrooms | Managed Wi-Fi | Teacher-to-student, group coordination |
| Off-grid / remote sites | Ships, oil rigs, field operations | Voice where no internet exists |

### 1.4 What this is not

- **Not a replacement for internet calling.** It only works on the local network.
- **Not encrypted.** Designed for trusted networks. See §8.
- **Not group calling.** 1-to-1 only.
- **Not a directory service.** You can only find peers currently (or recently) on your LAN.

---

## 2. Goals & non-goals

### 2.1 Goals

1. **Zero infrastructure.** No server, no cloud, no accounts. Works on a single home router.
2. **Automatic peer discovery.** Devices find each other within 1 second of opening the app.
3. **Human-friendly identity.** Users are `@username`s, not IP addresses.
4. **Fast call setup.** Tap → ring → connect in under 3 seconds on a healthy LAN.
5. **Clear voice quality.** Understandable speech with < 200 ms end-to-end latency.
6. **Graceful deletion.** Deleting the app frees the username for future use.
7. **Understandable failure.** When things break, users know why (bad network, wrong password, etc.).

### 2.2 Non-goals

1. Group calls (mesh or SFU) — future version
2. Video — future version
3. Encryption / E2E — out of scope for v1
4. Internet calling / NAT traversal — future version, WebRTC branch
5. Cross-subnet / VPN support — out of scope
6. Recording calls — out of scope
7. Echo cancellation — deferred (see §8)
8. Persistent user accounts across LANs — local persistence only

---

## 3. Feature list (prioritized)

### P0 — MVP (must ship for v1)

| # | Feature | Description |
|---|---|---|
| F1 | Device identity | Every device has a persistent `device_id` (uint32) + user-chosen `@username` |
| F2 | Auto-discovery | All devices on the same LAN appear in each other's peer lists within 1 s |
| F3 | Peer list | Home screen shows all online/recently-seen peers with status (online/busy/offline) |
| F4 | Search | Filter peers by username substring or device ID prefix |
| F5 | 1-to-1 voice call | Tap peer → ring → accept → two-way audio |
| F6 | Call controls | Mute, hang up, call timer |
| F7 | Incoming call UI | Full-screen incoming call notification with accept/decline |
| F8 | Presence status | Online / in-call / offline with "last seen X ago" |
| F9 | Username claim | User picks username; conflicts resolved via LAN protocol |
| F10 | Account deletion | User can delete account; device ID tombstoned; username freed |
| F11 | Network change handling | Moving to another Wi-Fi clears stale peers and rediscovers |

### P1 — Nice-to-have (v1.x)

| # | Feature | Description |
|---|---|---|
| F12 | Contacts / favorites | Persist `{username, deviceId}` pairs locally |
| F13 | Call history | List of past calls with duration |
| F14 | Display name change | Rename without deleting account |
| F15 | Do-not-disturb | Suppress incoming calls for X minutes |
| F16 | Ringtone + vibration | Standard call notification behavior |

### P2 — Future (v2+)

| # | Feature | Description |
|---|---|---|
| F17 | Group calls | 3+ participants |
| F18 | Internet calling | WebRTC branch with STUN/TURN |
| F19 | Encryption | DTLS or Noise protocol |
| F20 | Video | 1-to-1 video over LAN |
| F21 | Recording | Save calls locally |
| F22 | File/photo sharing | P2P transfer over LAN |

---

## 4. User stories & acceptance criteria

### US-1: First launch & identity setup

> **As a** new user, **I want to** set a username and start using the app, **so that** I can identify myself to others on my network.

**Acceptance criteria:**
- [ ] App requests Wi-Fi permission (if required by platform)
- [ ] App prompts user to choose a `@username`
- [ ] Format validation: 3–20 chars, starts with letter, `a-z0-9_` only
- [ ] App checks username availability (~1.5 s LAN query)
- [ ] On success: username persisted, home screen loads
- [ ] On conflict: helpful error with suggested alternatives
- [ ] App requests microphone permission before first call

### US-2: See peers on my network

> **As a** user, **I want to** see everyone on my Wi-Fi, **so that** I know who I can call.

**Acceptance criteria:**
- [ ] Within 1 second of opening the app, all online peers appear in the list
- [ ] Each peer row shows: `@username · deviceID · status`
- [ ] Status accurately reflects: online, in-call, offline-with-last-seen
- [ ] Peer list updates live (within ~3 s) as peers come/go/change status
- [ ] If the network blocks peer-to-peer traffic, show a clear hint

### US-3: Call someone

> **As a** user, **I want to** call a peer with one tap, **so that** I can start talking quickly.

**Acceptance criteria:**
- [ ] Tapping an online peer initiates a call
- [ ] Caller sees "Ringing…" with an option to cancel
- [ ] Callee sees a full-screen incoming call with accept/decline
- [ ] On accept, both transition to in-call within ~300 ms
- [ ] Two-way audio is intelligible
- [ ] End-to-end latency < 200 ms
- [ ] Either party can hang up; both return to IDLE within 1 s
- [ ] Call timer displays duration

### US-4: Search for a specific peer

> **As a** user, **I want to** filter the peer list by typing part of a username, **so that** I can find someone quickly on a busy network.

**Acceptance criteria:**
- [ ] Search bar at top of home screen
- [ ] Typing filters list in real-time (no search button)
- [ ] Matches on both `@username` substring and `device_id` prefix
- [ ] Empty query shows all peers
- [ ] Case-insensitive matching
- [ ] Results show status inline

### US-5: Delete my account

> **As a** user, **I want to** delete my account **so that** I can reset my identity or free my username.

**Acceptance criteria:**
- [ ] Settings screen has "Delete account" with clear confirmation
- [ ] Confirmation dialog warns: "This will delete your identity and free @username"
- [ ] On confirm:
  - [ ] Active call (if any) is ended
  - [ ] `USER_DELETE` broadcast to all peers (3× for reliability)
  - [ ] Local identity cleared
  - [ ] New `device_id` generated on next app launch
- [ ] Peers see the deleted peer vanish from their lists within ~100 ms
- [ ] The freed username is claimable by others immediately
- [ ] If the app crashes mid-deletion, the deletion retries on next launch

### US-6: Recover from network issues

> **As a** user, **I want to** know when the call is failing **so that** I can act (move closer to router, hang up).

**Acceptance criteria:**
- [ ] If audio stops flowing for 2 s, show "Connection lost" and end the call
- [ ] If a peer goes offline mid-ring, show "Peer unreachable"
- [ ] If the network changes (Wi-Fi switch), clear peer list and re-discover
- [ ] All errors have human-readable messages

---

## 5. Non-functional requirements

### 5.1 Performance

| Metric | Target | Limit |
|---|---|---|
| Peer discovery latency (cold) | < 500 ms | 2 s |
| Peer discovery latency (warm) | < 100 ms | 500 ms |
| Call setup (tap → in-call) | < 2 s | 5 s |
| End-to-end audio latency | < 200 ms | 400 ms |
| Frame cadence jitter | < 5 ms | 20 ms |
| Call drop rate on healthy LAN | < 0.1% | 1% |
| CPU during call | < 10% | 25% |
| Memory during call | < 30 MB | 60 MB |

### 5.2 Reliability

- **Presence**: peer considered offline after 10 s of silence (3 missed heartbeats)
- **Call loss detection**: 2 s of no audio → end call
- **Deletion recovery**: ungraceful deletions handled by 7-day auto-tombstone
- **Crash recovery**: pending-delete record survives restart

### 5.3 Compatibility

- **Platforms**: iOS 14+, Android 8+ (API 26+)
- **Flutter**: 3.19+
- **Networks**: Standard home/office Wi-Fi (802.11n/ac). Must handle networks that block peer-to-peer traffic gracefully.
- **Devices**: 2+ devices on the same subnet, port 5000 reachable

### 5.4 Accessibility

- All status changes must have non-color indicators (icons + text)
- Screen reader labels for peer rows
- Touch targets ≥ 48dp
- Dark mode support

### 5.5 Privacy

- **No data leaves the LAN.** No telemetry, no analytics, no crash reporting to external services.
- **No accounts.** No email, no phone number, no password.
- **No persistent logs** of call history unless the user explicitly enables it (P1 feature).
- Peer identity is visible to anyone on the same LAN. This is by design (it's a LAN app).

---

## 6. Constraints

### 6.1 Environmental constraints

- Both devices **must be on the same Wi-Fi network / subnet**
- The Wi-Fi network **must allow device-to-device communication**. Some routers (guest networks, enterprise Wi-Fi, hotel networks) block this via "AP isolation" or "client isolation." The app cannot work around this.
- Port **5000/UDP must be reachable** between devices

### 6.2 Identity constraints

- **`device_id`** is a random 32-bit integer. Collision probability is negligible at LAN scale.
- **`@username`** is user-chosen. It is **not guaranteed globally unique** — collision resolution is best-effort via LAN protocol.
- **Usernames are only findable if the device is online (or recently seen)**. There is no directory server.

### 6.3 Deletion constraints

- **Graceful deletion** (app online) → username freed immediately
- **Ungraceful deletion** (uninstall, crash) → username freed after **7 days** of silence by every peer
- Tombstones expire after **30 days**
- A deleted `device_id` **never returns** — reinstalls generate a new ID

### 6.4 Security constraints

- **No encryption in v1.** Suitable only for trusted LANs.
- **No authentication.** Any device can spoof any `device_id` or `username` on the LAN.
- **No authorization.** Any device can send any packet type.
- These are **documented limitations**, not bugs. v2 will add encryption.

---

## 7. User experience principles

1. **Zero-friction first launch.** The user should be talking within 30 seconds of installing the app.
2. **Show, don't hide.** Presence and connection state are always visible.
3. **Fail loudly.** When something doesn't work (network blocks P2P, mic denied), the user is told exactly why.
4. **No dead ends.** Every error state has a recovery action.
5. **Local-first.** No spinners waiting for a server. Everything is instant or fails within a bounded time.
6. **Trust the LAN.** Assume the user's Wi-Fi is trustworthy until told otherwise.

---

## 8. Known limitations (v1)

These are **by design**, not bugs. They'll be addressed in future versions or explicitly out of scope.

| Limitation | Reason | Workaround |
|---|---|---|
| **No encryption** | v1 ships without it | Use on trusted Wi-Fi only |
| **No group calls** | Complexity | Use 1-to-1 calls; group is v2 |
| **No internet calls** | Serverless design | WebRTC branch is future work |
| **Echo/feedback when phones are close** | No AEC in v1 | Use headphones or separate rooms |
| **Some Wi-Fi networks block P2P** | Router config | Try a home router or phone hotspot |
| **Manual username conflicts possible** | No central registry | 8-char `device_id` shown for disambiguation |
| **Auto-tombstone after 7 days may delete inactive users** | No server = no authoritative "still alive" signal | Keep users active; or add contacts (P1) |
| **App backgrounded = call drops** | OS suspends UDP | Foreground-only calls |
| **No cross-subnet** | Broadcast-based discovery | Single-router networks only |

---

## 9. Success metrics

These will be evaluated after a 4-week beta with 20+ users on various home/office networks.

| Metric | Target |
|---|---|
| First-call success rate | > 90% |
| Call setup time (p50) | < 2 s |
| Call setup time (p95) | < 4 s |
| Call audio quality (user-rated 1–5) | > 3.5 |
| Peer discovery success rate | > 95% on home networks |
| Crash-free sessions | > 99% |
| User-reported "didn't work" reasons categorized | 100% (all failures explainable) |

---

## 10. Milestones

| Phase | Deliverable | Features |
|---|---|---|
| **M1** | Discovery working on 2 phones | F1, F2 |
| **M2** | Peer list + search + presence | F3, F4, F8 |
| **M3** | Full call flow | F5, F6, F7 |
| **M4** | Identity + deletion | F9, F10 |
| **M5** | Network resilience | F11 |
| **M6** | Polish + testing | All P0, docs |
| **Beta** | Public testing | P0 + bug fixes |
| **v1.0** | App store release | P0 + selected P1 |

---

## 11. Open questions

| # | Question | Owner | Deadline |
|---|---|---|---|
| Q1 | Should we show peer device names (like "Pixel 7") in addition to `@username`? | Design | Before M2 |
| Q2 | What's the default ringout timeout — 30 s or 60 s? | Product | Before M3 |
| Q3 | Do we need a "silent ring" mode for quiet environments? | Product | v1.x consideration |
| Q4 | Should deletion have a "cooling off" period (7 days to undo)? | Product | Before M4 |
| Q5 | How do we handle two devices on the same LAN trying to claim the same username within milliseconds? | Engineering | During M4 |

---

## 12. Appendix: glossary

| Term | Meaning |
|---|---|
| **Device ID** | Persistent random 32-bit identifier for a physical device |
| **Username** | User-chosen `@handle`, human-readable, not guaranteed unique |
| **Peer** | Another device currently visible on the LAN |
| **Presence** | A heartbeat packet indicating "I'm here and what my status is" |
| **Tombstone** | A record that a `device_id` has been deleted, used to prevent resurrection |
| **Claim** | The act of reserving a username via LAN broadcast |
| **LAN** | Local Area Network — same Wi-Fi network |
| **P2P** | Peer-to-peer — direct device-to-device communication |
| **AEC** | Acoustic Echo Cancellation — not implemented in v1 |

---