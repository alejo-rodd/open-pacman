# AGENTS.md

Pac-Man–style arcade game in vanilla HTML/CSS/JS. No build system, no
package manager, no tests, no linter, no formatter. The whole app is the
contents of `src/`.

## Run it

Static files only. To play:

- Open `src/index.html` directly in a browser, **or**
- Serve the folder with any one-liner (e.g. `python3 -m http.server` from the
  repo root, then visit `/src/index.html`).

There is no `npm test`, no build step, no CI. Don't look for one.

## Source layout (the only one)

```
src/
  index.html        # entry; loads the four scripts in order
  css/style.css     # single stylesheet (overlay + canvas wrapper)
  js/maze.js        # MAZE_STR (ASCII grid) + MAZE, TUNNEL_ROW, starts
  js/game.js        # createGame, update, DIRS; mutates game.grid
  js/render.js      # draw; reads game.grid (never MAZE) for live dots
  js/main.js        # RAF loop, keyboard, overlay
```

## Things an agent will get wrong unless told

- **Script load order in `index.html` is mandatory.** `maze.js` → `game.js` →
  `render.js` → `main.js`. They communicate through globals attached to
  `window` (`MAZE`, `TUNNEL_ROW`, `PACMAN_START`, `GHOST_STARTS`, `createGame`,
  `update`, `draw`, `DIRS`). Do not convert to ES modules without rewriting
  every cross-file reference.
- **The maze grid is `MAZE_STR` in `src/js/maze.js`** — 31 ASCII rows of 28
  chars. Tiles: `#` wall (1), `.` dot (2), `-` pen door (3), space empty (0).
  Coordinates are `(x, y)`, origin top-left, `x ∈ [0,27]`, `y ∈ [0,30]`.
  `MAZE` is the pristine numeric copy; **never mutate it.** `createGame`
  deep-copies it into `game.grid`, which is what the renderer reads.
- **Canvas is 560×620 px and `TILE = 20` in `render.js`.** That is 28×31 cells.
  If you change one, change the other two (`index.html` width/height and the
  `#game-wrap` size in `style.css`).
- **Game state machine is a string on `game.state`**: `'start' | 'playing' |
  'won' | 'lost'`. Lives start at 3. Win when `dotsRemaining <= 0`. Lose when
  `lives <= 0` after a ghost collision. Collision uses `collides()` with a
  half-cell tolerance.
- **Pacman and ghosts only decide direction when aligned to a cell centre**
  (`aligned()` helper in `game.js`). Movement is sub-cell fractional per
  frame. Tunnel wrap only fires on `TUNNEL_ROW` (row 14).
- **Ghosts can't reverse direction mid-corridor.** `decideGhost()` filters out
  the opposite of `g.dir`; reversal is only allowed as a dead-end fallback.
  One ghost is `'hunter'` (chases via Manhattan distance), the other is
  `'random'`.
- **No external assets, no fonts loaded over the network.** The CSS font
  stack falls back to `"Courier New", monospace` if "Press Start 2P" is
  missing — don't add a webfont CDN without checking.
- **Touch / mobile input is not implemented.** Keyboard only
  (ArrowLeft/Right/Up/Down).

## Style conventions to match in new code

- 2-space indent, single quotes, spaces inside parentheses (`f( x )`).
- `const` / `let`, no `var`. Semicolons at end of statements.
- Plain `function` declarations, not arrow assignments, for top-level
  functions.
- Cross-file exports are `window.foo = foo` at the bottom of the file.
  Don't introduce a bundler or module system.
- Comments are short `//` headers at the top of each file stating what the
  file owns and which globals it depends on. Match that pattern.

## Spec-driven workflow (this repo's defining habit)

The two skills in `.agents/skills/` are the normal way to do work:

- **`/spec <feature>`** — design a feature. Writes `specs/NN-slug.md` in
  `Draft` state. The number is the next sequential zero-padded index in
  `specs/`. Do **not** write code during this command.
- **`/spec-impl <NN-slug>`** — implement an approved spec. It refuses to
  run unless the spec's state line means "Approved" (`Approved` /
  `Aprobado` / etc.). It creates a branch `spec-NN-slug` (controlled by
  `specs/.spec-config.yml` `AutoCreateBranch`, default `true`), shows the
  spec summary, and **pauses for confirmation after every plan step**.
  It never commits — committing is the human's call.

Conventions carried by both skills:

- Specs and comments in this repo mix Spanish and English; match the
  language of the most recent spec or the user's prompt.
- `specs/.spec-config.yml` may exist to tweak `AutoCreateBranch`. If absent,
  it defaults to `true`. Don't overwrite it once created.

## Git

- Default branch work happens on `spec-NN-slug` branches created by
  `/spec-impl`. No direct commits to `main` for spec-driven work.
- The agent never commits unless explicitly asked. Same for push, merge,
  and PR creation.