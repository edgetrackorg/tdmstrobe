# TDMStrobe

**Time-Division-Multiplexed (TDM) NIR strobe & trigger hub** for multi-camera hand/gesture capture. Designed for **2–4 stereo rigs** (i.e., **4–8 mono cameras**) with small, staggered baselines to improve **precision** and **occlusion robustness**.

- **Per-frame strobing:** supports **single-strobe** patterns (one pulse per exposure).
- **Optics:** the prototype uses **120° emitters** for fast bring-up; production targets **60° / 90°** for higher efficiency (less wasted light, better SNR in the target ROI).
- **Spectrum:** default **850 nm** NIR illumination. **850 nm and 940 nm are both IR-A and not inherently eye-safe**—safe operation depends on irradiance, duty cycle, distance, and exposure time (see **Safety** below).

> **Note:** Off-the-shelf strobe hubs with the required deterministic MCU timing/control are not available; this module is **custom-built**.

> **Safety:** Never look into emitters. Both **850 nm** and **940 nm** can be hazardous at sufficient intensity. **940 nm may reduce visible glow**, but it is **not automatically safer**. See the **Safety** section below.

> **Status:** Early prototype (electronics/firmware WIP). Interfaces, API, and connectors may change.

---

## Features

- **Simple host control over UART:** the MCU is configured from a Raspberry Pi 5 (or any Linux host) via a minimal UART protocol.
- **Deterministic timing on the MCU:** the host only sets parameters; the MCU executes trigger + strobe timing with microsecond-level determinism.
- **Deterministic lighting** for global-shutter stereo/triangulation.
- **TDM (A/B/C/D) phase scheduling** prevents cross-illumination between stereo pairs.
- **Low-latency trigger fan-out** with a simple 4-wire sync option (for chaining / multi-node setups).
- **Integrated dimming** (global & per-channel) for fine-grained brightness control.


## Design goals

- **Deterministic timing:** microsecond-level trigger + strobe timing is executed on the MCU (not in Linux userspace).
- **Low latency:** minimize buffering and keep the host path simple (UART config + status only).
- **Scalable multi-rig:** support **2–4 stereo rigs** by time-slicing illumination into phases (A/B/C/D).
- **Repeatable and debuggable:** explicit states (ARM/RUN/STOP), readable status, and clear fault reporting.
- **Hardware-first safety:** conservative defaults, explicit enable, and documented eye-safety constraints.

---

## What “TDM” means in practice

TDM = **Time Division Multiplexing**. The frame time is split into **phases** (A/B/C/D).  
In each phase, **only one rig (or one illumination group)** is allowed to emit NIR.  
This prevents **cross-illumination** (rig A lighting rig B’s exposure) and keeps stereo matching stable.

---

## Roadmap

coming soon. 

---

## License

**Apache‑2.0** (code, firmware, docs). Hardware files: to be added; may use CERN‑OHL‑S.

---

## Safety

### NIR Illumination (850 nm vs 940 nm) & Eye Safety

* **Never look into emitters.** Use black matte **baffles/shields**, aim emitters away from faces, and add **hardware interlocks** (LEDs off on loss of sync, open covers, or presence detection).
* Keep **exposure short** (strobe pulses strictly within camera exposure) and **average irradiance low**.
* Prefer **850 nm band-pass filters** on cameras to reduce the required LED output power.
* **850 nm and 940 nm are both IR-A** and are **not inherently eye-safe**; safety depends on irradiance, geometry, duty cycle, distance, and exposure time (IEC 62471).

### Solution Strategies

**Option A — Prefer more viewpoints over more power (recommended for 940 nm)**
- If **940 nm illumination** is preferred (reduced visible glow), the recommended approach is to **increase the number of stereo rigs (viewpoints)** to maintain SNR while keeping **irradiance low**, rather than compensating with higher-power NIR emitters.

**Option B — Side / rear placement (recommended)**
- Mount stereo pairs **left/right and slightly behind** the workspace, aimed toward the work area. Add **one or two top stereo pairs** for occlusion-free coverage. This directs NIR **away from the eyes** while maintaining uniform scene illumination.
Future refinement: recess-mount one stereo pair near the table center and another near the back edge for a slimmer, more robust setup.

**Option C — Front placement with HMD only**
- If stereo pairs must face forward, operate with a **closed VR headset** (no see-through optics) so the user’s eyes are **occluded**. Baffles and interlocks are still required to protect bystanders.

**Option D — IR-filtering safety eyewear**
- Use **visible-light-transmitting eyewear** that strongly attenuates **near-IR (≈ 780–950 nm)** (specified optical density at **850 nm / 940 nm**) so users retain normal vision while IR exposure is reduced.

**Option E — Side-shield eyewear (“horse-blinkers” concept)**
- Provide **IR-blocking safety glasses with side shields** for operators and visitors when emitters face forward. Ensure proper **near-IR attenuation ratings** and a snug fit to block off-axis radiation.

---

## Disclaimer

Prototype hardware. Use at your own risk. Ensure eye‑safety and proper thermal design in all setups.
