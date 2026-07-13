# Boardstrom Tank Sensor — firmware releases

Free firmware for the DIY **Boardstrom Tank Sensor**: a wireless radar tank
level sensor you build from off-the-shelf Acconeer parts (XM126 + XA122 +
CA122 + a CR2477 cell) — no holes drilled in the tank, no wires, no soldering.
It reads the liquid level through the tank wall with 60 GHz radar and
broadcasts over Bluetooth to the free
[Boardstrom app](https://boardstrom.com) (iOS / Android).

**Build guide, parts list, and flashing instructions:**
https://boardstrom.com/tank-sensor

**Download the firmware:** grab `boardstrom_tank_beacon.signed.bin` from the
[latest release](../../releases/latest). Verify the SHA-256 checksum listed in
the release notes if you like.

## Flashing (short version)

Full instructions with screenshots are in the build guide. In brief:

```
python -m pip install --upgrade acconeer-exptool[app]
python -m acconeer.exptool.flash flash XM126 -p COM6 --image boardstrom_tank_beacon.signed.bin
```

(Replace `COM6` with your XB122's serial port; on macOS it's
`/dev/tty.usbserial-…`.)

Re-flashing an updated release later works exactly the same way — the
module's bootloader stays in place, so your sensor is never locked to a
firmware version.

## What's in this repository

Release binaries and release notes only. The firmware source is not published
here.

## Support & community

Questions, builds, and problem reports: [r/boardstrom](https://www.reddit.com/r/boardstrom/)
or merten@boardstrom.com.

## License & disclaimer

The firmware binaries are free to download and use for personal,
non-commercial builds of the Boardstrom Tank Sensor. Provided **as-is,
without warranty of any kind**; you assemble, flash, and operate your build
at your own risk. Redistribution of modified binaries is not permitted.

Boardstrom is an independent project, not affiliated with, authorized, or
endorsed by Acconeer AB or Victron Energy B.V. Product names are trademarks
of their respective owners and are used only to identify compatible parts.

© 2026 Merten Stroetzel
