# MeshLink — Offline Emergency Messenger

**Status: Phase 3 of 8 complete** (real offline mesh communication layer)

## What this is right now

On top of Phase 2's Room persistence, MeshLink now has a real
device-to-device transport layer using **Google Nearby Connections**
(`com.google.android.gms:play-services-nearby`), which is what "Bluetooth
/ Wi-Fi Nearby APIs" points at: under the `P2P_CLUSTER` strategy it uses
BLE for discovery/advertising and Bluetooth/Wi-Fi for the actual payload
transfer — entirely on-device, no internet round trip, no server relaying
messages, no Firebase/WhatsApp/Telegram anywhere in this codebase.

- **Real discovery & advertising** — tap SCAN on the Nearby tab and this
  phone both looks for and announces itself to other MeshLink phones over
  BLE, using each phone's persistent local user ID (not Nearby's ephemeral
  session ID) so the same peer is recognized across app restarts
- **Real connection lifecycle** — AVAILABLE → CONNECTING → CONNECTED →
  DISCONNECTED, driven entirely by Nearby Connections callbacks, persisted
  to Room in real time
- **Real message transfer** — chat messages and emergency broadcasts are
  serialized to a small JSON envelope and sent as a Nearby BYTES `Payload`
  directly to the connected peer's device
- **Real SENT / DELIVERED / FAILED** — `SENT` means Nearby confirmed the
  bytes physically reached the peer's radio stack; `DELIVERED` means the
  receiving phone's app durably stored it in Room and sent an app-level
  ack back; `FAILED` means the transfer genuinely failed, was canceled, or
  expired past its TTL. None of these three states can be reached any
  other way — no code path fakes them
- **Automatic retry-on-connect** — a message that queued while a peer was
  offline is automatically sent the moment that peer connects
- **Runtime permissions** handled per Android version (see below), with a
  denial handled safely — no crash, no fake "connected", a visible
  "Permission needed" card with the exact reason for each permission
- **First real relay step** — EMERGENCY broadcasts are forwarded to every
  other currently connected peer (hop-count capped, duplicate-messageId
  guarded), which is genuinely working multi-hop for the broadcast case,
  while full arbitrary-topology relay for 1:1 chat is still Phase 5

## Files created (Phase 3)

```
app/src/main/java/com/meshlink/app/mesh/
├── MeshTransportManager.kt   (Nearby Connections wrapper — the real radio layer)
├── MeshEnvelope.kt           (JSON wire format sent as Payload bytes)
├── MeshTransportEvent.kt     (sealed events the transport reports up)
└── MeshPermissions.kt        (per-API-level permission list + rationale text)

app/src/main/java/com/meshlink/app/viewmodel/
└── NearbyViewModel.kt        (drives Nearby + Mesh screens' real state)
```

## Files modified (Phase 3)

- `app/build.gradle.kts` — added `play-services-nearby:19.3.0`
- `AndroidManifest.xml` — replaced the Phase 1 placeholder comment with the
  actual permissions (Bluetooth Scan/Advertise/Connect for API 31+, legacy
  Bluetooth for ≤30, fine location for ≤32, Nearby Wi-Fi Devices for 33+,
  Wi-Fi state, and two optional `<uses-feature>` so the app still installs
  on a device missing BLE or Wi-Fi hardware) — **still no `INTERNET`
  permission anywhere**
- `data/repository/MeshRepository.kt` / `MeshRepositoryImpl.kt` — added
  `observeMeshStatus`, `observeDiscoveredPeers`, `startDiscovery`,
  `stopDiscovery`, `connectToPeer`, `disconnectFromPeer`; `sendMessage`/
  `sendEmergencyMessage` now attempt a real transport send; a new
  transport-event pipeline persists discovery/connection/receive/ack/
  transfer-result events to Room; a periodic sweep marks past-TTL QUEUED
  messages FAILED
- `data/ServiceLocator.kt` — repository now also takes an application
  `Context` (needed by `Nearby.getConnectionsClient`)
- `data/local/dao/{MessageDao,ConversationDao,PeerDeviceDao}.kt` — added
  the handful of queries the transport pipeline needs (flush-on-connect,
  peer lookup by ID, connected-peer-ID list, TTL sweep)
- `data/Mappers.kt` — added `PeerDeviceEntity.toUi()`
- `viewmodel/HomeViewModel.kt` — `meshStatus` is now a real `StateFlow`
  from the repository instead of a hardcoded `OFFLINE` constant
- `viewmodel/EmergencyViewModel.kt` — `sendEmergency` now reports
  sent-vs-no-route based on the actual persisted result, not a guess
- `ui/screens/{HomeScreen,NearbyScreen,MeshScreen,EmergencyScreen}.kt` —
  wired to real ViewModel state; Nearby got a real SCAN/CONNECT/DISCONNECT
  flow with permission handling; Mesh shows real Connected/Available
  counts above its still-illustrative (clearly labeled) topology diagram
- `navigation/MeshLinkNavGraph.kt` — wires `NearbyViewModel` into the
  Nearby and Mesh routes

## Complete manual code check (this sandbox still cannot compile Android code)

Same limitation as Phase 2: **no Kotlin compiler, Gradle, or Android SDK
in this sandbox, and no network access to fetch them — I did not run
`./gradlew assembleDebug`, and I'm not claiming I did.** What I did this
pass, specifically for Phase 3:

1. Brace/paren balance check across all 40 Kotlin files (0 mismatches).
2. `xml.dom.minidom` parse of `AndroidManifest.xml` (well-formed).
3. Cross-referenced **every** DAO method called from
   `MeshRepositoryImpl.kt` against its DAO interface — exact 1:1 match on
   all 5 DAOs (`grep`-verified, shown in this session's tool output).
4. Cross-referenced every `transportManager.*` call in
   `MeshRepositoryImpl.kt` against `MeshTransportManager`'s actual public
   surface, and every `MeshTransportEvent.*` subtype referenced against
   the sealed class's 10 defined subtypes — exact match on both.
5. Verified `MessageEntity.toOutgoingEnvelope()`'s named arguments against
   both `MessageEntity`'s real fields and `MeshEnvelope`'s constructor
   parameters, field-by-field.
6. Checked the Nearby Connections API calls in `MeshTransportManager.kt`
   (`startDiscovery`, `startAdvertising`, `requestConnection`,
   `acceptConnection`, `sendPayload`, `disconnectFromEndpoint`,
   `stopAllEndpoints`, the `EndpointDiscoveryCallback` /
   `ConnectionLifecycleCallback` / `PayloadCallback` methods, `Payload`/
   `PayloadTransferUpdate` property access) against the real, published
   Nearby Connections API shape.
7. Confirmed `androidx.core:core-ktx` (needed for `ContextCompat` in
   `MeshPermissions.kt`) and `androidx.activity:activity-compose` (needed
   for `rememberLauncherForActivityResult` in `NearbyScreen.kt`) were
   already present from Phase 1 — nothing extra needed beyond the one new
   `play-services-nearby` dependency.

This is a careful, real review — but static reading still cannot catch
everything an actual compiler and annotation processor (Room's own
validation, AGP/SDK/Nearby version interactions) would. **A real build in
Android Studio is still required before you trust this on real hardware.**

## Known, documented simplifications (not hidden, not silently "fake")

- **Connections auto-accept.** There is no pairing-confirmation UI and no
  cryptographic identity check yet — any device advertising the MeshLink
  service ID is accepted automatically. This is explicitly what Phase 7
  (Security) exists to fix. It does not affect whether the underlying
  transfer is real.
- **Emergency relay is broadcast-only.** Phase 3's relay forwards
  EMERGENCY messages to every other connected peer (with hop-count +
  duplicate-ID guards). General 1:1 chat relay across an arbitrary,
  multi-hop mesh topology is Phase 5's job — the wire format already
  carries what that needs (`hopCount`, `expiryTime`), but the routing
  logic for it doesn't exist yet.
- **Requires Google Play services on the test device.** Nearby Connections
  is a Play services API. It will not work on a bare AOSP emulator image
  without Play services, or on a device with Play services disabled. Any
  regular consumer Android phone has this.

## Exact Android Studio setup

1. Unzip and open the `MeshLink/` folder in Android Studio (Koala/2024.1+).
2. Let Gradle sync — needs internet the first time, to fetch the Gradle
   wrapper, AndroidX/Compose dependencies, and the new
   `play-services-nearby` dependency.
3. Deploy to **two separate physical Android phones** (API 26+, with
   Google Play services — virtually any real phone qualifies). An emulator
   can run the app but **cannot** do real Bluetooth/Wi-Fi discovery with
   another device, so two-phone testing needs real hardware.

## Two-phone testing steps

1. **Install MeshLink on both phones**, from Android Studio (`Run ▶` with
   each phone selected as the target in turn, or `Build > Build Bundle(s)
   / APK(s) > Build APK(s)` and sideload the resulting `.apk` on both).
2. **On both phones:** open MeshLink, go to the **Nearby** tab, tap
   **SCAN**.
3. **Grant permissions when prompted** on each phone (Bluetooth
   Scan/Advertise/Connect and either Location or Nearby Wi-Fi Devices,
   depending on the phone's Android version — see the in-app "Permission
   needed" card if you deny by mistake, it explains exactly what's needed
   and lets you retry).
4. **Keep both phones within Bluetooth/Wi-Fi range** (a few meters) with
   both apps in the foreground and scanning.
5. Within a few seconds, each phone should show the other under **Real
   Devices** with status **Available**. If not, see Troubleshooting below.
6. **On Phone A**, tap **CONNECT** next to Phone B's entry.
7. Both phones should transition through **Connecting…** to **Connected**
   within a couple of seconds — this is a real Bluetooth/Wi-Fi link, not a
   simulation.
8. **On Phone A's Home tab**, tap **+**, but instead of naming a manual
   peer, go back to **Nearby** and note that a conversation was *not* yet
   auto-created — that happens the moment a message is actually
   exchanged. Simplest path: from Nearby, there's currently no "message
   this peer" shortcut, so the cleanest test is via Emergency (see step
   10) or by using the automatically-created conversation once Phone A
   sends the first message — **easiest test:** on Phone A, use the
   Emergency screen (step 10) first, since it doesn't require a pre-made
   conversation.

   *(If you'd like a direct "Message this real peer" button added to
   Nearby, tell me — Phase 3 didn't explicitly ask for that shortcut, but
   it wires up cleanly with what's already here.)*
9. **Verify persistence:** whatever conversation gets created (see step
   10), force-stop MeshLink on either phone and relaunch — the
   conversation and message history are still there (real Room
   persistence, unaffected by the transport layer at all).
10. **Test a real emergency broadcast:**
    - On Phone A, go to **Emergency**, type a message, tap the big button,
      confirm.
    - Because Phone A and B are connected, you should see **"SENT TO
      CONNECTED PEERS"** (green), not "NO ROUTE".
    - On **Phone B**, open the app (or it may already be open) — check
      **Home**: a new conversation/entry reflecting the incoming emergency
      message should appear, and the message should show status
      **Delivered** once Phone B's app-level ack reaches Phone A.
    - On Phone A, the message's status should move from `Sent` to
      `Delivered` within a second or two of Phone B receiving it — this
      is the real ack round-trip, not a timer or a fake transition.
11. **Test disconnect/reconnect:**
    - On either phone, tap **DISCONNECT** next to the peer.
    - Both phones should show **Disconnected**, and Home's "Nearby Peers"
      count should drop to 0.
    - Send a chat/emergency message now — it should stay **Queued**
      (there's genuinely no route).
    - Tap **CONNECT** again (or **RECONNECT** if it re-shows as
      disconnected) — once reconnected, the previously queued message(s)
      should automatically send and move past Queued.
12. **Test permission denial safety:** on a fresh install (or after
    revoking permissions in system Settings), tap SCAN and deny the
    permission prompt. Confirm the app does **not** crash, does **not**
    show any peer as connected, and shows the "Permission needed" card
    with a working "Grant permissions" retry button.

### Troubleshooting

- **Peers don't find each other:** confirm both phones have Bluetooth and
  Wi-Fi turned on (Nearby Connections needs both radios available even
  though it doesn't need an active Wi-Fi network/internet connection), and
  that neither phone denied a required permission (check the Nearby
  screen's permission card, or Android Settings → Apps → MeshLink →
  Permissions).
- **Stuck on "Connecting…":** back out and retry — this can happen on the
  first pairing attempt with some OEM Bluetooth stacks; it's a
  Nearby Connections/OEM-level quirk, not something this code can paper
  over.
- **Works on emulator only partially:** expected — emulators generally
  cannot do real BLE/Bluetooth Classic device-to-device discovery with
  each other or with a physical phone. Use two real phones.

## What works

Everything listed in Phase 1 and Phase 2's reports, still working
unchanged, plus everything under "What this is right now" above — all
verified by static/manual review as described; **not yet verified on
real hardware by me**, since I have no Android device access here.

## What is still missing / not yet implemented

| Missing | Arrives in |
|---|---|
| Full multi-hop relay for ordinary 1:1 messages (only EMERGENCY broadcast relays for now) | Phase 5 |
| Delivery acknowledgement UI polish / explicit retry button | Phase 6 |
| Connection pairing confirmation UI, device identity, message authentication, encryption | Phase 7 |
| AI emergency classification | Phase 8 |
| A direct "message this connected peer" shortcut from the Nearby screen | Small UX add — can be done anytime you want it |
| An actual two-phone hardware test run (I cannot do this myself) | You, per the steps above |
| A real Gradle build run (still can't run one in this sandbox) | You, in Android Studio |

## Next phase

Not starting until you say "Start Phase 4." Given how much of Phase 4
("Real Phone A ↔ Phone B communication") landed here already as part of
building a genuine transport layer, Phase 4 may mostly be about hardening
and polishing what exists (better reconnect UX, a direct message-peer
shortcut, clearer in-app connection diagnostics) rather than net-new
plumbing — happy to scope that with you when you're ready.
