# Stat Editor

A character respec / stat-allocation NPC that lets players spend a point pool across DC, MC, SC, HP, MP, AC, MAC etc., bought with credits.

## What it does

Each character earns or buys a fixed total of "stat points" and distributes them across an arbitrary set of stats. Each stat has its own cap and per-point credit price. The editor is server-authoritative: the client sends "add +1 to DC", the server validates remaining budget + caps + credit balance, then commits.

This is the closest the game has to a Diablo 2-style respec.

## Where to find it

- **Server.exe → Config → System → Stat Editor**
- Source: `Server.MirForms\Systems\StatEditorConfigForm.cs`
- Saves into `Setup.ini` under `[StatEditor]`.

## How to configure

| Field | What it does |
|---|---|
| **Enabled** | Master switch. |
| **Max entries** | How many distinct stat lines the editor shows (one row per editable stat). |
| **Per-stat cap** | The hard ceiling each stat can be pumped to. |
| **Total point pool** | The budget every character gets to start. (Optionally grows with level — see `LevelBonusPoints` field.) |
| **Base credit cost** | First point of any stat costs this many credits. |
| **Per-point increment** | Each subsequent point costs `base + (current * increment)`. Linear ramp. |
| **Reset cost** | One-shot credit cost to wipe and reallocate. |

## How players use it

1. Talk to the Stat-Editor NPC.
2. Spend points across DC/MC/SC/HP/etc. via UI buttons.
3. Confirm → server validates and deducts credits.
4. Stats are immediately applied (`Stats[Stat.X]` updated, `RefreshAll()` called).
5. To re-spec: pay the reset cost → all points returned to the pool, all bonuses cleared.

## How it works under the hood

- `Settings.StatEditor*` defines the available stats and pricing.
- The bonus values live on `CharacterInfo` in a separate `StatEditorBonuses` field so they survive level-ups and item swaps.
- On every `RefreshAll()` the editor adds its bonuses to the player's combat stats; the cost is at most one extra hashtable lookup per stat.

## Common pitfalls

- **Per-point increment of 0** makes the system pay-to-win — no diminishing returns. Use at least linear (e.g., increment = base) for balance.
- **Cap too high relative to vanilla itemization** breaks build diversity.
- **Reset cost should not be free.** Free resets encourage min-maxing each fight which slows the server. Charge enough to make it deliberate.

## Quick smoke test

Open the NPC dialog, add +1 to DC, confirm credit deducted, log out, log back in, verify stat persists.
