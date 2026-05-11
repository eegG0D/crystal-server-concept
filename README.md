[README.md](https://github.com/user-attachments/files/27605222/README.md)
# Tutorials Index

Operator and developer tutorials for the Crystal Mir2 server features.

## Server features (player-facing)

| # | Doc | What it covers |
|---|---|---|
| 1 | [Item Modifier](01-item-modifier.md) | Time-bounded stat blessings on items, tiered intensity, credit cost. |
| 2 | [Stat Editor](02-stat-editor.md) | Per-character stat allocation / respec system. |
| 3 | [Invisibility](03-invisibility.md) | Paid hide-from-monsters buff. |
| 4 | [Auto-Hunt](04-auto-hunt.md) | Passive auto-attack mode for AFK grinding. |
| 5 | [Mob Chaser](05-mob-chaser.md) | Pay-to-teleport to a named monster. |
| 6 | [Starting Skills](06-starting-skills.md) | Auto-grant skills on character creation. |
| 7 | [Map-Item Teleport](07-map-item-teleport.md) | Teleport to a dropped item on the ground. |
| 8 | [Auto-Farm](08-auto-farm.md) | Passive drop accumulator + retrieval dialog. |
| 9 | [Quest Tracker](09-quest-tracker.md) | In-game quest log with NPC teleport. |
| 10 | [PayPal Payments](10-paypal-payments.md) | Real-money credit purchases. |

## Server features (operator-facing)

| # | Doc | What it covers |
|---|---|---|
| 11 | [Trigger System](11-trigger-system.md) | JSON-defined event → condition → effect rules engine. |
| 12 | [Bulk Assign Monster Spells](12-bulk-assign-monster-spells.md) | One-shot: classify every mob by name, assign 4–5 themed spells + SkillCasterBoss AI. |
| 13 | [Bulk Scale Monster Stats](13-bulk-scale-monster-stats.md) | Derive Min/Max DC, MC, SC, HP from a mob's Level + Skills. |

## Client features

| # | Doc | What it covers |
|---|---|---|
| 14 | [Character Select (in-game)](14-character-select-in-game-scene.md) | Returning to SelectScene from GameScene without disconnecting. |
| 15 | [Map Visual Effects](15-map-visual-effects.md) | Fire / Lightning damage zones, weather particles, light setting. |
| 16 | [Match-3 Game](16-match-3-game.md) | Built-in client-side Match-3 puzzle minigame (`/m3`). |
| 17 | [EEG-to-English Label System](17-eeg-english-label-system.md) | Brain–computer-interface: TGAM headset → CSV-mapped English words + EXP + sound. |

## Related docs

| Doc | Purpose |
|---|---|
| [SECURITY-OPS.md](../SECURITY-OPS.md) | Operations runbook: pepper rotation, ACLs, log monitoring, incident response. |
| [SECURITY-TRIGGERS.md](../SECURITY-TRIGGERS.md) | Catalog of 300 pre-defined security triggers. |
| [THREAT-MODEL.md](../THREAT-MODEL.md) | STRIDE table + adversary matrix. |
| [TLS-PROXY.md](../TLS-PROXY.md) | nginx reverse-proxy setup for TLS termination + PROXY protocol. |
| [PACKET-HMAC-DESIGN.md](../PACKET-HMAC-DESIGN.md) | Deferred packet-HMAC implementation plan. |

## Reading order suggestions

**New operator** — start with `SECURITY-OPS.md` and `11-trigger-system.md`, then skim 1-10 to know what features exist.

**Developer extending features** — read 16 (Match-3) first; it's the most self-contained example of a feature with both server and client surfaces. Then 8 (Auto-Farm) for a server-authoritative dialog pattern.

**Performing a database bulk operation** — 12 (Bulk Assign Spells) then 13 (Bulk Scale Stats), in that order.

**Setting up monetization** — 10 (PayPal) and the security/TLS docs.
