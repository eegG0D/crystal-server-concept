# Mob Chaser

Player-initiated teleport-to-monster. Lets players spend credits (or gold) to warp to the nearest spawned instance of a named mob.

## What it does

Server looks up live monster instances of a chosen name across all maps (or the player's current map, depending on config), picks the closest match within an optional level-offset window, then teleports the player to that monster's tile.

It's the in-game equivalent of "fast travel to the next boss" — a quality-of-life feature that bypasses long traversal grinds, gated behind a credit cost so it doesn't trivialize content.

## Where to find it

- **Server.exe → Config → System → Mob-Chaser**
- Source: `Server.MirForms\Systems\MobChaserConfigForm.cs`
- `Setup.ini` section `[MobChaser]`.

## How to configure

| Field | What it does |
|---|---|
| **Enabled** | Master switch. |
| **Cost (credits or gold)** | Per-use price. Toggle between credit and gold pricing. |
| **Level offset min/max** | Restrict teleport targets to monsters within `[Player.Level + min, Player.Level + max]`. Set to (-99, +99) for unrestricted. |
| **Allow cross-map** | If false, only mobs on the player's current map are eligible. |
| **Cooldown (s)** | Per-player cooldown between uses. |

## How players use it

1. Open the Mob-Chaser NPC dialog.
2. Type the monster name (or pick from a list).
3. Pay → server teleports.
4. If no instance matches the level filter: server replies "No matching monster found."

## How it works under the hood

- Scans `Envir.MonsterInfoList` by name, then `Envir.MapList[].MonsterCount` for live instances.
- Validates the target's `Level` against the configured offset window.
- Uses the same `Teleport(map, location)` path as regular portal travel — respects `map.NoTeleport`, `Settings.DisabledMaps`, etc.

## Common pitfalls

- **Boss-tier monsters with map flags** like `NoReconnect` or `NoTeleport` reject the teleport but still charge credits. Wrap the cost in a "did the teleport actually succeed" check.
- **Cross-map teleports during conquest** can drop players into enemy guild territory mid-siege. Document or disable.
- **No cooldown** lets players rapid-chase a boss between every spawn cycle, trivializing scarcity. 30–60s cooldown is the typical balance.

## Quick smoke test

Spawn a test monster (`@MOB Skeleton`), open Mob-Chaser, type "Skeleton", pay, verify teleport. Try a non-existent name — should reply gracefully.
