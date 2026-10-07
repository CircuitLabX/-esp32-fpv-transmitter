# FPV  Drone Transmitter (ESP32 + ESP-NOW + EspFC)

A fully custom, hand-built FPV radio transmitter for a 3" micro brushless quadcopter.
No off-the-shelf TX, no PPM/SBUS/CRSF, no third-party RC library — this talks directly
to [EspFC](https://github.com/rtlopez/esp-fc) (a Betaflight-compatible flight controller
firmware for ESP32) over **raw ESP-NOW**, using a wire protocol reverse-engineered from
EspFC's own `espnow-rclink` library.

Built to compile on **mobile Arduino IDE (ArduinoDroid)** — no PlatformIO, no computer
required for day-to-day firmware changes.

![status](https://img.shields.io/badge/status-flying-success)
![platform](https://img.shields.io/badge/platform-ESP32-blue)
![firmware](https://img.shields.io/badge/compatible%20FC-EspFC%20(Betaflight)-orange)

---

## ✈️ Features

| Feature | Description |
|---|---|
| **Raw ESP-NOW RC link** | No external RC library — talks directly to EspFC's built-in ESP-NOW receiver (`SPI Rx` mode). Auto-pairing: scans WiFi channels 1–13 for the drone's `PAIR_REQ` broadcast. |
| **8-channel output** | Roll, Pitch, Throttle, Yaw (AETR order, matches EspFC/Betaflight default) + 4 AUX channels (ARM, Acro/Horizon, Find-My-Drone beeper, spare). |
| **Full-color touchless UI** | 2.8" 320×240 ILI9341 display via **TFT_eSPI**, flicker-free (only the pixels that changed are redrawn, never a full-screen clear during flight). |
| **Scrolling on-screen menu** | 10 menu items, button-navigated, with a scrollbar for overflow. |
| **Stick calibration wizard** | Guided calibration for Throttle (center + one-direction push — auto-detects whichever physical direction has more mechanical travel) and Roll/Pitch/Yaw (center + full extremes). Rejects a too-small captured range and asks you to redo it. |
| **Forward-only throttle mapping** | Spring-centered throttle stick: center (or behind it) = 0% (motor off, fail-safe default), only the calibrated "push" direction increases throttle. |
| **Throttle power limit** | Software-capped maximum throttle output (20–100%, adjustable) — lets 100% stick deflection correspond to less than 100% real power, for over-powered/high-voltage builds. |
| **Expo & Rate per axis** | Independent expo (0.0–1.0) and rate (0.5×–2.0×) for Roll, Pitch, Yaw, applied on the TX side (FC-side rates left neutral). |
| **Deadband** | Adjustable dead zone (0–300 raw ADC counts) around stick center to reject sensor noise/jitter. |
| **Live Monitor** | Real-time bar-graph view of all 4 stick channels (post-expo/post-rate, so you see the actual effect of your Expo/Rate settings) and both switches. Flags any axis whose calibration range is too small instead of showing a misleading reading. |
| **On-device flight timer / stopwatch** | DOWN = start / pause / resume, UP = stop, UP again = reset. Keeps counting correctly even while you're in the menu. |
| **"Find My Drone"** | Hold OK on the main screen to trigger a 2-second AUX3 beeper pulse on the drone (requires a BEEPER mode range on AUX3 in the FC's Modes tab). |
| **Telemetry display (unverified mapping)** | Receives the drone's `FC_DATA` packet and shows raw T1–T4 fields, plus a labeled **hypothesis** guess at battery voltage / RSSI — clearly marked as unconfirmed until cross-checked against a multimeter. |
| **TX battery monitor** | Its own battery %, measured via a resistor divider, shown top-left (not a guess — directly measured). |
| **Drone low-battery alarm** | Configurable total-pack voltage threshold (6.0–13.0 V) with a distinct repeating beep pattern + red banner, based on the telemetry hypothesis above. |
| **Link-loss / pairing status** | Live PAIRING / LINKED / LOST states, red "TELEM LOST" banner if the drone's heartbeat (`FC_ALIVE`) stops for >2.5 s. |
| **Persistent settings** | All calibration and tuning saved to flash (ESP32 `Preferences`/NVS), survives power cycles. |
| **Button/buzzer debug logging** | Every button/switch edge prints to Serial (115200 baud) for wiring/debounce diagnosis. |
| **Crosstalk-safe ADC reads** | Double-reads each analog pin (discard-first-sample) to work around a known ESP32 ADC1 channel-to-channel crosstalk erratum on adjacent reads. |

---

## 🔧 Hardware / Bill of Materials

| Component | Notes |
|---|---|
| ESP32-WROOM-32U dev board | External antenna (U.FL), used for better range |
| 3 dBi external antenna | Connects to the WROOM-32U's U.FL connector |
| 2.8" ILI9341 TFT display (SPI) | 320×240, used with the **TFT_eSPI** library |
| 2× analog joysticks (2-axis each) | One spring-centered (Roll + Pitch), one mixed (Throttle — no spring/self-centering; Yaw — spring-centered) |
| 2× 2-position toggle switches | SW1 = ARM, SW2 = Acro/Horizon flight mode |
| 3× momentary push buttons | UP / DOWN / OK, for menu navigation |
| 1× passive buzzer | Menu feedback + warnings |
| 1× status LED | *(wired but not yet used in firmware logic)* |
| TX LiPo/Li-ion battery + resistor divider | Powers the TX; divider scales battery voltage down for the ESP32 ADC |

> No potentiometer, no dedicated trim buttons — trim is done entirely in software via the
> Stick Calibration wizard.

### Pin Map (ESP32-WROOM-32U, 38-pin DevKit)

| Function | GPIO | Notes |
|---|---|---|
| TFT CS | 5 | Set in TFT_eSPI's `User_Setup.h`, **not** in the sketch |
| TFT DC | 2 | — same — |
| TFT RST | 4 | — same — |
| TFT SCK | 18 | VSPI |
| TFT MOSI | 23 | VSPI |
| TFT MISO | 19 | VSPI |
| Throttle (ADC1) | 36 | No mechanical spring/center |
| Yaw (ADC1) | 39 | Spring-centered |
| Roll (ADC1) | 34 | Spring-centered |
| Pitch (ADC1) | 35 | Spring-centered |
| TX battery sense (ADC1) | 32 | Through a resistor divider — **never feed raw battery voltage into an ESP32 ADC pin** |
| SW1 — ARM | 25 | Active LOW, internal pull-up |
| SW2 — Flight mode | 26 | Active LOW, internal pull-up |
| Button UP | 13 | Active LOW, internal pull-up |
| Button DOWN | 14 | Active LOW, internal pull-up |
| Button OK | 27 | Active LOW, internal pull-up |
| Buzzer | 33 | PWM tone via `ledcAttach`/`ledcWriteTone` (ESP32 core 3.x API) |
| Status LED | 21 | Wired, reserved for future use |

All ADC pins are on **ADC1** deliberately (ADC2 conflicts with WiFi, which this project
depends on for ESP-NOW). GPIO 0, 1, 3, 6–11, 12, 15 are avoided (flash/boot-strapping
pins).

---

## 📡 Protocol / Compatibility

- **Flight controller firmware:** [EspFC](https://github.com/rtlopez/esp-fc) (Betaflight-compatible, ESP32-native). On the drone's Receiver tab, set **Receiver Mode → `SPI Rx (e.g. built-in Rx)`**.
- **Link layer:** Raw `esp_now.h` (built into the ESP32 Arduino core) — no third-party RC library. Reverse-engineered from EspFC's `espnow-rclink` transport (`Protocol.h` / `Transmitter.h` / `Receiver.h`), which is **not** documented publicly beyond its source.
- **Message types:** `RC_DATA (0x01)`, `FC_ALIVE (0x10)`, `FC_DATA (0x11)` telemetry, `PAIR_REQ (0xFE)`, `PAIR_RES (0xFF)`. XOR checksum, seeded at `0x55`.
- **Channel order:** `AETR1234` (Roll, Pitch, Throttle, Yaw, then AUX1–4) — EspFC/Betaflight's standard default; no remap needed on the FC side.
- **Pairing:** the drone broadcasts `PAIR_REQ` (with its current WiFi channel) until a TX replies; this TX channel-hops 1–13 every 300 ms to find it, then tracks the drone's MAC from the received packet.

### ⚠️ Known limitation — this specific EspFC build

The EspFC build this project was tested against (`v0.2.1`) is a **minimal/stripped
compile**: `pin_led` and `feature_telemetry` CLI parameters both return
`param not found`. In other words, **this build has no LED-status output and no
telemetry feature compiled in**, even though the Receiver tab shows a Telemetry
toggle. If you build your own EspFC firmware with telemetry enabled, the T1–T4
hypothesis above is what you'll want to verify/correct first.

---

## 📲 Required Arduino Libraries

| Library | Why |
|---|---|
| **TFT_eSPI** (Bodmer) | Display driver. ⚠️ Pin configuration (CS/DC/RST/MOSI/MISO/SCLK + driver select) must be set **inside the library's own `User_Setup.h`**, not in the sketch — see `/docs/TFT_eSPI-setup.md` or the comment block at the top of the `.ino`. |
| `esp_now.h`, `WiFi.h`, `esp_wifi.h`, `Preferences.h` | Built into the ESP32 Arduino core — no install needed. |

No PlatformIO-only dependencies — this was a hard requirement so the project could be
built and iterated on entirely from a phone.

---

## 🚀 Getting Started

1. Wire everything per the **Pin Map** above.
2. Install the **TFT_eSPI** library, then edit its `User_Setup.h` with this panel's
   pin configuration (see the comment block at the top of the `.ino` file for the
   exact lines to paste in).
3. Flash the `.ino` to the ESP32-WROOM-32U.
4. On the drone, in EspFC/Betaflight Configurator → **Receiver tab**: set
   **Receiver Mode = `SPI Rx (e.g. built-in Rx)`**, save, reboot.
5. Power on the TX — it will channel-hop looking for the drone's pairing broadcast.
6. Once paired, open the TX's on-screen menu → **Stick Calibration** and follow the
   prompts (Throttle: release, then push fully in whichever direction the stick
   actually travels in; Roll/Pitch/Yaw: center, then move to all extremes).
7. In the FC's **Modes** tab, assign **AUX1 → ARM** and **AUX2 → HORIZON** (or
   ANGLE) with appropriate ranges matching your switch positions.
8. **Props off**, confirm arming and control response in Live Monitor before ever
   attaching propellers.

---


## 🙏 Acknowledgements

- [rtlopez/esp-fc](https://github.com/rtlopez/esp-fc) — the Betaflight-compatible
  ESP32 flight controller firmware this transmitter talks to.
- [Bodmer/TFT_eSPI](https://github.com/Bodmer/TFT_eSPI) — display driver.
