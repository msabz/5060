# G1-DirectChat Source Archive

`source.zip` at the repo root is the single source of truth. CI unpacks it,
runs `npm ci` + Jest, bundles, and builds debug/release APKs.

SHA-256 (this commit): `728caa93642ba98eae74c83774b543555f651d4875ba11e081b2389f8e39e41b`

## What this archive contains (on top of 46b3f10 field fixes)

### A) Connect-first P2P identity (fixes "Wi-Fi Direct never works without LAN")
- Provisional peer IDs (`p2p:<mac>`, `bt:<mac>`) let a Wi-Fi Direct / Bluetooth
  session start immediately; the real stable deviceId replaces it in-band
  (`PeerRegistry.rekeyPeer`, `ConnectionCoordinator.rekeyCurrentPeer` with
  hijack protection — only provisional IDs can be rekeyed).
- `buildCoordinatorP2pPeer` no longer throws when DNS-SD never delivered a
  stable identity.

### B) Add-by-number / QR from ANY transport
- BLE rotating number-hash hunt (`musabx-pub-v1:<digits>:<15min bucket>`,
  8-byte pubHash appended to the BLE advert) — privacy-preserving stranger
  discovery over Bluetooth.
- In-band number verification: identity messages now carry `num`, recomputed
  from the peer's pubkey (`IdentityKeysModule.musabIdFor`) — spoof attempts
  raise `identity-mismatch`.
- QR scan wired end-to-end: header camera icon / add-sheet → native ZXing
  scanner (`QrScannerModule`) → `addContactByCode` (accepts raw number,
  MusabX id, or a scanned `musabx://add?n=...&id=...` link).

### D) File-transfer route selection (fixes "LAN slower than Wi-Fi Direct")
- Presence beacon tracks per-interface hosts (`lanHost` / `p2pHost`).
- Sender prefers the peer's Wi-Fi Direct address when available; EWMA
  throughput memory (alpha 0.5) with 1.3x hysteresis flips back to LAN only
  when it is measurably faster.

### E) WhatsApp-literal UI
- Single home screen with bottom nav (الدردشات / القريبون / المكالمات),
  WhatsApp dark palette (#0B141A/#1F2C34/#005C4B/#00A884/#25D366),
  header with QR-scan/search/overflow, FAB, bottom add-sheet
  (مسح رمز QR / إضافة برقم / رمزي ورقمي), unread badges, ✓/✓✓ ticks,
  WhatsApp call-log rows.
- ChatScreen + global palettes re-skinned to real WhatsApp colors.

## Previous archive (46b3f10)
SHA-256: `06b3bb5e292cc4fb6a562161cd017a38121d07327e92cfd3539584c5a1f55211`
- ChatScreen missing `Avatar`/`TransportChip` imports (RedBox fix)
- `startPresence` startup guard (presence was nested behind LAN support check)
- Per-destination socket interface binding (192.168.49.x exempt → P2P works)
