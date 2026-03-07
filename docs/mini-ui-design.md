# UI Design (Edge Node) — Mini UART Control Panel (Prototype)

This document describes a minimal and practical UI for controlling an RP2040 over UART
directly from a Raspberry Pi 5 edge node.

Scope:
- **Single edge node**
- **Local UART connection**
- Quick parameter editing, start/stop control, and live status monitoring

> Out of scope: multi-node control, CoreFusion UI, network routing/tunneling, or remote Ethernet management.

---

## Goals

- Fast local bring-up on Pi5 (prototype workflow)
- Safe operation with explicit ARM / RUN / STOP states
- Deterministic timing remains inside RP2040 (UI only configures and reads status)
- Simple debugging (TX/RX frames, CRC errors, warnings)
- Works headless via SSH (terminal UI) or locally in a terminal window

---

## Recommended Implementation

- Python 3
- `pyserial`
- UI framework: `textual`

Why:

- Runs everywhere on Raspberry Pi 5
- Works over SSH
- Minimal dependencies
- Easy to package as a single CLI application

Binary name suggestion:

- `edgetrack-uart-ui`

---

## UI Layout (Single Screen)

### Top Bar — Connection & Safety

- **Serial Port**: dropdown (`/dev/ttyAMA0`, `/dev/serial0`, `/dev/ttyACM0`, etc.)
- **Baudrate**: dropdown (e.g. 115200, 230400, 460800, 921600)
- **Connect / Disconnect** button
- **CRC Status**: `OK` / `ERR` (live)
- **Link Stats**: `RX rate`, `TX rate` (frames/s)
- **E-STOP** (large button): immediately sends `STOP` and sets `ena=LOW` (if supported)

---

### Left Panel — Parameters

Grouped inputs with a **dirty state** (highlighted when modified but not yet applied).

#### Timing / Sync

- `timing_source`: `internal | sensor`
- `fps_request`: numeric input
- `apply_mode`: `immediate | next_frame`

#### Exposure / Strobe

- `strobe_ref`: `XVS | FSTROBE`
- `strobe_offset_us`: numeric input (+/-)
- `strobe_width_us`: numeric input
- `exposure_us`: numeric input
- `dimmer_pct`: slider (0..100)

#### Settle / Feedback

- `settle_count`: numeric input (0..10)
- `settle_mode`: `frames | time`
- `settle_time_ms`: numeric input

#### Parameter Controls

- **Send SET** (does not start running)
- **APPLY** (uses selected apply mode)
- **Save Preset** (store local JSON/YAML)
- **Load Preset** (restore local JSON/YAML)

---

### Center Panel — Run Control

Clear and explicit state-machine actions:

- **ARM** (sets `ena=HIGH`, sends ARM command)
- **RUN** (starts timing loop on RP2040)
- **STOP** (halts timing loop but keeps UART link alive)
- **DISARM** (sets `ena=LOW`, safe idle state)

Optional:

- **Single Shot** (one frame trigger, if supported)
- **Query Status** (manual REPORT request)

---

### Right Panel — Live Status / Telemetry

Updated continuously (e.g. 10–20 Hz or per REPORT):

- `state`: BOOT / IDLE / ARMED / RUN / FAULT
- `fps_actual`
- `frame_period_us`
- `status_ok`
- `warn_flags` (decoded + raw hex)
- `error_code` (decoded + raw)

Optional sensor indicators:

- `xvs_out_hz`
- `fstrobe_in_hz`
- `timing_source_active`
- `strobe_ref_active`

---

### Bottom Panel — Log / Frames

Tabbed logging view:

- **Events**: connect/disconnect, apply, arm/run/stop, faults
- **UART Frames (decoded)**: cmd_id, seq_id, parsed payload
- **Raw HEX**: raw bytes (optional toggle)
- **Errors**: CRC failures, sequence drops, timeouts, malformed frames

---

## UX Rules (Important)

### Safety-first actions

- RUN must always be separate from APPLY.
- Any FAULT state should:
  - optionally stop RUN automatically
  - keep UART alive for diagnostics
  - require explicit user action to clear (e.g. `CLEAR_FAULT`)

### Dirty Parameters

- Editing values must NOT immediately change hardware.
- Modified fields are visually highlighted.
- Only **SET + APPLY** updates active configuration.

### Live Feedback Behavior

If `settle_count > 0`, after APPLY the UI should:

- show a **SETTLING…** indicator
- collect N REPORT samples (frame- or time-based)
- display min / avg / max values for:
  - `frame_period_us`
  - `fps_actual`
- mark state as **STABLE** when complete

### Auto-Reconnect

If UART disconnects:

- show **DISCONNECTED**
- optionally retry every 2 seconds
- never auto-run after reconnect (safety rule)

---

## Presets (Local)

Presets allow reproducible testing and fast setup changes.

- Format: JSON
- Suggested path: