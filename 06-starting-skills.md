# Starting Skills

Bulk-grant a configured skill set to every new character on creation. Optionally restrict by class, charge credits, or announce to the world.

## What it does

When a fresh character finishes character creation, the server enumerates the configured starter skill list (filtered by class) and inserts each into the new character's `Info.Magics` with a configured starting level. Eliminates the "level 1 has nothing to cast" dead zone.

## Where to find it

- **Server.exe → Config → System → Starting Skills**
- Source: `Server.MirForms\Systems\StartingSkillsConfigForm.cs`
- `Setup.ini` section `[StartingSkills]`.

## How to configure

| Field | What it does |
|---|---|
| **Enabled** | Master switch. |
| **Announce on grant** | Broadcasts "<Name> received starter skills." to the server. |
| **Skill list per class** | One list per `MirClass` (Warrior / Wizard / Taoist / Assassin / Archer). Each entry is `(Spell, StartingLevel)`. |
| **Credit cost (optional)** | If non-zero, character creation costs this much credit. |
| **Default skill level** | 1–3. 1 = base, 2/3 = pre-trained. |

## How players use it

Just create a new character. The grant happens automatically before the player enters the world.

## How it works under the hood

- Hook is in `NewCharacter` packet handler (`Envir.NewCharacter`) right after the `CharacterInfo` is constructed and added to the account.
- For each entry in the class-matched list, the server adds a `UserMagic { Spell = X, Level = Y, Key = ... }` to `CharacterInfo.Magics`.
- The client renders these in the magic book on first login.

## Common pitfalls

- **Skills you grant must already be in `MagicInfoList`.** Otherwise `UserMagic` references a nonexistent spell and clients crash on render. Verify each Spell enum value exists in the Magic editor.
- **Starting Level too high** (e.g., level 3 ThunderBolt at character creation) makes early-game trivial. 1–2 is the standard.
- **Class restriction must match the spell's actual class.** Granting `FireBall` to a Warrior creates a "dead skill" in their book — castable but useless. The form should filter the list per class but check what your version actually does.

## Quick smoke test

Configure 2 starter skills for Wizard, create a fresh Wizard, log in, open the magic book — those skills should already be there at the configured level.
