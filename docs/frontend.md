# Frontend — `src/App.jsx`

All UI lives in a single file (~411 lines). It is organized as utility functions at the top followed by four Solid.js components.

## Utility Functions

### `draw_time(time)` (line 7)
Converts a raw microsecond value to a human-readable millisecond string.
```js
draw_time(1500) // → "2 ms"  (rounds to nearest ms)
```
Used everywhere a duration is displayed (stats table, bottom scroll bar).

---

### `getMeanAndVar(arr)` (line 11)
Returns `{ average, std_deviation }` for a non-empty number array. `std_deviation` is the population standard deviation (not sample).

---

### `getStats(duration_array)` (line 34)
Returns `{ median, min, max, average, std_deviation, samples }`.
- Returns all zeros if the array is empty (safe to call before any strafes are recorded).
- Sorts a copy of the input to find median/min/max without mutating state.
- Delegates mean + std deviation to `getMeanAndVar`.

---

### `getOccurance(duration_array)` (line 51)
Buckets an array of microsecond durations into a 41-element frequency array for the histogram.
- Each bucket covers 5,000 µs (5 ms).
- Bucket index = `Math.ceil(x / 5000)`, so index 0 = ~0ms, index 1 = 1–5ms, …, index 40 = 196–200ms.
- Returns `[0]` if the array is empty (Chart.js needs at least one data point).

---

## Components

### `MyChart` (line 66)

**Responsibility:** Renders a stacked bar histogram showing the distribution of strafe timings.

**Props:** `earlyStrafes`, `lateStrafes`, `perfectStrafes` (arrays of µs durations from the parent)

**Signals:**
- `chartData` — Chart.js data object, updated via `createEffect` whenever props change

**Chart details:**
- Type: stacked `Bar` (via `solid-chartjs`)
- X-axis labels: `[0, 5, 10, …, 200]` (ms) — 41 values, fixed at mount
- Datasets:
  - **Early** — `getOccurance(earlyStrafes)`, color `#a5c5ae` (secondary)
  - **Late** — `getOccurance(lateStrafes)`, color `#8cb5a8` (accent)
  - **Perfect** — `[perfectStrafes.length]` (single count at x=0), color `#b5ac8c`
- Chart.js modules registered once via `onMount`

> Note: Perfect strafes have duration 0, so they always land in bucket 0. The chart renders them as a single-bar count rather than bucketing them across the axis.

---

### `Stats` (line 138)

**Responsibility:** Displays a statistics table with columns for All / Early / Late strafes, plus a Perfect count below.

**Props:** `earlyStrafes`, `lateStrafes`, `perfectStrafes`

**Signals:**
- `stats` — `{ alls, early, late }` each a `getStats()` result object; `{ equals: false }` forces reactivity on object mutation
- `perfectCount` — integer count of perfect strafes

**Rows displayed:** Median, Average, Min, Max, Std. Deviation, Samples

**Update trigger:** `createEffect` watches all three prop arrays; recomputes on any change. `alls` is computed over the concatenation of all three arrays.

---

### `WASD` (line 213)

**Responsibility:** Shows A and D key visuals that animate on press/release. Also provides three demo buttons (hidden until hover) that simulate strafe types.

**Signals:**
- `aPressed` / `dPressed` — booleans that drive CSS class changes on the key visuals (translate down + teal tint when pressed)

**Tauri event listeners** (set up in `createEffect` with `onCleanup`):

| Event | Effect |
|-------|--------|
| `a-pressed` | `setAPressed(true)` |
| `a-released` | `setAPressed(false)` |
| `d-pressed` | `setDPressed(true)` |
| `d-released` | `setDPressed(false)` |

**Simulation functions** (demo only — manipulate local signals, do not interact with Rust):

| Function | Sequence |
|----------|----------|
| `simulateEarly()` | A down @ 0ms → A up @ 500ms → D down @ 850ms → D up @ 1350ms (350ms gap) |
| `simulateLate()` | A down @ 0ms → D down @ 500ms → A up @ 850ms → D up @ 1350ms (350ms overlap) |
| `simulatePerfect()` | A down @ 0ms → D down @ 500ms + A up @ 500ms (simultaneous) → D up @ 1000ms |

**Layout:** Three-column flex row. The demo buttons are the left column (opacity 0, slides in on parent `group-hover`). The key visuals are the center column. The right column is an empty spacer to keep keys centered.

---

### `App` (line 324) — Root Component

**Responsibility:** Owns all strafe state, subscribes to the `strafe` Tauri event, and lays out the four UI sections.

**Signals:**

| Signal | Type | Contents |
|--------|------|----------|
| `totalStrafes` | `{ type, duration }[]` | All strafes in newest-first order |
| `earlyStrafes` | `number[]` | µs durations of Early strafes, newest-first |
| `lateStrafes` | `number[]` | µs durations of Late strafes, newest-first |
| `perfectStrafes` | `number[]` | Durations (always 0) of Perfect strafes, newest-first |

**`resetStrafes()`:** Sets all four signals to `[]`.

**`strafe` event listener** (in `createEffect`):
Receives `Payload { strafe_type, duration }`, prepends `duration` to the matching typed array, and prepends `{ type, duration }` to `totalStrafes`.

**Layout sections:**
1. **Header** (section 1) — Title text; `PatrikZero's` in styled italic, `Strafe Evaluation` in normal weight
2. **Stats + Chart** (section 2) — Two equal-width columns; Stats card left (includes Reset button), MyChart right
3. **WASD display** (section 3) — 128px tall row, horizontally centered
4. **Bottom scroll bar** (section 4) — Horizontal overflow scroll of `totalStrafes`; each card shows type + `draw_time(duration)`
