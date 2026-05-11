# Character Select (in-game)

The Crystal client uses a **scene-based** flow: `LoginScene → SelectScene → GameScene`. Character select is a *scene*, not a dialog inside the game. Switching characters mid-game requires returning to `SelectScene`.

## What it does

After login, the client moves to `SelectScene` where the account's character list renders as 4 visible slots with model previews, level, class, and last-played map. The player picks one → `GameScene` boots. A new "Back to Character Select" button (or `/select` chat command) sends `C.LogOut` to drop the player back to `SelectScene` without disconnecting the socket.

## Where to find it

- **Scene:** `Client\MirScenes\SelectScene.cs`
- **Boot from login:** when `LoginScene` receives `S.LoginSuccess { Characters[] }` it instantiates `SelectScene` with those characters.
- **Boot from game:** `GameScene` calls `Network.Enqueue(new C.LogOut())` and on the server-side handler the player returns to character-select state (account remains connected).

## How players use it

**On login:**
1. Enter credentials → `LoginScene` validates → `S.LoginSuccess` arrives with `Characters[]`.
2. `SelectScene` renders. Click a character → click "Start Game" → `C.StartGame` → server creates `PlayerObject`, replies `S.StartGame`, `GameScene` loads.

**Switching mid-game:**
1. Open the in-game menu OR type `/select` (if your client wires that command).
2. Confirmation prompt (since unsaved auto-farm state etc. will pause).
3. Client sends `C.LogOut`; server unloads the `PlayerObject` from the world, keeps `Account` alive, sends back the character list.
4. Client switches back to `SelectScene`. Pick another character (or the same one) → back to game.

## How it works under the hood

- The connection's `GameStage` field transitions:
  - `Login` → `Select` (after successful login)
  - `Select` → `Game` (after `StartGame`)
  - `Game` → `Select` (after `LogOut`)
- Each transition swaps the active `MirScene` on the client.
- The server uses the same `Connection.Account` object across transitions; only the `Account.Connection.Player` field is cleared on `LogOut`.

## Common pitfalls

- **Don't use `Disconnect` instead of `LogOut`.** Disconnect drops the socket; LogOut just unloads the player. Mid-game character swap should never re-handshake.
- **Active trades / NPC dialogs.** If the player is mid-trade and clicks "Back to Select", the trade silently aborts on the partner's side. Block the switch button when `TradePartner != null` or display a warning.
- **Combat lockout.** Some servers refuse character-swap if the player is in combat for ~10 s — this prevents PvP escape via select-screen. Implement via `LastAttackedTime` check in the LogOut handler.
- **AutoFarm / AutoHunt state.** Decide policy: pause across swap (recommended) or persist across characters on the same account.

## Adding a `/select` command

Client-side toggle pattern (drop into `MainDialogs.cs` alongside the other client-only commands):

```csharp
if (string.Equals(msg, "/select", StringComparison.OrdinalIgnoreCase))
{
    Network.Enqueue(new C.LogOut());
    ChatTextBox.Visible = false;
    ChatTextBox.Text = string.Empty;
    LinkedItems.Clear();
    break;
}
```

The server `LogOut` handler already exists and transitions back to `Select` stage.

## Quick smoke test

Login → SelectScene → pick a character → in-game. Type `/select` (after wiring it). Client should drop back to SelectScene with the same character list. Pick another character → back in game with the new one's state.
