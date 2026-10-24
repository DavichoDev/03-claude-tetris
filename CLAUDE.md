# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Vanilla JS Tetris (HTML5 Canvas): `index.html`, `style.css`, `game.js`. No package.json, build, tests or linter. Run it with `open index.html` or a static server (`python3 -m http.server 8000`).

- Board size is duplicated: `COLS`, `ROWS`, `BLOCK` in `game.js` must equal the `width`/`height` of `<canvas id="board">` in `index.html` (`COLS × BLOCK` by `ROWS × BLOCK`, currently 300×600). Change both together.
- `README.md` and the HTML `lang` are in Spanish; keep docs and user-facing text consistent with that.

## Architecture

Everything lives in `game.js` (no modules, `'use strict'`, global mutable state declared in one `let` line: `board, current, next, score, lines, level, paused, gameOver, lastTime, dropAccum, dropInterval, animId, pieceCount`). `index.html` only provides the DOM ids it looks up at the top of the file (`board`, `next-canvas`, `score`, `lines`, `level`, `overlay`, `overlay-title`, `overlay-score`, `restart-btn`, `theme-toggle`); renaming one breaks the script.

- **Data model**: `board` is a `ROWS×COLS` matrix of `0` or a color/piece index 1–8. `PIECES[i]` and `COLORS[i]` share the same index (I,O,T,S,Z,J,L,N; N is the 3×3 "tuerca" ring with an empty center; index 9 = `BOMB`, a 1×1 piece that is never merged: `explode(cx, cy)` clears the 3×3 around it, applies per-column gravity and scores `10 × destroyed × level`; `drawBlock` draws it as a bomb); a piece's cells hold its own index, so the shape matrix doubles as its color map. `current`/`next` are `{ type, shape, x, y }`; `randomPiece()` copies the shape so `PIECES` is never mutated.
- **Rules layer** (pure-ish, operate on globals): `collide(shape, ox, oy)` is the single source of truth for movement, rotation, ghost and spawn checks (cells with `ny < 0` are allowed above the board). `rotateCW` returns a new matrix; `tryRotate` applies it with horizontal kicks `[0,-1,1,-2,2]` only (no vertical or SRS kicks).
- **Piece lifecycle**: `lockPiece()` = `merge()` (or `explode()` if the piece is the bomb) → `clearLines()` → `spawn()`. `spawn()` promotes `next`, rolls a new one (every `BOMB_EVERY`-th, counted by `pieceCount`, is the bomb via `makePiece(BOMB)`; `randomPiece()` never returns it), and calls `endGame()` if the new piece collides. All three drop paths (gravity in `loop`, `softDrop`, `hardDrop`) converge on `lockPiece()`.
- **Scoring/speed** live inside `clearLines()`: `LINE_SCORES[cleared] * level`, `level = floor(lines/10)+1`, `dropInterval = max(100, 1000 - (level-1)*90)`. Soft drop adds 1/row and hard drop 2/row (in `softDrop`/`hardDrop`). `init()` hardcodes the initial `1000` ms separately from this formula.
- **Loop**: `loop(ts)` uses `requestAnimationFrame`, accumulates `dropAccum` and drops one row per `dropInterval`; rendering (`draw`) runs every frame, state changes happen in the loop and in the `keydown` handler (which always ends with `updateHUD()`). Pause cancels the frame and re-enters `loop` on resume; `init()` (also the restart button handler) resets all state.
- **Rendering**: `draw()` repaints everything each frame (grid, board, ghost at alpha 0.2, current piece) via `drawBlock(context, x, y, colorIndex, size, alpha)`. `drawNext()` renders into the separate 120×120 canvas (4×4 cells of 30px) and is only called from `spawn()`. The HUD is DOM text updated by `updateHUD()`; the overlay (`.hidden` class) serves both PAUSA and GAME OVER.
- **Input**: single `keydown` listener using `e.code` (`ArrowLeft/Right/Down/Up`, `KeyX`, `Space`, `KeyP`). Key repeat from the OS is the only auto-repeat; there is no DAS/hold/lock delay.

### Tema claro/oscuro

Los colores de `style.css` son variables CSS (`:root` oscuro, `:root[data-theme="light"]` claro). El botón `#theme-toggle` alterna `document.documentElement.dataset.theme` y actualiza `gridColor` (leído de `--grid`) para `drawGrid`. No se persiste ni tiene atajo de teclado. Los colores de las piezas (`COLORS`) no cambian con el tema.

### Game over and the loop

`endGame()` is reached from inside `loop` (via `lockPiece` → `spawn`), where `cancelAnimationFrame` has nothing to cancel. `loop` therefore returns early when `gameOver` is set and only schedules the next frame if `!gameOver`. Keep both guards when touching the loop or `endGame`.
