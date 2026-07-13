# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Milky Way Atlas** — an interactive, hand-drawn survey of the galaxy. The application is a single self-contained `index.html` (~3,300 lines): no dependencies, no build step, no server, no network access, no package.json, no tests, no linter. The font is embedded as a base64 data URI.

The only files beside it are the PWA shell — `manifest.webmanifest`, `sw.js` (stale-while-revalidate service worker; a SW cannot be inlined), `icon-192.png`/`icon-512.png` — and `.github/workflows/deploy-pages.yml`, which publishes the repo root to GitHub Pages on every push to `main` (live at https://joewolly.github.io/universe/). Bump the `CACHE` version in `sw.js` only if a change to caching behaviour itself is needed; content updates propagate automatically.

To run or verify a change, open `index.html` in a browser:

```
open index.html
```

There is nothing else to build or install. When checking work, exercise the affected scene in the browser (Chromium + Playwright are available in remote sessions if a screenshot or scripted check is needed).

## Layout of index.html

One file, three blocks:

- **CSS** (`<style>`, ~lines 7–335) — organized by UI region with `/* ---------- name ---------- */` comment dividers (top bar, atlas drawer, survey card, controls, quiz, tour bar, welcome…). Single committed dark theme: the page is space; amber chrome imitates red-light observatory practice. Colors live in `:root` custom properties.
- **HTML body** (~lines 336–443) — a full-screen `<canvas id="sky">` plus fixed DOM overlays: top bar, atlas index drawer (left), survey card panel (right, with its own `<canvas id="plate">`), bottom controls, quiz dialog, tour bar, welcome overlay.
- **JS** (`<script>`, ~lines 444–2479) — organized into banner-commented sections, in order: CATALOG (galaxy scene, solar scene, night-sky extension, meteor showers), ENGINE (scene/camera/galaxy renderer, frame rendering, plate art/UI/input/main loop), NIGHT SKY EXTENSION engine, tonight, field exam, grand tour.

New features have historically been appended as "extension" sections that push additional entries into the existing catalog (see `SKY_EXT` and `SHOWERS`) rather than restructuring what exists. Follow that pattern.

## Architecture

### The catalog (`CAT`)

Everything clickable is an entry in the single `CAT` array. An entry looks like:

```js
{ id:'sgr-a', scene:'galaxy', group:'Heart of the galaxy', type:'blackhole',
  name:'Sagittarius A*', kicker:'Supermassive black hole', sub:'…',
  pos:[0,0], mark:8, plate:'blackhole',
  stats:[['Mass','4.3 million Suns'], …], blurb:'…', facts:['…'] }
```

- `scene` is `'galaxy'`, `'solar'`, or `'sky'` — filtered into `GALAXY_OBJS` / `SOLAR_OBJS` / `SKY_OBJS`.
- `group` buckets the entry in the atlas index; groups render in `GROUP_ORDER` (defined near `buildAtlas`), so a new group name must be added there.
- `type` picks the marker color from `TYPE_COLORS`.
- `plate` selects a procedural illustration in `drawPlate` (some take a parameter, e.g. `plate:'star:#cfe0ff'`).
- Position varies by scene: `pos` for fixed objects, `orbit`/`kepler` for solar bodies (resolved by `bodyPos`), `armPhi`/`armTheta` for spiral arms, `radiant` or `stars` (RA/Dec) for sky objects. `objWorldPos(o)` is the universal accessor.
- Optional `action` (e.g. `'enterSolar'`) adds a button to the survey card.

### Coordinate systems (one per scene)

- **Galaxy**: world units are light-years, galactic centre at (0,0), y grows downward (canvas convention), Sun at `SUN = [0, 26000]`. Helper `s(dx, dy)` offsets from the Sun.
- **Solar**: arbitrary "map px"; orbits are compressed, not to scale. Time is driven by `state.simDays` (`DAYS_PER_SEC = 8` at 1×).
- **Sky**: an equirectangular star chart in degrees — `x = (12h − RA) · 15`, `y = −Dec` (see `skyXY`). Constellation `stars` are `[RA_hours, Dec_deg, magnitude, name?, isRed?]`.

Positions throughout are approximate by design — drawn for navigation and learning, not measurement. Cards say so when an object is drawn far from its true distance.

### Engine

- Single `state` object holds the current scene, camera (`cam`/`tgt`, where `z` = px per world unit), selection, hover, follow target, time scale, and the visited set.
- The camera eases toward `tgt` each frame (`stepCam`); `flyTo` / `goTo` set targets; `w2s`/`s2w` convert world↔screen.
- The galaxy and sky backgrounds are pre-rendered once to offscreen sprites (`buildGalaxySprite`, `buildSkySprite`) and blitted per frame; only markers, labels, and solar bodies are drawn live. DPR is capped at 2, and `prefers-reduced-motion` is respected (`REDUCED`).
- Scene switching goes through `fadeTo(cb)` → `enterSolar`/`exitSolar`/`enterSky`/`exitSky` → `syncSceneUI` (which shows/hides scene-specific controls like the time buttons and Tonight). `sceneObjects()` returns the pickable set for the current scene.
- Hit-testing is `pickAt`; opening `openCard(o)` marks the object visited and redraws its plate illustration.

### Persistence

All persistence goes through the `store` wrapper (safe JSON localStorage). Keys: `mwa-visited` (surveyed object ids), `mwa-best` (quiz personal best), `mwa-welcomed` (welcome overlay dismissed).

### Quiz and tour

- `QUIZ` entries always put the correct answer at index 0 (`a:0`); options are shuffled at render time. `ref` is a catalog `id` used by the "Visit →" button — it must exist in `CAT`. Each round samples 8 of the bank.
- `TOUR` is an ordered array of catalog ids; each stop flies the camera there and opens the card.
- The "Tonight" feature computes the real Sun's RA/Dec for today (`sunRaDec`) and derives per-object visibility (`skyStatus`, `bestMonth`) shown on sky cards.

## Conventions

- Keep everything in the single `index.html`; do not introduce build tooling, external assets, or network fetches.
- Match the existing voice in catalog copy: `stats` are `[label, value]` pairs, `blurb` is one evocative paragraph, `facts` ("Field notes") are 1–3 concrete sentences each. Scientific accuracy matters; where the map distorts (distances, sizes), the copy acknowledges it.
- Use the section-banner comment style (`/* ============ NAME ============ */`, `/* ---------- sub ---------- */`) when adding code.
- Helpers to reuse: `$` (getElementById), `store`, `shuffle`, `mulberry` (seeded RNG for procedural art), `skyXY`.
