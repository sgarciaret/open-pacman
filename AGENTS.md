# AGENTS.md

## Project Overview

- **Stack**: Vanilla JS (ES6+), HTML5 Canvas, CSS. No build step, bundler, or `package.json`.
- **Methodology**: Spec-Driven Development via repo skills (`spec` and `spec-impl`).
- **Entrypoint**: `src/index.html`.

## Development & Running

- **Run/Preview**: Serve `src/` statically (e.g., `npx serve src` or open `src/index.html` in a browser).
- **Testing**: No automated test runner configured. Test changes manually in the browser.

## Architecture & Conventions

### Script Loading & Globals
Scripts are loaded sequentially via classic `<script>` tags in `src/index.html` and communicate through global variables (`window`):
1. `src/js/maze.js`: Grid constants (`MAZE_STR`, `MAZE`), `TUNNEL_ROW` (14), `PACMAN_START`, `GHOST_STARTS`.
2. `src/js/game.js`: State management (`createGame`), update loop (`update`), movement (`DIRS`), collision, and ghost AI.
3. `src/js/render.js`: Canvas drawing functions (`draw`, `drawWalls`, `drawPacman`, `drawGhost`, `drawHUD`).
4. `src/js/main.js`: Input listeners (`keydown`), overlay UI handling, and `requestAnimationFrame` game loop.

*Note: Do not convert scripts to ES modules (`type="module"`) unless explicitly requested, as `index.html` and script exports rely on globals.*

### Grid & Canvas Specs
- **Canvas Size**: 560 x 620 px.
- **Grid Dimensions**: 28 columns x 31 rows (20px per tile).
- **Tile Values**:
  - `0`: Walkable empty space.
  - `1`: Wall.
  - `2`: Dot (food).
  - `3`: Ghost pen door (permeable to ghosts, wall to Pac-Man).
- **Tunnel**: Located at row `14` (`TUNNEL_ROW`); actors wrap horizontally around edges.

## Spec-Driven Development Workflow

- Specs belong in the `specs/` directory.
- Use `/spec` to design new features or requirements before writing code.
- Use `/spec-impl` to implement approved specs. `spec-impl` requires the spec status to be explicitly marked as `Approved` (or `Aprobado`) before execution begins.
