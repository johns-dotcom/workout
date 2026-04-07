# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Phase 1 Training Tracker — a single-file PWA for tracking a structured 4-week workout program ("Boom Records — Phase 1 Training"). Tracks session completion, exercise performance (weight/reps/cardio), substitutions, PRI (Postural Restoration Institute) protocols, milestones, and progress charts. Fully offline-capable via service workers and localStorage.

## Repository Structure

The repo contains **6 parallel versions** of the same app in separate directories, each an iteration on UI/theme:

- **files/** — v1, dark theme (#0d1117), desktop sidebar layout, service worker `phase1-v1`
- **files 2/** — v2, light theme (#f5f0e8), mobile bottom-tab layout, no service worker
- **files 3/** — v2, light theme, adds service worker `phase1-v2`
- **files 4/** through **files 6/** — incremental iterations (v3–v5 service workers), light theme, mobile layout

All versions share the same JS logic and data model. **The latest/most complete version is `files 6/`** — default to editing this version unless told otherwise.

## Architecture

Each version is a **single `index.html`** (~2000 lines in v6) containing all HTML, CSS, and JS inline. No build tools, no frameworks, no dependencies beyond Google Fonts (Syne, DM Mono, Lora). The only separate file is `sw.js` (service worker).

### Key Data Structures (in-memory JS objects)

- **`CYCLE`** — Array of session objects defining the full training cycle. Each session has `blocks` (A/B/C/D) containing exercise arrays with properties: `id`, `n` (name), `p` (prescription), `c` (cues), `ds` (sets), `dr` (reps), `rest`, `ramp`, `cardio`.
- **`SUBS`** — Substitution exercises, filterable by availability (home, machine taken, outdoor).
- **`MILESTONES`** — Phase milestone definitions (Block 1–3, Exit).
- **`STATE`** — Global state persisted to localStorage under key `boom_p1_v5`. Shape: `{logs[], milestones{}, week, priChecked{}, skillLogs[], ptNotes[]}`. Loaded via `loadState()`, written via `saveState()`.

### Render Flow

The app uses imperative DOM construction — no templates or virtual DOM. The main `render()` function dispatches to panel-specific renderers based on a global `currentPanel` variable:

- `renderList()` — session list (home tab)
- `renderSession()` — active workout session with exercise blocks
- `renderPT()` — PRI/physical therapy protocols
- `renderSkills()` — skill tracking
- `renderHistory()` — session history log
- `renderProgress()` — Canvas-based charts via `drawChart()`
- `renderSubsView()` — exercise substitutions

DOM elements are built with helper functions: `el()`, `div()`, `span()`, `btn()`, `mk()` (~line 1266).

### UI Pattern

- Panel system: `.panel` divs toggled via `.show` class
- Tab navigation (sidebar in v1, bottom tabs in v2+) rendered by `renderTabBar()`
- Deload logic: every 6th week auto-reduces weights to 55% via `isDeloadWeek()` / `deloadWeight()`
- Rest timer overlay with start/pause/reset
- Muscle SVG overlays for exercise visualization

### Service Worker Strategy

Cache-first with network fallback (`sw.js`). Cache name in v6: `phase1-v8`. Pre-caches `index.html`, `manifest.json`, and Google Fonts. Old caches are cleaned on activate. Falls back to cached `index.html` when offline.

### CSS Conventions

Abbreviated class names: `.sb` (sidebar), `.wl` (week list), `.sl` (session list item), `.ph` (phase header), `.tab-*` (tabs), `.b-*` (badge colors). Colors use CSS custom properties (`--bg`, `--t`, `--gold`, `--gd`, etc.).

## Development

No build step. Open any version's `index.html` in a browser. For service worker testing, serve over HTTPS or localhost:

```bash
cd "files 6" && python3 -m http.server 8000
```

**When modifying, bump two version strings to force cache refresh:**
1. The `CACHE` constant in `sw.js` (e.g., `phase1-v8` → `phase1-v9`)
2. The localStorage key `SK` in `index.html` if the STATE shape changes (to avoid deserializing incompatible data)

**Caution:** Changing the `SK` localStorage key will reset all user data. Only bump it when the STATE schema is incompatible with the previous version.
