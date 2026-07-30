# Combined Flock-You unit — WiFi + BLE on Bruce

Design + build reference for merging the two Flock-You detection lines into a
single handheld running on the [Bruce](https://github.com/joshuawowk/esp32bruce)
firmware.

- **`main` branch** (this repo) → passive 2.4 GHz **WiFi** promiscuous detector
  (wildcard-probe + Information-Element fingerprint, gated by OUI).
- **`dev` branch** → **BLE** detector (Flock/Raven/SoundThinking MAC + name +
  manufacturer-ID + Raven service-UUID matching) with an on-device web dashboard.

The combined unit runs both at full quality **at the same time** by giving each
detection method its own radio.

---

## 1. Why two chips

A single ESP32-S3 has **one 2.4 GHz radio**. The WiFi method needs it in
*continuous promiscuous mode* with tight channel timing (250 ms dwell hopping
11 → 6 → 1, chasing Flock's ~125 ms probe bursts). The BLE method needs it for
active scanning. On one chip those two fight, and promiscuous + BLE is the
*worst* coexistence case — continuous/hopping RX collides with every BLE time
slice, so both detectors lose packets.

So the timing-critical, radio-monopolizing job gets a **dedicated chip**:

```
        ┌──────────────────────────────┐         ┌───────────────────────────────┐
        │  ESP32-S3 N16R8  (co-proc)    │  UART   │  T-Display-S3  (Bruce host)   │
        │  headless WiFi sniffer        │ ──────► │  Bruce + BLE + UI + logging   │
        │  • 250ms hop 11→6→1           │  (RX)   │  • BLE Flock/Raven scan       │
        │  • wildcard-probe + IE sig    │         │  • ingest co-proc JSON        │
        │  • this repo's main.cpp       │ ◄────── │  • merge W+B → one table      │
        │  • optional GPS puck          │  (TX,   │  • TFT list + SD log + alert  │
        │  • JSON over UART + USB CDC   │  cmds)  │  • optional soft-AP dashboard │
        └──────────────────────────────┘         └───────────────────────────────┘
             radio = WiFi only                         radio = BLE (+ light AP)
```

The elegant part: because promiscuous sniffing is **offloaded** to the co-proc,
the host's radio only ever does **BLE + (optionally) a light soft-AP** — which
is a *supported* coexistence combination the `dev` firmware already relies on.

---

## 2. The link protocol (co-proc → host)

Newline-delimited JSON, **115200 8N1**. The host parses each line with
ArduinoJson and **ignores any line that does not start with `{`** (so the
co-proc's human-readable `[flockyou] …` log lines are harmless on the wire).

| `event`     | Direction      | Payload (example)                                                       | Host action                                   |
|-------------|----------------|------------------------------------------------------------------------|-----------------------------------------------|
| `detection` | co-proc → host | existing WiFi schema (`detection_method`, `protocol:"wifi_2_4ghz"`, `mac_address`, `oui`, `rssi`, `channel`, `frequency`, `ssid`) | insert/update unified table (WiFi) |
| `status`    | co-proc → host | `{"event":"status","source":"wifi_coproc","channel":6,"mode":"CUSTOM","det":12,"uptime_ms":123456}` | update co-proc liveness row |
| `gps`       | co-proc → host | `{"event":"gps","valid":true,"lat":37.09,"lon":-94.51}`                 | update "latest fix"; stamp **all** detections |
| `cmd`       | host → co-proc | `{"cmd":"mute"}` / `{"cmd":"clear"}`                                    | *(phase 2)* control channel                   |

**GPS model:** the co-proc streams fixes; the **host** holds the latest fix and
applies it uniformly to both WiFi and BLE detections (same approach as
`api/flockyou.py`). The co-proc does *not* stamp detections itself, so
`emitDetectionJSON()` stays untouched.

The WiFi `detection` line is already emitted verbatim by `main.cpp`
(`emitDetectionJSON`) — and `dualPrintf` already mirrors it to **both** USB CDC
and the UART link — so the co-proc side was ~90 % done before any changes.

---

## 3. Hardware & wiring

### GPS lives on the co-processor
The T-Display-S3's GPIOs are almost all consumed by its parallel TFT (5–9,
38–48), so free external pins are scarce. The N16R8 has plenty. Putting the GPS
puck on the co-proc means the **host needs only one external UART** (the co-proc
link), which carries detections *and* GPS.

### Pins

| Signal                     | Co-proc (N16R8)      | Host (T-Display-S3)        | Notes |
|----------------------------|----------------------|----------------------------|-------|
| Detections/status/gps      | `LINK_TX_PIN` (2)    | link UART **RX** (43)      | co-proc → host |
| Commands (phase 2)         | `LINK_RX_PIN` (42)   | link UART **TX** (44)      | host → co-proc |
| GND                        | GND                  | GND                        | common ground required |
| 3V3                        | 3V3                  | 3V3                        | host may power co-proc if regulator has headroom |
| GPS puck (optional)        | `GPS_RX_PIN` (44)    | —                          | NMEA, default 9600 8N1 |

All 3.3 V S3 logic — no level shifting. Host link pins **43/44** are the
T-Display-S3's exposed Grove/`SERIAL` header in the default (non-touch,
non-MMC) build; they don't clash with the main SPI bus (10–13) that carries the
SD card / CC1101 / nRF24, so those remain usable.

### Alerting
The T-Display-S3 has no onboard speaker, so the `dev` branch's crow-caw audio
can't play on the host. Default plan: **visual alert on the TFT + LED**, with an
optional external piezo on a free host GPIO. (LoRa alert uplink via a T3-LoRa32
is a phase-2 option.)

---

## 4. Dashboard / telemetry (recommendation)

There are two distinct "dashboards" from the original project; only one lives on
the ESP:

- **Embedded dashboard** — HTML served *by the ESP* over its own soft-AP
  (`dev` branch's `FY_HTML` at `192.168.4.1`).
- **Flask dashboard** — `api/flockyou.py`, a **PC-side** Python app
  (`localhost:5000`) that reads the device over **USB serial**. The ESP never
  hosts Flask.

**Recommended for the combined unit:**

1. **On-device dashboard on the host's soft-AP**, reusing Bruce's existing web
   infrastructure (`src/core/wifi/webInterface.*`, `wifi_common` soft-AP helper,
   `ESPAsyncWebServer`) to serve the merged WiFi+BLE table + JSON/CSV/KML export.
   Keep it **lightweight** — see the coexistence caveats below.
2. **PC Flask path stays available for free**: because the co-proc's `dualPrintf`
   writes each detection to **both** the UART link *and* USB CDC, you can plug a
   PC into the **co-proc's USB** and run `api/flockyou.py` unchanged — but that
   feed is **WiFi-only** (the co-proc never sees BLE). For a *merged* feed to
   Flask, the host would re-emit its unified table over its own USB CDC (small
   serial-out hook); usually the on-device dashboard makes this unnecessary.

### Coexistence caveats (verified against Espressif's ESP32-S3 matrix)
Running the host soft-AP **while a client is connected** + BLE scanning is
Espressif's **`C1` tier = "supported but performance unstable"** (only an *idle*
beaconing AP is the stable `Y` tier). Expect: reduced BLE scan effective duty
cycle, some advertisements delayed/dropped during WiFi slices, added AP latency.
Fine for a lightweight status/export page; don't expect rock-solid throughput.
Notes:
- Soft-AP + BLE is *far* friendlier than promiscuous + BLE (which is why the
  promiscuous side is offloaded to the co-proc in the first place).
- Use **NimBLE** (Bruce already does) — Bluedroid is heavier and more crash-prone
  under coexistence.
- 8 MB PSRAM does **not** improve coexistence stability; the binding constraint
  is internal DMA-capable SRAM used by the WiFi/BT controllers.

Sources: Espressif [ESP32-S3 RF coexistence guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/coexist.html),
[coexistence FAQ](https://docs.espressif.com/projects/espressif-esp-faq/en/latest/software-framework/coexistence.html).

---

## 5. Deliverable A — co-processor firmware (this repo)

Additive changes to the existing `main.cpp` / `platformio.ini`; the WiFi
detection pipeline is unchanged.

| File            | Change |
|-----------------|--------|
| `platformio.ini`| New `[env:coproc_s3_n16r8]`: `board = esp32-s3-devkitc-1`, 16 MB flash, `qio_opi` PSRAM, `-DBOARD_COPROC_UART -DLINK_TX_PIN=2 -DLINK_RX_PIN=42` (optional `-DUSE_COPROC_GPS`). |
| `main.cpp`      | New `BOARD_COPROC_UART` config block (peer of `BOARD_LILYGO_T_DONGLE_S3`): no display/buzzer, `MIRROR_SERIAL` → the host link on `LINK_TX/RX`. |
| `main.cpp`      | `emitStatusJSON()` + a `status` line on the heartbeat (co-proc only). |
| `main.cpp`      | Optional self-contained GPS section (`USE_COPROC_GPS`): minimal `$xxRMC` NMEA parser on its own UART, emits `gps` lines. |
| `display_dongle.h` | Unchanged — already stubs `dongleDisplay*()` to no-ops when `USE_T_DONGLE_DISPLAY` is undefined, so the headless build needs no display guards. |

Build: `pio run -e coproc_s3_n16r8` · flash: `pio run -e coproc_s3_n16r8 -t upload`

The co-proc firmware can still be flashed to a XIAO or T-Dongle-S3 for
standalone use — the existing envs are untouched.

> **Build isolation (important):** this project pins the **official**
> `espressif32@^6.3.0` platform, while the Bruce host uses the **pioarduino**
> `espressif32` fork. Both install under the same platform *name*, so building
> them in a shared `~/.platformio` makes each clobber the other (broken
> `firmware.bin` / `FRAMEWORK_DIR=None`). This repo therefore sets
> `core_dir = .pio-core` in `[platformio]` so its toolchain lives beside the
> project and never collides with Bruce's global install. Requires **PlatformIO
> Core ≥ 6.x** (the ESP32-S3 platform won't resolve on older releases).

---

## 6. Deliverable B — Bruce "Flock-You" app (esp32bruce repo)

Following Bruce's `MenuItemInterface` + `modules/<category>/` convention.

**New module** `src/modules/flock/`:

| File                | Contents |
|---------------------|----------|
| `flockyou.h/.cpp`   | App lifecycle (`setup`/loop with `check(EscPress)` + `returnToMenu`), unified `FYDetection[]` table, on-screen list, SD logging, alert. |
| `flock_ble.cpp`     | BLE detection ported from `dev` main.cpp, **rewritten for NimBLE 2.x** (Bruce ships NimBLE 2.5 — the scan-callback API differs from the `dev` branch's 1.4). |
| `flock_link.cpp`    | `HardwareSerial` reader: parse co-proc lines with ArduinoJson → feed WiFi detections + GPS into the shared table. |
| `flock_patterns.h`  | OUI / name / manufacturer-ID / Raven-UUID tables (from `dev`). |

**New menu item** `src/core/menu_items/FlockMenu.h/.cpp` — `class FlockMenu :
public MenuItemInterface` (options: *Start (WiFi+BLE)*, *BLE only*, *WiFi only*,
*View session*, *Patterns*, *Export to SD*, *Clear*; simple `drawIcon`;
`hasTheme()→false`).

**Registration** (2 edits, mirroring every other menu):
- `src/core/main_menu.h` — `#include "menu_items/FlockMenu.h"` + member `FlockMenu flockMenu;`
- `src/core/main_menu.cpp` — add `&flockMenu,` to the `_menuItems` initializer.

**Reused Bruce infra:** `tft`/`padprintln`/`drawMainBorderWithTitle` (UI),
`sd_functions` (SD/LittleFS logging), `bruceConfig` (settings), NimBLE 2.5 scan,
`webInterface`/`wifi_common` soft-AP (optional dashboard). The `dev` branch's
AsyncWebServer dashboard is replaced by the TFT + SD (optionally revived via
Bruce's web interface).

---

## 7. Unified detection model

Both branches already emit the **same** JSON detection shape, which is what
makes the merge trivial — the `protocol` + `detection_method` pair is the merge
key:

```jsonc
// WiFi (co-proc):  protocol = "wifi_2_4ghz",  method e.g. "wifi_wildcard_probe_ie_sig"
// BLE  (host):     protocol = "bluetooth_le", method e.g. "mac_prefix" / "raven_uuid"
```

The host keeps one table keyed by MAC, tagged with `protocol`, so a WiFi hit and
a BLE hit render in the same list with a `W`/`B` badge and share GPS stamping.

---

## 8. Build order

1. **Co-processor firmware** — env + `BOARD_COPROC_UART` block; verify it emits
   `detection`/`status` lines on the link UART. *(Deliverable A — done first.)*
2. **Bruce link + unified table + `FlockMenu` skeleton** — prove the two-chip
   pipe end-to-end on the TFT.
3. **Bruce BLE detection** — port `dev`'s detection core to NimBLE 2.x; merge.
4. **GPS + SD logging + alert** — GPS from the co-proc stream, CSV/KML to SD,
   visual/piezo alert.
5. **Optional dashboard** — lightweight soft-AP page on the host (§4).
6. **Phase 2** — LoRa alert uplink (T3-LoRa32); host → co-proc command channel.

---

## 9. Open items

- **Audio:** host has no speaker — visual + optional external piezo (confirm).
- **"Moire":** unidentified board — provide a link to factor it in.
- **Upstreaming:** kept as a self-contained `flock/` module so it rebases cleanly
  onto BruceDevices/main.
- **Pin confirmation:** host link pins assume the default (non-touch, non-MMC)
  T-Display-S3 build; parameterized either way.
