# G1-DirectChat Source Archive

`source.zip` at the repo root is the single source of truth. CI unpacks it,
runs `npm ci` + Jest, bundles, and builds debug/release APKs.

SHA-256 (this commit): `354c6b60800f808e721e4b73c4a3d7ac3f59c01aa0467d435f7b58ece2e8c8d1`

## Field-failure instrumentation round — جولة «لا فشل صامت» (422 tests green, 76 suites)

Direct response to the field report ("يعمل على الشبكة المحلية فقط؛ الضغط لا
يفتح المحادثة؛ تلفزيون يظهر كجهة اتصال؛ QR/الرقم صامتان"):

- **Diagnostics subsystem**: `src/utils/diagLog.js` ring buffer (500 entries)
  + `src/components/DiagnosticsScreen.js` (live view, share/copy) reachable
  from the home ⋮ menu → «سجل التشخيص». Every BT/P2P/LAN/link event and every
  connect/send failure is logged with timestamps. Field failures stop being
  invisible.
- **Visible status**: `statusText` now renders as a banner on HomeScreen;
  connect failures from a contact tap raise an Alert instead of vanishing
  into a `.catch(() => {})`.
- **Nearby honesty (Bada/LocalSend lesson)**: native P2P layer now ships
  `primaryDeviceType`; unconfirmed (non-DNS-SD) devices that are not phones
  (category `10-`) — TVs, displays, printers — are hidden from «القريبون».
  Only confirmed MusabX devices and phone-category unknowns are actionable.
- **Bluetooth in the single window**: classic BT scan + device list now live
  in the home add-sheet (was stranded on IdleScreen); the "اتصال عبر IP" row
  is restored.
- **QR truth**: camera permission is requested at scan time (not only at
  boot), and scan/add outcomes are logged and surfaced.
- **Engine repair**: BT identity handler again registers the RFCOMM link into
  the LinkManager (lost in a prior snapshot) so `isReachable`/MBB see BT.

### Make-Before-Break transport engine — إصلاح جذري لنقل الطبقات (previous commit)

Built strictly on the reference study in `docs/REFERENCE_IMPLEMENTATIONS_STUDY.md`
(Briar / KDE Connect / Meshenger / Bada-QuickShare / LocalSend).

### M1) Bluetooth is a first-class citizen (الجذر: RFCOMM الآمن)
- Root cause of "البلوتوث غير موجود أبداً": the native module used SECURE RFCOMM
  (`createRfcommSocketToServiceRecord` / `listenUsingRfcommWithServiceRecord`)
  plus `requireBonded()` that threw on unpaired devices — strangers could never
  connect without Android pairing.
- Fix (Briar `AndroidBluetoothPlugin` model): INSECURE RFCOMM on both the
  listener and the connector; `requireBonded` deleted entirely. Security moves
  to the app layer where it already exists: Hello handshake + Ed25519 signed
  identity + TOFU pinning.
- File: `android/app/src/main/java/com/m200/bluetooth/BluetoothConnectionModule.kt`.

### M2) Multi-link engine — نموذج KDE Connect
- New `src/network/LinkManager.js`: per-peer simultaneous links keyed by
  transport, priority LAN(30) > P2P(20) > BLUETOOTH(10).
- `send()` = priority-ordered `links.any` semantics (KDE `sendPacketBlocking`):
  a send automatically lands on the best live link and falls through on error.
- `isReachable(peerId)` = at least one live link — the UI can never show
  "متصل" on a dead route again.
- Nonce tie-break (Briar `compareConnections`) for same-transport duplicate
  links: higher nonce wins, tie keeps the incumbent, loser is closed with
  reason `superseded`.
- `rekeyPeer` merges provisional identities into the stable peer id.
- 9 behavioral tests in `__tests__/linkManager.test.js`.

### M3) ترقية صامتة وسقوط تلقائي (Make-Before-Break)
- `App.js` wiring: `registerSessionLink` / `unregisterSessionLink` /
  `scheduleLowerLinkRetirement` / `sendOverLinks`. New links register on both
  identity paths (signaling + Bluetooth); after a higher-priority link is
  live, lower links get a `link-retire` frame + 8s drain, then close only if
  the better link is still alive.
- `sendMsg` routes through the link engine — one send path, any transport.
- `LinkSupervisor` extended: BT→P2P upgrade rule, LAN-loss→BT fallback,
  P2P-sick→BT fallback. Initiator-only rule retained (no simultaneous-switch
  race).
- `beginWifiNegotiation` gained `{ silent }` — background P2P upgrades no
  longer wipe the chat or flash status text.
- Chat rows get `live` from `linkManager.isReachable` — honest presence only.

### M4) نافذة واحدة + واجهة صادقة
- `src/components/ContactsScreen.js` (the old "DirectChat" window) DELETED;
  `showClassic` state and its Modal removed from `App.js`. One unified window.
- HomeScreen nearby rows: honest labels («عبر Bluetooth» / «عبر Wi-Fi Direct»
  / «Wi-Fi Direct · غير مؤكد» / «عبر الشبكة المحلية» / «عبر I2P (مجهول)»),
  P2P 6s display grace with «شوهد قبل لحظات — جارٍ التحقق».
- Chat row preview: `متصل الآن` only when a live link exists.
- Add-sheet gains "اتصال عبر IP" (direct LAN connect) inside HomeScreen.

### Test retargeting after the single-window merge
- `wifiDirectPeerVisibilityGrace.test.js`, `wifiDirectDiscoveryLifecycle.test.js`,
  `appUnifiedRuntimeWiring.test.js` now assert App.js / HomeScreen.js /
  IdleScreen.js instead of the deleted ContactsScreen.
- `appP2pCoordinatorOwnership.test.js` signature assertion updated for the new
  `{ silent = false }` negotiation option.

### I2P) طبقة I2P المستقبلية — بنية جاهزة بلا راوتر مدمج (previous commit)
- `TRANSPORTS.I2P` + `peerRegistry.upsertI2pPeer`: the I2P Destination is a
  STABLE identity — `isProvisionalPeerId` never matches `i2p:` (guarded).
- Identity messages carry an optional `i2p` field; DB v7 `peers.i2p_destination`.
- Contract + do-NOTs documented in `docs/I2P_TRANSPORT_READINESS.md`.

### R1–R7) Field-report root causes (earlier commit on this branch)

Field-report root-cause fixes ("app only works on LAN; no BT / no Wi-Fi Direct;
QR and number add fail; video PiP empty").

## Build contract (unchanged)

- CI unpacks the first `*.zip` at repo root → `G1-DirectChat/` → `npm ci` →
  Jest → bundle → debug/release APKs.
- No new native npm dependencies. No `bindProcessToNetwork`.
## Fallback-engine wiring round — جولة لحام محرك التراجع (425 tests green, 77 suites)

SHA-1 of `source.zip`: `819bdf5f28ee33752f71f3680626a68e73ef4305`

Field log root cause: `connectToContact` failed on every tap with
«Transport fallback engine is not configured». The coordinator singleton was
created without an engine and the shared `fallbackEngine` module was imported
only by an unused diagnostics component, so in the production bundle its body
never executed and the engine never attached. Fixes:
- App.js imports the shared engine and attaches it explicitly.
- `ConnectionCoordinator.connectPeer` lazily self-heals the shared engine
  (lazy require avoids the module cycle).
- Unexpected link loss now resets the auto-connect exponential backoff so
  reconnection is immediate (~1.5 s initiator) instead of 30 s+.
- Standby BT link send closure is honest: it refuses to write when the
  coordinator session is no longer on Bluetooth, so the link manager falls
  through to a live link instead of writing into a stale socket.
- Diagnostics: auto-connect plans/outcomes and supervisor migrations are
  logged to the in-app diagnostics screen.
- New regression suite `fallbackEngineWiring.test.js` (3 tests).
## Channel-liveness round — جولة «لا قناة زومبي» (431 tests green, 78 suites)

SHA-1 of `source.zip`: `13c6de0faa64ac1536b94fec68ed9273fd77e016`

Field-log evidence: chat header showed «متصل» while sends failed with
«لا قناة حية», and Wi-Fi Direct teardown could hang forever on
«جاري إنهاء اتصال Wi-Fi Direct وتنظيف المجموعة…».
Root fixes:
- LinkManager.send now retires a link whose write fails (emits link-down
  with reason `send-failed`) — the UI cannot show a fake «متصل» anymore.
- Chat header derives liveness from the link engine + signaling health +
  coordinator BT session (`peerLive`), showing «انقطع — يعيد الاتصال…» in
  amber when every link is dead.
- Total channel failure during a session forces the terminal-recovery path
  immediately (`handleTerminalSignalingDisconnect({ force: true })`) instead
  of waiting up to 18 s for the heartbeat.
- Hard 20 s watchdog on both disconnect paths + JS-level race around native
  `cleanupConnection` — «جاري الإنهاء» can never be permanent again.
- Stage-level diagnostics inside the Wi-Fi Direct negotiation (group formed /
  socket accept / socket connect / identity) so the next field log pinpoints
  any stall.
- New regression suite `channelLiveness.test.js` (6 tests).
