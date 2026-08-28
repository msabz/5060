# G1-DirectChat Source Archive

`source.zip` at the repo root is the single source of truth. CI unpacks it,
runs `npm ci` + Jest, bundles, and builds debug/release APKs.

SHA-256 (this commit): `6106276664f35f43b57bed3562b58ebcba24672f4522a874dc830c46a338`

## What this archive contains (on top of the I2P readiness round)

### Make-Before-Break transport engine — إصلاح جذري لنقل الطبقات (414 tests green, 75 suites)

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
