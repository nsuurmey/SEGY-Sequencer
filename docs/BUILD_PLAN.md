# SeisJog v0.1 — Build Plan

**Source PRD:** [MIDI Controller Seismic Navigation PRD](./MIDI%20Controller%20Seismic%20Navigation%20PRD.md)
**Status:** Greenfield — no implementation exists yet.
**Target Runtime:** Python 3.10+, desktop (Linux/macOS/Windows)

---

## 1. Architecture Decomposition

### 1.1 Core Components

| # | Module | Description |
|---|--------|-------------|
| 1 | **`seisjog.volume`** | Loads a pre-indexed NumPy `.npy` cube into memory. Owns the data array and exposes axis-length metadata (inline count, crossline count, sample count). |
| 2 | **`seisjog.renderer`** | Wraps a PyVista `BackgroundPlotter`. Creates the 3D volume actor, manages three orthogonal slice meshes, and exposes methods to update slice positions, opacity, colormap, and camera angle. |
| 3 | **`seisjog.midi_input`** | Opens a MIDI input port via `mido`, listens for CC and Note messages on a background thread, and dispatches them to registered callback functions. |
| 4 | **`seisjog.controller_map`** | Defines the mapping table between MIDI messages (CC numbers, Note numbers) and application actions. Reads from a static config dict (hardcoded for MVP; future: YAML/JSON file). |
| 5 | **`seisjog.app`** | Top-level orchestrator. Wires the controller map callbacks to the renderer methods, initializes all components, and runs the main event loop. |

### 1.2 Data Models / Schemas

| Model | Fields | Notes |
|-------|--------|-------|
| **SeismicVolume** | `data: np.ndarray` (3D), `n_inline: int`, `n_crossline: int`, `n_samples: int` | Immutable after load. Shape is `(n_inline, n_crossline, n_samples)`. |
| **SliceState** | `inline_idx: int`, `crossline_idx: int`, `time_idx: int` | Mutable. Clamped to volume bounds. Updated by MIDI callbacks. |
| **ViewState** | `opacity: float` (0.0–1.0), `colormap: str`, `camera_preset: str \| None` | Mutable. Opacity mapped from MIDI CC 0–127 → 0.0–1.0. |
| **MIDIMapping** | `message_type: str` ("cc" or "note"), `number: int`, `action: str` | Static lookup table. One entry per physical control. |

### 1.3 APIs / Integrations

- **No network APIs.** This is a standalone desktop application.
- **MIDI (USB/virtual):** Communication is one-directional: device → application. Uses the OS MIDI subsystem via `mido` + a backend (`python-rtmidi` or `portmidi`).
- **GPU:** PyVista delegates to VTK, which uses OpenGL for hardware-accelerated rendering. No direct GPU API calls from application code.

### 1.4 External Dependencies

| Dependency | Purpose | Version Constraint | Notes |
|------------|---------|--------------------|-------|
| `pyvista` | 3D rendering | >=0.43 | Requires VTK under the hood. |
| `numpy` | Seismic data array | >=1.24 | Peer dependency of PyVista. |
| `mido` | MIDI message parsing | >=1.3 | Lightweight, no binary deps. |
| `python-rtmidi` | MIDI I/O backend | >=1.5 | Compiles native extension; needs system MIDI headers on Linux (`libasound2-dev`). |
| `vtk` | Rendering engine | Pulled in by PyVista | Heavy install (~150 MB). Headless CI will require `vtk` with OSMesa or `xvfb`. |

**Assumption:** A sample NumPy cube (`.npy` file) will be created synthetically during development. No real SEG-Y data is needed for MVP.

---

## 2. Work Streams

### 2.1 Stream A — Data Layer

| # | Task | Acceptance Criteria | Complexity | Blockers |
|---|------|---------------------|------------|----------|
| A1 | Create synthetic 3D seismic volume generator | Running `python -m seisjog.sample_data` produces a `.npy` file with realistic-looking layered reflectors (e.g., sinusoidal horizons + noise). File is <50 MB. | Med | None |
| A2 | Implement `seisjog.volume` loader | `SeismicVolume.from_file(path)` returns a populated object. Raises clear error on missing file or wrong ndim. | Low | A1 (needs test data) |

### 2.2 Stream B — 3D Rendering

| # | Task | Acceptance Criteria | Complexity | Blockers |
|---|------|---------------------|------------|----------|
| B1 | Scaffold `BackgroundPlotter` window | Running the app opens a PyVista window with a black background and axis labels. Window can be closed cleanly. | Low | None |
| B2 | Render 3D volume from NumPy cube | Volume appears as a solid block with a default colormap. No interactivity yet. | Med | A2, B1 |
| B3 | Implement three orthogonal slice meshes | Three slice planes visible inside the volume. Each slice can be repositioned programmatically via `renderer.set_inline(idx)`, etc. Slice textures update correctly at boundaries (first index, last index). | High | B2 |
| B4 | Implement opacity control | `renderer.set_opacity(value)` adjusts volume transparency from 0% to 100%. Visual result: reflectors "glow through" at low opacity. | Med | B2 |
| B5 | Implement camera presets | `renderer.snap_to_view(name)` moves camera to one of at least 2 named angles: "inline" and "timeslice". Transition is instant (no animation for MVP). | Low | B1 |
| B6 | Implement colormap cycling | `renderer.cycle_colormap()` rotates through `['RdBu', 'Greys', 'viridis']` and re-applies to the volume and slices. | Low | B2 |

### 2.3 Stream C — MIDI Input

| # | Task | Acceptance Criteria | Complexity | Blockers |
|---|------|---------------------|------------|----------|
| C1 | Implement `midi_input` listener | Opens first available MIDI port (or a named port from env var). Logs incoming CC and Note messages to stdout. Runs on a background thread without blocking the renderer. Clean shutdown on app exit. | Med | None |
| C2 | Implement `controller_map` dispatch | Given a MIDI message, calls the correct registered callback. Unrecognized messages are silently ignored. Map matches the PRD hardware table (CC 16/17/18, CC 0, Notes 36/37/40). | Low | C1 |
| C3 | Scale CC values to domain ranges | CC 0–127 maps to slice index 0–(N-1) for knobs, and 0.0–1.0 for the opacity fader. Scaling is linear and clamped. | Low | A2 (needs volume dimensions), C2 |

### 2.4 Stream D — Integration & App Shell

| # | Task | Acceptance Criteria | Complexity | Blockers |
|---|------|---------------------|------------|----------|
| D1 | Wire MIDI callbacks to renderer methods | Turning a knob visually moves the corresponding slice in real-time. Fader changes opacity. Pads snap camera and cycle colormap. End-to-end test with hardware or a virtual MIDI loopback. | Med | B3, B4, B5, B6, C3 |
| D2 | CLI entry point | `python -m seisjog --data path/to/cube.npy` starts the app. `--port` flag optionally specifies a MIDI port. `--help` prints usage. | Low | D1 |
| D3 | Graceful shutdown & error handling | Closing the PyVista window stops the MIDI listener and exits cleanly (no zombie threads, no tracebacks on normal exit). Missing MIDI device prints a warning and enters keyboard-only fallback or exits gracefully. | Med | D1 |

### 2.5 Stream E — Project Scaffolding & DevOps

| # | Task | Acceptance Criteria | Complexity | Blockers |
|---|------|---------------------|------------|----------|
| E1 | Create project structure and `pyproject.toml` | Standard Python package layout (`src/seisjog/`, `tests/`). `pip install -e .` works. | Low | None |
| E2 | Add development tooling config | Linter (`ruff`), formatter (`ruff format`), type checker (`mypy`) configured. `make lint` or equivalent script passes on empty project. | Low | E1 |
| E3 | Write unit tests for volume loader and CC scaling | `pytest` suite with ≥5 tests covering: load valid cube, reject bad shape, CC-to-index mapping at boundaries (0, 64, 127). | Med | A2, C3 |
| E4 | GitHub Actions CI pipeline | On push: install deps, run lint, run tests. Uses `xvfb` for any PyVista tests that need a display. | Med | E1, E2, E3 |

---

## 3. Sequencing & Phases

### Phase 1 — Skeleton (MVP Foundation)

**Goal:** A window opens, a volume renders, and the project is installable.

| Order | Task(s) | Rationale |
|-------|---------|-----------|
| 1 | E1 | Everything else imports from the package structure. |
| 2 | A1, B1 (parallel) | Synthetic data and empty window are independent. |
| 3 | A2 | Volume loader needs the synthetic data from A1. |
| 4 | B2 | Volume rendering needs the loader (A2) and the plotter (B1). |
| 5 | E2 | Lock in linting/formatting before more code lands. |

**Exit Criteria:** `pip install -e .` works. Running the app displays a 3D seismic block in a PyVista window. No MIDI yet.

**Inter-phase dependency:** Phase 2 cannot begin MIDI wiring (D1) until B3, B4, B5, B6, and C3 are complete.

---

### Phase 2 — MIDI-Driven Interaction (Core Feature Set)

**Goal:** All seven MIDI controls from the PRD hardware table function end-to-end.

| Order | Task(s) | Rationale |
|-------|---------|-----------|
| 6 | B3, C1 (parallel) | Slice meshes and MIDI listener are independent of each other. |
| 7 | B4, B5, B6 (parallel) | Opacity, camera presets, colormap cycling are independent renderer features. |
| 8 | C2, C3 (sequential) | Dispatch map needs the listener; scaling needs the map and volume dimensions. |
| 9 | D1 | Full wiring — the first time hardware drives visuals. |
| 10 | D2, D3 (parallel) | CLI polish and shutdown handling are independent. |

**Exit Criteria:** A user can plug in a MIDI controller, run `python -m seisjog --data cube.npy`, and perform all actions listed in the PRD hardware table. Closing the window exits cleanly.

---

### Phase 3 — Quality & Stability (Polish)

**Goal:** The project is testable, linted, and CI-protected.

| Order | Task(s) | Rationale |
|-------|---------|-----------|
| 11 | E3 | Unit tests for the logic layers (volume, scaling). |
| 12 | E4 | CI pipeline to prevent regressions. |

**Exit Criteria:** `pytest` passes with ≥80% line coverage on `volume` and `controller_map` modules. CI is green on `main`.

---

### Dependency Graph (Summary)

```
E1 ──► A1 ──► A2 ──► B2 ──► B3 ──► D1 ──► D2
  │              │              │         │       ▲         │
  │              ▼              ▼         ▼       │         ▼
  ├──► B1 ──────►              B4        C3 ─────┘        D3
  │                            B5        ▲
  ├──► E2                      B6        │
  │                                  C1 ► C2
  ├──► E3 (after A2, C3)
  └──► E4 (after E3)
```

---

## 4. Role / Skill Requirements

### Roles

| Role | Owns | Count |
|------|------|-------|
| **Python Application Developer** | Streams A, C, D, E. Volume loader, MIDI layer, app wiring, project scaffolding, CI. | 1 |
| **3D Visualization Developer** | Stream B. PyVista rendering, slicing, opacity, camera logic. | 1 (can be same person if comfortable with VTK/PyVista) |

Given the small scope (48-hour sprint target in the PRD), a **single developer proficient in Python and PyVista** can own all streams sequentially. Two developers can parallelize Streams A+C and Stream B.

### Cross-Functional Handoff Points

1. **A2 → B2:** The volume loader's `SeismicVolume` object is the contract between data and rendering. Its interface (`.data`, `.n_inline`, etc.) must be agreed on before B2 starts.
2. **B3/B4/B5/B6 → D1:** The renderer's public method signatures (`set_inline`, `set_opacity`, `snap_to_view`, `cycle_colormap`) must be stable before MIDI wiring begins.
3. **C3 → D1:** The scaling functions must accept raw CC values (0–127) and return domain values. The output types (int index vs. float opacity) must match what the renderer expects.

---

## 5. Risk & Unknowns

### Technical Risks

| # | Risk | Severity | Mitigation |
|---|------|----------|------------|
| R1 | **PyVista `BackgroundPlotter` thread safety.** MIDI callbacks arrive on a background thread; PyVista rendering runs on the main thread. Directly calling renderer methods from the MIDI thread may crash VTK. | High | Use a thread-safe queue (e.g., `queue.Queue`) polled by a PyVista timer callback on the main thread. Prototype this in Phase 1 to validate. |
| R2 | **`python-rtmidi` installation on CI/headless.** Needs ALSA headers on Linux. May fail in minimal Docker images. | Med | Pin `libasound2-dev` in CI setup step. Test the install early (E1). |
| R3 | **PyVista slice update performance.** Updating three slice meshes at 30+ Hz (fast knob scrub) may cause flickering or lag if mesh recreation is expensive. | Med | Profile early. If slow, switch from removing/re-adding actors to updating mesh data in-place via `mesh.points` / `mesh.point_data`. |
| R4 | **No MIDI device available during development.** Developer may not always have hardware connected. | Low | Use a virtual MIDI loopback (`mido` supports virtual ports on macOS/Linux) or a script that programmatically sends test messages. Add a `--no-midi` flag for render-only testing. |

### Scope Creep Vectors

| Vector | Why It's Tempting | Why to Resist |
|--------|-------------------|---------------|
| SEG-Y file loading | "Just add `segyio` to read real data" | Parsing edge cases (byte-order, irregular traces) will derail the 48-hour sprint. Synthetic data is sufficient. |
| Animated camera transitions | "Smooth snap-to-view looks better" | Adds animation loop complexity. Instant snap is fine for MVP. |
| Configurable MIDI mappings (YAML/JSON) | "Makes it flexible" | Hardcoded map matches one controller and ships faster. Configurability is Phase 4+. |
| Multi-volume support | "Load two cubes side-by-side" | Doubles rendering complexity. One volume is the demo. |

### Open Questions (Need Clarification Before Building)

| # | Question | Who Decides | Impact If Unresolved |
|---|----------|-------------|---------------------|
| Q1 | What are the dimensions of the demo cube? (e.g., 200×200×400 samples) | Product owner / demo designer | Affects memory usage, render performance, and synthetic data generation. **Assumption for now: 200×200×400 float32 (~61 MB).** |
| Q2 | Which MIDI controller is the reference hardware? (e.g., Akai LPD8, Korg nanoKONTROL2) | Product owner | Affects whether CC numbers in the PRD match reality. **Assumption: PRD table is authoritative.** |
| Q3 | Is `BackgroundPlotter` or `Plotter` the right PyVista class? `BackgroundPlotter` allows non-blocking interaction but has threading caveats. | Developer (technical decision) | Affects R1 above. **Assumption: `BackgroundPlotter` for interactivity; fall back to timer-based `Plotter` if threading proves problematic.** |
| Q4 | Should the app exit or run in render-only mode when no MIDI device is detected? | Product owner | Affects D3 error handling. **Assumption: Print warning and run without MIDI so the demo still shows the visual.** |

---

## 6. Success Metrics

### Phase 1 — Skeleton

| Metric | Target |
|--------|--------|
| Package installable via `pip install -e .` | Yes/No |
| PyVista window opens and displays a colored 3D block | Yes/No |
| Synthetic `.npy` cube generates without error | Yes/No |

### Phase 2 — MIDI-Driven Interaction

| Metric | Target |
|--------|--------|
| All 7 hardware controls from PRD table produce correct visual response | 7/7 |
| Slice scrubbing latency (knob turn → visual update) | <100 ms perceived |
| Opacity fader range | Full 0%–100% visual transparency |
| App starts and runs for 5 minutes without crash | Yes/No |
| Clean exit on window close (no zombie threads, no traceback) | Yes/No |

### Phase 3 — Quality & Stability

| Metric | Target |
|--------|--------|
| Unit test pass rate | 100% |
| Line coverage on `volume` and `controller_map` | ≥80% |
| CI pipeline green on push to `main` | Yes/No |
| Lint / type-check passes | Zero errors |

### Overall "Done" Definition (from PRD)

> A non-technical stakeholder can plug in a USB MIDI controller, run a single command, and physically scrub through a 3D seismic volume — moving slices with knobs, fading opacity with a fader, and snapping camera views with pads — in a visually impressive, crash-free demo.
