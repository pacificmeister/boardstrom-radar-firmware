# Boardstrom tank sensor: BLE advertisement format v2

This is the broadcast sent by the released tank sensor firmware, **v1.0.0-beta.4
through v1.0.0-beta.6**. It replaces [ble-advert-v1.md](ble-advert-v1.md), which
describes the older beta.1 to beta.3 layout and the Boost wake signal that was
removed in beta.5.

It is a keyless plaintext manufacturer-data advert in the Ruuvi/Mopeka style:
the sensor broadcasts, any receiver scans passively, no pairing, no cloud, no
account. The sensor reports physics only (distance, battery, temperature). Fill
% is computed by the receiver after tank calibration.

## AD structure

Legacy, non-connectable advertisement with one manufacturer-specific data
structure (AD type 0xFF):

| Field | Value |
|---|---|
| Company ID | **0xFFFF** (the Bluetooth SIG "internal use / testing" id) during the beta. A move to a proper company id is planned; the magic byte, version byte and checksum exist so receivers survive that transition. |
| Payload | 14 bytes, layout below |

Total manufacturer data on the wire: 2 (company id) + 14 = 16 bytes.

The sensor is a pure broadcaster: it never scans and never accepts a
connection.

## Payload layout (14 bytes, multi-byte fields little-endian)

| Offset | Size | Field | Encoding |
|---|---|---|---|
| 0 | 1 | Magic | `0x42` ('B') |
| 1 | 1 | Version | `0x02` |
| 2 | 2 | Distance | uint16, millimetres from the sensor to the liquid surface. `0xFFFF` = no target / invalid measurement |
| 4 | 2 | Battery | uint16, millivolts. `0xFFFF` = n/a |
| 6 | 1 | Temperature | int8, °C (the radar chip's temperature). `0x7F` = n/a |
| 7 | 1 | Status | see below |
| 8 | 2 | Counter | uint16, steps once per measurement, wraps. Use it to drop repeated adverts of the same measurement and to spot missed ones |
| 10 | 2 | Sensor ID | uint16, stable per-device id. Use this, not the BLE address, to tell sensors apart |
| 12 | 1 | Firmware | uint8, firmware release number: 4 = beta.4, 5 = beta.5, 6 = beta.6 |
| 13 | 1 | Checksum | XOR of payload bytes 0..12 |

### Status byte

| Bits | Meaning |
|---|---|
| 0-2 | Signal quality 0-7 (0 = unusable, 7 = excellent) |
| 3 | Unstable: slosh detected or the reading has not settled yet |
| 4 | **Firmware 6 and later:** far-range mode. The distance came from the 0.3 to 3 m sweep that the sensor falls back to when the 3 cm to 1 m near sweep finds no surface. |
| 5-7 | Reserved. Ignore them; do not assume they are zero |

**Firmware 4 only:** bits 4 and 5 meant "Boost active" and "Boost disabled".
Boost was removed in firmware 5 and those bits were zero there. Read bit 4 as
far-range only when the firmware byte is 6 or higher.

## Validating a packet

0xFFFF is shared by definition, so other test devices use it too. Check the
whole chain before trusting a packet:

1. Manufacturer data is 16 bytes and starts with company id `0xFFFF` (`ff ff`).
2. Magic byte is `0x42`.
3. Version byte is `0x02`. Reject any other version you don't know; a new
   layout always gets a new version byte.
4. The XOR of payload bytes 0..12 equals byte 13.

### Example

Manufacturer data: `ff ff 42 02 e2 00 c2 0b 15 06 23 01 59 d2 06 d7`

- distance `e2 00` = 226 mm
- battery `c2 0b` = 3010 mV
- temperature `15` = 21 °C
- status `06` = quality 6, settled, near range
- counter `23 01` = 0x0123
- sensor id `59 d2` = 0xD259
- firmware `06` = beta.6
- checksum `d7` = XOR of the 13 bytes before it

## Older sensors (firmware beta.1 to beta.3)

Sensors still on beta.1 to beta.3 send version `0x01`: a 13-byte payload with
the same bytes 0..11, no firmware byte, and the checksum (XOR of bytes 0..11)
at offset 12. A receiver that wants to support them dispatches on the version
byte first, then applies the matching length and checksum position. Re-flashing
to the latest release moves a sensor to v2.

## Measurement rhythm

Every 10 s for the first 10 minutes after the battery goes in, then every 60 s.
Re-seat the battery for another 10-minute fast window. The advert repeats
between measurements; the counter only steps when there is a new measurement.

## Notes for receivers

- Fill % is not in the advert by design. Store the distance at "empty" and at
  "full" for each tank (or a multi-point curve) and convert on the receiver.
- A full tank simply reads a small distance.
- Home Assistant: an ESP32 running ESPHome as a Bluetooth proxy carries these
  adverts into HA, and a small decoder turns them into sensors. If you build
  one, tell us at [r/boardstrom](https://www.reddit.com/r/boardstrom/) and
  we'll link it.
- Victron GX (Cerbo, Venus OS): the
  [BoardstromTank](https://github.com/pacificmeister/BoardstromTank) driver
  already reads this format.
