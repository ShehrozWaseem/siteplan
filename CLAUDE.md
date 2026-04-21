# DC Floor Plan Manager — PLDT

A single-file HTML application for visually mapping data center equipment onto a floor plan image. Used by PLDT (Philippine Long Distance Telephone) teams to document physical equipment placement in data center facilities.

## What It Does

Users load a floor plan image as a background, populate an equipment library (via CSV upload or manual entry), then drag-and-drop items from the sidebar library onto the canvas to represent physical equipment locations. The layout can be saved, restored, and exported.

## Key Features

### Equipment Library (Sidebar)
- **Manual Add** — Fill in name, type, make, model, HW code, and config fields, then click "+ Add to Library"
- **CSV Import** — Uploads a CSV with columns: `alias`, `hardware_type`, `hw_make`, `hw_model`, `hw_code`, `processor_id`, `configuration`; deduplicates by name automatically
- **Filter Bar** — Click equipment type pills to filter the library list; "All" clears filters
- **Search** — Real-time search across name, make, and model fields
- **Placement Badge** — Each library item shows a count badge of how many times it's been placed on the canvas

### Canvas
- **Drag & Drop** — Drag items from library to canvas; drop position maps to canvas coordinates accounting for zoom/pan
- **Click to Place** — Click a library item to place it at the center of the current view with slight random offset
- **Move Items** — Click-drag placed items to reposition them
- **Double-click Remove** — Double-click a placed item to remove it from the canvas
- **Tooltip** — Hover over placed items to see full equipment details (type, make, model, HW code, processor ID, config)

### Viewport Controls
- **Zoom** — Mouse wheel zoom centered on cursor; +/- buttons zoom around canvas center; range: 8%–600%
- **Pan Mode** — Toggle "Pan" button to enable click-drag canvas panning
- **Snap Mode** — Toggle "Snap" button to snap item positions to a 20px grid
- **Fit / Reset** — "Fit" button fits the floor plan image into the viewport

### Persistence
- **Auto-Save** — Every state change writes to `localStorage` key `fp_autosave_v2`; auto-restored on page load
- **Save State** — Downloads a `.json` file containing the full state including the floor plan image (encoded as base64 data URL if under 7 MB)
- **Load State** — Uploads a previously saved `.json` state file and restores everything
- **Floor Plan Image** — Upload any image file as the background; blob URLs are flagged as un-embeddable in state files if over size limit

### Export
- **JSON Export** — Downloads `layout_<timestamp>.json` containing placed item positions and metadata (name, type, make, model, hw_code, x, y)
- **Image Export** — Uses `html2canvas` to capture the canvas at 2x resolution as a PNG (toolbar and status pills are hidden during capture)

### Clear
- Confirmation modal before clearing all placed items from canvas; library and background image are preserved

## State Object (`S`)

```js
const S = {
  lib: [],        // Array of library items (equipment loaded from CSV or added manually)
  placed: [],     // Array of placed items (items currently on the canvas)
  zoom: 1,        // Current zoom scale factor (0.08 – 6)
  px: 0,          // Viewport X translation in pixels
  py: 0,          // Viewport Y translation in pixels
  snap: false,    // Whether grid snap is active
  snapG: 20,      // Grid snap size in pixels
  pan: false,     // Whether pan mode is active
  filters: Set,   // Set of currently active equipment type filter strings
  nid: 1,         // Auto-incrementing ID counter for lib and placed items
  bgW: 0,         // Background image natural width in pixels
  bgH: 0,         // Background image natural height in pixels
};
```

### Library Item Schema (`S.lib[]`)
```js
{
  id: Number,          // Unique numeric ID
  name: String,        // Equipment alias / display name
  type: String,        // Equipment type (see types below)
  make: String,        // Manufacturer
  model: String,       // Model name/number
  hw_code: String,     // Hardware code
  processor_id: String,// Processor ID (CSV only)
  config: String,      // Configuration notes
}
```

### Placed Item Schema (`S.placed[]`)
```js
{
  id: Number,          // Unique numeric ID (different from libId)
  libId: Number,       // Reference to the source library item's id
  name: String,        // Copied from library at placement time
  type: String,
  make: String,
  model: String,
  hw_code: String,
  processor_id: String,
  config: String,
  x: Number,           // Left position in canvas (viewport) pixels, unscaled
  y: Number,           // Top position in canvas (viewport) pixels, unscaled
}
```

### Save File Schema (v2)
```js
{
  v: 2,                // Format version
  ts: Number,          // Unix timestamp of save
  zoom: Number,
  px: Number,
  py: Number,
  snap: Boolean,
  nid: Number,
  lib: [],             // Full library array
  placed: [],          // Full placed array
  bgSrc: String,       // Base64 data URL of floor plan image, or "__blob__" / "__large__" sentinel
}
```

## Equipment Types & Colors

| Type           | Color      |
|----------------|------------|
| `energy_meter` | Blue       |
| `hvac`         | Green      |
| `battery`      | Purple     |
| `io`           | Orange     |
| `dc_plant`     | Red        |
| `ups`          | Light blue |
| `pdu`          | Bright green |
| `server`       | Amber      |
| `network`      | Sky blue   |
| `other`        | Gray       |

## Key Functions

| Function | Description |
|---|---|
| `addManual()` | Reads manual add form inputs, validates, pushes to `S.lib` |
| `parseCSV(txt)` | Parses raw CSV text into a 2D array, handling quoted fields |
| `renderLib()` | Re-renders the sidebar library list with current filters and search |
| `rebuildFilters()` | Rebuilds the filter pill bar from current unique types in `S.lib` |
| `placeItem(libId, px, py)` | Adds an item to `S.placed` and mounts its DOM element |
| `mountEl(item)` | Creates and attaches the `.pi` div for a placed item; wires drag-to-move and dblclick-remove |
| `removeItem(id)` | Fades out and removes a placed item from DOM and `S.placed` |
| `clearAll()` | Removes all placed items from canvas and `S.placed` |
| `applyT()` | Applies `S.zoom`, `S.px`, `S.py` as a CSS transform to the `#vp` element |
| `fitView()` | Calculates zoom and translation to fit the floor plan image in the viewport |
| `zAround(cx, cy, d)` | Zooms by delta `d` centered on canvas point `(cx, cy)` |
| `toggleSnap()` | Toggles `S.snap` and updates the snap button state |
| `togglePan()` | Toggles `S.pan` and updates cursor/button state |
| `buildSave()` | Assembles the save object from current state |
| `autoSave()` | Serializes state to `localStorage`; skips large background images |
| `saveState()` | Downloads the full state as a `.json` file, converting blob bg to data URL |
| `applyLoadedState(obj)` | Restores all state from a parsed save object (v2 format only) |
| `exportJSON()` | Downloads placed item layout as a minimal JSON file |
| `exportImage()` | Captures canvas with html2canvas and downloads as PNG |
| `snap(v)` | Rounds a coordinate to the nearest grid step if snap is enabled |
| `showTT(e, item)` | Populates and shows the hover tooltip for a placed item |
| `toast(msg)` | Shows a brief notification message at the bottom of the screen |
