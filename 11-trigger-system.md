# Trigger System

Event-driven rules engine. Subscribes to in-game events (`PlayerLogin`, `PlayerDealtDamage`, `PlayerChat`, `ServerTick`, etc.), evaluates a condition tree, fires effects (give item, kick, ban, send chat). JSON-defined; hot-reloadable in principle.

## What it does

Each trigger is a JSON file in `./Envir/Triggers/` (recursively scanned). When the game raises an event, the manager checks all triggers subscribed to that event, evaluates their condition group, and executes the listed effects if it passes.

It's a low-code automation layer — operators write JSON instead of touching C#.

## Where to find it

- **Source:**
  - `Triggers\TriggerManager.cs` — loader + dispatcher.
  - `Triggers\TriggerDefinition.cs` — JSON model.
  - `Triggers\TriggerCondition.cs` — 200 ConditionType values.
  - `Triggers\TriggerEffect.cs` — 200 EffectType values.
  - `Triggers\ConditionEvaluator.cs` — condition switch.
  - `Triggers\EffectExecutor.cs` — effect switch.
- **JSON files:** `Build\Server\Debug\Envir\Triggers\**\*.json`
- **Server-side editor:** Server.exe → Database → Triggers (`Server.MirForms\Database\TriggerEditorForm.cs`).
- **Pre-built security triggers:** `docs\SECURITY-TRIGGERS.md` and `Tools\SecurityTriggerGen\Program.cs` produce 300 starter triggers.

## Trigger JSON shape

```json
{
  "Index": 1,
  "Name": "WelcomeOnLogin",
  "Description": "Sends a welcome hint when a player logs in.",
  "IsActive": true,
  "FireMode": "Repeating",
  "MaxFireCount": 0,
  "CooldownMs": 0,
  "Events": [ "PlayerLogin" ],
  "RootConditionGroup": {
    "Operator": "AND",
    "Conditions": [
      { "Type": "PlayerIsAlive", "Params": [] }
    ],
    "SubGroups": []
  },
  "Effects": [
    { "Type": "SendHint", "Params": [ "Welcome to the server!" ], "DelayMs": 0 }
  ]
}
```

| Field | Notes |
|---|---|
| `Index` | Unique. Operators should reserve [10000-19999] for generated security triggers; user-authored triggers below that. |
| `IsActive` | Disable without deleting the file. |
| `FireMode` | `Repeating` / `Once` / `RepeatingWithCooldown` / `OncePerPlayer` / `CountLimited`. |
| `Events` | One or more `TriggerEvent` enum names. |
| `RootConditionGroup` | `Operator` (AND/OR/NOT), `Conditions[]`, `SubGroups[]` (nested groups). |
| `Conditions[i].Type` | Any of 200 `ConditionType` values. `Params[]` are documented in the enum comments. |
| `Effects[i].Type` | Any of 200 `EffectType` values. `DelayMs` defers execution. |

## How to author a trigger

**Option A — Server editor (recommended)**
1. Server.exe → Database → Triggers.
2. Click `New…`, give it a name.
3. Check events, add conditions / effects from dropdowns.
4. Save. Restart server (or call `TriggerManager.Load()` from a GM hook).

**Option B — Hand-edit JSON**
1. Drop a `.json` file into `Build\Server\Debug\Envir\Triggers\` (or any subfolder).
2. Restart server. The loader recurses subdirectories.

## Tracing why a trigger does/doesn't fire

- Look at `MessageQueue` for `[Triggers] Unknown ConditionType X` — means you used a `ConditionType` the evaluator doesn't implement. After this session's patch, that line logs once per type then is silent.
- For misfires, set `FireMode = RepeatingWithCooldown` with a long cooldown to throttle while you debug.
- The trigger manager prints `Fired <Name>` (DEBUG builds only) on each successful firing — see `Triggers\TriggerManager.cs:TryFire`.

## Common pitfalls

- **`ServerTick` + `ConditionType.Always`** fires once per second. Add real predicates or set IsActive=false.
- **Destructive effects (`KickFromGame`, `BanAccount`, `ChatBan`) with `Always` conditions** will hit every player on every event match. The SecurityTriggerGen auto-disables this combo — manual triggers don't have that safety net.
- **Locking yourself out as GM.** A bad trigger can ban you on login. Always test on a throwaway account first. Recovery: stop the server, delete the offending `.json`, restart.
- **Nested condition groups** are supported by the runtime but the in-game editor only edits the root group. Hand-edit JSON for nested logic.

## What's NOT here

- Stateful conditions (sliding-window rate limits, per-account counters). The 200 ConditionType values are mostly point-in-time predicates. For "5 fails in 60s" style rules, write the logic in C# (e.g., the `Envir._loginAttempts` rate limiter) and use a trigger only for the alert effect.

## Related

- `docs/SECURITY-TRIGGERS.md` — 300-rule reference catalog.
- `docs/SECURITY-OPS.md` — operational runbook including trigger ops.
