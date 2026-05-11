# Auto-Farm

Passive drop accumulator. Server periodically rolls loot from a configured drop table and stuffs it into a per-player virtual bag. Players retrieve the items via the in-game Auto-Farm dialog.

## What it does

Auto-Farm is the "offline progression" feature. Unlike Auto-Hunt (which requires the player to be standing near monsters), Auto-Farm runs purely off a timer + drop policy: every `IntervalMs`, the server rolls 1+ items from a level-appropriate drop pool into the player's Auto-Farm bag. Players can be anywhere on the map — even AFK in a safe zone — and the system keeps ticking. When they want the loot, they open the dialog and right-click items to move them into real inventory.

## Where to find it

- **Server config form:** Server.exe → Config → System → Auto-Farm  
  Source: `Server.MirForms\Systems\AutoFarmConfigForm.cs`  
  `Setup.ini` section: `[AutoFarm]`.
- **Client dialog:** `/farm` in chat → opens `Client\MirScenes\Dialogs\AutoFarmDialog.cs` (10×6 grid).

## How to configure (server)

| Field | What it does |
|---|---|
| **Enabled** | Master switch. |
| **Interval (ms)** | Minimum 1000. Time between item rolls. |
| **Inventory slots** | 1–60. Bag size for accumulated drops. |
| **Level tolerance** | How far above/below the player's level the drop pool may be (item level filter). |
| **Max retries per tick** | If rolls keep producing items the player already has stacked at cap, retry up to N times before skipping. |

## How players use it

1. Type `/farm` to open the bag dialog.
2. Toggle the "Enabled" radio. The server begins rolling on the next tick.
3. As items accumulate, the grid populates; you see countdown bars for "next item" and "overall session."
4. Right-click an item to move it into real inventory.
5. Filter by ItemType via the dropdown (client-side filter — doesn't change server behavior).

## How it works under the hood

- Server keeps a `UserItem[]` per `CharacterInfo.AutoFarmInventory` (or similar field) and a `long NextAutoFarmTick`.
- On every server tick, if `Envir.Time >= NextAutoFarmTick` and Auto-Farm is enabled for that player, roll a drop. Match the player's level ± tolerance against the global drop pool.
- The roll uses the same drop probability tables as normal monster loot.
- Server pushes `S.AutoFarmState`, `S.AutoFarmAdd`, `S.AutoFarmRemove` packets — the dialog is server-authoritative.
- `C.AutoFarmTake` is what the right-click sends to move loot into the inventory.

## Common pitfalls

- **Interval too low** = item printer. 20 min per drop is typical.
- **Pool too generous** = AFK > active play. The bag should be a side income, not a replacement.
- **Inventory slot cap.** When the bag is full, the system has nowhere to put new rolls. Either pause auto-farming or drop the surplus.
- **Players leaving auto-farm running while logged off.** Decide policy: pause on logout (default), or continue for X hours after logout.

## Quick smoke test

Enable, set Interval to 5 s for testing, watch the dialog populate live. Right-click an item → it lands in real inventory. Disable when done.
