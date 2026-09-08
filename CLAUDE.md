# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A single-page Tetris implementation in vanilla JavaScript, HTML5 Canvas, and CSS. No build step, no package manager, no dependencies. The entire game lives in three files: `index.html`, `style.css`, `game.js`.

## Running the game

Open `index.html` directly in a browser, or serve it with any static file server:

```bash
python3 -m http.server 8000
# or
npx serve .
```

There is no build, lint, or test tooling in this repo — there's nothing to compile and no test suite to run. Verify changes by opening the page in a browser and playing.

## Architecture (`game.js`)

Everything is global state and top-level functions — no classes, no modules. Understanding the game requires tracing how these pieces interact:

- **Board model**: `board` is a `ROWS × COLS` matrix (20×10). Each cell is `0` (empty) or an integer `1–7` identifying which piece color occupies it (see `COLORS`/`PIECES`).
- **Pieces**: `PIECES` defines the 7 tetrominoes as small square matrices, indexed by the same 1–7 type used in `board`. A live piece is `{ type, shape, x, y }`. Rotation (`rotateCW`) is a transpose + row-reverse of `shape`, not a lookup table of rotation states.
- **Collision** (`collide`): the single source of truth for whether a shape can occupy a position — checks board bounds and cell occupancy. Every movement, rotation, spawn, and ghost-piece calculation goes through this function.
- **Wall kicks** (`tryRotate`): after rotating, tries horizontal offsets `[0, -1, 1, -2, 2]` via `collide` until one fits, otherwise the rotation is discarded.
- **Game loop** (`loop`): driven by `requestAnimationFrame`, accumulates elapsed time in `dropAccum` and advances the piece one row once `dropAccum >= dropInterval`. `dropInterval` shrinks as `level` increases (`max(100, 1000 - (level-1)*90)`).
- **Locking a piece** (`lockPiece`): `merge()` writes the current piece into `board`, `clearLines()` removes full rows (shifting from the bottom up and unshifting empty rows at top), then `spawn()` promotes `next` to `current` and generates a new `next`. If the newly spawned piece immediately collides, `endGame()` fires.
- **Scoring**: `LINE_SCORES = [0, 100, 300, 500, 800]` indexed by lines cleared at once, multiplied by `level`. Hard drop adds 2 points per row dropped; soft drop adds 1 point per row.
- **Rendering** (`draw`/`drawNext`): redraws the whole canvas every frame — grid, locked board cells, a semi-transparent ghost piece (projected via the same `collide` loop used for drop logic, at `globalAlpha = 0.2`), then the current piece.
- **Input**: a single `keydown` listener drives movement/rotation/drop/pause; arrow keys and `X`/`Space`/`P` per the controls in the README.

### Tunable constants

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `LINE_SCORES`, `dropInterval` are all defined at the top of `game.js`. If `COLS`, `ROWS`, or `BLOCK` change, the `<canvas id="board">` `width`/`height` in `index.html` must be updated to match (`COLS × BLOCK` and `ROWS × BLOCK`).
