# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Repan ("Stämnyckeln") is a single-file, offline-first web app (Swedish UI) for piano/guitar tuning, intonation practice, a metronome, and a drum/rhythm sampler. Everything — markup, CSS, and JS — lives in one file: `repan.html`. There is no build step, package manager, or test suite; it is opened directly in a browser.

## Development commands

There is no build/lint/test tooling. To work on this app:

- **Run it**: open `repan.html` directly in a browser, or serve the directory (e.g. `python3 -m http.server`) and visit it. Microphone features require a secure context (`https://` or `localhost`) — the app explicitly checks `window.isSecureContext` and shows an error otherwise.
- **Iterate**: edit `repan.html` and reload the browser; there is no transpilation or bundling.
- **Debug audio/pitch**: all DSP runs synchronously on the main thread via a `setInterval` tick (see Architecture below); use browser dev tools and `console.log` directly inside the relevant function.

## Architecture

The file is organized into three parts, in order:

1. **`<head>` / `<style>`** (lines ~1–340): CSS custom properties define a light/dark theme (`:root`, `prefers-color-scheme`, `[data-theme]`). Layout is a single-column "instrument panel": persistent 88-key strip + one active mode view below it.
2. **`<body>`**: static markup for every mode (`v-map`, `v-tune`, `v-guitar`, `v-intone`, `v-metro`, `v-sampler`, `v-over`), all present in the DOM at once and toggled with `hidden`.
3. **`<script>`** (lines ~690–end): all application logic, split into clearly marked sections (search for `/* ---------- Name ---------- */`):
   - **DSP** — FFT (`spectrum`), windowed single-bin DFT (`binDFT`), phase-difference frequency refinement (`refine`), partial-tracking harmonic analysis (`analyze`) that fits both fundamental frequency and inharmonicity coefficient B, and YIN pitch detection (`yin`) for the chromatic tuner.
   - **Tuning model** — `PROFILES` (mild/standard/wide stretch curves) and `computeTargets()`, which derives a per-key target frequency table (`f0[128]`) from the piano's own measured inharmonicity, propagating out from A4 using octave-type stretch targets.
   - **State** — `S` (current piano's live state: map, tuned notes, mode, current key) and `DB` (all pianos + shared prefs), persisted to `localStorage` under key `stamnyckeln.v2` (with migration from the older `stamnyckeln.v1` single-piano format). `save()`/`load()` are the single persistence boundary.
   - **Audio** — one `AudioContext` + `AnalyserNode` captures the mic into a `Float32Array`; a `setInterval(tick, 90)` polls it. A separate `outCtx` (or the same `AC` if the mic is running) plays reference tones/chords/metronome/sampler sounds, built additively from the piano's own stretched partials so playback matches what the tuner is listening for.
   - **Mapping** (`v-map`) → **Tuning** (`v-tune`): two-phase workflow. Mapping walks all 88 keys, analyzing each note's partials to build the inharmonicity/stretch model; tuning then walks the keys again (A4 first, then the temperament octave, then outward) showing a cents-deviation meter against the computed target, with "Ren"/"Varm" (clean/warm) unison and 3-string handling.
   - **Guitar** — a free-running chromatic tuner (YIN-based) independent of the piano's key-by-key model, with per-preset string sets and a lockable "nearest string" mode.
   - **Intonation practice** (`v-intone`) — a canvas-based pitch game ("Rymdskepp"/"Kalejdoskop") that scores sung/played pitch accuracy against a target note, with difficulty and instrument-transposition settings.
   - **Sampler** — a 16-step drum machine with recordable/importable sample slots, per-slot EQ/reverb, and waveform trim UI on an HTML canvas.
   - **Metronome** — classic click or drum-groove playback with per-signature/per-groove editable patterns.
   - **Overview**, **Settings & data**, **Pianos** — multi-piano management (add/switch/rename), import/export of all piano data as JSON, and chart rendering (inline SVG) of the stretch curve and inharmonicity.
   - **Modes** / **Wire up** (end of file) — `setMode()` is the single entry point that shows/hides the right section, starts/stops the mic and other mode-specific loops, and re-renders; all DOM event listeners are attached in one block at the very end, followed by `load()` → `setMode(S.mode)` to boot.

## Key conventions

- **One source of truth per piano**: each piano's `map`/`tuned`/`curMap`/`curTune`/tuning `settings` are stored under its own id in `DB.pianos`; shared app-wide preferences (sound choice, metronome state, etc.) live in `DB.prefs`. `storeActive()`/`loadActive()` move data between the live `S` object and `DB` when switching pianos.
- **Target frequencies are memoized**: `targets()` lazily computes and caches the per-key frequency/B table; call `invalidate()` (not a direct recompute) whenever settings or measured map data change, so the cache is rebuilt on next read.
- **UI text is in Swedish**; keep new user-facing strings consistent with the existing tone (plain, instructional, no jargon beyond what's already used).
- **No frameworks**: DOM updates are plain `document.getElementById` (aliased as `$`) and manual class/attribute toggling — follow this style rather than introducing a framework or virtual DOM.
