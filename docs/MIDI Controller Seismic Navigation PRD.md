# MIDI Seismic Navigator — Product Requirements Document

**Project Codename:** `SeisJog`
**Version:** 0.1 (Proof of Concept)
**Author:** Geophysical Tooling Initiative
**Status:** Draft — Hobby / Experimental
**Date:** April 2026

***

## Executive Summary

SeisJog is a lightweight Python proof-of-concept that connects a USB MIDI controller — the kind used for music mixing — to 3D seismic interpretation software, enabling hands-on, physical navigation of seismic volumes without touching a keyboard or mouse. The project is a creative experiment: the hypothesis is that the tactile, muscle-memory experience of moving faders and pressing pads to flip through inlines and crosslines could be more fluid and enjoyable than pecking arrow keys. This document defines the controls landscape, system architecture, functional requirements, and a suggested implementation path for a Python developer building this as a side project.

***

## 1. Background and Motivation

### 1.1 The Problem with Keyboard-Only Navigation

Navigating 3D seismic volumes is a physically repetitive, high-iteration task. A geophysicist scanning through a volume during regional exploration may step through hundreds of inlines per session — checking fault geometries, assessing stratigraphic packages, or looking for DHIs. Commercial platforms like Petrel (SLB), OpendTect (dGB), Kingdom (IHS), and DUG Insight all expose navigation through keyboard shortcuts and toolbar clicks[^1][^2]. These work, but they demand eyes-on-keyboard and interrupt interpretive flow.

MIDI controllers — physical hardware originally designed for music production — expose an array of faders (linear sliders), rotary knobs, and drum-pad buttons that generate continuous and discrete digital messages over USB[^3][^4]. The core idea of SeisJog is that a fader swept physically from bottom to top can scroll through all inlines in a volume, while a knob controls gain, and a row of pads jumps to named positions. No special APIs, no vendor plugins — just Python reading MIDI events and translating them into navigation actions.

### 1.2 Why Python

Python's ecosystem provides everything needed: `mido` or `python-rtmidi` for MIDI I/O[^5][^6], `pyautogui` or `pynput` for OS-level keyboard/mouse simulation[^7][^8], and `segyio` plus `matplotlib` for a standalone viewer mode[^9][^10]. The entire stack is open-source, cross-platform, and well-documented. The project does not require access to proprietary seismic software APIs.

### 1.3 Scope Boundaries

This is explicitly **not** an interpretation tool. SeisJog does not pick horizons, create faults, or write to interpretation databases. It is purely a **navigation and display control** layer. The goal is to demonstrate that physical hardware can make exploration workflows more tactile and fun.

***

## 2. Target Users

| User | Context | How They Use SeisJog |
|------|---------|---------------------|
| Exploration Geophysicist | Regional reconnaissance, scanning large volumes | Scroll inlines/crosslines with fader; tweak gain with knob |
| Geologist | Quick structural review | Jump to saved positions with pads; flip between IL and Z-slice views |
| Data Scientist / Geoscientist | Python-based seismic analysis workflow | Use standalone viewer mode with segyio + matplotlib |
| Hobbyist / Tinkerer | Learning MIDI + Python automation | Experiment with the bridge layer, customize mappings |

**Primary persona for v0.1:** A geoscientist with a Python development background who owns any generic USB MIDI controller and wants a weekend project that connects physical hardware to their seismic workflows.

***

## 3. Core Concepts: Seismic Volume Navigation Controls

Understanding what needs to be controlled is the foundation of this PRD. A 3D seismic volume is indexed in three orthogonal axes, and the interpreter controls both **where** they are looking and **how** the data is displayed[^11].

### 3.1 Spatial Navigation Controls

These are the primary navigation axes for any 3D seismic volume:

| Control | Description | Native UI Analog | Range |
|---------|-------------|-----------------|-------|
| **Inline (IL) Position** | Step forward/backward through N-S lines (survey-dependent) | Arrow keys, scroll wheel, spinner box | Survey IL min → IL max [^12] |
| **Crossline (XL) Position** | Step forward/backward through E-W lines | Arrow keys, scroll wheel, spinner box | Survey XL min → XL max [^12] |
| **Time/Depth Slice (Z)** | Step through horizontal slices (TWT ms or depth m) | Arrow keys, spinner box | 0 ms → max record length [^13] |
| **IL Step Size** | How many lines to advance per tick (1, 2, 5, 10…) | Step size field | 1 → N [^13] |
| **XL Step Size** | How many crosslines to advance per tick | Step size field | 1 → N |
| **Z Step Size** | How many samples to advance per Z-tick | Step size field | 1 → N |
| **Navigation Mode** | Which axis is "active" (IL, XL, or Z) | Mouse click on slice | Enum: IL / XL / Z |

In OpendTect, users assign user-defined keys (typically `F` and `B`) to move the active slice forward and backward[^2]. In Petrel, arrow keys navigate the active element, and the active inline/crossline is displayed in a spinner that updates in real-time[^1][^14].

### 3.2 Display / Rendering Controls

These controls affect how the seismic amplitude data is rendered — critical for exploration scanning because the interpreter often needs to dynamically adjust visibility as they scroll:

| Control | Description | Range / Type |
|---------|-------------|-------------|
| **Display Gain** | Linear amplitude scaling for wiggle/color display | 0.1× → 10× (float) [^15] |
| **Clip (Symmetrical)** | Clips amplitudes at ±N% of max; avoids saturation | 0.1 → 1.0 [^16] |
| **Bias / Polarity** | Shifts color scale midpoint; useful for AVO work | -1.0 → +1.0 [^15] |
| **Display Mode** | Wiggle + VA, Variable Density only, Wiggle only | Enum toggle [^17] |
| **Colormap / Palette** | Color scale applied to variable density display | Enum: seismic, greys, rainbow, etc. [^16] |
| **Vertical Stretch** | Z-axis exaggeration | 0.5× → 5× |
| **Zoom Level** | Horizontal/vertical zoom of the section view | Continuous [^13] |

The gain and clip controls are especially important: many interpreters dynamically scan with high gain to find subtle events, then reduce gain to see the strong reflectors in context[^16]. Having physical knobs for these parameters is a natural fit for a MIDI controller.

### 3.3 Multi-Volume / Attribute Controls

Modern interpretation workflows almost always involve co-rendering multiple seismic volumes (e.g., amplitude + coherence + sweetness attribute):

| Control | Description |
|---------|-------------|
| **Active Volume Selector** | Which loaded volume is being navigated | Cycle through loaded volumes [^18] |
| **Overlay Opacity** | Blend ratio between primary and attribute overlay | 0.0 → 1.0 |
| **Attribute Blend Mode** | How overlay renders on top of primary | Enum: blend, replace, colorize |

### 3.4 Viewport / Window Controls

| Control | Description |
|---------|-------------|
| **Linked Navigation** | Whether multiple views (IL + XL + Z) move together | Toggle |
| **Snapshot / Screenshot** | Save current view to PNG | Momentary button |
| **Bookmark / Jump to Position** | Store current IL/XL/Z and recall later | 4–8 pad buttons |
| **Reset View** | Return to home zoom and position | Button |

***

## 4. MIDI Controller — Controls Mapping Model

### 4.1 MIDI Message Primer (Relevant Subset)

A MIDI controller communicates over USB using three message types relevant to SeisJog[^5]:

| MIDI Message | Status Byte | Data Bytes | Physical Source |
|-------------|------------|-----------|-----------------|
| `note_on` | 0x90 + channel | note (0–127), velocity (0–127) | Pads, piano keys |
| `note_off` | 0x80 + channel | note (0–127), velocity (0–127) | Pad released |
| `control_change` (CC) | 0xB0 + channel | CC number (0–127), value (0–127) | Knobs, faders, wheels |

All faders and knobs produce CC messages with a value between 0 and 127[^19][^3]. This 7-bit resolution translates to 128 discrete positions, which is more than sufficient for stepping through seismic volumes or adjusting gain.

### 4.2 Suggested Control Mapping (Reference Controller: 8-Fader / 8-Knob Layout)

This mapping assumes a common MIDI controller layout such as the Behringer BCF2000, Akai APC, or similar 8-channel mixer-style device. CC numbers are illustrative; the implementation uses a config file for remapping[^20].

| Physical Control | MIDI Message | Seismic Action | Notes |
|-----------------|-------------|---------------|-------|
| **Fader 1** | CC 0 (0–127) | Inline position | Maps 0–127 → IL min–max (normalized) |
| **Fader 2** | CC 1 (0–127) | Crossline position | Maps 0–127 → XL min–max (normalized) |
| **Fader 3** | CC 2 (0–127) | Time/Depth slice | Maps 0–127 → Z min–max |
| **Fader 4** | CC 3 (0–127) | Display Gain | Maps 0–127 → 0.1×–10× |
| **Fader 5** | CC 4 (0–127) | Clip level | Maps 0–127 → 1%–100% |
| **Fader 6** | CC 5 (0–127) | Bias / Color shift | Maps 0–127 → -1.0 → +1.0 |
| **Fader 7** | CC 6 (0–127) | Overlay opacity | Maps 0–127 → 0%–100% |
| **Fader 8** | CC 7 (0–127) | Vertical stretch | Maps 0–127 → 0.5×–5× |
| **Knob 1** | CC 16 | IL step size | Detented: 1, 2, 4, 8, 16, 32 |
| **Knob 2** | CC 17 | XL step size | Detented: 1, 2, 4, 8, 16, 32 |
| **Knob 3** | CC 18 | Z step size | Detented: 1, 2, 4, 8, 16 |
| **Knob 4** | CC 19 | Colormap selector | Cycles through palette list |
| **Pad 1** | Note 36 | Step IL forward (+step) | Momentary trigger |
| **Pad 2** | Note 37 | Step IL backward (-step) | Momentary trigger |
| **Pad 3** | Note 38 | Step XL forward (+step) | Momentary trigger |
| **Pad 4** | Note 39 | Step XL backward (-step) | Momentary trigger |
| **Pad 5** | Note 40 | Step Z forward (+step) | Momentary trigger |
| **Pad 6** | Note 41 | Step Z backward (-step) | Momentary trigger |
| **Pad 7** | Note 42 | Cycle active volume | Toggle through volumes |
| **Pad 8** | Note 43 | Snapshot current view | Save PNG to output dir |
| **Pad 9–12** | Notes 44–47 | Recall bookmarks 1–4 | Jump to saved IL/XL/Z |
| **Pad 13–16** | Notes 48–51 | Store bookmarks 1–4 | Save current IL/XL/Z |
| **Transport: Play** | Note 116 | Toggle linked navigation | Sync all views |
| **Transport: Stop** | Note 117 | Reset view | Home zoom and position |
| **Mod wheel** | CC 1 | Fine IL scroll (velocity-sensitive) | Wheel rate → step speed |

> **Note:** These are defaults. All mappings live in a YAML or JSON config file and are fully remappable at runtime.

***

## 5. System Architecture

### 5.1 Operating Modes

SeisJog operates in two modes, selectable at startup:

**Mode A — Standalone Viewer (Self-Contained)**
Reads a SEG-Y file directly using `segyio`, renders slices in a `matplotlib` window, and responds to MIDI input to update what is displayed. No external seismic software needed. Best for demos, testing, and Python-heavy workflows.

**Mode B — Application Bridge (Keyboard/Mouse Emulation)**
Listens to MIDI input and translates events into OS-level keyboard shortcuts or mouse scroll events via `pyautogui`[^7][^8], forwarding them to whichever seismic application has focus (Petrel, OpendTect, Kingdom, etc.). The Python process acts as an invisible middleware. No SEG-Y file is needed — the interpreter works directly inside their native tool.

### 5.2 Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         SeisJog Runtime                         │
│                                                                 │
│  ┌──────────────┐    ┌──────────────────┐    ┌───────────────┐ │
│  │  MIDI Input  │───▶│  Event Router    │───▶│  Action       │ │
│  │  Thread      │    │  (mapping layer) │    │  Dispatcher   │ │
│  │  (mido/      │    │                  │    │               │ │
│  │  rtmidi)     │    │  Reads YAML cfg  │    │  Mode A:      │ │
│  └──────────────┘    │  CC → action     │    │  Matplotlib   │ │
│                       │  Note → action  │    │  viewer       │ │
│  ┌──────────────┐    └──────────────────┘    │               │ │
│  │  Config      │                            │  Mode B:      │ │
│  │  YAML/JSON   │                            │  pyautogui    │ │
│  │  (CC→action  │                            │  keyboard/    │ │
│  │   mappings)  │                            │  mouse sim    │ │
│  └──────────────┘                            └───────────────┘ │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Seismic State Manager                                   │  │
│  │  (current IL, XL, Z, gain, clip, step sizes, bookmarks)  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

External:
  USB MIDI Controller ──────────────────────────────────────────▶
  Petrel / OpendTect / Kingdom (Mode B, focus window) ◀─────────
  SEG-Y File on disk (Mode A) ◀────────────────────────────────
```

### 5.3 Threading Model

MIDI I/O is inherently event-driven and must not block the display/UI thread. The recommended pattern uses `mido`'s callback interface, which invokes the callback from a background thread managed by mido[^21]. The callback pushes events onto a thread-safe `queue.Queue`. The main thread (or matplotlib event loop) drains this queue on each tick[^22].

```
Thread 1 (mido background)  ──── CC/Note event ──▶  Queue
Thread 2 (main/matplotlib)  ◀─── drain queue ──────  Queue
                                  │
                                  ▼
                            Apply to state
                            Update plot / send keystroke
```

This approach avoids asyncio complexity while maintaining responsiveness. For Mode A, `matplotlib`'s `FuncAnimation` or `canvas.draw_idle()` provides non-blocking redraw[^23][^24].

***

## 6. Functional Requirements

### 6.1 MIDI Input Layer (FR-MIDI)

| ID | Requirement |
|----|-------------|
| FR-MIDI-01 | The system SHALL detect all available MIDI input ports at startup and present them to the user for selection |
| FR-MIDI-02 | The system SHALL process `control_change` messages (CC 0–127, values 0–127) from any channel |
| FR-MIDI-03 | The system SHALL process `note_on` messages (notes 0–127) as momentary trigger events |
| FR-MIDI-04 | MIDI input SHALL run in a background thread and SHALL NOT block the main application loop |
| FR-MIDI-05 | The system SHALL log all received MIDI messages to console in a verbose debug mode |
| FR-MIDI-06 | The system SHALL NOT crash if the MIDI controller is disconnected; it SHALL log an error and wait |

### 6.2 Configuration Layer (FR-CFG)

| ID | Requirement |
|----|-------------|
| FR-CFG-01 | All MIDI-to-action mappings SHALL be stored in a human-readable YAML or JSON file |
| FR-CFG-02 | The config file SHALL support mapping any CC number to any seismic action |
| FR-CFG-03 | The config file SHALL support specifying value scale ranges (e.g., map CC 0–127 to IL 100–800) |
| FR-CFG-04 | The config SHALL include a `mode` field: `standalone` or `bridge` |
| FR-CFG-05 | The config SHALL include a `dead_zone` parameter for faders (default: ±2 CC units) to suppress jitter |
| FR-CFG-06 | Profile presets for common controllers (Akai MPK Mini, Behringer BCF2000) SHALL be included as example configs |

### 6.3 Seismic State Manager (FR-STATE)

| ID | Requirement |
|----|-------------|
| FR-STATE-01 | The state manager SHALL track current IL, XL, Z index, gain, clip, bias, step sizes, active volume, and bookmarks |
| FR-STATE-02 | IL, XL, and Z SHALL be clamped to valid survey bounds at all times |
| FR-STATE-03 | The state manager SHALL support 8 user-assignable bookmarks (store and recall IL/XL/Z triplets) |
| FR-STATE-04 | Step size SHALL apply to all increment/decrement pad actions |
| FR-STATE-05 | State SHALL be serializable to JSON for session persistence |

### 6.4 Standalone Viewer Mode (FR-VIEW)

| ID | Requirement |
|----|-------------|
| FR-VIEW-01 | The viewer SHALL open a SEG-Y file via `segyio` and expose IL, XL, and Z slices |
| FR-VIEW-02 | The viewer SHALL display the current slice as a `matplotlib imshow` plot with correct aspect ratio |
| FR-VIEW-03 | Colormap SHALL update in real-time from MIDI input without closing the window |
| FR-VIEW-04 | Gain and clip SHALL apply as amplitude scaling to the displayed slice data |
| FR-VIEW-05 | The viewer SHALL display a heads-up overlay showing current IL/XL/Z, gain, clip, and active MIDI CC values |
| FR-VIEW-06 | Slice navigation SHALL redraw within 100 ms on a standard laptop for a typical 3D dataset (up to ~5 GB loaded by IL) |
| FR-VIEW-07 | The viewer SHALL support a three-panel layout: IL section, XL section, Z time-slice displayed simultaneously |
| FR-VIEW-08 | The snapshot button SHALL save the current figure to a timestamped PNG file |

### 6.5 Application Bridge Mode (FR-BRIDGE)

| ID | Requirement |
|----|-------------|
| FR-BRIDGE-01 | The bridge SHALL translate MIDI events into keystrokes using `pyautogui` directed at the focused application window[^7] |
| FR-BRIDGE-02 | The bridge SHALL support configuring the target application name (e.g., "Petrel", "OpendTect") for window-focus verification |
| FR-BRIDGE-03 | Keystroke sequences SHALL be configurable per action (e.g., IL forward = `PageDown`, or `F` key as in OpendTect[^2]) |
| FR-BRIDGE-04 | Continuous faders (CC) SHALL map to repeated keystroke bursts, with rate proportional to fader position from center (jog-wheel mode) |
| FR-BRIDGE-05 | The bridge SHALL NOT send keystrokes if the target application does not have focus, to prevent accidental input in other apps |
| FR-BRIDGE-06 | Mouse scroll simulation SHALL be supported as an alternative to key presses for applications that respond to scroll wheel navigation |

***

## 7. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| **Performance** | MIDI event-to-action latency SHALL be < 20 ms end-to-end (MIDI in → keystroke/redraw dispatched) |
| **Compatibility** | SHALL run on Windows 10/11, macOS 12+, and Linux (Ubuntu 22.04+); MIDI via class-compliant USB |
| **Dependencies** | Core deps: `mido`, `python-rtmidi`, `pyautogui`, `segyio`, `matplotlib`, `numpy`, `pyyaml` |
| **Portability** | Single Python file entry point with `requirements.txt`; no compiled extensions beyond `python-rtmidi` |
| **Safety** | Bridge mode SHALL include a configurable panic key (e.g., `Esc`) that immediately halts all output |
| **Reliability** | MIDI reconnection retry SHALL be attempted every 5 seconds if the port is lost |
| **Observability** | A live status panel in the terminal SHALL display the last received CC message, current state, and active mode |

***

## 8. Implementation Guide

### 8.1 Recommended Python Stack

```python
# Core dependencies
mido>=1.3.0          # MIDI I/O - clean Pythonic API, CC message handling
python-rtmidi>=1.5   # Low-level MIDI backend for mido on all platforms
pyautogui>=0.9       # OS keyboard/mouse simulation for bridge mode
segyio>=1.9          # SEG-Y file reading for standalone viewer mode
matplotlib>=3.7      # Interactive display with widget sliders
numpy>=1.24          # Array math for amplitude scaling
pyyaml>=6.0          # Config file parsing
rich>=13.0           # Optional: beautiful terminal status output
```

### 8.2 Project Structure

```
seijog/
├── seijog.py              # CLI entry point
├── config/
│   ├── default.yaml       # Default CC→action mapping
│   ├── akai_mpk_mini.yaml # Profile for Akai MPK Mini
│   └── behringer_bcf.yaml # Profile for Behringer BCF2000
├── seijog/
│   ├── __init__.py
│   ├── midi_input.py      # MIDI listener thread + event queue
│   ├── event_router.py    # Maps MIDI messages to action names
│   ├── state.py           # SeismicState dataclass + bookmarks
│   ├── actions.py         # Action handler registry
│   ├── viewer.py          # Mode A: matplotlib 3-panel viewer
│   ├── bridge.py          # Mode B: pyautogui keyboard emulator
│   └── utils.py           # CC value scaling, clamping, dead zones
├── requirements.txt
└── README.md
```

### 8.3 Core Code Patterns

**MIDI Input Thread (midi_input.py)**

```python
import mido
import queue
import threading

class MidiInputThread:
    def __init__(self, port_name: str, event_queue: queue.Queue):
        self.port_name = port_name
        self.queue = event_queue
        self._thread = threading.Thread(target=self._run, daemon=True)

    def start(self):
        self._thread.start()

    def _run(self):
        with mido.open_input(self.port_name) as port:
            for msg in port:             # blocks until message arrives
                self.queue.put(msg)      # thread-safe put
```

**CC Value Scaling (utils.py)**

```python
def scale_cc(cc_value: int, out_min: float, out_max: float,
             dead_zone: int = 2) -> float:
    """Map a 0–127 CC value to [out_min, out_max], with optional dead zone."""
    # Clamp center dead zone
    if abs(cc_value - 64) <= dead_zone:
        cc_value = 64
    return out_min + (cc_value / 127.0) * (out_max - out_min)
```

**Jog-Wheel Mode for Faders in Bridge Mode (bridge.py)**

```python
import pyautogui

JOG_CENTER = 64  # fader center position = no movement

def fader_to_jog_rate(cc_value: int) -> int:
    """Convert fader position to step rate. Center = 0, extremes = max."""
    deviation = cc_value - JOG_CENTER
    # Quadratic acceleration: small deviations = slow, large = fast
    return int((deviation / 64.0) ** 2 * 10 * (1 if deviation > 0 else -1))

def send_il_step(direction: int, key_forward: str = 'pagedown',
                 key_backward: str = 'pageup'):
    if direction > 0:
        pyautogui.press(key_forward)   #
    elif direction < 0:
        pyautogui.press(key_backward)
```

**Three-Panel Matplotlib Viewer (viewer.py, sketch)**

```python
import matplotlib.pyplot as plt
import matplotlib.gridspec as gridspec
import segyio
import numpy as np

class SeismicViewer:
    def __init__(self, segy_path: str):
        with segyio.open(segy_path, ignore_geometry=False) as f:
            self.data = segyio.tools.cube(f)       # (IL, XL, Z) array
            self.inlines = f.ilines
            self.xlines = f.xlines
            self.zrange = np.arange(f.samples.size)

        self.fig = plt.figure(figsize=(16, 8))
        gs = gridspec.GridSpec(1, 3, figure=self.fig)
        self.ax_il = self.fig.add_subplot(gs)   # Inline section
        self.ax_xl = self.fig.add_subplot(gs[^1])   # Crossline section
        self.ax_z  = self.fig.add_subplot(gs[^2])   # Time slice

    def update(self, state):
        """Redraw all panels from current SeismicState."""
        il_idx = np.searchsorted(self.inlines, state.il)
        xl_idx = np.searchsorted(self.xlines,  state.xl)
        z_idx  = int(np.clip(state.z, 0, self.zrange.size - 1))

        vmax = np.nanpercentile(np.abs(self.data), state.clip * 100) * state.gain

        self.ax_il.imshow(self.data[il_idx, :, :].T, aspect='auto',
                          cmap=state.colormap, vmin=-vmax, vmax=vmax)
        self.ax_xl.imshow(self.data[:, xl_idx, :].T, aspect='auto',
                          cmap=state.colormap, vmin=-vmax, vmax=vmax)
        self.ax_z.imshow(self.data[:, :, z_idx].T, aspect='auto',
                         cmap=state.colormap, vmin=-vmax, vmax=vmax)

        self.fig.canvas.draw_idle()   # non-blocking redraw
```

### 8.4 Config File Schema (YAML)

```yaml
# seijog/config/default.yaml
mode: standalone           # standalone | bridge
segy_file: /path/to/data.segy
midi_port: "MPK mini 3"   # partial match supported

survey:
  il_min: 100
  il_max: 800
  xl_min: 1000
  xl_max: 1500
  z_min: 0
  z_max: 3000

defaults:
  gain: 1.0
  clip: 0.98
  colormap: RdBu_r
  il_step: 1
  xl_step: 1
  z_step: 4

dead_zone: 2              # CC jitter suppression (±units)

mappings:
  control_change:
    0:  { action: set_inline,    scale: [100, 800] }
    1:  { action: set_crossline, scale: [1000, 1500] }
    2:  { action: set_z,         scale: [0, 3000] }
    3:  { action: set_gain,      scale: [0.1, 10.0] }
    4:  { action: set_clip,      scale: [0.5, 1.0] }
    5:  { action: set_bias,      scale: [-1.0, 1.0] }

  note_on:
    36: { action: step_il_forward  }
    37: { action: step_il_backward }
    38: { action: step_xl_forward  }
    39: { action: step_xl_backward }
    40: { action: step_z_forward   }
    41: { action: step_z_backward  }
    42: { action: cycle_colormap   }
    43: { action: snapshot         }
    44: { action: recall_bookmark, bookmark: 0 }
    45: { action: recall_bookmark, bookmark: 1 }
    48: { action: store_bookmark,  bookmark: 0 }
    49: { action: store_bookmark,  bookmark: 1 }

# Bridge mode keystroke map (Mode B)
bridge:
  target_app: "OpendTect"
  keymap:
    step_il_forward:   "f"     # OpendTect F key
    step_il_backward:  "b"     # OpendTect B key
    step_xl_forward:   "right"
    step_xl_backward:  "left"
    step_z_forward:    "down"
    step_z_backward:   "up"
```

***

## 9. Development Milestones

| Milestone | Description | Effort Estimate |
|-----------|-------------|----------------|
| **M0 — MIDI Hello World** | List ports, print CC messages to terminal, confirm controller is responsive | 1–2 hours |
| **M1 — Config + Routing** | Load YAML config, route CC/note to named actions, unit test scaling | 3–4 hours |
| **M2 — Standalone Viewer v0** | Open SEG-Y with segyio, display single IL section in matplotlib, update on MIDI CC | 4–6 hours |
| **M3 — Three-Panel Viewer** | IL + XL + Z panels, all three updated together from MIDI input | 4–6 hours |
| **M4 — Display Controls** | Gain, clip, colormap cycle, snapshot — all wired to MIDI | 2–3 hours |
| **M5 — Bookmarks** | Store/recall IL/XL/Z triplets on pad buttons; persist to JSON | 2 hours |
| **M6 — Bridge Mode v0** | pyautogui keystroke emulation, target-app focus check, OpendTect test | 3–4 hours |
| **M7 — Jog-Wheel Mode** | Fader deviation → burst keystroke rate for smooth scrolling | 2–3 hours |
| **M8 — Polish** | Rich terminal status panel, error handling, panic key, README | 3–4 hours |

**Total estimated effort for a proficient Python developer: ~25–35 hours** across a few weekends.

***

## 10. Known Limitations and Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **SEG-Y file size** | Large volumes (>10 GB) cannot be fully loaded into memory with `segyio.tools.cube()` | Use `segyio` trace-by-trace or memmap access; cache only the ±N inlines around current position |
| **CC jitter** | Physical faders/knobs generate noisy values that cause display flickering | Dead zone filter (±2 CC units) + debounce timer (50 ms hysteresis) |
| **pyautogui focus** | Keystrokes sent to wrong window if user switches apps | Check target window title before each key press; abort if focus lost[^7] |
| **macOS accessibility** | pyautogui requires accessibility permissions on macOS | Document setup steps; consider `pynput` as alternative |
| **Matplotlib redraw speed** | Large Z slices may be slow to render | Pre-index with numpy; use `set_data()` on existing `AxesImage` instead of re-`imshow()` |
| **MIDI CC resolution** | 128 steps across a 1000-inline survey = ~8 IL per step | Use pad buttons for fine ±1 stepping; fader for coarse position |
| **Controller variety** | CC numbers differ between controllers | YAML config + detection mode logs all CCs to help user identify their controller's numbers |

***

## 11. Future Extensions (Out of Scope for v0.1)

- **OSC (Open Sound Control) support**: Higher resolution than MIDI (float32), useful for precision gain control
- **Velocity-sensitive navigation**: Harder pad hit = larger step size
- **segyio-based attribute computation**: On-the-fly envelope, instantaneous phase, computed and displayed live
- **OpendTect Python plugin**: Instead of key emulation, use OpendTect's Python scripting API for direct state control (eliminates focus dependency)
- **Web-based HUD**: Flask/FastAPI server + browser overlay displaying current state on a second monitor
- **Multi-controller support**: Different controllers for different axes (e.g., X-Y pad for IL/XL, separate faders for display)
- **Record and playback**: Capture a MIDI navigation session and replay it for reproducible exploration paths

***

## Appendix A: Reference — Seismic Software Keyboard Navigation

| Software | IL Forward | IL Backward | XL Forward | XL Backward | Z Forward | Z Backward |
|----------|-----------|------------|-----------|------------|---------|----------|
| **OpendTect** | `F` (user-defined) | `B` (user-defined) | `F` | `B` | `F` | `B` [^2] |
| **Petrel** | `→` Arrow | `←` Arrow | `↑` Arrow | `↓` Arrow | — | — [^1][^14] |
| **Kingdom** | `Page Down` | `Page Up` | `Page Down` | `Page Up` | — | — |
| **DUG Insight** | Scroll wheel | Scroll wheel | Scroll wheel | Scroll wheel | Scroll wheel | Scroll wheel [^12] |

> Note: OpendTect allows users to define their own `F`/`B` shortcut keys for moving slices forward and backward[^2]. This is the cleanest integration point for bridge mode.

***

## Appendix B: Recommended MIDI Controllers for This Project

| Controller | Faders | Knobs | Pads | Street Price | Notes |
|-----------|--------|-------|------|-------------|-------|
| Akai MPK Mini MkIII | 0 | 8 | 16 | ~$100 | Highly portable; pads and knobs only |
| Behringer BCF2000 | 8 motorized | 8 | 8 | ~$150 | Best fit: motorized faders provide visual feedback on position |
| Novation Launch Control XL | 3×8 faders | 3×8 knobs | 0 | ~$130 | Maximum continuous control surface |
| Akai APC40 MkII | 8 | 8 | 40 pads | ~$200 | Mix of pads and faders; good for bookmarks |
| Arturia BeatStep | 0 | 16 | 16 | ~$100 | Budget option; all rotary encoders |

**Recommendation for SeisJog:** The **Behringer BCF2000** is the ideal match — its 8 motorized faders mean the physical fader position resets to match the software state when switching volumes or loading sessions, eliminating position confusion.

---

## References

1. [Petrel Shortcut Keys Overview | PDF - Scribd](https://www.scribd.com/document/282244208/Shortcut-Keys-in-Petrel) - In Petrel, arrow keys move selected objects, Ctrl-S saves, and F1 opens the manual. Keys also contro...

2. [Keyboard Shortcuts To Move IL-XL-Z-slices Forward Backward in ...](https://www.youtube.com/watch?v=EafeBoJnQ6Y) - Displayed in OpendTect 4.6. The active inline / crossline / z ... Keyboard Shortcuts To Move IL-XL-Z...

3. [Assigning FX Parameters to your MIDI controller's Faders/Knobs](https://support.akaipro.com/en/support/solutions/articles/69000854827-akai-pro-mpc-beats-assigning-fx-parameters-to-your-midi-controller-s-faders-knobs) - This article will walk through the process of assigning FX parameters to MIDI controller faders and ...

4. [What are the MPK mini CC and PROG CHANGE buttons? - Reddit](https://www.reddit.com/r/makinghiphop/comments/2p6igo/what_are_the_mpk_mini_cc_and_prog_change_buttons/) - Bank numbers and program numbers are both specified using CC messages, and the Program Select messag...

5. [Messages — Mido 1.3.2 documentation](https://mido.readthedocs.io/en/stable/messages/index.html) - A Mido message is a Python object with methods and attributes. The attributes will vary depending on...

6. [How do you get midi events using python-rtmidi - Stack Overflow](https://stackoverflow.com/questions/61814607/how-do-you-get-midi-events-using-python-rtmidi) - My first goal is to get the midi events then write a program that calculates whether or not the even...

7. [PyAutoGUI](https://pypi.org/project/PyAutoGUI/) - PyAutoGUI lets Python control the mouse and keyboard, and other GUI automation tasks. For Windows, m...

8. [Keyboard Control Functions¶](https://pyautogui.readthedocs.io/en/latest/keyboard.html)

9. [segyio | Skills Marketplace - LobeHub](https://lobehub.com/skills/steadfastasart-geoscience-skills-segyio) - Read, write, and manipulate SEG-Y seismic data files. Fast C library with Python bindings for trace,...

10. [Open and create — segyio 2.0.0-alpha.1 documentation](https://segyio.readthedocs.io/en/latest/segyio.html) - Opens a segy file and tries to figure out its sorting, inline numbers, crossline numbers, and offset...

11. [Inline/Crossline - Seequent Product Help](https://help.seequent.com/GMSYS/2024.1/Content/ss/gmsys_2d/utilities/SEG-Y/p/inline_crossline.htm) - Inline/Crossline. 3D seismic surveys have internal inline and crossline coordinates, which specify l...

12. [Using the IL/CL View - DUG Insight User Manual](https://help.dugeo.com/m/Insight/l/732040-using-the-il-cl-view) - The IL/CL View allows you to view the trace information for a selected inline (IL) and crossline (CL...

13. [2.6 2D Viewer](https://doc.opendtect.org/6.6.0/doc/od_userdoc/content/getting_started/viewer_2d.htm) - For 2D seismic lines you can open a crossing 2D line using right-click on the annotated cross-line. ...

14. [Petrel keyboard shortcuts - DefKey](https://defkey.com/petrel-reservoir-software-shortcuts) - Seismic interpretation (43 shortcuts) ; 0. Z · Zoom (Interpretation window) ; 0. H · Start Horizon I...

15. [Seismic Gains Tab](http://help.divestco.com/WinPICS/5.9/Content/Topics/SeismicWindow/DisplayOptions/Seismic_Gains_Tab.htm) - Wiggle gain can be adjusted for excursion or clip. Color gain can be adjusted for gain or bias. Adju...

16. [Seismic Reflection Interpretation: 1-7 Seismic Data Display - YouTube](https://www.youtube.com/watch?v=CTryqnKlx5I) - This lecture discuss a few essential techniques for effectively displaying and navigating through se...

17. [[PDF] Composite Density Displays - CREWES](https://www.crewes.org/Documents/ResearchReports/2003/2003-56.pdf) - The typical modern computer-generated seismic display is some combination of colour/greyscale variab...

18. [2.1 System Overview](https://doc.opendtect.org/7.0.0/doc/od_userdoc/content/getting_started/system_overview.htm?TocPath=2+Getting+Started%7C2.1+System+Overview%7C_____0) - OpendTect v7.0 is a project-oriented seismic interpretation system. Projects are organized in Survey...

19. [How to force a MIDI device to report control status? - Stack Overflow](https://stackoverflow.com/questions/56093682/how-to-force-a-midi-device-to-report-control-status) - I'm using python-rtmidi to read a MIDI device with sliders and knobs. ... How do I send midi Control...

20. [How would you map 27 buttons, knobs and faders on a midi ...](https://vi-control.net/community/threads/how-would-you-map-27-buttons-knobs-and-faders-on-a-midi-controller.102555/) - It has 9 faders, 9 buttons and 9 knobs that I can map. These are grouped into 3 control banks that y...

21. [How can I integrate Python mido and asyncio?](https://stackoverflow.com/questions/56277440/how-can-i-integrate-python-mido-and-asyncio) - I have a device which does file I/O over MIDI. I have a script using Mido that downloads files but i...

22. [Is there a way to wrap pygame.midi.Input.read() in an asynchronous ...](https://stackoverflow.com/questions/75131490/is-there-a-way-to-wrap-pygame-midi-input-read-in-an-asynchronous-task-without) - Another possible choice is to utilize a library that furnishes an asynchronous interface for MIDI in...

23. [Slider — Matplotlib 3.10.8 documentation](https://matplotlib.org/stable/gallery/widgets/slider_demo.html) - Sliders are used to control the frequency and amplitude of a sine wave. See Snap sliders to discrete...

24. [Matplotlib - Slider Widget - GeeksforGeeks](https://www.geeksforgeeks.org/python/matplotlib-slider-widget/) - Matplotlib provides several widgets to make interactive plots. Among these widgets, the Slider widge...

