# Porting per-hardware OTA auto-update to another board

Reference for adding this firmware-host's OTA auto-update to another LED
flight-tracker hardware repo. Everything that changes per board is marked
`<board>`.

## Architecture (shared across all boards)

- **One firmware-host repo**: `tuiterwyk/LEDFlightTracker-firmware`. All boards publish here.
- **Manifest per board, by filename** — committed to `main`, raw-served:
  `version-<board>.json` (e.g. `version-presto.json`).
- **Binary per board+version** — a GitHub **Release asset** (keeps binaries out of git):
  tag `<board>-vX.Y.Z`, asset `firmware-<board>-vX.Y.Z.bin`.
- Each firmware bakes in its **own** `OTA_HARDWARE_ID` and default `_updateUrl`,
  and **refuses any manifest that isn't for its board** (fail-closed).

## Per-board values (the only things that differ)

| Token | Example (i75) | Example (Presto) |
|---|---|---|
| `OTA_HARDWARE_ID` | `interstate75w` | `presto` |
| `_updateUrl` default | `.../main/version-interstate75w.json` | `.../main/version-presto.json` |
| Release tag / asset | `interstate75w-v1.5.3` / `firmware-interstate75w-v1.5.3.bin` | `presto-v...` / `firmware-presto-v....bin` |

`OTA_HARDWARE_ID`, the manifest filename suffix, and the manifest's `"hardware"`
field **must all be the same string.**

## `version-<board>.json` schema

```json
{
  "hardware": "<board>",
  "version": "v1.5.3",
  "firmware_url": "https://github.com/tuiterwyk/LEDFlightTracker-firmware/releases/download/<board>-v1.5.3/firmware-<board>-v1.5.3.bin",
  "md5": "<md5 of that .bin>"
}
```

## Firmware code changes

**1. Board identity (a constants header):**
```cpp
// version.json must carry a matching "hardware" field or the image is
// refused (fail-closed). Change this one line per board.
constexpr const char *OTA_HARDWARE_ID = "<board>";
```

**2. Default update URL (config/globals):**
```cpp
String _updateUrl =
    "https://raw.githubusercontent.com/tuiterwyk/LEDFlightTracker-firmware/main/version-<board>.json";
bool   _autoUpdate = false;   // keep auto-apply OPT-IN
```

**3. Fail-closed hardware guard** — in the version-check function, after parsing
the manifest and **before** any download:
```cpp
const char *hardware = doc["hardware"] | "";
// ... after the missing-fields check ...
if (strcmp(hardware, OTA_HARDWARE_ID) != 0) {
    dlog("hw mismatch '%s'!='%s'", hardware, OTA_HARDWARE_ID);
    return String("hardware mismatch: ") + hardware;   // absent field = reject
}
```

**4. WDT-safe redirect download (REQUIRED on RP2350/arduino-pico with a
hardware watchdog).** GitHub release URLs `302` to S3 — two TLS handshakes in
one blocking `GET()`. Follow redirects *manually*, feeding the watchdog
between hops, and use this for **both** the manifest GET and the firmware GET:
```cpp
static int otaGetFollowingRedirects(HTTPClient &http, BearSSL::WiFiClientSecure &sec,
                                    WiFiClient &plain, String url, int timeoutMs) {
  constexpr int MAX_REDIRECTS = 5;
  for (int hop = 0; hop <= MAX_REDIRECTS; hop++) {
    bool isHttps = url.startsWith("https://");
    if (!(isHttps ? http.begin(sec, url) : http.begin(plain, url))) return -1;
    http.setFollowRedirects(HTTPC_DISABLE_FOLLOW_REDIRECTS);
    http.setTimeout(timeoutMs);
    diag::watchdogFeed();                 // fresh window per handshake
    int code = http.GET();
    diag::watchdogFeed();
    if (code >= 300 && code < 400) {
      String loc = http.getLocation();    // populated on 3xx even with follow off
      http.end();
      if (loc.length() == 0) return code;
      url = loc;
      continue;
    }
    return code;                          // 2xx still connected, or hard error
  }
  return -1;
}
```

**5. Cap the BearSSL handshake under the watchdog budget** — once at boot,
before any HTTPS:
```cpp
BearSSL::WiFiClientSecure::setTLSConnectTimeout(6000);  // < your WDT timeout (8s)
```
`http.setTimeout()` does **not** bound the handshake — BearSSL uses a separate
15 s default. Without this cap a single stalled connect rides straight through
an 8 s watchdog.

**6. Feed the watchdog around every other blocking network call** (`GET()`,
`getString()`, NTP polls) in the data-fetch paths — feed immediately before and
after each.

## Publishing a build

```bash
# 1. bump VERSION_STRING, build
pio run
cp .pio/build/<env>/firmware.bin firmware-<board>-vX.Y.Z.bin

# 2. binary -> release asset
gh release create <board>-vX.Y.Z firmware-<board>-vX.Y.Z.bin \
  --repo tuiterwyk/LEDFlightTracker-firmware --title "<board> vX.Y.Z"

# 3. update version-<board>.json (version, firmware_url, md5) and commit to main
md5 -q firmware-<board>-vX.Y.Z.bin
```

## Safety invariants (do not violate)

- **`version` in the manifest must equal the binary's real `VERSION_STRING`.**
  Advertise higher and the device re-downloads forever (it never "reaches" the
  version) — an update loop.
- **`md5` must match the exact uploaded binary.** The device verifies and aborts
  on mismatch.
- **Auto-apply stays opt-in** (`_autoUpdate=false` default); the manual web
  `/update` upload path needs none of the redirect/handshake work (no outbound TLS).
- The download follows the GitHub->S3 redirect; `setInsecure()` is intentional
  (LAN-trust / no cert pinning).

## Per-board checklist

- [ ] `OTA_HARDWARE_ID` set to `<board>`
- [ ] `_updateUrl` default -> `version-<board>.json`
- [ ] hardware guard added before download
- [ ] redirect helper used for manifest + firmware GETs (if watchdog present)
- [ ] `setTLSConnectTimeout()` at boot
- [ ] `version-<board>.json` committed + `<board>-vX.Y.Z` release published, md5 verified
- [ ] confirm: fetch manifest, download binary, md5 matches, device reports new version
