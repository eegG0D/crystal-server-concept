# Crystal Server — Security Operations Runbook

This document is operator-facing, not developer-facing. It covers everything you need to do *outside* of the source tree to deploy the server safely.

---

## 1. Service account (least privilege)

Create a dedicated local Windows user for the game server. Never run as `Administrator` or your own daily account.

```powershell
# As Administrator on the game-server host:
$pw = Read-Host -AsSecureString "Password for 'crystal' service account"
New-LocalUser -Name "crystal" -Password $pw -PasswordNeverExpires -UserMayNotChangePassword `
              -Description "Crystal game server service account"

# DO NOT add this user to Administrators or any privileged group.
# It belongs to the default 'Users' group only.
```

For interactive testing, log in as `crystal` over `runas /user:.\crystal "PowerShell"`.

---

## 2. Filesystem ACLs

| Path | Access for `crystal` | Why |
|---|---|---|
| `Build\Server\Debug\` (executable) | **Read & Execute** only | Defeats RCE → binary rewrite. |
| `Build\Server\Debug\Configs\` | **Modify** | `Setup.ini`, `2fa.snapshot`, `2fa.journal` rewritten at runtime. |
| `Build\Server\Debug\Logs\` | **Modify** | Hash-chained logs; tamper-EVIDENT (see §5). |
| `Build\Server\Debug\Server.MirDB*` | **Modify** | Database file. |
| `Build\Server\Debug\Maps\`, `Exports\` | **Read** only | |
| Everything else | No access | |

```powershell
$root = "C:\crystal\crystal source fresh\Crystal-master\Build\Server\Debug"
$acct = "$env:COMPUTERNAME\crystal"
icacls "$root" /inheritance:r
icacls "$root" /grant:r "Administrators:(OI)(CI)F" "SYSTEM:(OI)(CI)F" "${acct}:(OI)(CI)RX"
icacls "$root\Configs"   /grant:r "${acct}:(OI)(CI)M"
icacls "$root\Logs"      /grant:r "${acct}:(OI)(CI)M"
icacls "$root\Server.MirDB"  /grant:r "${acct}:M" 2>$null
```

⚠️ The `crystal` account has Modify on `Logs\`, which means it could in theory rewrite the audit log. The chain-of-hashes (§5) makes any in-place edit detectable; you also need to ship logs off-host (§5) so an attacker who pops the service can't quietly rewrite history before discovery.

---

## 3. Windows Firewall

```powershell
New-NetFirewallRule -DisplayName "Crystal Game"      -Direction Inbound -Action Allow -Protocol TCP -LocalPort 7000
New-NetFirewallRule -DisplayName "Crystal Login TLS" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 443
Disable-NetFirewallRule -DisplayName "Remote Desktop*"
Disable-NetFirewallRule -DisplayName "File and Printer Sharing*"
```

Remote management via VPN / jumpbox only.

---

## 4. Required environment variables

Set in the **service account's** environment **before** `Server.exe` starts. The server reads them at process start.

| Variable | What it does |
|---|---|
| `CRYSTAL_PASSWORD_PEPPER` | The CURRENT pepper. Applied to every new hash. |
| `CRYSTAL_PASSWORD_PEPPER_P<N>` | Named historical pepper (`_P1`, `_P2`, …) kept for verification of unmigrated hashes. See *Pepper rotation* below. |
| `CRYSTAL_GM_PASSWORD` | Overrides the `.ini` default. Production must override. |

### Generating secrets (CSPRNG, not `Get-Random`)

```powershell
$bytes = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
[Convert]::ToBase64String($bytes)
```

### Verifying

```powershell
# In a NEW PowerShell window after setting Machine-scope:
[Environment]::GetEnvironmentVariable("CRYSTAL_PASSWORD_PEPPER", "Machine")
```

### Pepper rotation (the SAFE way)

The system supports any number of named pepper versions. Each stored hash records which pepper version produced it. To rotate cleanly:

1. **Generate a new secret.**
2. **Move the current pepper to a numbered slot.** If `CRYSTAL_PASSWORD_PEPPER` was `<old>`, set `CRYSTAL_PASSWORD_PEPPER_P1 = <old>` (or `_P2`, `_P3`, etc. — pick a number you haven't used).
3. **Set the new pepper.** `CRYSTAL_PASSWORD_PEPPER = <new>`.
4. **Restart the server.** Existing hashes are still verifiable via the kept variable; new hashes use the new pepper.
5. **Wait.** As users log in, the `NeedsRehash` path silently re-hashes their password under the new pepper. There is no forced disruption.
6. **Optional cleanup.** Once you're satisfied (e.g., monitor login traffic — when the rate of unmigrated-pepper logins falls to ~zero, say after 30–60 days), unset `CRYSTAL_PASSWORD_PEPPER_P<N>`. Any user who *still* hasn't logged in by then will need a password reset, OR you can keep the old variable indefinitely with no security impact.

**Backup the active pepper to your password vault.** Losing it locks out accounts that have not migrated to a newer version.

---

## 5. Logs to monitor (and how to verify tamper-evidence)

| Path | What's in it | Watch for |
|---|---|---|
| `Logs\GMAudit\gm-YYYY-MM-DD.log` | Every GM `@` command attempt, hash-chained | Unusual `@MAKE`, `@MOB <bossname>`, account changes |
| Server console / `Logs\Server*` | Login failures, IP blocks, flood disconnects | Repeated "Too many failed login attempts from this IP"; "packet flood"; "Rejected connection (bad PROXY header)" |
| `Logs\Chat*` | All in-game chat (if enabled) | Coordinated dupe-attempt chatter |
| `Error.txt` | Unhandled exceptions from network code | Anything new |

**Daily**: ship `Logs\GMAudit\*` off-host before the day rolls over. Verify the chain on the off-host copy:

```csharp
// One-liner if you're scripting against the Server.Library assembly:
var (ok, badLine) = Server.Security.HashChainLog.Verify(@"C:\offhost\gm-2026-05-09.log");
if (!ok) Console.WriteLine($"AUDIT TAMPERING DETECTED at line {badLine}");
```

A chain break anywhere indicates either disk corruption or an attacker rewriting history. Both warrant immediate investigation.

The chain is tamper-EVIDENT, not tamper-PROOF — an attacker with continuous write access can rewrite from any chosen point and recompute hashes forward. The defense is **the off-host copy you took before the breach.**

---

## 6. Backup policy

Daily:

1. `Build\Server\Debug\Server.MirDB`
2. `Build\Server\Debug\Configs\` (`2fa.snapshot`, `2fa.journal`, `Setup.ini`)
3. `Build\Server\Debug\Logs\` (especially `GMAudit\`)

**Encrypt at rest. Never store the password pepper in the same backup as the DB** — defeats the entire point.

---

## 7. Patch / update channel

The launcher verifies SHA-256 per downloaded file (from `PList.gz`) and, if a public key is embedded, an RSA-PSS signature over the entire manifest.

One-time, on a secure build box (not the game server):

```powershell
$rsa = [System.Security.Cryptography.RSA]::Create(2048)
$rsa.ExportSubjectPublicKeyInfoPem()  | Out-File -Encoding ascii pubkey.pem
$rsa.ExportRSAPrivateKeyPem()         | Out-File -Encoding ascii privkey.pem
```

Paste `pubkey.pem` into `Client\Forms\AMain.cs:ManifestPublicKeyPem`. Rebuild client. Keep `privkey.pem` only on the build box.

Per-release:

```powershell
dotnet run --project Tools\PatchPublisher -- `
    .\Build\Client\Release `
    .\Publish\PList.gz `
    --privkey C:\Vault\privkey.pem
```

---

## 8. Lock hierarchy (developer-facing)

Code that takes more than one lock MUST take them in this order to avoid deadlock. Reviewers: block PRs that introduce a new lock without adding it to this list.

```
AccountLock                          (outermost — broad)
  ↓
LoadLock
  ↓
Envir._loginAttemptLock             (rate limiter)
  ↓
Envir._ipPacketRateLock             (per-IP packet rate)
  ↓
TotpStore._diskLock                 (innermost — short critical section, file I/O)
```

If you need locks in a different order, refactor — don't reorder the hierarchy.

---

## 9. Incident response one-pager

1. **Disconnect the game-server NIC.** Stop further damage; DB is what matters.
2. **Snapshot disk.** Image the volume before forensics.
3. **Verify the off-host audit logs.** `HashChainLog.Verify` on every day in the suspected window; any chain break is a confirmed compromise.
4. **Rotate every secret.** New pepper (via the §4 rotation procedure — kept old in `_P<N>` so users can still log in during transition). New `CRYSTAL_GM_PASSWORD`. New manifest signing key.
5. **Force password reset on every account.** Set `RequirePasswordChange = true` via the AccountInfo editor or a maintenance script.
6. **Audit GM logs** for the day before, day of, and day after, on the off-host copy.
7. **Re-deploy** server binary from a known-good build.

---

## 10. Quarterly drills

- Restore the DB from backup to a fresh VM. Confirm logins work.
- Rotate `CRYSTAL_GM_PASSWORD`. Verify new value works.
- **Test pepper rotation in staging using the §4 procedure** before doing it in prod. Confirm a peppered account that you log in BEFORE the rotation works AFTER the rotation (verify via kept `_P<N>`), and that a fresh login after rotation produces a `-p2$` hash.
- Read this document. Update anything that has drifted.
