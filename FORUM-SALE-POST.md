# 🏰 [WTS] Production-Grade Crystal Mir2 Server — Fully Customized, Secured, Monetized — $24,950

> Custom-built Crystal Mir2 server with 4826 monsters, 3910 maps, real-money monetization, enterprise-grade security, and 17 documented gameplay systems. **Source code, tools, and operator docs included.** Not another stock package — every layer has been audited and extended. Built for someone who wants to launch a real revenue-generating server, not a hobby project.

---

## What you get

### 🧠 The big-ticket custom features

| System | What it is | Where most servers fail |
|---|---|---|
| **EEG-to-English Brain Interface** | Players plug in a NeuroSky MindWave / TGAM headset. The server reads raw brainwaves over serial, maps each value to an English word + EXP + sound via a 6000-entry CSV, and shows it in real-time above the HUD. **Nobody else has this.** | N/A — this is unique. |
| **Match-3 Idle Minigame** | Built-in Bejeweled-style minigame with bombs, super gems, cascades, hints, shuffles. Fully client-side engine + draggable GUI. Pulls game tile icons from your live `items.lib`. | Most servers ship zero minigames. |
| **Server-Side Trigger System** | JSON rule engine. 50 event types × 200 conditions × 200 effects. Hot-loadable. Ships with **300 pre-built security triggers** (catalog in `docs/SECURITY-TRIGGERS.md`). | Stock Mir2 has no rules engine; you'd hand-code each one. |
| **Bulk Monster Spell Assigner** | One-shot tool that classifies all 4826 monsters by name into 18 archetypes (Undead, Demon, Dragon, Ice, Mage, …) and stamps 4–5 themed spells + the SkillCasterBoss AI on each. **2000+ unique spell combinations across the population.** | Vanilla servers have ~100 mob types using ~5 AIs. |
| **Bulk Monster Stat Scaler** | Derives Min/Max DC, MC, SC, HP per monster from its Level + the spell mix you just assigned. Caster mobs get high MC, melee get DC, support get SC. Idempotent, previewable, backup-before-save. | Manual stat tuning of 4826 mobs is several weeks of work. |
| **PayPal Live Payments** | Real-money credit purchases via embedded HTTPS webhook. Sandbox + live modes, deduplicated transaction log, configurable credit packages. **You can charge real money on day one.** | Stock Mir2 has no monetization. |

### 🛡️ Enterprise-grade security (rare in the Mir2 scene)

- **Password hashing**: PBKDF2-HMAC-SHA256 at OWASP-2024 minimum (600,000 iterations), with **server-side pepper** read from env var (not the DB). Crypto-agile via versioned prefix (`$pbkdf2-sha256-p1$…`) — you can rotate the pepper without locking out a single user.
- **Login rate limiting**: Per-IP (10 failures / 5 min = 15-min block) **and** per-account (exponential backoff: 2 → 4 → 8 → … → 240 min cap).
- **TOTP 2FA**: RFC 6238 compliant. Per-account seeds stored DPAPI-sealed on disk (Windows) with AES-GCM fallback. In-game enrollment via `@2fa setup`.
- **GM audit log**: Every `@`-prefixed command is logged into a **hash-chained, tamper-evident** daily log file. Edit one line, the chain breaks at a known offset.
- **Packet integrity**: Per-IP packet-rate aggregator, retry-queue caps, slow-loris socket timeouts, PROXY-protocol v1 parser for TLS-terminated deployments.
- **Patch manifest signing**: RSA-PSS over the entire client patch manifest + SHA-256 per file. Operator gets a publisher tool (`Tools\PatchPublisher`) that signs releases. Prevents trojaned-client distribution.
- **300 security triggers** pre-configured (most disabled by default with reasoned safety logic; flip on as you're ready).
- **60 automated tests** for every primitive — password hashing, TOTP RFC 6238 test vectors, manifest tampering rejection, hash-chain verification, Base32 RFC 4648 vectors.

### ⚙️ Operational tooling

- **Trigger Editor GUI** — Database → Triggers in the Server form app. List, edit, save without touching JSON.
- **Bulk Assign Spells GUI** — preview before commit, archetype breakdown, backup before save.
- **Bulk Scale Stats GUI** — DC/MC/SC/HP multiplier knobs, idempotent.
- **PatchPublisher CLI tool** — signs and packages client manifests for release.
- **Console tools** — `SecurityTriggerGen` (regenerates the 300 security triggers from source).

### 🗺️ Content scale

- **4826 monsters** (10× a vanilla server)
- **3990 map definitions**
- **57,240 inter-map routes** in `MapInfoExport.txt`
- **3910 binary `.map` files** (3.0 GB total)
- **111,460 monster spawn lines** in `spawns.txt`
- Spawns use level-banded mob names (`Mob_LvNNN`) — built-in difficulty curve from day one.

### ⚡ Performance hardening (built in this round)

- **Parallel map loading** — startup time on the multiplied 10× content brought down from ~minutes to seconds via `Parallel.ForEach` cell pre-parsing.
- **NPC lookup** — O(n²) per-map linear scan → O(1) dictionary keyed by MapIndex.
- **Walkable-cell filter** — O(W×H) LINQ per respawn → O(spread²) bounding-box scan. ~100× speedup for the 111k spawn lines.
- **Per-IP packet aggregator** — DoS-resistant at the connection layer.

### 📖 Documentation

The repo ships **18 tutorial Markdown files**, a STRIDE threat model, a security-ops runbook, a TLS reverse-proxy guide, a packet-HMAC design doc, the 300-rule security trigger catalog. **`docs/tutorials/README.md` is the index.** A new operator can read for an afternoon and have full understanding of every system.

---

## What it's worth

Honest accounting at conservative junior-dev rates ($75/hr) for the custom work:

| Component | Hours | Value |
|---|---:|---:|
| Security infrastructure (hashing+pepper+rotation+2FA+rate limit+audit log+manifest signing) | 60h | $4,500 |
| Trigger system + editor + 300-rule catalog | 80h | $6,000 |
| Bulk monster spell assignment (18 archetypes, GUI, backup) | 16h | $1,200 |
| Bulk monster stat scaler (formula + multiplier GUI) | 12h | $900 |
| PayPal live integration with dedupe ledger | 24h | $1,800 |
| Match-3 minigame (engine + GUI + cascades) | 20h | $1,500 |
| EEG-to-English BCI integration (serial reader + mapper + dialog + 6000-row CSV) | 32h | $2,400 |
| Content multiplication (4826 mobs, 3910 maps, 111k spawns) | 24h | $1,800 |
| Performance optimization (parallel loading, NPC dict, bounding-box filter) | 16h | $1,200 |
| Operational forms (Auto-Farm, Auto-Hunt, Stat Editor, Item Modifier, Mob Chaser, Quest Tracker, Map-Item TP, Invisibility, Starting Skills) | 50h | $3,750 |
| Documentation (18 tutorials, threat model, ops runbook) | 16h | $1,200 |
| Test suite (60 tests) | 8h | $600 |
| **Total dev value** | **358h** | **$25,950** |

Add a **base Crystal Mir2 source license / equivalent buildout** ($3,000–$5,000 market floor) and you're at **$28,950–$30,950 of work**.

### 💰 Asking price: $24,950

That's a discount on the dev-time accounting alone. You're getting the base server, the customizations, the security audit, the docs, **and the right to commercialize via the integrated PayPal layer** — for less than building any single one of these subsystems from scratch.

For comparison, the going rate for full custom Mir2 servers on Ragezone / EliteSinatra / FlyForum:

- Stock package with minor reskin: $500–$2,000
- Custom-balanced server with a few new systems: $3,000–$8,000
- Fully custom + monetized + secured + documented: **$15,000–$40,000**

This sits in the upper bracket because the security work alone is worth what most "fully custom" servers charge for everything combined.

---

## Why you should pull the trigger

1. **First-mover advantage on the EEG mechanic.** No other Mir2 server has it. Marketing write-up is going to print itself. "Mir2 that responds to your brain" sells.
2. **Real monetization on day one.** Plug in your PayPal sandbox credentials, ship a $5/500-credit package, flip to live. Most servers spend months on this.
3. **You can hire less for sysops.** The security primitives are production-grade. The docs are operator-facing. You don't need a security engineer; you need someone who can read a runbook.
4. **It compiles clean.** `dotnet build` on the solution returns 0 errors. No "we'll fix that warning later" debt rotting in your branch.
5. **It scales.** Tested under 10× content multiplication with the parallel loader. Tested at 4826 mobs with the bulk tools. Tested under flood with the packet-rate aggregator.
6. **The hard part is done.** Reskinning, retuning numbers, swapping artwork — all easy. Building security, monetization, and tooling from scratch — that's the cost most operators don't see until they're three months into a launch that won't ship.

---

## How to buy

- **Price**: $24,950 USD
- **Payment**: PayPal (logical, given the integration); crypto on request; escrow for amounts over $10K is fine.
- **Delivery**: Full git history of the repo, `.MirDB` snapshots, the 6000-row EEG CSV, the 300 security trigger JSON files, all tooling, all docs.
- **Support**: 30 days of post-sale email Q&A included. Code walkthrough call on request.
- **Exclusivity**: Single buyer. Once sold, listing comes down. No reseller will get this.

DM for sample server access. Serious inquiries only — please bring a clear vision of what you intend to ship, not "considering a server."

> **Why am I selling it?**
> I built the engine I wanted to build. Operating a live server is a different job than building one, and I'd rather hand the keys to an operator who can run a community than half-run one myself. Code is meant to ship.

---

*Posted: 2026-05-11. Listing valid until sold. Repo state: clean, tagged, documented. All claims in this post are verifiable from the included `docs/` folder.*
