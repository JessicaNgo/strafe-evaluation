# Data Flows

## Flow 1: Real Keypress → Classification → UI Update

```
User presses D key (while A was previously held then released)
  │
  ▼
[Rust — main.rs loop tick]
  - DKey.is_pressed() = true, right_pressed = false
  - Sets right_pressed = true
  - Emits: "d-pressed" (no payload)
  - Reads left_released_time (set when A was released)
  - Calls eval_understrafe(elapsed, &mut left_released_time, handle)
    ├── elapsed < 1,600 µs  →  emit "strafe" { strafe_type: "Perfect", duration: 0 }
    ├── elapsed 1,600–200,000 µs  →  emit "strafe" { strafe_type: "Early", duration: µs }
    └── elapsed ≥ 200,000 µs  →  no event
  - Resets left_released_time = None
  │
  ▼
[Tauri IPC]
  - "d-pressed" event → WASD component
  - "strafe" event    → App root component
  │
  ▼
[Solid.js — WASD component]
  - setDPressed(true) → D key visual animates (translate down + teal tint)

[Solid.js — App component]
  - Receives { strafe_type: "Early", duration: 42000 }
  - setEarlyStrafes(a => [42000, ...a])
  - setTotalStrafes(a => [{ type: "Early", duration: 42000 }, ...a])
  │
  ▼
[Solid.js reactivity — Stats + MyChart]
  - Stats createEffect re-runs: recomputes getStats for earlyStrafes
  - MyChart createEffect re-runs: recomputes getOccurance, updates chartData signal
  - Chart.js re-renders histogram bar
  - Stats table cells re-render with new values
  - Bottom scroll bar prepends new card
```

---

## Flow 2: Late Strafe (Both Keys Overlapping)

```
User holds A, then presses D before releasing A
  │
  ▼
[Rust — loop tick when both pressed]
  - left_pressed = true, right_pressed becomes true
  - both_pressed_time = Some(SystemTime::now())   ← overlap starts
  │
  ▼
[Rust — loop tick when one key released]
  - e.g. A released: left_pressed = false
  - both_pressed_time is Some → call eval_overstrafe(elapsed, &mut both_pressed_time, handle)
    ├── elapsed < 200,000 µs  →  emit "strafe" { strafe_type: "Late", duration: µs }
    └── elapsed ≥ 200,000 µs  →  no event (ignored)
  - both_pressed_time = None
  │
  ▼ (same IPC + reactivity as Flow 1, but updating lateStrafes)
```

---

## Flow 3: Simulation Button Click

The demo buttons in `WASD` animate key visuals locally — **no Rust involvement, no strafe events emitted**.

```
User clicks "Early" button
  │
  ▼
[simulateEarly() — src/App.jsx line 253]
  - setAPressed(true)   immediately
  - setAPressed(false)  after 500ms
  - setDPressed(true)   after 850ms   ← 350ms gap
  - setDPressed(false)  after 1350ms
  │
  ▼
[Solid.js — WASD render]
  - A key visual animates pressed → released
  - 350ms later D key visual animates pressed → released
  (Stats and chart are NOT updated — no strafe data is generated)
```

Simulation timelines for all three types:

| Type | A down | D down | A up | D up | Gap/Overlap |
|------|--------|--------|------|------|-------------|
| Early | 0ms | 850ms | 500ms | 1350ms | 350ms gap |
| Late | 0ms | 500ms | 850ms | 1350ms | 350ms overlap |
| Perfect | 0ms | 500ms | 500ms | 1000ms | simultaneous release+press |

---

## Flow 4: Reset

```
User clicks "Reset" button (inside Stats card in App)
  │
  ▼
[resetStrafes() — src/App.jsx line 330]
  - setEarlyStrafes([])
  - setLateStrafes([])
  - setPerfectStrafes([])
  - setTotalStrafes([])
  │
  ▼
[Solid.js reactivity]
  - Stats: all getStats() calls return zeros
  - MyChart: getOccurance([]) returns [0] for each dataset — bars disappear
  - Bottom scroll bar: For loop renders nothing (empty array)
  (WASD key listeners remain active — Rust backend is unaffected)
```
