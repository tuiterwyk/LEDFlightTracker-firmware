# LEDFlightTracker firmware host

OTA update host for the LED flight-tracker family. One repo, every board.

## How a device finds updates

Each board's firmware defaults its `_updateUrl` to its own manifest here:

```
https://raw.githubusercontent.com/tuiterwyk/LEDFlightTracker-firmware/main/version-<board>.json
```

`<board>` matches the firmware's compiled-in `OTA_HARDWARE_ID` (e.g. `interstate75w`).

## Layout

| Thing | Where | Naming |
|---|---|---|
| Manifest | committed to `main` | `version-<board>.json` |
| Binary | GitHub **Release** asset | tag `<board>-vX.Y.Z`, asset `firmware-<board>-vX.Y.Z.bin` |

Manifests are committed (small, raw-served, stable URL); binaries are release
assets (no git bloat; downloaded via the device's WDT-safe redirect path).

## Manifest shape

```json
{
  "hardware": "interstate75w",
  "version": "v1.5.3",
  "firmware_url": ".../releases/download/interstate75w-v1.5.3/firmware-interstate75w-v1.5.3.bin",
  "md5": "<md5 of that .bin>"
}
```

The device **refuses** a manifest whose `hardware` != its own `OTA_HARDWARE_ID`
(fail-closed, before downloading) and verifies the image against `md5`.

## Publishing a new version

1. Build the firmware (bump `VERSION_STRING`).
2. `gh release create <board>-vX.Y.Z firmware-<board>-vX.Y.Z.bin --repo tuiterwyk/LEDFlightTracker-firmware`
3. Edit `version-<board>.json`: new `version`, `firmware_url`, `md5` (`md5 -q <bin>`). Commit to `main`.

Auto-check devices pull it within 12 h (or on next boot).
