# Bulk Assign Monster Spells

One-shot tool that walks every `MonsterInfo`, classifies it by name into an "archetype" (Undead, Demon, Dragon, Ice, Beast, Mage, Warrior…), and stamps 4–5 themed spells onto its `Skills` list plus the `SkillCasterBoss` AI (id 224). Backs up `Server.MirDB` before saving.

## What it does

A vanilla Mir2 server has ~4800 monsters but most of them have empty `Skills` lists, so the spell-casting AI (224) has nothing to rotate through. This tool fills that gap by:

1. Classifying each monster by keyword in `Name` (case-insensitive substring) into one of 18 archetypes (+ a Generic fallback).
2. For each, deterministically picking 4–5 spells from the archetype's themed pool (seeded by `MonsterInfo.Index` so the assignment is reproducible).
3. Setting `AI = 224` (SkillCasterBoss) so the monster actually USES the spells in combat.
4. Skipping reserved AIs (Guards, Gates, conquest archers/walls) so sieges aren't broken.

End result: 4826 monsters with thousands of unique spell-set combinations, all combat-ready.

## Where to find it

- **Server.exe → Database → Bulk Assign Monster Spells…**
- Source: `Server.MirForms\Database\BulkAssignMonsterSpellsForm.cs`

## How to use it

1. Stop the running game server (the editor and the server share `Envir.Edit`, but you want a clean save).
2. Open Server.exe → Database → **Bulk Assign Monster Spells…**.
3. Optional: tick **"Overwrite monsters that already have ≥1 spell"** to clobber hand-tuned bosses. Default off — preserves existing custom work.
4. Click **Preview**. The log shows:
   - Counts (updated / skipped reserved AI / skipped because they had skills).
   - Archetype distribution.
   - First 25 sample rows.
5. Review.
6. Click **Apply + Save DB**. Confirm in the dialog. A timestamped backup of `Server.MirDB` is written first.
7. Restart your game server to pick up the new DB.

## Archetype keyword map

| Archetype | Keywords (substring, case-insensitive) | Spell themes |
|---|---|---|
| Undead | skeleton, zombie, ghoul, mummy, wraith, lich, boneking, … | Plague, Curse, Vampirism, SoulFireBall, Poisoning, HellFire |
| Demon | demon, devil, imp, evil, fiend, hellknight, … | HellFire, FireWall, MeteorStrike, FlameDisruptor, Curse |
| Dragon | dragon, drake, wyrm, wyvern | GreatFireBall, FireBang, MeteorStrike, FlameField |
| Ice | ice, frost, snow, glacier, frozen, penguin, yeti | IceStorm, IceThrust, FrostCrunch, Blizzard, Hiding |
| Thunder | thunder, lightning, storm, shock, electric | ThunderBolt, ThunderStorm, Lightning, ElectricShock |
| Beast | wolf, tiger, bear, boar, cat, rat, leopard, ape, … | ShoulderDash, Vampirism, ThunderBolt, Hiding, MagicBooster |
| Insect | spider, scorpion, worm, ant, bee, wasp, centipede, … | Poisoning, PoisonCloud, Plague, Hallucination, Curse |
| Plant | tree, treant, ent, plant, flower, grass, vine, shroom | PoisonCloud, EnergyShield, Healing, MagicShield, FlameField |
| Mage | wizard, mage, sorcerer, sorceress, warlock, magi, elemental | FireBall, GreatFireBall, IceStorm, ThunderStorm, MagicShield |
| Warrior | warrior, knight, soldier, fighter, samurai, general, … | ShoulderDash, LionRoar, BattleCry, BlessedArmour, UltimateEnhancer |
| Archer | archer, hunter, ranger, sniper, bowman | BindingShot, ElectricShock, Hiding, Vampirism |
| Taoist | taoist, priest, monk, shaman, cleric, healer, sep | Healing, MassHealing, Purification, BlessedArmour |
| Assassin | assassin, rogue, ninja, thief, shadow, blade | Hiding, MassHiding, PoisonCloud, ShoulderDash |
| Snake | snake, serpent, naga, viper, cobra, hydra, yimoogi | Poisoning, Plague, PoisonCloud, Hiding |
| Aquatic | fish, shark, kraken, octopus, turtle, frog, crab | IceThrust, IceStorm, Hiding, FrostCrunch |
| Spirit | ghost, spirit, phantom, specter, soul, haunt, banshee | Hiding, MoonLight, Curse, Hallucination, Vampirism |
| Stone | stone, golem, rock, statue, iron, metal, obsidian | MagicShield, BlessedArmour, ImmortalSkin, MeteorStrike |
| Boss | king, queen, lord, lady, boss, ancient, great, elder, mir | GreatFireBall, MeteorStrike, Blizzard, ThunderStorm, MassHealing |
| Generic (fallback) | (no keyword match) | FireBall, IceStorm, ThunderBolt, Poisoning, MagicShield, Healing |

## Skipped AIs

These are never touched:

```
6   Guard           58  TaoGuard        72  SiegeGate       73  GateWest
80  Archer          81  Gate            82  Wall            113 GuardBoss
```

Touching them would break sieges / quests.

## How it works under the hood

- The tool reads `Envir.Edit.MonsterInfoList` (the editor's parallel `Envir`, populated on form startup).
- For each monster: regex/substring keyword match → archetype → spell pool → Fisher-Yates pick of 4 or 5 spells.
- Mutates `mi.Skills = picked.ToList()` and `mi.AI = 224` in memory.
- Calls `Envir.Edit.SaveDB()` to persist.

## Common pitfalls

- **Apply requires preview first.** The Apply button is disabled until you've previewed.
- **Reserved AIs are hardcoded.** If your server uses custom AI numbers in the reserved range, edit `ReservedAIs` in the form source.
- **Restart required.** The running game server's `Envir.Main.MonsterInfoList` is separate from `Envir.Edit`. Your changes only take effect after a server restart reads the new DB.
- **Pair with Bulk Scale Monster Stats** (`docs/tutorials/13-bulk-scale-monster-stats.md`). Spells without DC/MC/SC stats do reduced damage — run both tools in sequence.

## Quick smoke test

Spawn a freshly-assigned monster (`@MOB ZombieGeneral` or similar) in-game. Engage. Watch it cast 4–5 spells from its archetype pool. If it just basic-attacks: `AI != 224` (it was probably a reserved AI), or `Skills.Count == 0` (overwrite was off and it already had stuff).
