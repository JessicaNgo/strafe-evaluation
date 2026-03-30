# Strafe Evaluation — Developer Documentation

A Tauri desktop app that captures A/D key presses and evaluates CS:GO strafe timing in real time, classifying each strafe as **Perfect**, **Early**, or **Late** and displaying live statistics and a histogram.

## Docs Index

| File | Contents |
|------|----------|
| [architecture.md](architecture.md) | System overview, tech stack, directory layout, timing thresholds, component interaction table |
| [backend.md](backend.md) | Rust backend: input capture loop, state machine, evaluation functions, Tauri IPC events |
| [frontend.md](frontend.md) | Solid.js frontend: components, signals, props, Tauri event listeners, utility functions |
| [data-flow.md](data-flow.md) | End-to-end flows for real keypresses, simulation buttons, and reset |

## Quick Reference

- **Rust entry point:** `src-tauri/src/main.rs`
- **Frontend entry point:** `src/App.jsx`
- **Dev:** `npm run tauri dev`
- **Build:** `npm run tauri build`
- **Window size:** 800×620px, non-resizable
- **Polling rate:** 1ms loop in Rust backend
