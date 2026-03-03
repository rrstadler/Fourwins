# CLAUDE.md — 4 Wins (Connect Four)

This file provides context for AI assistants working in this repository.

---

## Project Overview

A single-file browser Connect Four game.
Human (Red) plays against a CPU opponent (Yellow).
The first player to connect **4 discs** in a row — horizontally, vertically, or diagonally — wins.

## Repository Structure

```
Fourwins/
├── index.html   # Entire game: HTML layout + embedded CSS + embedded JS
├── README.md    # Project description and feature list
└── CLAUDE.md    # This file
```

There is no build step, no package manager, no framework.
Open `index.html` directly in any modern browser to play.

---

## Game Rules

- Board: **7 columns × 6 rows** (standard Connect Four dimensions)
- Players alternate turns; Human always goes first
- A piece falls to the **lowest empty row** of the chosen column
- Win condition: **4 consecutive pieces** in any direction (horizontal / vertical / diagonal)
- Draw: board is full with no winner

---

## Architecture (`index.html`)

All code lives in a single file with five clearly delimited sections, each preceded by a banner comment of the form:

```js
// ─────────────────────────────────────────────
//  Section Name
// ─────────────────────────────────────────────
```

The five sections in order are: **Constants → State → DOM references → Board logic → AI — Heuristic → AI — Minimax → Rendering → Game flow → Event listeners → Start**.

### HTML structure

```
<body>
  <h1>                        — gradient title
  <p.subtitle>                — tagline
  <div.difficulty-row>        — Easy / Medium / Hard buttons (.diff-btn[data-level])
  <div.explain-row>           — Explain Mode toggle (#explainToggleBtn)
  <div.board-wrapper>
    <div#colArrows>           — 7 ▼ arrow divs (.col-arrow[data-arrow=c])
    <div#boardGrid>           — 42 cell divs (.cell[data-r][data-c])
  <div#statusBar>             — coloured dot + <span#statusText> + #explainInfoBtn (ⓘ)
  <div.score-row>             — <span#scoreP1> <span#scoreDraw> <span#scoreP2>
  <button#newGameBtn>
  <div.legend>
  <div#explainPopup>          — fixed overlay; contains .explain-popup-card with reason + lookahead
```

### Constants

```js
ROWS  = 6,  COLS = 7
HUMAN = 1,  CPU  = 2,  EMPTY = 0
DEPTH = { easy: 0, medium: 3, hard: 7 }
// Note: DEPTH.easy is never used — Easy mode bypasses minimax entirely
```

### State variables

| Variable | Type | Purpose |
|---|---|---|
| `board` | `number[][]` | 6×7 grid; values 0 / 1 / 2 |
| `currentTurn` | `1\|2` | Whose turn it is |
| `gameOver` | `boolean` | Blocks input after win/draw |
| `difficulty` | `'easy'\|'medium'\|'hard'` | Controls AI strategy |
| `hoveredCol` | `number` | Column index under the cursor (-1 = none); drives hover highlighting |
| `animating` | `boolean` | Blocks clicks during drop animation or AI delay |
| `score` | `{p1,p2,draw}` | Cumulative score across games (never resets on New Game) |
| `explainMode` | `boolean` | Whether explanation mode is enabled |
| `lastCpuExplanation` | `{reason, lookahead}\|null` | Plain-language explanation of the most recent CPU move |

### Key Functions

#### Board logic

| Function | Signature | Description |
|---|---|---|
| `createBoard()` | `→ number[][]` | Returns a fresh 6×7 zero-filled array |
| `getValidCols(b)` | `→ number[]` | Returns columns where `b[0][c] === EMPTY` |
| `dropPiece(b, col, player)` | `→ number` | Mutates `b`; returns row where piece landed, -1 if full |
| `undropPiece(b, col)` | `→ void` | Clears the topmost filled cell in `col` (used by minimax backtracking) |
| `checkWinner(b)` | `→ {player, cells}\|null` | Scans all 4 directions; returns winning player and the 4 winning `[r,c]` pairs |
| `isBoardFull(b)` | `→ boolean` | `true` when every cell in row 0 is filled |

#### AI — Heuristic

| Function | Signature | Description |
|---|---|---|
| `scoreWindow(window, player)` | `→ number` | Scores a 4-element array for `player` (see weights below) |
| `evaluateBoard(b, player)` | `→ number` | Aggregates `scoreWindow` over all horizontal, vertical, and diagonal windows; adds center-column bonus |

**`scoreWindow` weights:**

| Pattern | Score |
|---|---|
| 4 of player | +100 000 |
| 3 of player + 1 empty | +10 |
| 2 of player + 2 empty | +3 |
| 3 of opponent + 1 empty | −80 |
| anything else | 0 |

**`evaluateBoard` center-column bonus:** +6 per CPU piece in column 3.

#### AI — Minimax

| Function | Signature | Description |
|---|---|---|
| `minimax(b, depth, alpha, beta, maximizing)` | `→ number` | Recursive minimax with alpha-beta pruning; returns heuristic score |
| `getBestMove(b, depth)` | `→ number` | Tries each valid column (ordered center-out), runs minimax at `depth-1`, returns best column |
| `chooseComputerCol()` | `→ number` | Easy → random valid column; Medium/Hard → `getBestMove` at the configured depth |

**AI difficulty:**

| Level | `DEPTH` | Strategy |
|---|---|---|
| Easy | — | `Math.random()` pick from `getValidCols` |
| Medium | 3 | `getBestMove` → minimax depth 3 |
| Hard | 7 | `getBestMove` → minimax depth 7 with alpha-beta |

Column iteration order in `getBestMove` is center-out (`sort by |col - 3|`) to maximise alpha-beta cut-offs.

#### Rendering

| Function | Signature | Description |
|---|---|---|
| `renderBoard(winCells?)` | `→ void` | Rebuilds all `.cell` class lists from `board` state; applies `col-hover` and `win-cell` |
| `setStatus(text, dotClass)` | `→ void` | Sets `#statusText` and the coloured dot class (`dot-player1 / dot-player2 / dot-draw`) |
| `updateScoreDisplay()` | `→ void` | Writes `score.p1 / p2 / draw` into the three `<span>` score elements |

#### Explanation mode

| Function | Signature | Description |
|---|---|---|
| `generateExplanation(col)` | `→ {reason, lookahead}` | Called inside `chooseComputerCol()` **before** the piece is placed. Checks for immediate win, blocking move, or strategic play; returns plain-language strings for the popup |

`generateExplanation` priority order:
1. **Immediate win** — placing CPU in `col` → `checkWinner` returns CPU
2. **Block** — placing HUMAN in `col` → `checkWinner` returns HUMAN
3. **Strategic fallback** — best position after lookahead

After each CPU move, when `explainMode === true` and `lastCpuExplanation !== null`, the popup (`#explainPopup`) opens **automatically** with the explanation pre-filled. The `#explainInfoBtn` (ⓘ) is also revealed so the user can re-open the popup if they closed it. The popup is dismissed by the close button or clicking the overlay backdrop.

#### Game flow

| Function | Signature | Description |
|---|---|---|
| `initBoard()` | `→ void` | Resets `board`, `currentTurn`, `gameOver`, `animating`, `hoveredCol`, `lastCpuExplanation`; builds DOM on first call only |
| `handleClick(col)` | `→ void` | Guards (`gameOver`, `animating`, `currentTurn`, full column) then calls `playMove` |
| `playMove(col, player)` | `→ void` | Calls `dropPiece`, adds CSS drop animation, calls `afterMove` on `animationend` |
| `afterMove(player)` | `→ void` | Checks win/draw → updates score/status; otherwise switches turn and schedules CPU move via `setTimeout` |

**CPU think delay:** 200 ms (easy) / 300 ms (medium) / 400 ms (hard) — ensures the board re-paints before blocking JS computation.

---

## CSS / Visual Design

### CSS custom properties (`:root`)

| Property | Value | Purpose |
|---|---|---|
| `--cell` | `clamp(32px, min(calc((100vw-120px)/7), calc((100vh-300px)/6)), 60px)` | Fluid cell size — respects both viewport width and height |
| `--gap` | `8px` | Gap between cells and arrow columns |

- Dark blue gradient background (`#1a1a2e → #16213e → #0f3460`)
- Red discs (`#e94560`) = Human; Yellow discs (`#f5a623`) = CPU
- Cell colours use `radial-gradient` for a 3-D sphere look
- Column hover: cells lighten (`#122040`); arrow turns red
- Drop animation: `@keyframes drop` — translateY from `calc(var(--cell) * -6)` with a two-stage bounce, 380 ms
- Win cells: `@keyframes pulse-win` — scale 1 → 1.12 + white glow, infinite alternate, 700 ms
- No external assets, images, or web fonts — uses `'Segoe UI', system-ui`

### CSS class reference

| Class | Applied to | Meaning |
|---|---|---|
| `.player1` | `.cell` | Human disc (red gradient) |
| `.player2` | `.cell` | CPU disc (yellow gradient) |
| `.win-cell` | `.cell` | One of the 4 winning discs (pulse animation) |
| `.col-hover` | `.cell` | Empty cell in the hovered column |
| `.dropping` | `.cell` | Cell currently animating |
| `.hovered` | `.col-arrow` | Arrow above the hovered column |
| `.active` | `.diff-btn` | Currently selected difficulty button |
| `.active` | `.explain-toggle-btn` | Explain mode is enabled |

---

## Development Conventions

### Editing the game
- Everything is in `index.html` — `<style>` for CSS, `<script>` for JS.
- Do **not** split into separate files unless explicitly asked; the single-file approach is intentional.
- Preserve the `// ─────────────────────` section-header comments so the file stays navigable.
- DOM is built exactly **once** inside `initBoard()` (guarded by `if (boardGrid.children.length === 0)`). Subsequent calls to `initBoard()` only reset state and re-render.

### Testing
No automated test suite. Test manually in the browser after any change:

1. Human wins — horizontal, vertical, diagonal ↘, diagonal ↗
2. CPU wins at each difficulty level
3. Draw — fill the board completely
4. New Game resets the board; cumulative score is preserved
5. Difficulty buttons restart the game and use the correct AI depth
6. Clicking a full column is ignored
7. No input is accepted while `animating === true` (during drop or CPU think delay)
8. Hovering columns highlights them; arrow appears only on Human's turn

### Adding features
- **New difficulty level:** add an entry to `DEPTH` and a `.diff-btn[data-level]` button in the HTML.
- **Sound effects:** hook `playMove` (on drop) and `afterMove` (on win/draw).
- **Two-player mode:** bypass `chooseComputerCol()` and switch `currentTurn` for both players.
- **Network multiplayer:** replace `chooseComputerCol()` with a WebSocket call.
- **Responsive layout:** cell size is driven by the `--cell` CSS custom property on `:root` — a `clamp(32px, min(vw-formula, vh-formula), 60px)` expression. All grid column/row definitions, `.cell` width/height, and the `@keyframes drop` translateY reference `var(--cell)`, so changing the formula automatically updates the entire layout.

---

## Branch & Git

- Active feature branch: `claude/connect-four-game-foEK3`
- Push target: `origin claude/connect-four-game-foEK3`
- Do **not** push directly to `main` / `master`.
