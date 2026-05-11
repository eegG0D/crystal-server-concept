# Quest Tracker

In-game quest log + NPC teleport. Browse every quest in the world with color-coded status, jump to the giving NPC with one click.

## What it does

Quest Tracker is the client-side dialog that replaces "wandering town looking for ! markers." It pulls the full quest list from the server (chunked), shows them in a searchable 5-column ListView (Quest name, Chapter, Status, NPC, Index), and offers a Teleport button that moves the player to the quest's NPC.

Status is color-coded so high-value quests jump out at a glance.

## Where to find it

- **Client dialog:** `/qt` (or `/quests`) → toggles `Client\MirScenes\Dialogs\QuestTrackerDialog.cs`.
- **Server packets:** `S.QuestTrackerList` (chunked quest data), `S.QuestTrackerResult` (teleport / action result).
- **Server request packets:** `C.QuestTrackerListReq`, `C.QuestTrackerGo`.

## How players use it

1. Type `/qt` in chat.
2. The dialog requests the list; server sends in chunks (default ~50 per chunk to keep packets small).
3. Each row shows: Quest title, Chapter/Group, Status, NPC name, the quest's internal Index.
4. Filter by typing in the search box.
5. Click a quest → click the "Teleport" button → server validates and moves you to the NPC's map+location.

## Status colors

| Status | Color | Meaning |
|---|---|---|
| Locked | Grey | Level/class/prereq not met. |
| Available | Cyan | Can be picked up right now. |
| In progress | Yellow | Started but not turned in. |
| Completed | Green | Already turned in. |

## How it works under the hood

- Server-side: `Envir.QuestInfoList` is the source of truth. The list-build code computes `Status` for each quest per-player.
- `S.QuestTrackerList` carries Offset, Total, and an Entry[] (QuestIndex, Order, Status, RequiredMinLevel, RequiredMaxLevel, Name, Group, NpcName, NpcMapIndex, NpcX, NpcY, MapName, ProgressSummary).
- Teleport uses normal `Teleport(map, location)` path; if NPC is on `NoTeleport` map, the server replies with `S.QuestTrackerResult { Success=false, Message="..." }`.

## Common pitfalls

- **Locked status is computed by the server.** If the client filters "available only" but the server still sends locked entries, latency feels weird — pre-filter on the server.
- **Quest progress summary** ("Slay Hen 4/13") must be recomputed on the fly when the dialog opens; it's not cached. Open-close-open spam can pin CPU.
- **NPC on an instance/private map.** The `NpcMapIndex` field can be `-1` (unspawned). The teleport button should be disabled for those rows.

## Quick smoke test

Type `/qt`, verify the list loads. Filter for an available quest. Click Teleport — should land at the NPC. If no NPC exists for that quest, the row should be grayed out for teleport.
