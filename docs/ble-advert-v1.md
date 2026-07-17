# Boardstrom tank sensor — BLE advertisement format v1

Keyless plaintext manufacturer-data advert in the Ruuvi/Mopeka style: the
sensor broadcasts, the app passively scans, no pairing, no cloud, no account.
Fill % is computed app-side after tank calibration; the sensor only reports
physics (distance, battery, temperature).

## AD structure

Legacy advertisement, manufacturer specific data (AD type 0xFF):

| Field | Value |
|---|---|
| Company ID | **0xFFFF** (the Bluetooth SIG "internal use / testing" id) during the beta. A migration to a proper company id is planned; the magic byte, version byte, and checksum exist so receivers survive that transition. |
| Payload | 13 bytes, layout below |

Because 0xFFFF is shared-by-definition, the parser must validate the magic
byte, version byte, and checksum before trusting a packet (same defensive
posture as the app's Mopeka parser, which requires service UUID + known
model byte).

## Payload layout (13 bytes, little-endian multi-byte fields)

| Offset | Size | Field | Encoding |
|---|---|---|---|
| 0 | 1 | Magic | `0x42` ('B') |
| 1 | 1 | Version | `0x01` |
| 2 | 2 | Distance | uint16 LE, millimetres from sensor reference plane to liquid surface. `0xFFFF` = no target / invalid measurement |
| 4 | 2 | Battery | uint16 LE, millivolts. `0xFFFF` = n/a |
| 6 | 1 | Temperature | int8, °C. `0x7F` = n/a |
| 7 | 1 | Status | bit 0-2: signal quality 0-7 (0 = unusable, 7 = excellent, mapped from detector strength); bit 3: unstable flag (slosh detected / median not settled); **bit 4: fast/boost mode active** (app-triggered wake-on-demand, see below); bits 5-7 reserved, zero |
| 8 | 2 | Sync counter | uint16 LE, increments once per measurement cycle, wraps. Lets the app dedupe repeated adverts of the same measurement and detect missed cycles |
| 10 | 2 | Sensor ID | uint16 LE, stable per-device id (derived from MAC/serial) |
| 12 | 1 | Checksum | XOR of payload bytes 0..11 |

Total manufacturer data on the wire: 2 (company id) + 13 = 15 bytes.
Advert budget: 3 (flags) + 17 (mfr data structure) = 20 of 31 bytes — room
left for a shortened local name if we want one later.

## Design notes

- **Distance, not fill %.** The sensor doesn't know the tank. Calibration
  (empty/full distances) lives in the app, mirroring how the app already
  handles Mopeka tank geometry.
- **Sensor ID in payload** rather than relying on the BLE MAC: nRF52
  privacy features and iOS MAC randomization on the scanner side make the
  MAC unreliable as identity. Same reasoning as Mopeka putting MAC bytes in
  the payload. 16 bits is enough at "a handful of sensors per RV" scale;
  collisions are further disambiguated by the (random) advertising address
  during a session.
- **Checksum**: BLE link layer already CRCs packets; the XOR byte guards
  against 0xFFFF company-id collisions with other test devices, not against
  bit errors.
- **v1 freeze:** distance/battery/temp/status/counter is deliberately
  minimal. Anything else (e.g. RSSI-independent quality metrics, config
  echo) waits for v2; the version byte exists so v2 can happen.

## Wake signal (phone → sensor, "boost mode")

The reverse direction: the **app** advertises a wake signal and the sensor,
which opens a short passive scan window every 2 min, switches to fast (5 s)
measurements for 10 min on a match. Keyless and connectionless — no GATT, no
pairing.

**Mechanism: a 128-bit Service UUID.** iOS only reliably advertises service
UUIDs in the foreground (manufacturer data is stripped), and Android advertises
them trivially, so one mechanism covers both. The target sensor's id is embedded
in the UUID so exactly one puck reacts.

**Wake UUID** (fixed prefix + the target sensor's 16-bit id as the last 4 hex
digits, i.e. the id printed big-endian `%04x`):

```
21a1b057-adb0-0065-6b61-772d5342XXXX
```

- `XXXX` = the sensor's 16-bit Sensor ID (payload offset 10) as 4 lowercase hex
  digits. Example: sensor id `0xD259` → `21a1b057-adb0-0065-6b61-772d5342d259`.
- Nothing else is needed in the advert — just this one service UUID. The sensor
  matches the fixed 14-byte prefix and its own id; other pucks ignore it.
- On the wire a 128-bit UUID is little-endian, i.e. the reverse of the string.
  Hand the UUID string to your platform's advertiser API and the OS
  byte-orders it (nRF Connect's advertiser reproduces the wake exactly).

**App confirms the wake landed** by watching **status bit 4** (fast mode) go high
on that sensor's advert v1, and by the observed ~5 s cadence. The sensor holds
fast mode only while it KEEPS receiving the wake (in boost it re-checks every
10 s and exits after two consecutive misses, ~20 s), with a hard 10 min cap per
session — so the app advertises continuously for the whole session, at low
latency (~100 ms interval; a ~1 s advert interval gets missed by the 110 ms
listen window). Give up waking after ~2.5 min (covers the worst-case 2 min
steady-state listen latency). This UUID is provisional until the payload company
id migrates off `0xFFFF` (they migrate together); treat both as a coordinated
change.

## Notes for implementers

- Validate the full chain before trusting a packet: length (15 bytes of
  manufacturer data), company id 0xFFFF, magic 0x42, version 0x01, XOR
  checksum. 0xFFFF is shared by definition; skipping validation WILL
  misparse other devices.
- Fill % is not in the advert by design: the sensor reports the distance to
  the liquid surface, and the receiver applies the tank's empty/full
  calibration (or a multi-point curve).
- Home Assistant: an ESP32 running ESPHome as a Bluetooth proxy carries
  these adverts into HA; a small decoder turns them into sensors. If you
  build one, tell us at r/boardstrom and we'll link it.
