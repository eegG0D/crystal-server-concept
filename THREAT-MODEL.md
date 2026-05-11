# Crystal Server — Threat Model

A STRIDE-style threat model. Names what we defend, who we defend against, what we explicitly accept, and what each piece of code is load-bearing for.

---

## Scope

The Crystal server (`Server.exe`) and launcher (`Mir2.exe`), the patch host that serves `PList.gz`, and the operator's deployment environment. **Out of scope**: the game client's anti-cheat (a player running a tampered client owns their machine), the player's PC, the player's email account.

## Assets (ranked by attacker payoff)

1. **Account database** (`Server.MirDB`) — usernames + password hashes + character data. If dumped: account takeover at scale.
2. **TOTP seeds** (`Configs\2fa.snapshot/journal`) — if dumped: 2FA bypass forever for affected accounts.
3. **Pepper** (env var) — adds cost to any DB-only crack.
4. **GM password** (env var) — privilege escalation in-game.
5. **Patch signing private key** — RCE on every player via tampered binary push.
6. **Game economy state** (gold, items in `Server.MirDB`) — dupes/edits monetizable in secondary markets.

## Adversaries

| Adversary | Capabilities | Motivation |
|---|---|---|
| **Script kiddie** | Off-the-shelf tools, modified clients from GitHub, public exploit kits. | Cheating, dupes, status. |
| **Credential stuffer** | Botnet + leaked password lists. | Account takeover for resale. |
| **Motivated player attacker** | Reverse engineers client, writes own packet tools, time. | Cheating, dupes, vendettas. |
| **Network MITM** | Owns a hop on the wire (rogue Wi-Fi, hostile ISP). | Steals credentials, injects packets. |
| **Insider** | A GM, or someone with shell on the host. | Theft, sabotage, fraud. |
| **Operator-host attacker** | RCE on the game-server VM (e.g., via SSH brute force, supply-chain compromise). | Full game compromise. |

We make no claim about defending against nation-state-level adversaries.

## STRIDE table

| Threat | Vector | Defense | Residual risk |
|---|---|---|---|
| **S**poofing — wrong account login | Stolen password | `PasswordHasher` (PBKDF2-SHA256 600k + pepper). 2FA via `Totp`. Per-IP + per-account rate limit. | Phishing of TOTP code over a short window; lost device + recovery. |
| **S**poofing — stolen session | Hijacked TCP socket | App-level disconnect on dup login; socket bound to PROXY-protocol-validated IP. | A confederate on the same NAT can ride after disconnect (rare). |
| **S**poofing — MITM | Plaintext on the wire | `docs/TLS-PROXY.md` (TLS termination at nginx + client cert pinning when client is updated). | Pre-pinning client builds vulnerable to credential sniff on hostile networks. |
| **T**ampering — DB at rest | Disk theft, backup leak | Hashes (not plaintext) + pepper (not in DB). Encrypted backups (operator). | If backup AND env var are exfiltrated together, hashes are crackable. |
| **T**ampering — TOTP store | Disk theft of `Configs\2fa.*` | DPAPI-sealed (Windows) / AES-GCM fallback. | If a sealed blob AND the unsealing identity are stolen together, seeds recoverable. |
| **T**ampering — patch binary | Hostile patch host or MITM on download | Per-file SHA-256 + RSA-PSS signature over manifest. `PatchManifest.Read` rejects on signature fail, length cap, malformed entries. | Until clients are rebuilt with a real `ManifestPublicKeyPem`, only the SHA-256 stops per-file tampering; manifest swap is undefended. |
| **T**ampering — audit log | Insider editing `gm-*.log` to hide actions | `HashChainLog` — every entry includes SHA-256 of previous. Verified daily on off-host copy. | Real-time attacker with continuous write can rewrite forward; defense is the off-host copy taken before the breach. |
| **R**epudiation — "I didn't do that" | GM denies abuse | Hash-chained audit log + off-host shipping. | Pre-incident off-host shipping must be in place. |
| **I**nformation disclosure — account enumeration | Probing `Login` for `Result=3` vs `Result=4` | Per-IP rate limiter increments on `account == null` too (this iteration). | The two response codes are still different on a single probe; only RATE is bounded. |
| **I**nformation disclosure — TOTP existence | Probing `Login` for "needs TOTP" vs "doesn't" | Verifier runs password check FIRST; TOTP fail returns same `Result=4` as password fail. | Stable response for unknown accounts also returns `Result=3`, distinguishable from `Result=4` for existing-but-wrong; mitigated only by rate limit. |
| **I**nformation disclosure — secrets in logs | `MessageQueue.Enqueue(p.Password)` etc. | None at primitive level; audited callsites do not log credentials. | Future careless logging could leak. Code review must enforce. |
| **D**enial of service — packet flood | Single connection spam | `MaxReceivePerTick=200`, `MaxRetryQueue=100`. | A coordinated 50-IP attack at 199 pkt/tick/connection passes both per-conn caps. |
| **D**enial of service — connection flood | Many concurrent connections | `Settings.MaxIP=5` (per-IP). `Settings.MaxUser=50` (global). | An attacker with thousands of source IPs is undefended at the app level — needs a WAF / Cloudflare at the edge. |
| **D**enial of service — per-IP packet flood across N connections | Pace each of MaxIP conns under per-conn cap | `Envir.RecordIPPackets` aggregates per-IP at `MaxIPPacketsPerSecond=500`. | An attacker that splits across many IPs still passes; same mitigation as above. |
| **D**enial of service — slow-loris | Open connection, never send | `_client.ReceiveTimeout = max(30000, Settings.TimeOut*3)` + app-level `TimeOutTime`. | OS kill is slower than ideal for adversarial close-then-reopen; bounded by `MaxIP`. |
| **D**enial of service — login rate-limit memory exhaustion | Probe from 10M unique IPs | `_loginAttempts` capped at `LoginAttemptMaxEntries=200_000`, oldest evicted; idle entries swept after 1h. | LRU eviction can let a non-blocked attacker get back in by waiting; acceptable tradeoff. |
| **D**enial of service — hostile manifest | Patch host serves 1 GB malformed manifest | `PatchManifest.MaxBytes = 20 MiB`, `MaxEntries = 200k`, `MaxFileNameLength = 512` enforced before parse. | A 20 MiB load is still expensive; offset by HTTPS + signature pinning in production. |
| **E**levation — GM via password chat | Players guessing `@LOGIN` + GMPassword | `CRYSTAL_GM_PASSWORD` env var (long random), audited via `GMAuditLog` chain. Plaintext over wire when TLS not active. | Until TLS is in place, GM password is sniffable on a hostile network. |
| **E**levation — RCE via NPC script | Hostile NPC script content | NPC scripts come from operator-controlled `.txt` files, not players. `NPCConfirmInput` caps `p.PageName ≤ 30`, `p.Value ≤ 200`, rejects stale session. | Scripts that use `$NPCInputStr` in unsafe contexts are a script-author bug, not a server bug. |
| **E**levation — privilege via dupe | Inventory/gold/NPC packet handlers | All audited (`PlayerObject.DropGold`, `TradeGold`, `BuyItem`, `SellItem`, `ConsignItem`, `MarketBuy`, `SendMail`, `SplitItem`, `DropItem`, `Magic`) — all server-authoritative with bounds. | A novel handler added without the same pattern is the failure mode. |

## Explicitly accepted risks

- **Trojaned client on a player's PC**: nothing the server can do. Anti-cheat would require kernel-mode hooks (EAC/BattlEye); out of scope.
- **Phishing of TOTP**: a real-time phishing site can capture password + TOTP and replay within the 90-second window. Mitigated only by user education.
- **Operator-host RCE**: an attacker with shell on the server can do anything. Mitigated by least-privilege OS user (`SECURITY-OPS.md` §1) so blast radius is the `crystal` account, not root.
- **2FA UX via password suffix**: `password:NNNNNN` is hostile to password managers. Acknowledged tradeoff to avoid a protocol change. Proper fix is a separate TOTP packet field, deferred to the same release window as packet HMAC (`docs/PACKET-HMAC-DESIGN.md`).
- **Lazy legacy-hash collision**: the original Crystal hash function's UTF-8 entropy corruption means two distinct passwords could produce the same stored bytes (vanishingly rare in practice). On a colliding wrong password, the lazy migration would cement the wrong password. We log every migration; ops would see the spike if a coordinated attack tried to exploit it.

## What this model does NOT cover

- Side-channel attacks (timing of `FixedTimeEquals` is the only mitigation we attempt; CPU-cache attacks are out of scope).
- Hardware faults / Rowhammer.
- Supply-chain compromise of the .NET SDK, log4net, ProtectedData NuGet, etc.
- Long-term cryptographic obsolescence (SHA-1 in PBKDF2 legacy, SHA-1 in TOTP per RFC 6238). Migration plan: bump algo string in `HashHeader`, parser auto-handles.

## What to revisit when

- **A new packet handler is added**: add a row above describing what's authoritative.
- **A new lock is introduced**: add it to the hierarchy in `SECURITY-OPS.md` §8.
- **An adversary capability changes** (e.g., a public Crystal speedhack tool drops): re-rank the table.
- **A defense fails in incident response**: write the post-mortem under `docs/incidents/` and update this table.
