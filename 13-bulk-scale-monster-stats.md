# Bulk Scale Monster Stats

Companion to **Bulk Assign Monster Spells**. Walks every `MonsterInfo` and rewrites `Min/Max DC, MC, SC, HP` based on its `Level` and the spell mix in `Skills`. Higher level → higher stats. Spell mix → stat axis (caster mobs get MC, melee mobs get DC, support mobs get SC).

## What it does

For each monster:
1. Classifies its spell list into `physical / magic / support` axis weights.
2. Computes Min/Max DC, MC, SC by scaling those weights against the monster's `Level`.
3. Optionally rescales `HP` as `Level^1.6 × 5` (×3 for bosses).
4. Clamps all values into the `ushort` field width of the DB.
5. Backs up `Server.MirDB` and saves.

## Where to find it

- **Server.exe → Database → Bulk Scale Monster Stats…**
- Source: `Server.MirForms\Database\BulkScaleMonsterStatsForm.cs`

## How to use it

1. **First run** `Bulk Assign Monster Spells…` (so each monster has a Skills list).
2. Server.exe → Database → **Bulk Scale Monster Stats…**.
3. (Optional) tune the multipliers:
   - **DC×, MC×, SC×, HP×** — global scaling knobs. Default `1.0`. Bump `MC×` to `1.5` for harder casters, `HP×` to `0.6` for shorter fights, etc.
4. Tick / untick:
   - **Overwrite even when current Min/Max stats are non-zero** — default ON. Untick to preserve hand-tuned bosses.
   - **Also scale HP** — default ON. Untick to leave HP untouched.
   - **Skip Guard / Gate AIs** — default ON.
5. Click **Preview**. Log shows count, multipliers, first 30 sample rows.
6. Click **Apply + Save DB**. Backup is written first.
7. Restart game server.

## Formula

```
For each monster (level L):
  weights      = ClassifySpells(Skills)
                 → (phys, mag, sup) summing to ~1.5; baseline phys ≥ 0.20

  minDC = round(L * 0.30 * phys * DCMul)
  maxDC = round(L * 0.55 * phys * DCMul)
  minMC = round(L * 0.35 * mag  * MCMul)
  maxMC = round(L * 0.65 * mag  * MCMul)
  minSC = round(L * 0.35 * sup  * SCMul)
  maxSC = round(L * 0.65 * sup  * SCMul)

  HP    = round(L^1.6 * 5 * HPMul * bossBonus)
          bossBonus = 3 if mi.IsBoss else 1
```

All values clamped to `[0..ushort.MaxValue]` (DB field width).

## Spell axis examples

| Spell | phys | mag | sup |
|---|---|---|---|
| ShoulderDash, LionRoar, BindingShot | 1.0 | 0 | 0 |
| FireBall, MeteorStrike, IceStorm, ThunderBolt, HellFire | 0 | 1.0 | 0 |
| Healing, MassHealing, Purification | 0 | 0 | 1.0 |
| MagicShield, BlessedArmour, EnergyShield | 0 | 0.2 | 0.8 |
| Vampirism | 0.6 | 0.4 | 0 |
| Poisoning, Plague, PoisonCloud | 0 | 0.7 | 0.3 |
| (no spell / unknown) | 0.33 | 0.33 | 0.34 |

## Worked examples (default multipliers)

| Monster | Lv | Archetype | DC | MC | SC | HP |
|---|---:|---|---|---|---|---:|
| Plain melee mob | 10 | physical | 3-6 | 1-2 | 1-2 | 200 |
| Skeleton w/ Plague, Curse, Vampirism, SoulFireBall | 25 | magic | 4-7 | 6-12 | 2-4 | 1,016 |
| Wizard archetype | 40 | magic | 2-5 | 12-22 | 0-1 | 1,902 |
| Taoist healer | 30 | support | 2-4 | 1-3 | 8-15 | 1,225 |
| Boss, mixed | 80 | magic | 6-10 | 22-40 | 6-12 | 9,121 (boss×3) |

## Safeties

- **Preview first.** Apply is disabled until Preview runs.
- **Backup before save.** Timestamped `Server.MirDB.backup-YYYYMMDD-HHMMSS`.
- **Skip Level == 0.** Those are usually scripted entities; scaling would produce zeros.
- **Idempotent.** Running it twice with the same knobs produces the same output.

## Recommended order of operations

```
1. Bulk Assign Monster Spells     ← populates Skills
2. Bulk Scale Monster Stats        ← derives DC/MC/SC/HP from Skills + Level
3. Server restart                  ← runtime picks up the new DB
4. Combat-test a few sample monsters at low / mid / high level
5. If too easy/hard, adjust the *Mul knobs and re-run Step 2
```

## Common pitfalls

- **Running stats-scale before spell-assign** produces "neutral" stat splits (every monster gets phys = mag = sup) because there are no spells to classify by. Always run them in order.
- **Bossbonus of 3** can make HP overflow the ushort for very high-level bosses. The tool clamps but the clamped value means the boss is "infinite HP" relative to the displayed stat. Cap your bosses at Level 100 or pre-scale HP outside the form.
- **Overwriting non-zero stats** clobbers hand-tuned bosses. Untick the box if you care about preserving them.

## Quick smoke test

Run both Bulk tools. Spawn `@MOB ZombieGeneral` and a `@MOB Wizard`. Verify the Zombie has high DC + decent MC (it uses Vampirism), the Wizard has high MC + low DC (pure caster).
