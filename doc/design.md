# FATRUG3 – Design Document

Status: **draft**, version 3 started 2026-10-09.

FATRUG is a pair of battery-powered timing gates for sprint training
([FAT](https://en.wikipedia.org/wiki/Fully_automatic_time)). One unit stands at the start, one at
the finish, and they talk over ESP-NOW. Version 3 is a from-scratch redesign of
[fatrug2](https://github.com/bardolf/fatrug2). The main goal is reliable runner detection.

---

## 1. Requirements

| # | Requirement | Note |
|---|---|---|
| R1 | Sprint timing **indoors and outdoors**, including direct sunlight | outdoor sun is the hard case |
| R2 | **One box per gate on a tripod**, no reflector on the other side of the lane | rules out retro-reflective light barriers |
| R3 | Gate-to-gate distance up to ~100 m | ESP-NOW Long Range |
| R4 | Timing error ≤ ~5 ms, excluding the "which body part broke the beam" ambiguity | v2 had a 20–50 ms systematic error |
| R5 | No false starts or missed finishes from sensor glitches | the main pain point of v2 |
| R6 | Battery life of several hours (a training session) | |
| R7 | **Web UI on a phone**: live time, run history, settings, export | served by the unit itself |
| R8 | **One firmware** for both units; role and peer stored in NVS | v2 needed two builds and hard-coded MACs |

## 2. Lessons from v2

**Detection with the HC-SR04 ultrasonic sensor**
- Pinging every ~10 ms is far faster than the ~60 ms the module needs. Echoes from walls or trees
  2 m away land in the next ping's window and look like a nearby object.
- After a missed echo, the module holds ECHO high for ~38 ms and ignores triggers.
- A missing echo (timeout) was treated as "no object". Soft clothing absorbs ultrasound, and an angled
  torso deflects it. So a lost echo looked like "runner left", i.e. a false start. The 100 ms
  `RUN_CHECK` state was a workaround for this.
- Bug: `abs(1 - _prevPrevDistance / _prevPrevDistance)` is always 0, so the second tolerance check
  never applied.
- At the finish, the logic required 3 consecutive samples within 20 % of each other. This fails for
  a running body: arms, torso and legs sit at different distances, and the body is in the cone
  for only ~40–60 ms.

**Timing and communication**
- The start unit computed the result from its own clock **when the finish message arrived**. Radio
  latency, retries and queueing were all added to the result, and this error was not constant.
- There was no application-level ack or retry. A lost FINISH message left the finish display
  running.
- MAC addresses were hard-coded and the role was chosen at compile time.
- `currentState` was shared between tasks without synchronization.
- The receive callback checked neither message length nor sender.

**Earlier attempt:** the VL53L0X laser sensor (v1) failed in sunlight because its emitter is too weak.

## 3. System overview

```
   START unit                                  FINISH unit
 ┌─────────────┐   ESP-NOW (LR, fixed channel)   ┌─────────────┐
 │ LiDAR       │ ◄─────────────────────────────► │ LiDAR       │
 │ ESP32       │   clock sync, events, acks      │ ESP32       │
 │ display/LED │                                 │ display/LED │
 └──────┬──────┘                                 └─────────────┘
        │ Wi-Fi SoftAP (same channel)
        ▼
   phone (web UI)
```

- Both units are identical hardware running identical firmware. The role (START/FINISH) is a setting.
- Each unit **timestamps its own detection locally** (µs, `esp_timer_get_time()`).
- The units keep a **shared timebase** through continuous clock synchronization over ESP-NOW.
  The result is `t_finish − t_start` in that timebase, so radio latency does not affect it.
- One unit (configurable, by default the one near the coach) runs a SoftAP with the web UI.

## 4. Hardware

### 4.1 Distance sensor – single-point LiDAR (decision)

All Benewake models below share the same 9-byte UART frame (`0x59 0x59`, distance, signal strength,
temperature, checksum). The firmware can therefore support all of them, and swapping the sensor is cheap.

| Sensor | Rate | Ambient light | Range @10 % refl. | Supply | Approx. price | Role |
|---|---|---|---|---|---|---|
| **TFmini-S** | up to 1000 Hz (UART) | 70 klux | 7 m | 5 V ±0.1 V | 1 245 CZK (RPishop, DFRobot SEN0259) | **primary choice** |
| TF-Luna | up to 250 Hz | 70 klux | 2.5 m | 3.7–5.2 V | ~30 USD | budget option |
| TF02-Pro | up to 1000 Hz | 100 klux, IP65 | 13.5 m | 5–12 V, ~200 mA | 2 698 CZK (LaskaKit) | fallback if sun is a problem |

- We need only ~0.5–1.5 m of range, so there is large margin even when range drops above the rated klux.
- 70 klux is roughly a bright day. Direct summer noon sun is ~100 klux. **Must be tested outdoors.**
- Mounting rules: do not aim into a low sun, and avoid a bright sunlit wall right behind the lane.
- UART logic is 3.3 V, so the sensor connects directly to the ESP32.
- At 1000 Hz the stream is ~90 kbit/s. Raise the baud rate from the default 115200 if the sensor allows it.
- Ultrasonic sensors and the VL53Lx family are rejected (see §2).

### 4.2 Other components

| Part | Choice | Notes |
|---|---|---|
| MCU | **Olimex ESP32-S3-DevKit-Lipo-EA** (ordered from olimex.com; ships with a small 2.5 dBi u.FL PCB antenna; battery connector polarity must be checked before connecting a LiPo) | ESP32-S3 dual core (radio on one core, LiDAR and detection on the other), u.FL with antenna included, LiPo charger, battery measurement, USB/LiPo auto switch, open-source HW, EU seller. Alternatives: FireBeetle 2 ESP32-S3-U, Unexpected Maker FeatherS3[D] |
| Antenna | 2.4 GHz 3–5 dBi SMA-male whip on a u.FL → SMA-female bulkhead pigtail through the box wall (LaskaKit; SMA vs. RP-SMA must match) | vertical polarization on both units. Mount high: at 100 m the 1st Fresnel zone radius is ~1.8 m, so the ground near a low box blocks part of it |
| Display | TBD | TM1637 0.56" is small; consider larger 7-segment digits or a sunlight-readable display |
| Status LED | WS2812 | as in v2 |
| Buzzer | new | audible feedback for armed, start, finish, and a future audio start |
| Button | 1× (reset/pair) | long press = pairing |
| Battery | LiPo 2500 mAh (as in v2) | |
| 5 V boost | for LiDAR (and display) | must handle LiDAR peak current. TFmini-S wants 5 V ±0.1 V, so use a fixed-output boost, not a trimmer-set MT3608 |
| Sensor power switch | P-MOSFET | turn the LiDAR off when idle to save battery |
| Battery measurement | divider + `analogReadMilliVolts()` | replaces the ad-hoc formula |

## 5. Detection

### 5.1 Signal processing (both roles)
1. **Learn the background** when the unit is armed: the median distance and signal strength over
   ~1 s. "No return" (open space behind the lane) is a valid background.
2. A sample counts as **object** when its distance is clearly shorter than the background
   (e.g. `< background − margin`) and its signal strength is above a minimum.
3. Use **hysteresis and a minimum duration** (N consecutive samples) to change state, so single
   bad samples are ignored.
4. **Timestamp the event at the first sample of the confirming run**, not at the decision.
   Each sample is stamped when its UART frame arrives, minus the constant frame transfer time.
   The sensor's internal latency is constant and largely cancels out between the two units
   (same sensor type).
5. **Log the raw stream** (time, distance, strength) on demand, so thresholds can be tuned on real data.

### 5.2 Start modes
- **Leave beam (default):** the runner stands ~0.5–1 m in front of the start unit. After a stable
  presence, the clock starts when the beam clears. A false-start guard checks that the runner does
  not reappear within X ms.
- *Future:* an audio start (beep after a random delay; the time then includes reaction time), and
  a flying start through the beam.

### 5.3 Finish
- The clock stops at the first confirmed object sample. Mount the unit at chest height.
- *Future, if arms trigger early:* a second LiDAR ~10–20 cm away, with both beams required within
  a short window (dual-beam logic).

## 6. Timing

### 6.1 Clock synchronization
NTP-style exchange over ESP-NOW, repeated about every second:

```
A: t1 ──req──► B: t2
A: t4 ◄──resp── B: t3
offset = ((t2 − t1) + (t3 − t4)) / 2
rtt    = (t4 − t1) − (t3 − t2)
```

- Keep the **minimum-RTT** samples, which have the least scheduling and retry jitter.
- Estimate drift by linear regression over a sliding window. Crystals drift ~10–20 ppm, which is
  ≤1.2 ms per minute uncorrected.
- Expected offset accuracy is well below 1 ms. This will be verified in the prototype.

### 6.2 Error budget (target)
| Source | TFmini-S @1000 Hz | TF-Luna @250 Hz |
|---|---|---|
| Sample quantization | ±0.5 ms | ±2 ms |
| Clock sync | <1 ms | <1 ms |
| Start vs. finish detection asymmetry | constant, to be calibrated | constant, to be calibrated |
| **Total (excl. body-part ambiguity)** | **~1–2 ms** | **~3 ms** |

## 7. Radio protocol (ESP-NOW)

- **Pairing:** a long button press on both units starts a broadcast exchange. Each unit stores the
  peer MAC (the correct interface MAC, STA vs. AP) and the channel in NVS.
- **Fixed Wi-Fi channel** shared by ESP-NOW and the SoftAP. A single radio cannot use two channels.
- **Long Range:** the ESP-NOW peer rate is set to LR (`esp_now_set_peer_rate_config`), and the
  SoftAP stays in 802.11b/g/n so phones can connect. **Coexistence must be verified (prototype P1).**
- **Message header:** magic, protocol version, message type, sequence number, run ID. The length is
  checked on receive, and only the paired peer is accepted.
- **Reliability:** every state-changing message is acked and retried. Receivers are idempotent per
  `(run ID, seq)`.
- **Message types (draft):** `PAIR_REQ/RESP`, `SYNC_REQ/RESP`, `ARM`, `DISARM`,
  `START_EVENT(t)`, `FINISH_EVENT(t)`, `RESULT`, `ACK`, `STATUS` (battery, sensor health, signal
  strength).

## 8. Web UI

- A SoftAP on the hosting unit, with a captive portal (DNS redirect) so the page opens by itself.
  Phones will report "no internet".
- Live updates over WebSocket: state, running time, result.
- Run history stored in flash (LittleFS), with optional athlete names, CSV export and clear.
- Settings: role, start mode, thresholds, display brightness, and a raw-data logging toggle.
- Static assets are gzipped and served from flash.

## 9. Firmware

- PlatformIO with Arduino-ESP32 3.x (ESP-IDF 5.x underneath), C++.
- Modules: `lidar` (UART parser + sample timestamps), `detector`, `timesync`, `link` (ESP-NOW protocol),
  `runcontroller` (state machine), `web`, `display`, `led`, `buzzer`, `battery`, `storage`.
- The state machine runs in a single task and consumes events from a queue. Shared state is accessed
  only through that task.
- Pure logic (detector, timesync, protocol codec, state machine) is kept hardware-free so it can be
  **unit-tested on the host** (`pio test -e native`).

## 10. Roadmap

| Phase | Content | Exit criterion |
|---|---|---|
| **P0** | Buy LiDARs (2× TFmini-S, or TF-Luna) | sensors on hand |
| **P1** | Prototype on v2 hardware: SoftAP + ESP-NOW LR coexistence and range; clock sync accuracy | phone connected and 100 m LR link at the same time; sync error measured |
| **P2** | Prototype: LiDAR raw logging, indoor and outdoor (full sun), real sprints | datasets recorded; detection parameters chosen |
| **P3** | Firmware v3 (all modules, web UI) | full runs on v2 hardware |
| **P4** | Hardware v3: PCB, box for the LiDAR, display, buzzer | two finished units |

## 11. Open questions / risks

- LiDAR behaviour in direct summer sun (P2).
- Whether ESP-NOW LR and a phone-facing SoftAP coexist on one ESP32 (P1).
- Display choice: size and sunlight readability.
- Is a single beam enough, or do arms trigger the finish early (dual beam)?
- Power budget with the LiDAR always on vs. switched off.
