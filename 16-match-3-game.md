# Match-3 Game

Built-in Match-3 puzzle minigame, fully client-side. Open with `/m3`, click adjacent gems to swap, cascade chains for score. Includes power-ups (Bomb 3×3 explosion, Super color-nuke), Hint, Shuffle, and New Game.

## What it does

A drop-in Bejeweled-style minigame embedded in the Mir2 client. The whole game (board state, swap logic, cascade resolution, gravity, shuffle, power-up creation, scoring) runs in the player's process — no server roundtrip per move. Independent of all other features.

## Where to find it

- **Engine:** `Client\MirGames\Match3Engine.cs`
- **Dialog:** `Client\MirScenes\Dialogs\Match3Dialog.cs`
- **Open in-game:** `/m3` (or `/match3`) in chat → toggles the window.
- **Wired in:** `GameScene` constructs the dialog (`Visible=false` initially), `MainDialogs.cs` parses the `/m3` chat command.

## Gameplay

1. Type `/m3`. A draggable window opens, centered top-left.
2. The 8×8 grid fills with 6 base gem colors (Red, Blue, Green, Yellow, Purple, White).
3. Left-click a gem to select (gold border).
4. Left-click an **adjacent** gem (one orthogonal step) to attempt a swap.
   - If the swap creates a match of 3+ → cascade triggers, score climbs.
   - If it doesn't → engine reverts the swap.
5. Right-click anywhere → clears selection.
6. Buttons:
   - **Hint** — highlights a cell that has a legal swap available (cyan border, 2 seconds).
   - **Shuffle** — randomizes the board (use when stuck).
   - **New** — resets the board, score, and combo.
   - **X** (top-right) — closes the dialog (state preserved across reopen).

## Scoring

- Base: **10 × Combo** per cleared cell. Combo starts at 1 on each player swap, increments per chain step during cascade.
- Match-4 → spawns a **Bomb** at the match center (50-point bonus on detonation).
- Match-5 → spawns a **Super** (`★`) at the match center (100-point bonus).
- L / T intersection of same-color matches → spawns a **Bomb** at the intersection.
- Bomb in cascade → 3×3 explosion, chain-reactive (bombs caught in another bomb's blast also pop).

## Power-up interactions

| Move | Result |
|---|---|
| Bomb + adjacent swap | Detonates 3×3 area on cascade. |
| Super (`★`) + color gem swap | Destroys every gem of that color on the board. |
| Super + Super swap | Nukes the entire board. |
| Bomb + Bomb (cascade overlap) | Cascading chain explosion. |

## Random spawns

When the board refills (gravity drops new gems at the top), there's a small chance the new gem is a power-up:
- 2% chance per new gem → Bomb.
- 0.5% chance per new gem → Super.

Tunable in `Match3Engine.ApplyGravity()`.

## How it works under the hood

`Client.MirGames.Match3Engine` is pure logic, no I/O. Public API:

```csharp
new Match3Engine()                    // Constructs + InitializeGrid (no starting matches, guaranteed playable)
bool TrySwap(x1, y1, x2, y2)          // Returns true if swap created a match
List<Point> FindMatches()             // Returns all currently-matched cells
int RemoveMatches(List<Point>)        // Clears + scores + spawns power-ups; returns points gained
bool ApplyGravity()                   // Drops gems down; refills top; returns true if anything moved
Point GetHint()                       // Returns coords of a cell with a valid swap, or (-1,-1)
bool HasPossibleMove()                // GetHint().X != -1
void Shuffle()                         // Fisher-Yates shuffle of all gems; falls back to full reset if no valid move after
```

The dialog drives a 2-stage cascade loop on `BeforeDraw`:

```
stage 0: FindMatches → RemoveMatches → next stage
                    → no matches → exit resolve, auto-shuffle if stuck
stage 1: ApplyGravity → back to stage 0
```

One stage per `ResolveStepIntervalMs` (default 100 ms) so cascades animate.

## Rendering

Each cell is a child `MirControl` with a `BackColour` mapped from `GemType`:

| Gem | Color |
|---|---|
| Red | `(200, 60, 70)` crimson |
| Blue | `(60, 120, 220)` royal blue |
| Green | `(60, 180, 80)` lime |
| Yellow | `(220, 200, 60)` gold |
| Purple | `(160, 80, 200)` violet |
| White | `(230, 230, 240)` |
| Bomb | dark with "B" label |
| Super | bright magenta with `★` |

Selected cells get a **Gold** border; hinted cells get a **Cyan** border.

## Customizing the visuals

To swap colored cells for `items.lib` icons, replace the `ColorFor`/`SymbolFor` maps in `Match3Dialog.cs` with an `ImageFor` map → call `Libraries.Items.Draw(image, x, y)` from a `BeforeDraw` handler instead of setting `BackColour`. Pick 8 item indices that visually distinguish the gem types.

## Common pitfalls

- **Dialog state survives close/reopen** by design. To reset on close, call `NewGame()` from the close button or `Hide()` override.
- **Power-up spawn locations.** `_pendingPowerups` is populated during `FindMatches()` and consumed during `RemoveMatches()`. If you call them in a different order the power-ups appear in the wrong cells. Always pair them.
- **Bomb in a corner** explodes a smaller area (cells outside the grid are skipped). Expected.
- **Movable + cell clicks.** The dialog has `Movable = true` but the grid cells are child controls that handle their own MouseDown. Drag the header area (between title and close-X) or the footer area to move the window.

## Quick smoke test

1. `/m3` in chat → dialog opens.
2. Click two adjacent gems of the same color → swap should reject (no match) and snap back.
3. Click two adjacent gems where one creates a row of 3 → cascade fires, score climbs.
4. Wait for a Match-4 → a Bomb appears (dark cell with "B").
5. Detonate the Bomb (swap it adjacent) → 3×3 explosion.
6. Press Hint → cyan-bordered cell glows briefly. Verify it has a valid swap.
7. Press Shuffle → board reshuffles, score unchanged.
8. Press New → score resets to 0, fresh board.
