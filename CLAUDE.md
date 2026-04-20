# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Phase 1 Training Tracker — a single-file PWA for tracking a structured 4-week workout program ("Boom Records — Phase 1 Training"). Tracks session completion, exercise performance (weight/reps/cardio), substitutions, PRI (Postural Restoration Institute) protocols, milestones, and progress charts. Fully offline-capable via service workers and localStorage.

## Repository Structure

The repo contains **6 parallel versions** of the same app in separate directories plus a **root-level copy** used for deployment:

- **`/` (root)** — deployed copy. `index.html`, `sw.js`, `manifest.json`, icons are byte-identical mirrors of `files 6/`. Served by Railway via `npm start` (`package.json` runs `npx serve`).
- **files/** — v1, dark theme (#0d1117), desktop sidebar layout, service worker `phase1-v1`
- **files 2/** — v2, light theme (#f5f0e8), mobile bottom-tab layout, no service worker
- **files 3/** — v2, light theme, adds service worker `phase1-v2`
- **files 4/**, **files 5/** — iterations with service workers `phase1-v3`, `phase1-v4`
- **files 6/** — current version, service worker `phase1-v12` (latest as of 2026-04)

All versions share the same JS logic and data model. **Default to editing `files 6/` and mirror the change to the root copy** (`index.html` and `sw.js`) in the same commit — recent commit history shows both are always touched together. Don't edit earlier `files N/` directories unless explicitly asked.

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

Cache-first with network fallback (`sw.js`). Pre-caches `index.html`, `manifest.json`, and Google Fonts. Old caches are cleaned on activate. Falls back to cached `index.html` when offline. Current cache name is `phase1-v12` (in both `files 6/sw.js` and root `sw.js`) — bump this on every user-visible change or clients won't pick it up.

### CSS Conventions

Abbreviated class names: `.sb` (sidebar), `.wl` (week list), `.sl` (session list item), `.ph` (phase header), `.tab-*` (tabs), `.b-*` (badge colors). Colors use CSS custom properties (`--bg`, `--t`, `--gold`, `--gd`, etc.).

## Development

No build step. Open any version's `index.html` in a browser. For service worker testing, serve over HTTPS or localhost.

Local dev (either works):
```bash
cd "files 6" && python3 -m http.server 8000
# or, from the repo root (matches Railway):
npm start   # runs: npx serve -l tcp://0.0.0.0:${PORT:-3000} --single
```

Railway deploys the **repo root**, not `files 6/`. The root `index.html`/`sw.js`/`manifest.json` must stay in sync with `files 6/` — treat them as a single edit.

**When modifying, bump version strings to force cache refresh:**
1. The `CACHE` constant in **both** `files 6/sw.js` and root `sw.js` (e.g., `phase1-v12` → `phase1-v13`). Without this, users keep the old cached `index.html`.
2. The localStorage key `SK` in `index.html` (currently `boom_p1_v5`) **only** if the STATE shape changes incompatibly.

**Caution:** Changing `SK` wipes all user data — logs, milestones, PRI checkmarks, PT notes. Only bump for schema-incompatible changes, and prefer a migration path when feasible.
