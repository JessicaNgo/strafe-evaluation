# Backend — `src-tauri/src/main.rs`

The entire Rust backend lives in a single file (~177 lines). It runs a 1ms polling loop that detects A/D key state transitions, measures timing gaps and overlaps, classifies each strafe, and emits IPC events to the frontend.

## Data Structures

### `Payload` (line 11)
Serialized over Tauri IPC as JSON for every classified strafe.

```rust
struct Payload {
    strafe_type: String,  // "Early" | "Late" | "Perfect"
    duration: u128,       // microseconds; always 0 for Perfect
}
```

## Functions

### `eval_understrafe(elapsed, released_time, app)` (line 16)

**When called:** When a new key is pressed and the other key was recently released (understrafe — there was a gap between the two keys).

**Input:** `elapsed` = time since the other key was released.

**Logic:**
- `elapsed < 1,600 µs` → emit `strafe` with type `"Perfect"`, duration `0`
- `1,600 µs ≤ elapsed < 200,000 µs` → emit `strafe` with type `"Early"`, duration in µs
- `elapsed ≥ 200,000 µs` → no event (gap was too long to be a real strafe attempt)

**Side effect:** Resets `*released_time` to `None` so the same release isn't evaluated twice.

---

### `eval_overstrafe(elapsed, both_pressed_time, app)` (line 43)

**When called:** When one key is released after both were held simultaneously (overstrafe — keys overlapped).

**Input:** `elapsed` = duration both keys were held at the same time.

**Logic:**
- `elapsed < 200,000 µs` → emit `strafe` with type `"Late"`, duration in µs
- `elapsed ≥ 200,000 µs` → no event (overlap was too long, ignored)

**Side effect:** Resets `*both_pressed_time` to `None`.

---

### `is_azerty_layout()` (line 62)

**Purpose:** AZERTY keyboards (French/Belgian) use Q where QWERTY uses A. This function detects the active Windows keyboard layout so the loop can check the correct key.

**Implementation:** Calls `GetKeyboardLayout(0)` (Windows API), masks to the low 16 bits (the language ID), and returns `true` for layout IDs `0x040C` (FR), `0x080C` (BE), `0x140C`, `0x180C`.

---

## Main Event Loop (line 70)

Runs inside `tauri::async_runtime::spawn` so it doesn't block the Tauri UI thread.

### State Variables

| Variable | Type | Meaning |
|----------|------|---------|
| `left_pressed` | `bool` | Whether A/Q/Left is currently held |
| `right_pressed` | `bool` | Whether D/Right is currently held |
| `both_pressed_time` | `Option<SystemTime>` | Timestamp when both keys first became simultaneously held |
| `right_released_time` | `Option<SystemTime>` | Timestamp of the most recent D key release |
| `left_released_time` | `Option<SystemTime>` | Timestamp of the most recent A key release |
| `is_azerty` | `bool` | Whether the keyboard layout is AZERTY (checked once at startup) |

### Keys Monitored

| Physical key | Logical role | AZERTY note |
|-------------|--------------|-------------|
| A / Left Arrow | Strafe left | Q on AZERTY |
| D / Right Arrow | Strafe right | Same on all layouts |

### Loop Tick Order (per 1ms)

1. **D released** (line 87): `right_pressed && !DKey && !RightKey` → set `right_pressed = false`, emit `d-released`, record `right_released_time`
2. **A released** (line 93): `left_pressed && key-not-held` (handles AZERTY/QWERTY/Arrow) → set `left_pressed = false`, emit `a-released`, record `left_released_time`
3. **A pressed** (line 104): key newly held && `!left_pressed` → set `left_pressed = true`, emit `a-pressed`, if `right_released_time` is set → call `eval_understrafe`
4. **D pressed** (line 127): key newly held && `!right_pressed` → set `right_pressed = true`, emit `d-pressed`, if `left_released_time` is set → call `eval_understrafe`
5. **Overlap start** (line 147): both pressed & `both_pressed_time == None` → record `both_pressed_time`
6. **Overlap end** (line 151): one key released & `both_pressed_time != None` → call `eval_overstrafe`

> Releases are checked before presses each tick so that a same-tick release+press is ordered correctly.

## Tauri IPC Events Emitted

| Event name | Payload | When |
|-----------|---------|------|
| `a-pressed` | `()` | A/Q/Left newly pressed |
| `a-released` | `()` | A/Q/Left newly released |
| `d-pressed` | `()` | D/Right newly pressed |
| `d-released` | `()` | D/Right newly released |
| `strafe` | `Payload { strafe_type, duration }` | After every classified strafe |
