# Map-Item Teleport

Teleport-to-a-dropped-item. The server tracks all items lying on the ground (per map) and lets players warp directly to a specific item from a UI list.

## What it does

When players want to recover loot they didn't grab in time (or want to teleport to expensive boss drops they saw in chat), they open the Map-Item TP dialog, see a list of every ground item within the configured scope, click one → server validates ownership / proximity / cost → teleports.

It's "find my stuff" + "fast-track to high-value drops on shared maps."

## Where to find it

- **Server.exe → Config → System → Map-Item TP**
- Source: `Server.MirForms\Systems\MapItemTeleportConfigForm.cs`
- `Setup.ini` section `[MapItemTeleport]`.

## How to configure

| Field | What it does |
|---|---|
| **Enabled** | Master switch. |
| **Require GM** | If true, only GMs can use the feature. |
| **Restrict to current map** | If true, items on other maps don't appear in the list. |
| **Allow gold payment** | If true, accepts gold instead of credits. |
| **Cost per teleport** | Credits (or gold) per use. |
| **Max items per map listed** | 1–60, caps the list size sent to the client. |
| **Refresh cooldown (ms)** | Minimum 100. Prevents spamming list refreshes. |

## How players use it

1. Open the Map-Item TP dialog (chat command or hotkey).
2. Browse the list — each entry shows the item name, quantity, the map, and a quality indicator.
3. Click → pay → teleport to the item's tile.
4. Pick up the item the normal way.

## How it works under the hood

- The server maintains a per-map `List<ItemObject>` (already exists for ItemObjects rendered on tiles).
- A `S.MapItemTpList` packet sends the filtered window to the client.
- `C.MapItemTpGo` packet carries the chosen `ObjectID`; server validates the item still exists, the player can pay, and the destination is a `NoTeleport == false` map.
- Teleport uses the same `Teleport(map, location)` path as portals.

## Common pitfalls

- **Item list freshness.** If many items get picked up between list-send and player click, the chosen ObjectID may be stale. Server must re-validate.
- **Per-map item caps too low.** On a packed loot map, the list might omit the very item the player wants. Cap at 60 and consider pagination.
- **Allowing gold payment on a loot-heavy server** makes the feature spam-cheap. Credit-only is the safer default.

## Quick smoke test

Drop a stack of gold on a map, walk away, open Map-Item TP, find the gold pile in the list, teleport back, pick it up.
