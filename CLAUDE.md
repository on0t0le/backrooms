# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Backrooms horror game that is **one self-contained file**: `index.html`. All HTML,
CSS, and JS are inline. No build step, no dependencies, no external assets — every
texture, sound, and bit of geometry is generated procedurally at runtime. Anything
that would be an "asset" in a normal game is code here.

## Commands

There is no build/lint/test tooling. Workflow is edit `index.html`, reload.

```bash
open index.html                 # quickest local run
python3 -m http.server 8000     # serve (needed for Pointer Lock / Web Audio gesture rules)
```

Deploy is automatic: push to `main` → `.github/workflows/deploy-pages.yml` builds and
publishes to GitHub Pages (https://on0t0le.github.io/backrooms/). The whole repo root
is the artifact (`path: .`), so `index.html` at root is the deployed page.

## Verifying changes

There are no tests. Verify visually with the chrome-devtools-mcp tools (load
`file:///.../index.html`, `evaluate_script`, `take_screenshot`, `list_console_messages`).

- A headless `requestPointerLock` rejection ("Uncaught (in promise)") is **expected and
  benign** — pointer lock needs a real user gesture. It is not a code bug.
- The whole script is an IIFE, so internals are private. To exercise game state
  headlessly (force an event, jump to win/lose, inspect the exit cell), temporarily
  expose a `window.__dbg = {...}` hook before the closing `})();`, test, then **remove
  it before committing**.
- Two visual invariants to re-check after any render/lighting change: the frame must
  never clip to white or go to black (sample pixels: `clipRatio`/`blackRatio` ≈ 0), and
  the palette must stay canonical Backrooms mono-yellow (avg channel roughly R>G>B,
  e.g. ~[132,115,63]), not cream/white.

## Architecture (all inside `index.html`'s IIFE)

The code reads top-to-bottom as a pipeline; sections are marked with banner comments.

- **Determinism.** Everything world-related derives from a single mutable `SEED` via
  `hash2` → `valueNoise` → `fbm`. Same seed = same world. `SEED` is `let` (not const)
  so `restartRun()` can swap it for a brand-new maze. Seed and player save persist in
  `localStorage` (`backrooms_seed`, `backrooms_save`).

- **Procedural textures.** `makeWallpaperTex/Carpet/Ceil` write 64×64 packed-RGBA
  `Uint32Array`s via `packRGBA` (little-endian: `(a<<24)|(b<<16)|(g<<8)|r`). Sampled
  directly during rendering — no canvas image objects.

- **Infinite world.** `cellIsWall(gx,gy)` is the single source of truth for geometry.
  It checks transient `overrides` (spatial-anomaly cells), else an LRU-cached 16×16
  chunk (`getChunk`). A guaranteed open "street" grid (`isStreet`, every
  `STREET_SPACING`) keeps the maze traversable; `rawIsWall` fills the rest from `fbm`.
  Lights live only on street cells (`isLightCell`).

- **Rendering** writes into a 480×270 `Uint32Array` (`pix32`) upscaled to the screen
  canvas with `image-rendering: pixelated`. Per frame `renderFrame()`:
  `castColumn` (DDA wall raycast, fills `zbuffer`) → `renderFloorCeilingRow` (per-row
  floor/ceiling cast, also where emissive light fixtures, the exit tile glow, and the
  red breadcrumb trail are drawn) → `drawSkybeam` → `drawExit` → `drawActiveSprites` →
  `drawStalker` → `drawHunt`. The exit pillar (`drawExit`) and `drawSkybeam` are
  proximity-gated by `EXIT_NEAR` (~6 cells) — they stay hidden until you're nearly on
  the exit, so they orient without revealing the route. All billboards go through
  `projectSprite`, which respects `zbuffer` for occlusion. Lighting is `AMBIENT` +
  `lightContribution` (inverse-square over `nearbyLights`) + flashlight, blended to
  warm-yellow fog in `shadeColor` — tuned to **always stay lit, never to white/black**.

- **Audio** is fully synthesized Web Audio (`initAudio`): constant hum/whine, HRTF-
  spatialized footsteps via `PannerNode` + `AudioListener` (`updateListenerOrientation`
  syncs to the player each frame). Events call into these, never play files.

- **Horror events** (`tickEvents`) fire on a randomized, cooldown-gated, `escalation()`-
  scaled timer — never a fixed loop. Visual events (silhouette/shadow) use
  `visiblePoint()` (a `castRayDist` line-of-sight check) so a figure is only spawned
  where there is actually open corridor; the figure shape is `inHumanoid(u,v)` masked.

- **Game loop / objective.** `loop()` drives everything. Win = reaching the deterministic
  `exitCell` (computed by `computeExitCell`, rendered as a green beacon). Lose = the
  **hunt**: walking too straight grows `straightWalkTimer` past `huntThreshold()` →
  `startHunt()`; the player must bank enough turning (`huntTurnAccum` vs
  `huntTurnNeeded`) before the countdown to shake it, else `finishRun(false)`.
  `runOver` freezes input/physics while an end overlay is up. `restartRun()` reseeds and
  resets all of this.

## Conventions

- Keep it a single dependency-free file. Do not add a bundler, framework, or external
  asset — procedural generation is the point.
- New geometry must go through `cellIsWall`; new billboards through `projectSprite`;
  new sounds through the Web Audio graph in the audio section.
- `docs/architecture.md` is the original design brief (the spec/prompt) and is
  **gitignored** (`.gitignore` is just `docs`) — local-only, do not commit it.
