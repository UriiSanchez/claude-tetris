# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A classic Tetris implementation in vanilla JavaScript with HTML5 Canvas and CSS. No dependencies, no build process, no package.json.

## Running the game

There is no build/lint/test tooling. Open `index.html` directly in a browser, or serve it statically:

```bash
python3 -m http.server 8000   # or: npx serve .   /   php -S localhost:8000
```

Then visit `http://localhost:8000`. To verify a change works, actually open the page in a browser and play — there are no automated tests.

## Architecture

Three files, all logic lives in `game.js` (~300 lines, single global scope, no modules):

- `index.html` — DOM structure: main `<canvas id="board">` (300×600, i.e. `COLS×BLOCK` by `ROWS×BLOCK`), a `<canvas id="next-canvas">` for the next-piece preview, score/lines/level panel, and a shared overlay div used for both PAUSE and GAME OVER states.
- `style.css` — dark/retro arcade visual theme only.
- `game.js` — all game state and logic, described below.

### Core model

- **Board**: `ROWS × COLS` matrix (`createBoard`), each cell is `0` (empty) or a color index `1–7` identifying which piece locked there.
- **Pieces**: `PIECES` array of square matrices (index 0 unused/null so piece type doubles as color index into `COLORS`). Rotation is done geometrically via `rotateCW` (transpose + reverse), not via pre-baked rotation states. `randomPiece` picks a type via weighted random using `PIECE_WEIGHTS` (index 1-8) rather than a uniform 1-in-8 — the 3×3 nut piece (8) has a lower weight than the seven standard tetrominoes since its shape is harder to place.
- **Collision** (`collide`): checks board bounds and overlap with locked cells; used for movement, rotation, and ghost-piece projection.
- **Wall kicks** (`tryRotate`): after rotating, tries offsets `[0, -1, 1, -2, 2]` columns until one doesn't collide.
- **Locking** (`lockPiece` → `merge` + `clearLines` + `spawn`): merges the current piece into `board`, clears completed rows, then spawns the next piece.
- **Line clearing & gravity** (`clearLines`): finds every full row in one pass, removes exactly those rows, and pushes new empty rows in at the top — a rigid shift, not per-cell gravity. Everything above a cleared row drops by the number of rows cleared, in place, so leftover fragments of a broken piece (e.g. only part of an L or Z was used to complete the line) fall straight down within their own column and only fill the gap the clear just opened; pre-existing holes elsewhere in the board are left untouched instead of being collapsed.
- **Scoring**: `LINE_SCORES = [0, 100, 300, 500, 800]` × current `level` on line clears; hard drop adds 2 points per row dropped, soft drop adds 1 point per row.
- **Leveling/speed**: level = `floor(lines / 10) + 1`; `dropInterval = max(100, 1000 - (level-1)*90)` ms.
- **Ghost piece** (`ghostY`): projects the current piece straight down until collision, drawn at `globalAlpha = 0.2`.

### Game loop

`init()` creates the board, seeds `next` via `randomPiece()`, calls `spawn()`, then starts `requestAnimationFrame(loop)`. `loop(ts)` accumulates elapsed time into `dropAccum`; once it exceeds `dropInterval` the piece drops one row (or locks if blocked), then `draw()` redraws grid + board + ghost + current piece every frame. `spawn()` promotes `next` to `current`, generates a new `next`, and calls `endGame()` if the new piece immediately collides (classic top-out game over).

Input is a single `keydown` listener mapping arrow keys / X / Space / P to movement, rotation, soft drop, hard drop, and pause — no input buffering or DAS/ARR handling.

### Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, update the `<canvas id="board">` `width`/`height` in `index.html` to match (`COLS×BLOCK`, `ROWS×BLOCK`).
