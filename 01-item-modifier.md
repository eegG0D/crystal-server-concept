# Item Modifier

A time-bounded "blessing" system that re-rolls stat bonuses on equipped items at configurable intensity tiers. The boosted stats decay back to base when the timer expires.

## What it does

When a player invokes the modifier on an item, the server picks a tier (Intensity 1..N), rolls bonus stats inside the tier's caps, and stamps them onto the item for the duration. The intensity tier determines both the magnitude and the per-roll count — Intensity 1 might add 1 minor stat for 30 minutes; Intensity 5 might add 5 major stats for 8 hours.

This is the in-game equivalent of a Path-of-Exile-style "blessing" or a Diablo-style temporary affix roll.

## Where to find it (server form)

- **Server.exe → Config → System → Item Modifier**
- Source: `Server.MirForms\Systems\ItemModifierConfigForm.cs`
- Settings persist into `Build\Server\Debug\Configs\Setup.ini` under the `[ItemModifier]` section.

## How to configure

| Field | What it does |
|---|---|
| **Enabled** | Master switch. Off → the in-game modifier NPC silently no-ops. |
| **Duration (minutes)** | How long the rolled bonuses last on an item after applying. |
| **Tier list** | One row per Intensity tier. Each row defines: max rolls (how many stats), per-stat cap, and credit cost. |
| **Daily cap (per character)** | Max applies per real-time day. |
| **Refund %** | If a player cancels before expiry, percentage refunded. |

## How players use it

1. Find the Item-Modifier NPC (configure its spawn in the map editor).
2. Open the dialog → drop an item into the slot.
3. Pick an Intensity tier (each costs more).
4. Pay → server applies the bonus → item now glows in inventory until the timer runs out.
5. Equipped item with active mod functions normally; expiry removes the bonus, base stats remain.

## How it works under the hood

- `Settings.ItemModifier*` static fields are read on server boot.
- The roll happens server-side in the NPC's `IF MODIFIERTRY ...` action; the engine clamps each rolled value to the tier cap.
- The modifier metadata is serialized into the `UserItem` save format with an `ExpiryTime` field — load/save handles it on every server restart.

## Common pitfalls

- **Intensity caps must be tuned to your itemBase.** Setting Intensity 5 to "+200 DC" on a vanilla server makes endgame trivial.
- **Refund % above 100 is a money printer.** Configurable but always set ≤ 100.
- **Duration affects DB save load.** Massive durations on many items inflate save sizes; the DB doesn't compress.
- **Daily cap is per character, not per account.** Rerolling an alt account is intended for testing only.

## Quick smoke test

```
@ITEMMODIFIER status   (if you've added a GM command for it)
```
Or just apply an Intensity 1 mod on a test character and verify in chat: `[ItemModifier] applied: +3 MaxDC, +2 MaxAC (Intensity 1, expires in 30m).`
