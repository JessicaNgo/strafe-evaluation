# Architecture

## System Diagram

```
┌─────────────────────────────────────────────────────┐
│  FRONTEND  (Solid.js + Vite, served by Tauri WebView)│
│                                                      │
│  App (root)                                          │
│  ├── Stats         — statistics table               │
│  ├── MyChart       — histogram                      │
│  ├── WASD          — key display + simulations      │
│  └── bottom bar    — recent strafes scroll list     │
└────────────────────────┬────────────────────────────┘
                         │  Tauri IPC  (app.emit_all / listen)
┌────────────────────────┴────────────────────────────┐
│  BACKEND  (Rust + Tauri)                             │
│                                                      │
│  main()                                              │
│  ├── 1ms polling loop                                │
│  ├── inputbot   — global keyboard state              │
│  ├── eval_understrafe()  — gap analysis              │
│  └── eval_overstrafe()   — overlap analysis          │
└─────────────────────────────────────────────────────┘
```

## Tech Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Desktop shell | Tauri 1.x | Lightweight alternative to Electron; Rust backend |
| Backend language | Rust | High-performance, sub-millisecond input polling |
| Input capture | inputbot 0.6 | Global keyboard hooks without focus requirement |
| Windows API | winapi 0.3.9 | Keyboard layout detection (AZERTY check) |
| Serialization | serde + serde_json | Tauri IPC payload serialization |
| Frontend framework | Solid.js 1.x | Fine-grained reactivity, minimal re-renders |
| Charting | Chart.js 4 + solid-chartjs | Stacked bar histogram |
| Styling | Tailwind CSS | Utility-first, consistent color palette |
| Bundler | Vite | Fast dev server on :1420 (required by Tauri) |

## Directory Structure

```
strafe-evaluation/
├── index.html              # Vite HTML entry point
├── vite.config.js          # Vite config (port 1420, Solid plugin)
├── tailwind.config.js      # Custom color tokens
├── postcss.config.js       # Tailwind + autoprefixer
├── package.json            # Frontend deps + npm scripts
│
├── src/
│   ├── index.jsx           # Solid.js render root
│   ├── App.jsx             # All UI components (~410 lines)
│   └── App.css             # Tailwind imports + custom classes
│
└── src-tauri/
    ├── Cargo.toml          # Rust deps
    ├── tauri.conf.json     # App name, window size, build paths
    ├── build.rs            # Tauri build hook
    └── src/
        └── main.rs         # Entire Rust backend (~177 lines)
```

## Timing Thresholds

These values are hard-coded in `src-tauri/src/main.rs` and drive all strafe classification.

| Strafe Type | Condition | Duration |
|-------------|-----------|----------|
| **Perfect** | Gap between key releases ≤ threshold | < 1,600 µs (1.6 ms) |
| **Early** | Gap between keys is noticeable but short | 1,600 – 200,000 µs (1.6 – 200 ms) |
| **Late** | Both keys held simultaneously | 0 – 200,000 µs (0 – 200 ms) |
| **Ignored** | Both keys held too long | > 200,000 µs (> 200 ms) |

For **understrafe** (Early/Perfect): the gap is measured from when one key is released to when the other is pressed.
For **overstrafe** (Late): the overlap is measured from the moment both keys are simultaneously held to when one is released.

## Component Interaction

| Component | Responsibility | Receives | Emits/Mutates |
|-----------|---------------|----------|---------------|
| `main.rs` | Capture input, classify strafes | System keyboard | Tauri events: `a-pressed`, `a-released`, `d-pressed`, `d-released`, `strafe` |
| `App` (root) | Owns all strafe state, routes events | `strafe` Tauri event | `earlyStrafes`, `lateStrafes`, `perfectStrafes`, `totalStrafes` signals |
| `Stats` | Displays statistics table | `earlyStrafes`, `lateStrafes`, `perfectStrafes` props | Rendered table |
| `MyChart` | Renders histogram | Same as Stats | Canvas chart via Chart.js |
| `WASD` | Shows live key state; demo animations | `a/d-pressed/released` Tauri events | Visual key highlight |

## Custom Tailwind Colors

Defined in `tailwind.config.js`:

| Token | Hex | Used For |
|-------|-----|----------|
| `primary` | `#88a56f` | Reset button |
| `secondary` | `#a5c5ae` | Stats card background, Early strafe color |
| `accent` | `#8cb5a8` | Late strafe color, bottom bar tint |
| `dark` | `#3a3f36` | Default text color |
| `bright` | `#f4f4f4` | Page background, title text |
