# G1-DirectChat Source Archive

`source.zip` at the repo root is the single source of truth. CI unpacks it,
runs `npm ci` + Jest, bundles, and builds debug/release APKs.

SHA-256 (this commit): `319d1e17663eaba19b83c7759e003c2daba249bf5a725fa2d7446a0c93d2e95e`

## What this archive contains (on top of 6cb077c)

Field-report root-cause fixes ("app only works on LAN; no BT / no Wi-Fi Direct;
QR and number add fail; video PiP empty") — 399 Jest tests, 73 suites, all green.

### R1) القريبون (nearby) is no longer LAN-only
- `refreshContacts` merges three sources: LAN presence, raw Wi-Fi Direct
  discoveries (sorted MusabX-first, `p2p:<mac>` provisional ids,
  `unconfirmed` badge for non-MusabX devices), and BLE stranger sightings
  (`bleStrangerSnapshot`, `bt:<mac>` ids). Saved contacts and their known
  addresses are excluded.
- Home rows show the transport: «عبر الشبكة المحلية» / «عبر Wi-Fi Direct» /
  «Wi-Fi Direct · غير مؤكد» / «عبر Bluetooth».

### R2) BLE stranger listener registered at startup
- `setBleStrangerListener` was nested inside the onPeer callback (never fires
  for strangers). It now registers before `startBlePresence`, so rotating
  number-hashes are matched the moment a nearby device advertises.
- `BlePresence` tracks stranger sightings (address/at/rssi), reaps them on
  the same interval as adverts, and clears them on stop.

### R3) Stranger P2P invitations are answered
- `maybeAnswerIncomingInvitation` used to require a trusted saved peer —
  strangers were silently rejected. `findAnyIncomingInvitation` now accepts
  any INVITED device; a stranger gets a provisional `p2p:<mac>` id, a status
  hint («جهاز قريب يتصل بك عبر Wi-Fi Direct…»), and the normal connect-first
  → in-band-identity flow.

### R4) musabNumber is persisted (DB v6)
- `peers` gains `musab_number TEXT` (SQLiteOpenHelper v5 → v6,
  `addColumnIfMissing` migration, guarded column mapping).
- `savePeer(peerId, name, lastMessage, musabNumber)` stores it; the identity
  handler persists `msg.num`, and add-by-code first matches saved contacts by
  normalized number before falling back to the BLE hash hunt.

### R5) Video-call PiP actually renders
- react-native-webrtc requires `zOrder` to stack SurfaceViews: remote
  `zOrder={0}`, local PiP `zOrder={1}`, both keyed by stream URL.
- `captureRemoteStream` rebuilds a NEW MediaStream per track event (merged
  live tracks, dedup by id, ended tracks skipped) — mutating the old stream
  kept the same toURL and RTCView never re-rendered late video tracks.

### R6) In-band P2P-MAC learning (replaces dead DNS-SD route learning)
- `DirectConnection.getOwnP2pDeviceAddress()` (WifiP2pManager.requestDeviceInfo,
  API 29+ guarded) is sent right after identity as `{type:'p2p-address'}` over
  signaling, or over the coordinator for Bluetooth sessions.
- Receivers persist it (`savePeerAddress`) and upsert the P2P peer — so a
  contact first met on LAN can later connect over Wi-Fi Direct with no LAN.
- Guarded by `isProvisionalPeerId` so provisional ids can't be re-mapped.

### R7) Identity hunt spoof-guard everywhere
- Both identity handlers (LAN/P2P signaling and Bluetooth) now verify
  `num` against the advertised pubkey (`verifyAdvertisedNumber`);
  `numOk === false` aborts the hunt with a warning instead of completing.
- Bluetooth identity is sent with full pubKey/musabId/num/nonce enrichment
  (`buildOutgoingIdentityMessage`).

### UI) MessageActionSheet is now WhatsApp-style
- Compact centered floating card (was a bulky 2×2 grid bottom sheet):
  dimmed backdrop, message preview echo, single rounded card with RTL rows
  (رد / نسخ / مشاركة / حذف محلي in red), hairline dividers, 48pt targets.
- Full accessibility contract preserved (all testIDs, hints, disabled state).

## Previous archive (6cb077c)
SHA-256: `728caa93642ba98eae74c83774b543555f651d4875ba11e081b2389f8e39e41b`
- Connect-first P2P identity (provisional `p2p:`/`bt:` ids, in-band rekey)
- BLE rotating number-hash hunt + QR add end-to-end
- File-transfer route selection (P2P-preferred, EWMA hysteresis)
- WhatsApp-literal UI (home, palette, ticks, call log)

## Previous archive (46b3f10)
SHA-256: `06b3bb5e292cc4fb6a562161cd017a38121d07327e92cfd3539584c5a1f55211`
- ChatScreen missing `Avatar`/`TransportChip` imports (RedBox fix)
- `startPresence` startup guard (presence was nested behind LAN support check)
- Per-destination socket interface binding (192.168.49.x exempt → P2P works)
