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
- Controls: WASD **or** arrow keys to move/strafe, mouse look (Pointer Lock API),
  `F` toggles a flashlight, `Esc` opens the pause menu (and frees the mouse).

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
behind the camera, light flickers, a single light going dark behind you, **full
blackouts** (lights cut to near-dark with breathing, then snap back), a glimpsed
silhouette down a corridor, shadow entities, spatial anomalies (a hallway that
wasn't there before), a once-per-session distant scream, and an extremely rare
(<1%/minute) jumpscare. Frequency and intensity escalate the longer you walk. Note:
the blackout beat is a deliberate, short, temporary deviation from the otherwise
"always lit" rule — the steady state stays fully lit between beats.

### Objective & game loop
- **Goal:** find the **exit** — one rare, deterministic tile per world that glows as a
  green beacon column. Step onto it to escape and win.
- **Wayfinding (guides without trivializing):**
  - A faint **red breadcrumb trail** glows on the floor along the open street grid
    toward the exit, but only the next ~14 cells light up (fading at the far end), and
    it **goes dark while you're being hunted** — you get "which way now," not the whole
    solution.
  - The exit's **pillar and sky-beam** only reveal once you're within ~6 cells — a
    short-range "you found it" cue, not a map marker that gives away the route.
  - A minimal HUD (top-left) shows distance + rough heading as a backup.
  - The exit is 40+ tiles away through the maze; difficulty is *surviving the walk*,
    not solving a labyrinth.
- **The stalker:** an entity that appears at the edges and **advances toward you
  whether or not you're looking at it** — facing it no longer makes it vanish, it
  just keeps closing the distance. If it reaches you it lunges into a full hunt. (A
  short grace period at the start of each run lets you get your bearings first.)
- **The hunt:** triggered by the stalker closing in, or by walking in a straight line
  too long. A red warning closes in with a countdown — keep **changing direction** to
  accumulate enough turning and shake it. If the countdown runs out it lunges to
  point-blank with a scream and you lose.
- **Pause / restart:** `Esc` opens a pause menu (Resume / Restart Run) showing time
  survived and distance walked. Win and lose both show an end screen with stats and a
  Play Again button. **Every restart generates a brand-new maze** (fresh seed, new
  layout, new exit location).

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

- Soften the horror sprites further: they now render as a solid humanoid outline
  (head/shoulders/torso via `inHumanoid` in `index.html`), which is clearly visible;
  a soft alpha falloff at the edges would make them more ambiguous and dreamlike.
- Add a directional audio cue or visual tell so the hunt's origin is clearer before
  the entity is on top of you; tune `huntThreshold`/`huntTurnNeeded` from playtesting.
- Per-pixel (rather than per-column/per-row) lighting for floor/ceiling would remove
  the remaining banding at close range, at a performance cost.
- Tune chunk/light density and event pacing based on real playtesting — current
  values are reasonable first guesses, not tuned against player feedback.
