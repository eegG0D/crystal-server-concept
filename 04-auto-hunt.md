# Auto-Hunt

Passive auto-attack mode. The server polls the player's surroundings on a fixed cooldown and triggers an attack on the nearest hostile monster in range. Lets players AFK-grind low-tier mobs.

## What it does

Server-side timer per player: every `IntervalMs` ms (minimum 250), if the player is online and not in combat-locked state, find the closest monster within `Range` cells that is `IsAttackTarget(this) == true`, fire the player's basic attack, gain XP / loot per normal combat rules. Optionally a percentage XP multiplier is applied to incentivize subscribing to the feature.

This is the AFK farming feature most Mir2 private servers offer.

## Where to find it

- **Server.exe → Config → System → Auto-Hunt**
- Source: `Server.MirForms\Systems\AutoHuntConfigForm.cs`
- `Setup.ini` section `[AutoHunt]`.

## How to configure

| Field | What it does |
|---|---|
| **Enabled** | Master switch. |
| **Range (cells)** | Max distance from the player to consider a monster. |
| **Interval (ms)** | Minimum 250. Lower = faster auto-attacks (loadbearing). |
| **XP bonus %** | Multiplier applied to gained XP while auto-hunting is on. 0 = no bonus, 50 = +50%. |
| **Stop on full inventory** | Suspend auto-hunt when bag is full (otherwise drops get lost on the floor). |

## How players use it

1. Toggle via chat command (`@AUTOHUNT ON` / `@AUTOHUNT OFF`) or NPC.
2. Stand somewhere with monsters in range.
3. Server ticks every `IntervalMs`, fires attacks for them.
4. Logging out cancels auto-hunt; logging back in does NOT auto-resume (intentional).

## How it works under the hood

- A per-`PlayerObject` flag `AutoHunting` plus a `long AutoHuntNextAttack` timestamp.
- The Envir/MapManager process loop calls `Player.ProcessAutoHunt()` if `AutoHunting && Envir.Time >= AutoHuntNextAttack`.
- The method runs a small `FindAllNearby(Range)` and picks the lowest-HP attack target, then calls `Attack(direction, spell)` the same way a manual click would.

## Common pitfalls

- **Interval too low** turns the server into one big combat thread. 500–1000ms is the safe range.
- **Range too high** lets players AFK from off-screen, breaking the social loop. 5–8 cells is sane.
- **No "stop on full inventory" lets items vanish.** Always enable that flag in production.
- **Auto-hunt + invisibility** is a logical conflict: hidden players can't be aggroed but their auto-attacks DO reveal them. Document the interaction or disable invisibility while auto-hunting is on.

## Quick smoke test

Spawn a low-level mob (`@MOB Skeleton`), toggle auto-hunt on, walk near, watch the player auto-attack at the configured interval.
