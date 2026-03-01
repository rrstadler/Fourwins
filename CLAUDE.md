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
├── README.md    # Brief project description
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

All code lives in a single file, split into clearly commented sections:

### Constants
```
ROWS = 6, COLS = 7
HUMAN = 1, CPU = 2, EMPTY = 0
DEPTH = { easy: 0, medium: 3, hard: 7 }
```

### State variables
| Variable | Type | Purpose |
|---|---|---|
| `board` | `number[][]` | 6×7 grid; values 0/1/2 |
| `currentTurn` | `1\|2` | Whose turn it is |
| `gameOver` | `boolean` | Blocks input after win/draw |
| `difficulty` | `'easy'\|'medium'\|'hard'` | Controls AI depth |
| `animating` | `boolean` | Blocks clicks during drop animation or AI delay |
| `score` | `{p1,p2,draw}` | Cumulative score across games |

### Key Functions

| Function | Description |
|---|---|
| `createBoard()` | Returns a fresh 6×7 zero-filled array |
| `getValidCols(b)` | Returns columns where row 0 is still empty |
| `dropPiece(b, col, player)` | Mutates `b`, returns the row where piece landed |
| `undropPiece(b, col)` | Reverses `dropPiece` (used by minimax) |
| `checkWinner(b)` | Returns `{player, cells}` or `null` |
| `isBoardFull(b)` | Returns `true` when no moves remain |
| `evaluateBoard(b, player)` | Heuristic score for minimax leaf nodes |
| `scoreWindow(window, player)` | Scores a 4-cell window for one player |
| `minimax(b, depth, α, β, max)` | Recursive minimax with alpha-beta pruning |
| `getBestMove(b, depth)` | Runs minimax for all valid columns, returns best |
| `chooseComputerCol()` | Dispatches to random (easy) or minimax (medium/hard) |
| `initBoard()` | Resets state, rebuilds DOM on first call only |
| `handleClick(col)` | Entry point for human moves |
| `playMove(col, player)` | Drops piece + triggers CSS drop animation |
| `afterMove(player)` | Checks win/draw, switches turn, triggers CPU |
| `renderBoard(winCells?)` | Syncs DOM classes with board state |
| `setStatus(text, dotClass)` | Updates the status bar below the grid |

### AI Design

| Difficulty | Strategy | Lookahead |
|---|---|---|
| Easy | Pure random from valid columns | — |
| Medium | Minimax + alpha-beta | depth 3 |
| Hard | Minimax + alpha-beta | depth 7 |

**Heuristic** (`evaluateBoard`):
- +6 per CPU piece in the center column
- Scores every 4-cell window (horizontal, vertical, both diagonals)
- Per window: +100 000 for 4-in-a-row, +10 for 3+empty, +3 for 2+2 empty, -80 for opponent 3+empty

**Alpha-beta**: standard negamax-style; columns ordered center-out for better pruning.

---

## CSS / Visual Design

- Dark blue theme with a gradient background
- Red discs = Human player; Yellow discs = CPU
- Drop animation: `@keyframes drop` (bounce effect, 380 ms)
- Winning discs: `@keyframes pulse-win` (scale + glow, infinite)
- Column hover: arrow turns red, column cells lighten
- No external assets or fonts (uses `system-ui`)

---

## Development Conventions

### Editing the game
- Everything is in `index.html`. Use a `<style>` block for CSS, `<script>` for JS.
- Do **not** split into separate files unless explicitly asked — the single-file approach is intentional.
- Keep the JS comment section headers (`// ── Section ──`) so the file stays navigable.

### Testing
- No automated test suite. Test manually in the browser.
- Key scenarios to verify after any change:
  1. Human wins (horizontal, vertical, diagonal ↘, diagonal ↗)
  2. CPU wins
  3. Draw (fill the board)
  4. New Game resets board and score display remains
  5. Difficulty buttons restart the game with correct AI depth
  6. Clicking a full column is ignored
  7. No input is accepted while the CPU is "thinking" (animating=true)

### Adding features
- New difficulty level: add an entry to `DEPTH` and a `.diff-btn` in the HTML.
- Sound effects: hook into `afterMove` and `playMove`.
- Network multiplayer: replace `chooseComputerCol()` with a WebSocket call.

---

## Branch & Git

- Active feature branch: `claude/connect-four-game-foEK3`
- Push target: `origin claude/connect-four-game-foEK3`
- Do **not** push directly to `main` / `master`.
