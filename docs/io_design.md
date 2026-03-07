# IO Design — RP2040 ↔ Pi5 Control & Sync Interface

This document describes the electrical wiring, GPIO signals, and UART protocol
used between the Raspberry Pi 5, RP2040, and connected modules (sensor, LED driver,
and RS-485 bus).  
The goal is deterministic timing, simple debugging, and scalable multi-module sync.

---

## Hardware Wiring

### RP2040 ↔ Raspberry Pi 5

Main communication (parameters, mode, dimmer, delays, etc.)

| RP2040 | Raspberry Pi 5 | Description |
|---|---|---|
| UART0 TX (1) | UART RX (GPIO15 / pin 10) | UART communication |
| UART0 RX (2) | UART TX (GPIO14 / pin 8) | UART communication |
| GND (3) | GND (pin 14) | Ground |
| GP2 (4) | GPIO5 (pin 29) | ARM / enable signal |
| GP3 (5) | GPIO6 (pin 31) | Status heartbeat |

---

### RP2040 ↔ SP3485 (RS-485)

Used for module-to-module communication.

| RP2040 | SP3485 | Description |
|---|---|---|
| 3V3 (36) | VCC (3–5V) | Power |
| GND (13) | GND | Ground |
| UART1 TX (11) | RXI | Data TX |
| UART1 RX (12) | TXO | Data RX |
| GP7 (10) | RTX | TX/RX direction control |

---

### RP2040 ↔ AL8843Q LED Driver

PWM dimmer control for LED strobe.

| RP2040 | AL8843Q | Description |
|---|---|---|
| GND (28) | GND | Ground |
| GP21 (27) | CTRL (via RC filter) | PWM dimmer (0.3–2.5 V) |

---

### RP2040 ↔ OV9281 (UC-788 Rev.B)

Trigger and feedback signals.

| RP2040 | OV9281 | Description |
|---|---|---|
| GND (23) | GND | Ground |
| GP17 (22) | FSTROBE | Sensor feedback (output from sensor) |
| GP16 (21) | XVS | Trigger signal (input to sensor) |

---

## IO Signal Design (Initial Version)

### GPIO

- **ena**  
  Enable / run permission.  
  **HIGH = run allowed**, LOW = safe idle / stop.

- **status** (Heartbeat pattern)  
  Fast diagnostic signal without UART.

  - **1 Hz**  = RUN OK (normal operation)
  - **5 Hz**  = WARN (running with warnings)
  - **10 Hz** = FAULT (latched)
  - **constant HIGH/LOW** = BOOT / DEAD / wiring issue  
    - if constant for **> 2 s** → treat as FAULT

---

### UART Parameters (SET)

- **timebase_hz**: `1..1000000`
  Internal timing base in Hz for the scheduler (used to quantize offsets/width/exposure). Choose higher for finer timing resolution (at higher CPU cost).

- **dimmer_pct**: `0..100`  
  LED / strobe brightness percentage.

- **fps_request**: `1..10000` *(clamped to hardware limits)*  
  Requested frame rate. Real value is reported via `fps_actual`.

- **timing_source**: `internal | sensor`  
  Select timing reference:
  - `internal` → RP2040 internal timer (no sensor feedback required)
  - `sensor` → timing measured from sensor feedback (e.g. FSTROBE)

- **strobe_ref**: `XVS | FSTROBE`  
  Timing reference signal:
  - `XVS` = trigger reference
  - `FSTROBE` = real sensor exposure timing (recommended if available)

- **strobe_offset_us**: `-100000..+100000`  
  Time offset relative to `strobe_ref`.

  - **positive (+)** → strobe occurs **after** reference
  - **negative (−)** → strobe occurs **before** reference

- **strobe_width_us**: `0..100000`  
  Strobe pulse width (clamped to exposure if required).

- **exposure_us**: `0..100000`  
  Requested exposure time.

- **apply_mode**: `immediate | next_frame`  
  Parameter apply behavior:
  - `immediate` → apply instantly
  - `next_frame` → apply on next frame boundary

- **settle_count**: `0..10`  
  Number of stability REPORT samples after APPLY.  
  `0` = no automatic feedback.

- **settle_mode**: `frames | time` *(default: frames)*  
  Defines sample generation:
  - `frames` → per frame
  - `time` → periodic in time

- **settle_time_ms**: `0..30000`  
  Only relevant when `settle_mode = time`.  
  Interval between REPORT samples.

---

### UART Feedback (REPORT)

- **state**: `BOOT | IDLE | ARMED | RUN | FAULT`
- **fps_actual** — measured / real frame rate
- **frame_period_us** — real frame timing (ground truth)
- **status_ok**: `0 | 1`
- **warn_flags** — bitmask (clamping, missing feedback, settling, etc.)
- **error_code** — `0 = OK`, otherwise error reason

---

### UART Frame Format

- **cmd_id** — command type (SET, APPLY, REPORT, etc.)
- **seq_id** — sequence counter (drop / duplicate detection)
- **payload** — command data
- **CRC16-CCITT** — frame integrity check

---

## IO Signal Design (Professional Version)

Additional parameters for synchronization between multiple RP2040 nodes.

### UART Parameters (SET)

- **sync_role**: `master | slave | auto`
- **sync_mode**: `simultaneous | phased`
- **node_id**: `0..255`
- **phase_count**: `1..N`
- **phase_index**: `0..N-1`
- **guard_us**: `0..200`

---

### UART Feedback (REPORT)

- **sync_locked**: `0 | 1`
- **slot_us**
- **phase_index / phase_count (active)**
- **strobe_ref (active)**

---

### UART Integrity / Diagnostics

- **firmware_version**
- **last_crc_error_count**

---

## Notes

- XVS = sensor trigger input  
- FSTROBE = sensor exposure feedback output  
- Timing should be generated locally on RP2040 for deterministic behavior.
- UART is used for configuration and status only (not real-time timing).
