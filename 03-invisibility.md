# Invisibility

Paid hide-from-target buff. Players pay credits to become invisible to monsters (and optionally other players) for a configurable duration.

## What it does

Applies a server-side buff that:
1. Sets the player's `Hidden = true` flag.
2. `HideFromTargets()` clears any in-flight monster aggro.
3. Monsters' `FindTarget` skips hidden players unless they have `CoolEye` + sufficient level.

This is the same engine path as the Assassin's Hiding skill; the difference is the trigger (credits, not a skill).

## Where to find it

- **Server.exe → Config → System → Invisibility**
- Source: `Server.MirForms\Systems\InvisibilityConfigForm.cs`
- Saves into `Setup.ini` under `[Invisibility]`.

## How to configure

| Field | What it does |
|---|---|
| **Enabled** | Master switch. |
| **Cost (credits)** | Per-activation price. |
| **Duration (minutes)** | How long the buff lasts. |
| **Refund %** | If a player cancels before expiry, percent of credits refunded. |
| **Target scope** | Monsters only / Players only / Both. (Caveat: PvP hiding can be abused — see pitfalls.) |

## How players use it

1. Talk to the Invisibility NPC (or use a configured chat command if you wired one).
2. Pay → server adds `BuffType.Hiding` for the configured duration.
3. Player appears semi-transparent to themselves, fully invisible to others (depending on scope and your client's draw code).
4. Attacking breaks invisibility immediately (server removes the buff in `Walk`/`Magic` paths).

## How it works under the hood

- `AddBuff(BuffType.Hiding, this, durationMs, Stats)` is the core call.
- Monster targeting check (`MonsterObject.FindTarget`): `if (ob.Hidden && (!CoolEye || Level < ob.Level)) continue;`
- The client renders hidden objects via `MonsterObject.Draw` — depending on which build of the client renderer you have, hidden objects either skip drawing or render at 50% opacity. See `Client\MirObjects\MonsterObject.cs:Draw`.

## Common pitfalls

- **PvP scope is exploit-prone.** If players can hide in PvP zones, they can stack hide + camp a respawn. Default scope to "Monsters only" unless your server is PvE only.
- **Hidden + Sneaking buffs stack** unintentionally. Double-check `Stat` totals aren't accidentally summed.
- **CoolEye monsters break hiding silently.** Higher-tier boss mobs almost always have CoolEye. Players will report "Invisibility didn't work" and they'll be right. Document which monsters see through it.

## Quick smoke test

Buy invisibility, walk into a normal mob's view radius, confirm it doesn't aggro. Walk into a CoolEye mob's view radius, confirm it does (this is the intended behavior).
