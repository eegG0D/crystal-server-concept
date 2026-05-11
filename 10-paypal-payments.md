# PayPal Payments

Real-money credit purchases. Players click a button in a web wrapper, complete checkout on paypal.com, the server captures the order and grants in-game credits.

## What it does

This is the live monetization path. The server runs a small embedded HTTP listener that exposes `/create-order` and `/capture-order` endpoints. The launcher (or any embedded WebView) opens the PayPal sandbox / live URL with the returned `orderId`; PayPal redirects back; the server captures and credits the player's account.

A transaction log (`PayPalTransactionLog.cs`) prevents double-crediting if PayPal retries the webhook.

## Where to find it

- **Server-side classes:** `Server\PayPal\PayPalService.cs`, `Server\PayPal\PayPalHttpServer.cs`, `Server\PayPal\CreditPackage.cs`, `Server\PayPal\PayPalTransactionLog.cs`.
- **Server config form:** **Server.exe → Config → PayPal**  
  (defines packages, prices, granted credit amounts).
- **`Setup.ini` section:** `[PayPal]` (`ClientId`, `ClientSecret`, sandbox/live mode toggle).

## How to configure

| Field | What it does |
|---|---|
| **ClientId / ClientSecret** | From the PayPal developer dashboard. Live and sandbox credentials are different — never mix them. |
| **Sandbox mode** | If true, hits `api-m.sandbox.paypal.com`. Use this for testing. |
| **Packages** | Each row: `Name`, `PriceUSD`, `Credits`. Sent to the client menu. |
| **Webhook URL** | Public-facing URL that PayPal hits on PAYMENT.CAPTURE.COMPLETED. Must be reachable from PayPal's servers (behind your reverse proxy / firewall — see `docs/TLS-PROXY.md`). |

## How players use it

1. Click "Buy Credits" in the in-game shop UI.
2. The launcher opens an embedded browser to `paypal.com/checkoutnow?token=<orderId>`.
3. Player completes checkout.
4. PayPal redirects back to your `/capture-order` URL with the orderId.
5. Server captures, looks up the package, grants `Account.Credit += package.Credits`, writes the transaction log entry.
6. Client shows "Credits added: 500" (or whatever).

## How it works under the hood

- `PayPalService.cs` wraps the REST API: `CreateOrder(packageId) → orderId`, `CaptureOrder(orderId) → captureId + amount`.
- `PayPalHttpServer.cs` is a thin `HttpListener` exposing the two endpoints. Runs on a dedicated thread.
- `CreditPackage.cs` is the package registry (id, name, USD price, credits).
- `PayPalTransactionLog.cs` is the dedupe ledger: every captureId gets appended to disk before credits are granted. On restart the log is replayed into an in-memory hashset.

## Common pitfalls

- **`/capture-order` MUST be idempotent.** PayPal retries on transient errors. If you grant credits before writing the log entry, a retry double-credits. The shipped code writes the log entry first and bails on dup.
- **ClientSecret in plaintext in `Setup.ini`** is the default. Move it to an env var (`CRYSTAL_PAYPAL_SECRET`) and override on load — same pattern as the password pepper. See `docs/SECURITY-OPS.md` §4.
- **Webhook endpoint must be HTTPS in production.** PayPal won't deliver webhooks to plain HTTP. Terminate TLS via nginx (`docs/TLS-PROXY.md`).
- **Sandbox vs live mix-up.** Pushing sandbox credentials to live = silent failure (no orders captured). Always verify the URL the launcher hits matches the credential's environment.
- **Amount tampering.** Validate that PayPal's reported amount matches the package's expected price BEFORE granting credits. Never trust the client to tell you which package they bought — the orderId is the only source of truth.
- **Currency.** All package prices default to USD. Multi-currency support is not built in; you'd need to extend `CreditPackage` and PayPal's order-create call.

## Quick smoke test (sandbox)

1. Set sandbox credentials.
2. Create a `$1 = 100 credits` test package.
3. Click "Buy" in-game.
4. Log into the PayPal sandbox buyer account → approve.
5. Check the log line `[PayPal] Captured ORD-XXX → granted 100 credits to Acct-Y`.
6. Verify `Account.Credit` increased.
7. Manually replay the webhook → should NOT double-credit.
