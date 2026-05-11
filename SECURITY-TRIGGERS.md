# Crystal Server — 300 Security Triggers

A catalogue of 300 server-side detection rules for the Crystal Mir2 server. Each entry follows the same shape so it can be implemented either as a `TriggerEvent` rule inside the existing `Triggers/` runtime, or shipped to an external SIEM that watches log streams.

```
T### [Category] WHEN <event>  IF <predicate>  THEN <action>   — <one-line rationale>
```

**Actions**: `LOG` (info), `ALERT` (operator notification), `THROTTLE` (rate-limit further actions), `KICK` (disconnect), `BAN_SHORT` (15 min), `BAN_LONG` (24 h), `BAN_PERMANENT`, `LOCK_ACCOUNT`, `QUARANTINE_ITEM`, `ROLLBACK` (revert state), `AUDIT` (write hash-chained audit entry).

Multiple actions allowed (`ALERT + KICK`). Predicates reference field names in the codebase (`account.WrongPasswordCount`, `Player.Stats[Stat.MaxDC]`, etc.); operator should treat as pseudocode.

---

## 1. Authentication / Login (T001–T030)

```
T001 [Auth] WHEN Login              IF account.WrongPasswordCount >= 3 AND login_succeeds AND ip != account.LastIP   THEN ALERT + LOG          — Credential-stuffing success after multiple fails from a new IP.
T002 [Auth] WHEN Login              IF same_account_id seen from >= 3 distinct IPs within 60s                         THEN ALERT + THROTTLE     — Concurrent geo-impossible login attempts.
T003 [Auth] WHEN Login              IF ip not in account.LastIP AND account.has_2fa == false                          THEN LOG                  — Encourage 2FA enrollment via mail.
T004 [Auth] WHEN Login              IF ip in known_tor_exit_list                                                      THEN ALERT + KICK         — Tor exits are rarely legitimate for game clients.
T005 [Auth] WHEN Login              IF ip in known_vpn_provider_list AND account_age < 24h                            THEN ALERT                — Fresh accounts via VPN are dupe-farm signals.
T006 [Auth] WHEN LoginFailure        IF same_ip fails for >= 5 distinct account_ids within 60s                         THEN BAN_SHORT(ip)        — Username enumeration in progress.
T007 [Auth] WHEN LoginFailure        IF same_account fails 10 times within 60s                                         THEN LOCK_ACCOUNT(2h)     — Targeted brute force on one account.
T008 [Auth] WHEN LoginSuccess        IF account.LastDate < now - 365d                                                  THEN ALERT + LOG          — Dormant account reactivation; possibly stolen.
T009 [Auth] WHEN LoginSuccess        IF account.AdminAccount == true AND ip not in admin_whitelist                     THEN ALERT + KICK         — Admin from unexpected IP.
T010 [Auth] WHEN PasswordChange      IF previous_change_within(5m)                                                     THEN ALERT + KICK         — Rapid password churn = takeover-in-progress.
T011 [Auth] WHEN PasswordChange      IF new_password matches account.AccountID                                         THEN REJECT + AUDIT       — Trivial password.
T012 [Auth] WHEN PasswordChange      IF new_password matches account.SecretAnswer                                      THEN REJECT               — Reuse of recovery answer.
T013 [Auth] WHEN PasswordChange      IF account.has_2fa AND no TOTP supplied this session                              THEN REJECT               — 2FA required for sensitive ops.
T014 [Auth] WHEN AccountCreate       IF ip in account_creation_count_for_ip(1h) >= 5                                   THEN BAN_SHORT(ip)        — Mass account creation = dupe farm.
T015 [Auth] WHEN AccountCreate       IF AccountID matches blacklisted_phrases                                          THEN REJECT               — Profanity / impersonation filter.
T016 [Auth] WHEN AccountCreate       IF EMailAddress domain in disposable_email_list                                   THEN ALERT                — Mailinator-style throwaway.
T017 [Auth] WHEN 2FA_Setup           IF account.AdminAccount AND completed_without_secret_displayed                    THEN REJECT               — Replay of stale setup state.
T018 [Auth] WHEN 2FA_Verify_Fail     IF >= 3 fails within 60s                                                          THEN BAN_SHORT(account)   — TOTP brute force (~33% of code space).
T019 [Auth] WHEN 2FA_Disable         IF account.has_unread_security_mail OR account.recent_login_anomaly               THEN REJECT + ALERT       — Disabling 2FA right after takeover signal.
T020 [Auth] WHEN Logout              IF session_duration < 5s AND no actions taken                                     THEN LOG                  — Probing / fingerprinting.
T021 [Auth] WHEN Connect             IF ip.concurrent_connections > Settings.MaxIP                                     THEN KICK                 — Already enforced; trigger as belt-and-braces.
T022 [Auth] WHEN Connect             IF ip in pre_existing_ipblocks AND block_just_lapsed_<5s_ago                      THEN BAN_LONG(ip)         — Repeat-offender re-connects at threshold edge.
T023 [Auth] WHEN Login               IF account.Banned == true AND login_attempt                                       THEN AUDIT                — Track banned-account probing.
T024 [Auth] WHEN Login               IF account_id length not in [Globals.MinAccountIDLength .. Max]                   THEN KICK                 — Malformed; already covered by regex but log.
T025 [Auth] WHEN ChangePassword      IF account != caller (admin path) AND no GM_PASSWORD                              THEN REJECT + AUDIT       — Account modification not gated by GM auth.
T026 [Auth] WHEN Login               IF supplied TOTP code reused within 90s                                           THEN REJECT               — Replay defence (TOTP codes valid for 90s window).
T027 [Auth] WHEN HTTPLogin           IF user-agent missing or empty                                                    THEN REJECT               — All real clients send UA.
T028 [Auth] WHEN HTTPLogin           IF HTTPLogin called by anyone (currently dead path)                                THEN ALERT                — Dead code being exercised = reconnaissance.
T029 [Auth] WHEN Connect             IF ProxyProtocolEnabled and missing PROXY header                                  THEN KICK                 — Direct connection bypassing the proxy.
T030 [Auth] WHEN Login               IF same hardware-fingerprint logs in across >= 3 accounts in 1h                   THEN ALERT                — Multi-account farmer.
```

## 2. Movement / Speed (T031–T060)

```
T031 [Move] WHEN Walk                IF positions_per_second > 2.5                                                     THEN KICK                 — Walk is gated at MoveDelay=600ms; >2 steps/s is speedhack.
T032 [Move] WHEN Run                 IF positions_per_second > 4                                                       THEN KICK                 — Run is faster but bounded.
T033 [Move] WHEN Walk                IF dest_cell not adjacent to current_cell                                         THEN KICK + AUDIT         — Teleport-walk hack.
T034 [Move] WHEN Walk                IF dest_cell.CellAttribute != Walk                                                THEN KICK                 — Walking into a non-walkable cell.
T035 [Move] WHEN Run                 IF Player.CurrentBagWeight > Stats[BagWeight]                                     THEN KICK                 — Run with overweight; existing client check bypassed.
T036 [Move] WHEN Movement            IF Player has Frozen/Paralysis poison AND moves                                   THEN KICK                 — Status-effect bypass.
T037 [Move] WHEN Walk                IF distance traveled in 10s > legal_max(MoveSpeed)                                 THEN KICK                 — Accumulated drift detection.
T038 [Move] WHEN MapChange           IF target map not adjacent via known portal                                       THEN KICK + AUDIT         — Teleport to arbitrary map.
T039 [Move] WHEN MapChange           IF target map has NeedBridle AND no_horse                                         THEN KICK                 — Bypassing bridle gate.
T040 [Move] WHEN Login               IF saved CurrentLocation invalid for CurrentMap                                   THEN move to map start    — Crash-corruption recovery, not malicious.
T041 [Move] WHEN PositionMove        IF NPCObjectID == 0 AND not in PvP-arena flow                                     THEN KICK                 — Self-teleport without NPC session.
T042 [Move] WHEN PositionMove        IF dest in different map and map.NoTeleport                                       THEN KICK                 — Map flag bypass.
T043 [Move] WHEN PositionMove        IF dest in safe zone for opposing guild during conquest                           THEN KICK                 — War-shelter abuse.
T044 [Move] WHEN Walk                IF in NoRandomMove map AND multiple positions per 200ms                            THEN KICK                 — Anti-shuffle for boss arenas.
T045 [Move] WHEN MountUse            IF target item.UniqueID not in inventory                                          THEN REJECT               — Slot spoofing.
T046 [Move] WHEN MountUse            IF mount.Level > Player.Level + 5                                                 THEN ALERT                — Level-gated mount.
T047 [Move] WHEN AnyAction           IF Player.Dead == true                                                            THEN REJECT               — Dead-player action exploit.
T048 [Move] WHEN MapChange           IF map.NoRecall AND used recall ring                                              THEN REJECT               — Item-effect bypass.
T049 [Move] WHEN Movement            IF in StonedMode (boss mechanic) AND moves                                        THEN KICK                 — Crowd-control bypass.
T050 [Move] WHEN Walk                IF cell.SafeZone change without entering portal                                   THEN AUDIT                — Safe-zone scumming.
T051 [Move] WHEN MapChange           IF traversal speed across portals > 1/sec sustained                               THEN ALERT                — Portal-spam pet farming.
T052 [Move] WHEN Move                IF dist > Globals.DataRange in one packet                                         THEN KICK                 — Wall-crossing.
T053 [Move] WHEN Move                IF dest cell has SpellObject of opposing player (trap)                            THEN ALLOW + LOG          — Normal; logged to investigate trap effectiveness.
T054 [Move] WHEN Walk                IF direction not in MirDirection enum                                             THEN KICK                 — Out-of-range enum.
T055 [Move] WHEN Walk                IF Sneaking AND ActiveSwiftFeet                                                   THEN KICK                 — Mutually-exclusive states active.
T056 [Move] WHEN MapChange           IF target_map.Banned_chars contains caller                                        THEN REJECT               — Per-map ban list.
T057 [Move] WHEN Walk                IF FastRun true AND no equipped FastRun item                                      THEN KICK                 — Buff without source item.
T058 [Move] WHEN MapChange           IF target_map.MinLevel > Player.Level                                             THEN REJECT               — Level gate.
T059 [Move] WHEN Move                IF map is Conquest area AND not member of any participating guild                 THEN ALERT                — Spectator in arena.
T060 [Move] WHEN PositionMove        IF teleport scroll consumed but inventory shows no decrement                      THEN KICK + AUDIT         — Consumable dupe.
```

## 3. Combat / Damage (T061–T090)

```
T061 [Cmbt] WHEN Attack              IF attacks_per_second > 1000 / AttackSpeed                                       THEN KICK                 — Attack-speed hack.
T062 [Cmbt] WHEN Magic               IF spell not in player's UserMagic list                                           THEN KICK                 — Casting unlearned spell.
T063 [Cmbt] WHEN Magic               IF cast time < magic.GetDelay()                                                   THEN KICK                 — Cast-speed hack.
T064 [Cmbt] WHEN Magic               IF target_location outside magic.Info.Range                                       THEN REJECT               — Range exceeded; already server-enforced but log.
T065 [Cmbt] WHEN Attack              IF target.Race == Player AND map not PvP                                          THEN REJECT               — PvP outside arena.
T066 [Cmbt] WHEN Attack              IF target is GM AND attacker isn't                                                THEN ALERT + KICK         — Targeting administration.
T067 [Cmbt] WHEN Damage              IF damage > 5 * Stats[MaxDC]                                                      THEN ALERT + AUDIT        — Damage outside legal bounds.
T068 [Cmbt] WHEN Damage              IF AttackerHero exists AND damage delivered while owner offline                   THEN AUDIT                — Hero-bot operation.
T069 [Cmbt] WHEN Damage              IF target.HP unchanged for 60s while taking continuous damage                     THEN ALERT                — God-mode flag.
T070 [Cmbt] WHEN Death               IF same Player dies > 30 times in 5min in PvP                                     THEN ALERT                — XP-feed alt.
T071 [Cmbt] WHEN Death               IF dropped items had bind DontDrop                                                THEN ROLLBACK             — Bind bypass.
T072 [Cmbt] WHEN Magic               IF mana cost less than MagicCost(magic)                                           THEN REJECT               — Mana-cost bypass.
T073 [Cmbt] WHEN Magic               IF SpellTime not yet elapsed since previous cast                                  THEN REJECT               — Cooldown skip.
T074 [Cmbt] WHEN Attack              IF Player.Stats[Stat.HP] increases via passive while attacking                    THEN AUDIT                — Vampirism abuse.
T075 [Cmbt] WHEN Magic               IF spell == Teleport AND used in map.NoTeleport                                   THEN REJECT               — Map flag bypass.
T076 [Cmbt] WHEN Damage              IF defender.in safe zone                                                          THEN REJECT               — Safe-zone shooting.
T077 [Cmbt] WHEN Damage              IF defender.PoisonList grows > 5 stacks                                           THEN AUDIT                — Poison overflow.
T078 [Cmbt] WHEN Magic               IF spell.target requires friendly AND target.IsAttackTarget == true               THEN REJECT               — Friendly-fire of buffs.
T079 [Cmbt] WHEN Attack              IF attacker.CurrentMap != target.CurrentMap                                       THEN KICK                 — Cross-map attack.
T080 [Cmbt] WHEN Magic               IF spell == TrapHexagon AND already cast within 18s                               THEN REJECT               — Cooldown enforcement.
T081 [Cmbt] WHEN Attack              IF Direction not consistent with attacker→target vector                           THEN AUDIT                — Aimbot signal.
T082 [Cmbt] WHEN Damage              IF Player solos boss whose recommended level > Player.Level + 20                  THEN ALERT                — Boss-carry exploit.
T083 [Cmbt] WHEN Attack              IF same target dies repeatedly via low-DPS character                              THEN ALERT                — Drop-farm bot.
T084 [Cmbt] WHEN Magic               IF cast within map.NoFight                                                        THEN REJECT               — Map-flag enforcement.
T085 [Cmbt] WHEN Magic               IF same spell cast 100 times in 60s                                               THEN THROTTLE             — Macro signature.
T086 [Cmbt] WHEN Damage              IF damage applied to BindMode.Invulnerable target                                  THEN REJECT               — Invuln-target hit.
T087 [Cmbt] WHEN Damage              IF target is QuestNPC                                                              THEN REJECT               — Quest-griefing.
T088 [Cmbt] WHEN Death               IF death within 1s of map entry                                                   THEN AUDIT                — Bounce-die exploit for buffs.
T089 [Cmbt] WHEN Attack              IF holding item with durability == 0 (broken) deals normal damage                  THEN AUDIT                — Broken-item exploit.
T090 [Cmbt] WHEN Magic               IF Plague cast with negative SC                                                   THEN REJECT               — Underflow into massive damage.
```

## 4. Inventory / Items (T091–T120)

```
T091 [Inv ] WHEN MoveItem            IF from_slot.Item == null                                                          THEN KICK                 — Slot spoofing.
T092 [Inv ] WHEN MoveItem            IF item.Bind has DontStore AND moving to Storage                                  THEN REJECT               — Bind bypass.
T093 [Inv ] WHEN SplitItem           IF Count >= source.Count                                                          THEN REJECT               — Already enforced; log if hit.
T094 [Inv ] WHEN DropItem            IF item.UniqueID changes within 5s of drop                                        THEN ROLLBACK + AUDIT     — Dupe via swap.
T095 [Inv ] WHEN DropItem            IF same item.UniqueID dropped twice                                               THEN ALERT + AUDIT        — Definitive dupe.
T096 [Inv ] WHEN UseItem             IF item.Info.Type != consumable AND consumed                                      THEN AUDIT                — Use-fail handling.
T097 [Inv ] WHEN PickUp              IF item.Owner not in {self, party}                                                THEN REJECT               — Loot-stealing.
T098 [Inv ] WHEN RefineItem          IF success_rate computed > legal_max                                              THEN REJECT               — Refine-rate hack.
T099 [Inv ] WHEN UpgradeItem         IF stat increase > legal_max(item.tier)                                           THEN ROLLBACK             — Upgrade overflow.
T100 [Inv ] WHEN CombineItem         IF same UniqueID for from and to                                                  THEN REJECT               — Self-combine dupe.
T101 [Inv ] WHEN MergeItem           IF resulting Count > ushort.MaxValue                                              THEN REJECT               — Stack-count overflow.
T102 [Inv ] WHEN MergeItem           IF items have different ItemInfo.Index                                            THEN REJECT               — Mix-merge.
T103 [Inv ] WHEN StoreItem           IF storage_index not in [0..80)                                                   THEN REJECT               — Bounds-bypass.
T104 [Inv ] WHEN TakeBackItem        IF storage[index] == null                                                         THEN REJECT               — Phantom take.
T105 [Inv ] WHEN EquipItem           IF item.RequiredLevel > Player.Level                                              THEN REJECT               — Level-gate bypass.
T106 [Inv ] WHEN EquipItem           IF item.RequiredClass != Player.Class                                             THEN REJECT               — Class-gate bypass.
T107 [Inv ] WHEN EquipItem           IF item.RequiredGender != Player.Gender                                           THEN REJECT               — Gender-gate bypass.
T108 [Inv ] WHEN RemoveSlotItem      IF unique IDs don't both exist                                                    THEN REJECT               — Dual-ID spoof.
T109 [Inv ] WHEN GainItem            IF inventory_count_for_item exceeds bind stack limit                              THEN REJECT               — Stack-bypass dupe.
T110 [Inv ] WHEN Awakening           IF type not in valid AwakeType enum                                               THEN REJECT               — Enum overflow.
T111 [Inv ] WHEN Awakening           IF same item awakened 10+ times in 60s                                            THEN THROTTLE             — Macro spam.
T112 [Inv ] WHEN DisassembleItem     IF source item.Info.Bind has BindMode.NoDisassemble                                THEN REJECT               — Bind enforcement.
T113 [Inv ] WHEN UseItem             IF cooldown not yet elapsed                                                       THEN REJECT               — Pot-spam exploit.
T114 [Inv ] WHEN UseItem             IF effect resets while previous still active (overlap)                            THEN REJECT               — Buff-stacking abuse.
T115 [Inv ] WHEN GainItem            IF item not in ItemInfoList (unknown index)                                       THEN REJECT + AUDIT       — Forged item.
T116 [Inv ] WHEN ItemInfo            IF item.UniqueID == 0 in any code path                                            THEN REJECT               — Sentinel-as-real exploit.
T117 [Inv ] WHEN RentalReturn        IF item.RentalInformation.RentalLocked                                            THEN REJECT               — Rental escape.
T118 [Inv ] WHEN MoveItem            IF moving cursed item out of equip slot during PvP cooldown                       THEN REJECT               — Curse bypass.
T119 [Inv ] WHEN MoveItem            IF MoveItem rate > 10/sec                                                         THEN THROTTLE             — Inventory-shuffle macro.
T120 [Inv ] WHEN Inventory_Diff      IF count of items grew without corresponding GainItem trigger                     THEN ALERT + AUDIT        — Server bug or exploit.
```

## 5. Gold / Trade / Auction / Mail (T121–T150)

```
T121 [Gold] WHEN DropGold            IF Amount > Account.Gold                                                          THEN REJECT               — Already enforced; trigger logs.
T122 [Gold] WHEN DropGold            IF Amount == 0                                                                    THEN REJECT               — Wasted packet, scripted.
T123 [Gold] WHEN GainGold            IF Account.Gold + Amount overflows uint                                           THEN AUDIT                — Already capped; record cap-hits.
T124 [Gold] WHEN GainGold            IF gold delta > 10M in single event                                               THEN ALERT                — Mega drop.
T125 [Gold] WHEN TradeGold           IF Amount > Account.Gold                                                          THEN REJECT               — Already enforced.
T126 [Gold] WHEN TradeGold           IF trade partner offline mid-trade                                                THEN ROLLBACK             — Disconnect dupe.
T127 [Gold] WHEN TradeAccept         IF accepting party didn't move within last 30s (likely afk-bot)                   THEN ALERT                — Auto-trade pattern.
T128 [Gold] WHEN BuyItem             IF Player.NPCPage not Buy/BuySell/BuyBack/PearlBuy/BuyNew/BuySellNew/BuyUsed       THEN KICK                 — Page check enforcement.
T129 [Gold] WHEN BuyItem             IF Count * unit_price overflows                                                   THEN REJECT               — Multiplication overflow.
T130 [Gold] WHEN SellItem            IF item.RentalInformation != null AND BindingFlags.DontSell                       THEN REJECT               — Rental sell.
T131 [Gold] WHEN ConsignItem         IF price < MinConsignment OR > MaxConsignment                                      THEN REJECT               — Already enforced.
T132 [Gold] WHEN ConsignItem         IF Account.Auctions.Count > Globals.MaxAuctionsPerAccount                          THEN REJECT               — List-spam DoS.
T133 [Gold] WHEN MarketBuy           IF bidPrice < auction.CurrentBid                                                  THEN REJECT               — Already enforced.
T134 [Gold] WHEN MarketBuy           IF auction.SellerInfo == caller.Info                                              THEN REJECT               — Wash-trade.
T135 [Gold] WHEN MarketBuy           IF same auction bid 10+ times within 60s                                          THEN THROTTLE             — Bid-spam attack.
T136 [Gold] WHEN SendMail            IF Account.Gold < gold + parcel_cost                                              THEN REJECT               — Already enforced.
T137 [Gold] WHEN SendMail            IF gold > 100M in single mail                                                     THEN ALERT                — Outsized mailed gold.
T138 [Gold] WHEN SendMail            IF recipient is same Account.Index as sender                                      THEN REJECT               — Self-mail laundering.
T139 [Gold] WHEN SendMail            IF mails_sent_in_1h > 50                                                          THEN THROTTLE             — Mail-spam.
T140 [Gold] WHEN SendMail            IF item.Bind has NoMail                                                           THEN REJECT               — Already enforced.
T141 [Gold] WHEN GuildStorageGold    IF caller.GuildRank.CanModifyTreasury == false                                    THEN REJECT               — Permission bypass.
T142 [Gold] WHEN GuildStorageGold    IF Amount > Guild.GoldStorage                                                     THEN REJECT               — Withdraw above balance.
T143 [Gold] WHEN GuildStorageGold    IF withdraw_total_in_1h > 100M                                                    THEN ALERT                — Officer pulling treasury.
T144 [Gold] WHEN Pay                 IF same gold ID flows A→B→A within 60s                                            THEN ALERT                — Round-trip wash.
T145 [Gold] WHEN GameShopBuy         IF Account.Credit < cost                                                          THEN REJECT               — Credit bypass.
T146 [Gold] WHEN GameShopBuy         IF item.Info.NotInGameShop                                                        THEN REJECT               — Catalog tampering.
T147 [Gold] WHEN PayPalCallback      IF capture_id seen before                                                         THEN REJECT               — Replay; already enforced by PayPalTransactionLog.
T148 [Gold] WHEN PayPalCallback      IF amount mismatch vs server-side cart                                            THEN ALERT                — Tampered checkout.
T149 [Gold] WHEN DonateGold          IF donation > 0 AND no scripted UI flow                                           THEN AUDIT                — Direct donate packet.
T150 [Gold] WHEN GoldChange          IF cumulative gold gain in 24h > 1B                                               THEN ALERT                — Whale or exploit.
```

## 6. NPC / Shop / Quest (T151–T180)

```
T151 [NPC ] WHEN CallNPC             IF p.Key length > 30                                                              THEN KICK                 — Already enforced.
T152 [NPC ] WHEN CallNPC             IF target ObjectID not in CurrentMap.NPCs                                         THEN REJECT               — Cross-map NPC.
T153 [NPC ] WHEN CallNPC             IF distance(npc, player) > Globals.DataRange                                      THEN REJECT               — Already enforced.
T154 [NPC ] WHEN NPCConfirmInput     IF p.Value length > 200                                                           THEN KICK                 — Already enforced.
T155 [NPC ] WHEN NPCConfirmInput     IF p.PageName length > 30                                                         THEN KICK                 — Already enforced.
T156 [NPC ] WHEN NPCConfirmInput     IF Player.NPCObjectID == 0                                                        THEN REJECT               — Already enforced.
T157 [NPC ] WHEN NPCInteract         IF Player rapid-cycles NPCs at >5/sec                                             THEN THROTTLE             — NPC-script abuse.
T158 [NPC ] WHEN NPCBuy              IF price calculated server-side != client-shown                                   THEN AUDIT                — Tamper detection.
T159 [NPC ] WHEN NPCSell             IF item RentalInformation indicates loaned                                        THEN REJECT               — Loan dupe.
T160 [NPC ] WHEN QuestAccept         IF level < quest.MinLevel                                                         THEN REJECT               — Level gate.
T161 [NPC ] WHEN QuestComplete       IF Quest.Status invalid for current Player state                                  THEN REJECT               — State-machine skip.
T162 [NPC ] WHEN QuestComplete       IF quest already completed AND repeatable == false                                THEN REJECT               — Reward double-claim.
T163 [NPC ] WHEN QuestReward         IF reward Gold > computed_max                                                     THEN REJECT               — Reward-inflation.
T164 [NPC ] WHEN QuestUpdate         IF quest progress moves backwards                                                 THEN AUDIT                — State corruption.
T165 [NPC ] WHEN NPCScript           IF script uses $NPCInputStr in unbounded shell-like command                       THEN ALERT (script author) — Script injection vector.
T166 [NPC ] WHEN NPCScriptError      IF error rate > 100/min for any script                                            THEN ALERT                — Probe attempts.
T167 [NPC ] WHEN CraftItem           IF materials don't actually exist in inventory                                    THEN REJECT               — Recipe forging.
T168 [NPC ] WHEN CraftItem           IF resulting item not in recipe table                                             THEN REJECT               — Out-of-recipe creation.
T169 [NPC ] WHEN CraftItem           IF chance_to_succeed bumped above legal                                           THEN REJECT               — Macro-tier hack.
T170 [NPC ] WHEN ExpHall             IF EXP gain rate > computed_max                                                   THEN ALERT                — XP-storage hack.
T171 [NPC ] WHEN NPCConsign          IF same item.UniqueID consigned twice                                             THEN REJECT               — Dupe via consign-stack.
T172 [NPC ] WHEN NPCBuyBack          IF item not in BuyBack list                                                       THEN REJECT               — Phantom buyback.
T173 [NPC ] WHEN NPCBuy              IF index doesn't match valid Goods entry                                          THEN REJECT               — Catalog spoof.
T174 [NPC ] WHEN GameShopBuy         IF GameShopItem.Stock == 0 AND purchase succeeds                                  THEN REJECT               — Stock-counter bypass.
T175 [NPC ] WHEN GameShopBuy         IF item count >= GameShopItem.iStock per account                                   THEN REJECT               — Per-account cap.
T176 [NPC ] WHEN NPCBuy              IF buyer Hero AND Hero not present on map                                         THEN REJECT               — Hero-shadow buy.
T177 [NPC ] WHEN MarriageProposal    IF either party is married already                                                THEN REJECT               — State-check.
T178 [NPC ] WHEN MarriageProposal    IF same party proposes 20+ times in 1h                                            THEN THROTTLE             — Harassment.
T179 [NPC ] WHEN AdoptHero           IF Player.Heroes.Count >= Globals.MaxHeroes                                       THEN REJECT               — Cap.
T180 [NPC ] WHEN Mentor              IF mentor.Level < mentee.Level                                                    THEN REJECT               — Reversed mentor.
```

## 7. Chat / Communication / Social (T181–T200)

```
T181 [Chat] WHEN Chat                IF message length > Globals.MaxChatLength                                        THEN KICK                 — Already enforced.
T182 [Chat] WHEN Chat                IF message contains URL AND account_age < 24h                                    THEN ALERT                — Spam-bot signature.
T183 [Chat] WHEN Chat                IF same message repeated 5+ times in 60s                                         THEN THROTTLE             — Spam.
T184 [Chat] WHEN Chat                IF message contains profanity                                                    THEN REDACT + WARN        — Filter.
T185 [Chat] WHEN Chat                IF message contains GMPassword string                                             THEN REJECT + ALERT       — Plaintext leak; secrets never echoed.
T186 [Chat] WHEN Whisper             IF target not online                                                              THEN REJECT               — Probe to confirm presence.
T187 [Chat] WHEN Whisper             IF target has blocked sender                                                      THEN REJECT               — Block-list.
T188 [Chat] WHEN ChatItems           IF item-link UniqueID not in inventory                                            THEN REJECT               — Item-link forgery.
T189 [Chat] WHEN ChatItems           IF link count > 5 per message                                                     THEN REJECT               — Spam.
T190 [Chat] WHEN GuildChat           IF caller not in target Guild                                                     THEN REJECT               — Cross-guild eavesdrop.
T191 [Chat] WHEN GroupChat           IF Player.Group == null                                                           THEN REJECT               — No-group send.
T192 [Chat] WHEN RelationshipChat    IF Player.LoverIndex == 0                                                         THEN REJECT               — No lover.
T193 [Chat] WHEN MentorChat          IF Player.MentorIndex == 0                                                        THEN REJECT               — No mentor.
T194 [Chat] WHEN GuildInvite         IF inviter doesn't have InviteRank                                                THEN REJECT               — Permission.
T195 [Chat] WHEN GuildInvite         IF invitee already in another guild                                               THEN REJECT               — State.
T196 [Chat] WHEN GuildInvite         IF invites_in_1h_from_caller > 50                                                 THEN THROTTLE             — Spam-invite.
T197 [Chat] WHEN FriendAdd           IF friend list >= max                                                             THEN REJECT               — Cap.
T198 [Chat] WHEN ReportPlayer        IF reporter rate > 5/h                                                            THEN THROTTLE             — Abuse-of-report.
T199 [Chat] WHEN AnnounceChat        IF caller.IsGM == false                                                           THEN REJECT               — Already enforced.
T200 [Chat] WHEN TriggerChat         IF script tries to log message containing 'password'/'pepper'/'secret' literal     THEN REJECT + ALERT       — Log poisoning.
```

## 8. Spell / Magic / Buff (T201–T220)

```
T201 [Spel] WHEN AddBuff             IF same BuffType added 10+ times in 10s                                           THEN THROTTLE             — Buff-stack DoS.
T202 [Spel] WHEN AddBuff             IF buff.Caster is dead/disconnected                                               THEN REJECT               — Caster-gone.
T203 [Spel] WHEN AddBuff             IF buff.ExpireTime > 30 * 60 * 1000 (30min) AND not infinite                      THEN REJECT               — Excessive duration.
T204 [Spel] WHEN AddBuff             IF buff.Stats contains negative values  with positive source                      THEN AUDIT                — Stat-flip exploit.
T205 [Spel] WHEN BuffStack           IF duration computed > Int32.MaxValue/2                                           THEN REJECT               — Overflow-near.
T206 [Spel] WHEN RemoveBuff          IF removing buff Player doesn't own                                               THEN REJECT               — Cross-player tampering.
T207 [Spel] WHEN BuffSerialize       IF buff.Info.Visible == false AND broadcasted                                     THEN AUDIT                — Visibility leak.
T208 [Spel] WHEN Hiding              IF Player attacks while Hidden                                                    THEN auto-remove Hidden + LOG — Pattern enforcement.
T209 [Spel] WHEN Hiding              IF Player remains hidden in safe zone > 60s                                       THEN LOG                  — Stalking signal.
T210 [Spel] WHEN MagicShield         IF same Player has MagicShield + ImmortalSkin + BlessedArmour simultaneously       THEN ALERT                — Stack-aggregation.
T211 [Spel] WHEN UltimateEnhancer    IF target.Class is incompatible                                                   THEN REJECT               — Class-gate.
T212 [Spel] WHEN Purification        IF target.PoisonList empty AND caster bills cost                                  THEN AUDIT                — Wasted cast.
T213 [Spel] WHEN Mass*               IF affected count > radius*radius*4                                               THEN AUDIT                — Aoe-area math sanity.
T214 [Spel] WHEN TrapHexagon         IF map.NoFight                                                                    THEN REJECT               — Map flag.
T215 [Spel] WHEN SoulFireBall        IF amulet count consumed mismatches cast                                          THEN REJECT               — Consumable bypass.
T216 [Spel] WHEN Concentration       IF Interrupted set externally without buff flag                                   THEN AUDIT                — State corruption.
T217 [Spel] WHEN MoonLight           IF cast cost == 0                                                                 THEN REJECT               — Free-cast.
T218 [Spel] WHEN ManaShield          IF Player.MP reduced below zero by shield absorption                              THEN AUDIT                — Underflow exploit.
T219 [Spel] WHEN BindingShot         IF affected mob already BindingShotCenter==true                                   THEN REJECT               — Re-stun loop.
T220 [Spel] WHEN Reincarnation       IF cast on player whose Death.Time > 2min ago                                     THEN REJECT               — Window enforcement.
```

## 9. Network / Protocol (T221–T250)

```
T221 [Net ] WHEN PacketRate          IF per-connection per-tick > MaxReceivePerTick                                    THEN KICK                 — Already enforced.
T222 [Net ] WHEN PacketRate          IF per-IP > MaxIPPacketsPerSecond                                                 THEN KICK                 — Already enforced.
T223 [Net ] WHEN PacketHeader        IF size field > 64KB                                                              THEN KICK                 — Spec violation.
T224 [Net ] WHEN PacketHeader        IF size field < 4                                                                 THEN KICK                 — Spec violation.
T225 [Net ] WHEN PacketParse         IF deserialize throws                                                             THEN KICK                 — Malformed.
T226 [Net ] WHEN PacketParse         IF unknown packetId                                                               THEN KICK                 — Reconnaissance / version mismatch.
T227 [Net ] WHEN PacketReceive       IF same packetId in a row > 200                                                   THEN KICK                 — Stuck client OR macro.
T228 [Net ] WHEN PacketSend          IF outbound queue > 10 MB                                                         THEN KICK                 — Slow consumer DoS.
T229 [Net ] WHEN PacketEnqueueRate   IF retry queue grows > MaxRetryQueue                                              THEN KICK                 — Already enforced.
T230 [Net ] WHEN Connection          IF Receive fires faster than 200 µs (impossibly fast)                             THEN KICK                 — Local proxy bot.
T231 [Net ] WHEN Disconnect          IF same IP reconnects within 100 ms                                               THEN BAN_SHORT            — Reconnect-storm.
T232 [Net ] WHEN ConnectionEstablish IF SYN-burst > 100/sec from one IP                                                THEN BAN_SHORT(ip)        — Connection flood.
T233 [Net ] WHEN PROXYHeaderParse    IF source IP in header is private RFC1918 from public proxy                       THEN REJECT               — Header spoof.
T234 [Net ] WHEN PROXYHeaderParse    IF parse error                                                                    THEN KICK + AUDIT         — Bad proxy or attacker direct-connect.
T235 [Net ] WHEN ConnectionLifetime  IF connection alive > 24h                                                         THEN AUDIT                — Persistent bot.
T236 [Net ] WHEN PacketTimeout       IF TimeOutTime exceeded                                                           THEN KICK                 — Already enforced.
T237 [Net ] WHEN Connection          IF same SessionID requested by client (replay)                                    THEN REJECT               — Session-id spoof.
T238 [Net ] WHEN HTTPRequest         IF user-agent in known scanner list                                                THEN BAN_SHORT(ip)        — Recon.
T239 [Net ] WHEN HTTPRequest         IF path != known endpoints                                                        THEN 404 + LOG            — Probing.
T240 [Net ] WHEN HTTPLogin           IF rate > 5/min from one IP                                                       THEN BAN_SHORT(ip)        — Brute.
T241 [Net ] WHEN UnencryptedLogin    IF behind TLS proxy AND PROXY header missing                                      THEN KICK + ALERT         — Bypass attempt.
T242 [Net ] WHEN ChatTraffic         IF outbound bytes > 1 MB/s for one player                                         THEN THROTTLE             — Chat-amp DoS.
T243 [Net ] WHEN Compression         IF compressed packet >= raw size +20%                                             THEN AUDIT                — Wrong compression.
T244 [Net ] WHEN PacketLatency       IF response time to server packet < 1ms (impossible RTT)                          THEN AUDIT                — Loopback-bot.
T245 [Net ] WHEN Sequence            IF KeepAlive sent < 100ms after previous                                          THEN KICK                 — Spam.
T246 [Net ] WHEN MultipleConnections IF account.Connection already set AND new login arrives                            THEN disconnect_old (current) — Race-condition window minimized.
T247 [Net ] WHEN BurstStream         IF receive_buffer fill > 50% sustained                                            THEN ALERT                — Approaching DoS.
T248 [Net ] WHEN FragmentedPacket    IF reassembly across > 10 receives                                                THEN KICK                 — Slow-trickle attack.
T249 [Net ] WHEN PacketSize          IF size mismatch vs declared                                                       THEN KICK                 — Length-prefix exploit.
T250 [Net ] WHEN TCPState            IF half-open RST-flood detected                                                   THEN log + rely on OS     — IDS hint.
```

## 10. Map / Position / Spawn (T251–T270)

```
T251 [Map ] WHEN RespawnAttempt      IF respawn cell.CellAttribute != Walk                                             THEN AUDIT                — Spawn-in-wall.
T252 [Map ] WHEN MonsterSpawn        IF spawn count exceeds map respawn cap                                            THEN AUDIT                — Spawn overflow.
T253 [Map ] WHEN ItemDrop            IF same item.UniqueID created twice                                               THEN ALERT                — Server bug or dupe.
T254 [Map ] WHEN MapEntry            IF map.Players.Count > MaxPlayersPerMap                                           THEN REJECT               — Soft-cap DoS.
T255 [Map ] WHEN MapWeather          IF lightning damage > Info.LightningDamage                                        THEN REJECT               — Damage cap.
T256 [Map ] WHEN MapMine             IF Mine harvest by player > rate-cap/min                                          THEN THROTTLE             — Mining bot.
T257 [Map ] WHEN ConquestActive      IF non-participant moves to gate                                                  THEN REJECT               — Conquest scumming.
T258 [Map ] WHEN MapSafeZone         IF same player camps SZ > 30min and AFK                                            THEN AUTO_KICK            — AFK farming.
T259 [Map ] WHEN GroundEffect        IF SpellObject persists past its ExpireTime                                       THEN AUDIT                — State leak.
T260 [Map ] WHEN ItemPickup          IF cross-map pickup detected                                                      THEN REJECT               — Coordinate exploit.
T261 [Map ] WHEN MapInstance         IF instance has zero population for 5min                                          THEN CLEANUP              — Memory hygiene.
T262 [Map ] WHEN MapResize           IF dynamic map > Width*Height bounds                                              THEN REJECT               — OOM.
T263 [Map ] WHEN PortalUse           IF cooldown not elapsed                                                           THEN REJECT               — Portal-spam.
T264 [Map ] WHEN PortalUse           IF portal disabled                                                                THEN REJECT               — State.
T265 [Map ] WHEN BindMap             IF bind to map.NoBind                                                             THEN REJECT               — Flag.
T266 [Map ] WHEN MoveTo              IF dest cell has CellAttribute.HighWall AND HighWallWalk == false                 THEN KICK                 — Wall-hack.
T267 [Map ] WHEN MoveTo              IF dest cell has CellAttribute.LowWall AND LowWallJump == false                   THEN KICK                 — Wall-hack.
T268 [Map ] WHEN DoorOpen            IF door.Locked AND key not in inventory                                           THEN REJECT               — Key bypass.
T269 [Map ] WHEN GateAttack          IF target Gate not in current siege                                               THEN REJECT               — Gate griefing.
T270 [Map ] WHEN MapEntry            IF entry of a player who is currently dead on another map                         THEN ROLLBACK             — State desync.
```

## 11. GM / Admin (T271–T285)

```
T271 [GM  ] WHEN @MAKE               IF count > 100                                                                    THEN AUDIT                — Mass-item conjure.
T272 [GM  ] WHEN @MOB                IF AI flag in {conquestAIs}                                                       THEN REJECT               — Already enforced.
T273 [GM  ] WHEN @KICK               IF target is also GM with higher rank                                             THEN REJECT               — GM hierarchy.
T274 [GM  ] WHEN @BAN                IF reason empty                                                                   THEN REJECT               — Audit hygiene.
T275 [GM  ] WHEN @CLEARGOLD          IF target.Account != caller AND caller.Rank < SuperGM                             THEN REJECT               — Privilege gate.
T276 [GM  ] WHEN @GIVE               IF item not in ItemInfoList                                                       THEN REJECT               — Forged item.
T277 [GM  ] WHEN @LEVEL              IF target_level > Globals.MaxLevel                                                THEN REJECT               — Cap.
T278 [GM  ] WHEN @LOGIN              IF GMPassword input contains non-ASCII                                            THEN REJECT               — Encoding attack.
T279 [GM  ] WHEN @LOGIN              IF login attempt rate >5/min                                                      THEN BAN_SHORT(account)   — GM-password brute.
T280 [GM  ] WHEN @SUMMON             IF target is GM                                                                   THEN REJECT               — Anti-harassment.
T281 [GM  ] WHEN @ANY                IF non-GM matched IsGM == false but on TestServer                                  THEN AUDIT                — Test-server abuse.
T282 [GM  ] WHEN GMPasswordChange    IF without re-prompt                                                              THEN REJECT               — Stale-session.
T283 [GM  ] WHEN GMAuditTamper       IF HashChainLog.Verify fails                                                      THEN ALERT (URGENT)       — Log integrity.
T284 [GM  ] WHEN GMSessionDuration   IF >12h continuous                                                                THEN ALERT                — Stolen GM acct.
T285 [GM  ] WHEN GMOnlineCount       IF > expected_gm_count                                                            THEN ALERT                — Privilege escalation.
```

## 12. Patcher / Client (T286–T300)

```
T286 [Patch] WHEN PatchDownload       IF file hash mismatch                                                            THEN REJECT + ALERT       — Already enforced.
T287 [Patch] WHEN PatchManifest       IF signature invalid                                                             THEN REJECT + ALERT(URGENT) — MITM.
T288 [Patch] WHEN PatchManifest       IF entry count > MaxEntries                                                      THEN REJECT               — Already enforced.
T289 [Patch] WHEN PatchManifest       IF file size > MaxBytes                                                          THEN REJECT               — Already enforced.
T290 [Patch] WHEN Launcher            IF version mismatch with Server                                                  THEN REJECT               — Stale client.
T291 [Patch] WHEN ClientFingerprint   IF executable hash not in approved manifest                                      THEN ALERT                — Modified client.
T292 [Patch] WHEN Library             IF DLL injected into Mir2.exe                                                    THEN ALERT                — Cheat-engine signature (best-effort).
T293 [Patch] WHEN Window              IF more than one Mir2 window per machine fingerprint                             THEN ALERT                — Multi-box farmer.
T294 [Patch] WHEN ServerInfo          IF client sends inconsistent screen resolution mid-session                       THEN AUDIT                — Window-injection.
T295 [Patch] WHEN PatchHost           IF non-HTTPS                                                                     THEN warn in docs         — Operator notice.
T296 [Patch] WHEN ClientStartup       IF launched without launcher                                                     THEN AUDIT                — Standalone exe.
T297 [Patch] WHEN ClientResources     IF .wil checksum mismatch                                                        THEN AUDIT                — Wallhack texture swap.
T298 [Patch] WHEN ClientLatency       IF measured RTT < cable-physical-minimum                                         THEN AUDIT                — Loopback bot.
T299 [Patch] WHEN VersionCheck        IF Settings.CheckVersion AND mismatch                                            THEN REJECT               — Already enforced.
T300 [Patch] WHEN AntiDebug           IF Mir2.exe detects attached debugger                                            THEN report to server     — Implement via timing-check trampoline.
```

---

## How to deploy

These rules are intended as a checklist + concrete spec, not a drop-in module. Implementation paths:

1. **Triggers/ runtime (in-process)** — Map each `WHEN <event>` to the existing `TriggerEvent` enum and write the predicate in the codebase's condition DSL. Best for actions that need to deny a packet (`REJECT`/`KICK`/`THROTTLE`).
2. **External SIEM** — Stream server logs (chat, GM audit, MessageQueue) to Loki/Splunk/Graylog and implement each predicate as a saved search + alert. Best for `ALERT`/`AUDIT` rules where action is human-driven.
3. **WAF / edge** — Anything network-layer (T221–T250) belongs at the reverse proxy / WAF, not the C# server.

Use the `Category` column to budget rollout: deploy the **Auth + Net** rules first (T001–T030 and T221–T250) — those defend against opportunistic attackers. **Inv + Gold** (T091–T150) next — those defend against the realm's economy. **Combat + Move** (T031–T090) are slower to weaponize against you and acceptable to deploy as a follow-up.

## Maintenance

- Every new packet handler MUST add at least one trigger in this table.
- Every confirmed incident MUST be retro-fitted into a trigger here so a recurrence is detected automatically.
- Quarterly: pick 10 triggers at random, fire them with a test harness, confirm the configured action triggers.
- When the **per-IP packet aggregator** (`Envir.RecordIPPackets`) tightens (e.g., new `MaxIPPacketsPerSecond`), update T222.
- When **`HashChainLog.Verify`** is automated, update T283 to "automated daily verification" rather than a manual trigger.
