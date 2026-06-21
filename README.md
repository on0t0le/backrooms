# The Backrooms

A single-file, self-contained Backrooms horror game running entirely in the browser.
No build step, no dependencies, no assets — everything (engine, textures, audio,
events) is generated procedurally inside `index.html`.

**Play it live:** https://on0t0le.github.io/backrooms/

## What it is

Inspired by [The Backrooms](https://en.wikipedia.org/wiki/The_Backrooms) liminal-space
concept: an infinite, mono-yellow office-basement maze lit by humming fluorescents,
with the unsettling sense that you're never quite alone — but never quite sure either.

The full design brief lives in `docs/architecture.md` (gitignored, local-only spec
doc used to drive the implementation). Summary of what's implemented:

### Engine
- Raycaster (DDA) with textured walls, per-row floor/ceiling casting.
- Internal render buffer 480×270, upscaled with `image-rendering: pixelated`.
- Infinite world on 16×16 chunks, deterministic value-noise/fBm generation, with a
  guaranteed open "street" grid (every 8th row/column) so the maze always stays
  traversable. Distant chunks are evicted with an LRU cache.
- Procedural textures (mono-yellow wallpaper with damp stains/seams, damp carpet,
  ceiling tiles) generated into typed arrays at load time — no image assets.
- Controls: WASD/arrows to move, mouse look (Pointer Lock API), `F` toggles a
  flashlight, `Esc` releases the pointer lock.

### Atmosphere
- Dynamic fluorescent lighting with inverse-square falloff; lights are on by default
  and the scene is always clearly lit — distance fades into a warm yellow haze, never
  to black.
- Flashlight cone, vignette, film grain, scanlines, subtle chromatic edges.
- Web Audio synthesis: constant low hum (58/116 Hz + saw grit), high-frequency
  fluorescent whine, HRTF-spatialized footsteps, electrical buzz tied to flickers.
- Player position/heading persisted to `localStorage`.

### Horror events
A randomized, cooldown-gated scheduler drives psychological-horror beats tied into
the existing audio/lighting systems (never separate jump-scare overlays): footsteps
behind the camera, light flickers, a single light going dark behind you, a glimpsed
silhouette down a corridor, rare shadow entities, spatial anomalies (a hallway that
wasn't there before), a once-per-session distant scream, and an extremely rare
(<1%/minute) jumpscare. Frequency and intensity slowly escalate the longer you walk.

## Running locally

No build step — just open the file:

```bash
open index.html
```

or serve it (needed for some browsers' Pointer Lock / Web Audio gesture rules):

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Deployment

Deploys automatically to GitHub Pages on every push to `main` via
`.github/workflows/deploy-pages.yml` (`actions/deploy-pages`). No manual steps
needed — push to `main` and the live site updates.

## Project structure

```
index.html                       # the entire game: HTML + CSS + JS, no deps
.github/workflows/deploy-pages.yml  # CI/CD: deploy to GitHub Pages on push to main
docs/architecture.md             # original design brief (gitignored, local-only)
```

## Ideas for future improvement

- Fix the horror-event silhouette/shadow sprites: they currently render as flat
  blocky rectangles instead of soft, ambiguous silhouettes (see `projectSprite` /
  `silhouetteColor` in `index.html`).
- Per-pixel (rather than per-column/per-row) lighting for floor/ceiling would remove
  the remaining banding at close range, at a performance cost.
- Tune chunk/light density and event pacing based on real playtesting — current
  values are reasonable first guesses, not tuned against player feedback.
