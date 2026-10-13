# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Asteroids arcade clone built with raw HTML5 Canvas and vanilla ES6+ JavaScript. No build tools, no bundler, no dependencies, no package.json. The entire game lives in a single file: `game.js` (~420 lines), loaded directly by `index.html`.

## Running / developing

There is no build step. Open `index.html` directly in a browser, or serve it locally:

```bash
npx serve .
```

Then visit `http://localhost:3000`. To test a change, reload the page in the browser — there is no compile/watch step and no test suite.

## Architecture

Everything is in `game.js`, structured top-to-bottom as:

1. **Input** — a single `keydown`/`keyup` listener pair populates two globals: `keys` (held state) and `justPressed` (edge-triggered, consumed via `pressed(code)`). `pressed()` clears the flag on read, so it must be polled exactly once per frame per key (used for `Space` to fire/restart).
2. **Utils** — `wrap(v, max)` implements toroidal wrap-around (the space is a torus: objects exiting one edge appear on the opposite edge), `dist`, `rand`, `randInt`.
3. **Entity classes** — `Bullet`, `Asteroid`, `Ship`, `Particle`. Each has its own `update(dt)` and `draw()`; `dt` is seconds, not ms. Entities mark themselves `dead = true` instead of removing themselves from arrays; arrays are swept with `.filter(e => !e.dead)` by the game loop.
   - `Asteroid` size is `1|2|3` (small/medium/large) indexing into parallel arrays `RADII`, `SPEEDS`, `POINTS` (index 0 unused). `split()` produces two smaller asteroids at the same position; size 1 asteroids don't split.
   - Asteroid and ship shapes are hand-drawn polygons in local space, transformed via `ctx.translate/rotate` in `draw()`.
4. **Game state** — module-level globals (`ship`, `bullets`, `asteroids`, `particles`, `score`, `lives`, `level`, `state`, `deadTimer`). `state` is one of `'playing' | 'dead' | 'gameover'` and gates what `update()` does each frame. `initGame()` resets everything for a new game; `nextLevel()` is called from `update()` when `asteroids.length === 0`.
5. **Update loop** (`update(dt)`) — branches first on `state`:
   - `gameover`: only waits for `Space` to call `initGame()` again.
   - `dead`: counts down `deadTimer` (respawn delay) while still animating existing asteroids/particles.
   - `playing`: handles shooting, moves all entities, then does collision detection (bullet↔asteroid, ship↔asteroid) using plain circle-distance checks (`dist(a, b) < a.radius + b.radius`-style), splits/destroys asteroids, spawns explosion particles, and calls `killShip()`/`nextLevel()` as needed.
6. **Draw loop** (`draw()`) — clears the canvas, draws particles → asteroids → bullets → ship (back to front), then HUD/overlay text.
7. **Main loop** — `requestAnimationFrame` loop computing `dt` from timestamps, clamped to `0.05`s max to avoid physics blowups on tab-switch/lag, then calling `update(dt)` and `draw()`.

Canvas is fixed-size: `W = 800`, `H = 600`, matching the `<canvas>` element's `width`/`height` attributes in `index.html`. There is no resizing/scaling logic.

## Notes

- `README.md` (Spanish) describes power-ups and a "shooting star" asteroid variant as features; these are not present in the current `game.js` — treat the README's feature list as aspirational/outdated relative to the code, not as a spec.
- Tuning constants (ship thrust/rotation/drag, bullet speed, asteroid speeds/radii/points) are inlined as local `const`s near their point of use rather than collected in one config block — check inside the relevant class/function first.
