# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Clone of the arcade game *Asteroids* on plain HTML5 Canvas. No dependencies, no bundler, no build step, no tests, no linter. The whole project is five files: `index.html`, `game.js`, `meteor-sprite.js`, `favicon.svg`, `README.md`.

## Running

Open `index.html` directly in a browser, or serve the folder:

```bash
npx serve .        # then http://localhost:3000
```

There is nothing to build or install — edits to `game.js` take effect on reload. Deployed as a static site to GitHub Pages (`https://klerith.github.io/claude-asteroids/`).

## Architecture (`game.js`)

One file, `'use strict'`, no modules — everything is top-level in one script scope. Sections are delimited by `// ── Name ───` banner comments in this order: Input, Utils, meteor sprite, Bullet, Asteroid, Ship, Particle, game state, `update`, `draw`, main loop. Keep new code inside the matching section.

**Entity contract.** Every entity class (`Bullet`, `Asteroid`, `Ship`, `Particle`) exposes `update(dt)`, `draw()`, a `radius`, and a `dead` boolean. Nothing self-destructs: `update` sets `dead`, and the top-level `update()` prunes with `arr = arr.filter(e => !e.dead)`. Adding an entity type means adding the class, a module-level array, reset in `initGame()`, and calls in `update()`/`draw()`.

**Time.** All motion is in units per second and multiplied by `dt`; the loop clamps `dt` to 0.05 s so a backgrounded tab cannot tunnel objects through collisions. Never hardcode per-frame deltas.

**Toroidal space.** Positions wrap through `wrap(v, max)`. `Ship`, `Bullet` and `Asteroid` wrap; `Particle` deliberately does not, since it expires within ~1 s.

**Input.** Two channels, and picking the wrong one is the usual bug source:
- `keys[code]` — held state, polled every frame for continuous actions (rotate, thrust).
- `pressed(code)` — edge-triggered, **consumes** the flag on read, so call it at most once per frame per key (shoot, restart).

**Game state.** Module-level `ship, bullets, asteroids, particles, score, lives, level, state, deadTimer`. `state` is `'playing' | 'dead' | 'gameover'`, dispatched by early returns at the top of `update()` — the `dead` and `gameover` branches keep animating particles/asteroids but skip input and collisions.

**Size-indexed tables.** `RADII`, `SPEEDS`, `POINTS` are indexed by asteroid `size` 1–3 with a dummy `0` slot. Changing the size range means extending all three arrays consistently. `Asteroid.split()` returns two asteroids one size smaller (none at size 1); the caller collects them into `newAsteroids` and appends after the collision loop, so never mutate `asteroids` while iterating it.

**Collisions** are circle-distance only (`dist`). Ship-vs-asteroid applies a `0.82` leniency factor on the asteroid radius; the ship is immune while `ship.invincible > 0` (3 s after every respawn, rendered as a blink in `Ship.draw`).

**Level flow.** Clearing all asteroids calls `nextLevel()`, which spawns `3 + level` large ones; `spawnAsteroids` rejects positions within 130 px of the center so the player never respawns onto a rock.

**Rendering** is vector — white strokes on a black `fillRect`, monospace HUD text, no audio — with one exception: size-3 asteroids. Draw order in `draw()` is particles → asteroids → bullets → ship → HUD → overlay. Entities draw in local coordinates inside `ctx.save()/translate/rotate/restore`.

**Meteor sprite.** Asteroids with `size === 3` set `this.meteor` and render through `Asteroid.drawMeteor()` as a flaming meteor bitmap instead of a polygon; sizes 1–2 stay vector. The image is a transparent PNG embedded as a base64 data URI in `meteor-sprite.js` (`const METEOR_SPRITE`, loaded by `index.html` **before** `game.js`), so no fetch happens and the game still runs from `file://`. `MET_CX`/`MET_CY`/`MET_R` are the rock's center and radius as fractions of the sprite side, and `MET_HEADING` is the direction the drawing points in; `drawMeteor` scales so the painted rock matches the collision `radius`, centers it on the entity, and rotates by `heading - MET_HEADING` so the flame always trails the velocity. If the image fails to load, `meteorReady` stays `false` and `draw()` falls back to the polygon, which is why meteors still build `verts`. The flashing look is three layers: an additive radial-gradient halo, twinkling `drawSparkle` stars, and a second additive pass of the sprite — all driven by `this.flick`, plus `spawnEmber` ember particles shed from the tail. `Particle` now takes an optional `rgb` string (`'255,150,45'`, defaulting to white) that `explode()` forwards, so explosions and embers can be warm-colored.

## Conventions

- Canvas is 800×600 in **two** places — the `<canvas>` attributes in `index.html` and the `W`/`H` constants in `game.js`. Change both together.
- Code comments and all on-screen text (`SCORE`, `NIVEL`, `GAME OVER`, `PUNTAJE`) are in Spanish; keep new strings and comments in Spanish to match.
- Tuning constants (`SPEED`, `ROT`, `THRUST`, `DRAG`, `NOSE`, cooldowns) live as `const`s inside the method that uses them rather than in a global config block.
- The README is out of date: it still advertises power-ups and a "estrella fugaz" asteroid that were removed in commit `13e713f`. Do not restore those features from the README description.
