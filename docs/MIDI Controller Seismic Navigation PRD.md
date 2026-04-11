# SeisJog v0.1 — The "Radical Core" (PyVista Edition)

## 1. Problem Statement
Traditional seismic navigation via keyboard is a high-iteration, physically repetitive task that creates a cognitive bottleneck. Standard shortcuts interrupt interpretive flow and lack the tactile feedback necessary for fluid spatial exploration.

---

## 2. MVP Scope: "The 3D Sculptor"
The objective is a high-impact "wow factor" demo enabling physical interaction with a 3D seismic volume using entry-level MIDI hardware.

* **Standalone 3D Environment**: A single PyVista-based window rendering a pre-loaded 3D seismic volume as a NumPy array.
* **The Trinity Slices**: Three orthogonal slices (Inline, Crossline, and Time/Depth) that update dynamically.
* **Continuous Knob Scrubbing**: Mapping MIDI CC (knobs) to slice positions for smooth, cinematic scrolling.
* **Tactile "Ghosting" (Opacity Fader)**: A physical fader controlling global volume transparency, allowing reflectors to "glow" or fade.
* **Snap-to-View Pads**: MIDI pads mapped to preset camera angles (e.g., "Inline View," "Top-Down") for instant perspective shifts.

---

## 3. The Rationale
* **PyVista Integration**: Removes the overhead of proprietary APIs and ensures hardware-accelerated rendering.
* **Knobs vs. Keys**: Continuous rotation provides higher information density per movement than repetitive key-tapping, creating a "fluid" professional feel.
* **Volume Opacity**: Physical fader control transforms a solid cube into a navigable internal structure—a task that is unintuitive with a mouse.
* **Pre-Loaded Data**: Using a pre-indexed NumPy cube eliminates SEG-Y header errors during live demonstrations.

---

## 4. The Vicious Cut List (Out of Scope)
* **Application Bridge Mode**: No keyboard/mouse emulation for Petrel or OpendTect; focus is on the standalone "Hero App".
* **SEG-Y File Loading**: Real-time file parsing is shelved to ensure demo stability.
* **Session Bookmarks**: Saving/loading coordinate triplets is deferred to post-MVP.
* **Bidirectional MIDI**: No motorized fader feedback or MIDI lighting logic.

---

## 5. Hardware Mapping (Reference)

| Physical Control | MIDI Message | Action |
| :--- | :--- | :--- |
| **Knob 1** | CC 16 | **Inline Position**: Scrub the X-axis slice. |
| **Knob 2** | CC 17 | **Crossline Position**: Scrub the Y-axis slice. |
| **Knob 3** | CC 18 | **Time Slice Position**: Scrub the Z-axis slice. |
| **Fader 1** | CC 0 | **Global Opacity**: 0% (Hidden) to 100% (Solid). |
| **Pad 1** | Note 36 | **Camera Preset**: Snap to Inline View. |
| **Pad 2** | Note 37 | **Camera Preset**: Snap to Time-Slice View. |
| **Pad 5** | Note 40 | **Colormap**: Cycle through presets (RdBu, Greys, Viridis). |

---

## 6. Implementation Path (48-Hour Sprint)
1.  **M1**: Setup PyVista background plotter and load the NumPy cube.
2.  **M2**: Wire `mido` callback to slice position functions.
3.  **M3**: Implement the "Opacity Slider" logic using volume properties.
4.  **M4**: Map "Hero" camera angles and colormaps to pads.
