# Crystal Server — TLS via Reverse Proxy

The Crystal wire protocol is plain TCP. Wrapping it in TLS at the protocol level would mean a synchronized client+server change and a protocol version bump. The pragmatic path is to **terminate TLS in front of the server** with a reverse proxy:

```
Player ─── TLS (443) ──▶  nginx (host:443)  ─── plain TCP (loopback) ──▶  Server.exe (127.0.0.1:7000)
```

- The plaintext login/game socket only exists on the loopback interface. Outside the host, everything is TLS.
- Game packets ride the same encrypted tunnel — there is no longer a "login channel" vs "game channel" distinction from the network's POV.
- No source change to Crystal is needed.

## Step 1 — Make the Server bind to localhost AND turn on PROXY protocol

Edit `Build\Server\Debug\Configs\Setup.ini`:

```ini
[Network]
IPAddress            = 127.0.0.1
Port                 = 7000
ProxyProtocolEnabled = True
```

**Both keys are mandatory together.** Without `ProxyProtocolEnabled=True`:

- The `RemoteEndPoint` the server sees is always `127.0.0.1` (the proxy).
- The per-IP cap (`Settings.MaxIP=5`) becomes a global cap on simultaneous connections.
- The audit log records `127.0.0.1` for every event, making forensics worthless.
- The login rate limiter blocks the proxy, not the attacker, so one bad IP can lock out everyone.

With it on, the server reads a one-line PROXY-v1 header from nginx before any game packet, parses out the real client IP, and uses that everywhere.

Conversely, **do not enable `ProxyProtocolEnabled` while the port is exposed directly to the internet.** A naked TCP client doesn't send a PROXY header, so every connection will be rejected as malformed.

Restart the server. Confirm with `netstat`:

```powershell
netstat -ano | findstr :7000
# You should see ONLY 127.0.0.1:7000, not 0.0.0.0:7000.
```

## Step 2 — Install nginx with the `stream` module

Windows builds of nginx ship with stream support by default. Download from <https://nginx.org/en/download.html>, extract to `C:\nginx\`.

## Step 3 — Get a TLS certificate

For a public-facing server, use Let's Encrypt via `win-acme` (<https://www.win-acme.com/>). It will produce two files:

```
C:\nginx\ssl\fullchain.pem
C:\nginx\ssl\privkey.pem
```

Renewals happen automatically (90-day rotation). For internal/dev, generate a self-signed cert and pin it in the client (see "Client pinning" below).

## Step 4 — nginx config

Replace `C:\nginx\conf\nginx.conf` with:

```nginx
worker_processes auto;
events { worker_connections 4096; }

# Game protocol — raw TCP wrapped in TLS.
stream {
    upstream crystal_game {
        server 127.0.0.1:7000;
    }

    server {
        listen 443 ssl;
        proxy_pass crystal_game;
        # MUST match ProxyProtocolEnabled=True in Setup.ini, otherwise the
        # server sees every connection as 127.0.0.1 and the per-IP cap
        # collapses to a global cap.
        proxy_protocol on;

        ssl_certificate           C:/nginx/ssl/fullchain.pem;
        ssl_certificate_key       C:/nginx/ssl/privkey.pem;

        # Modern TLS only. No SSLv3/TLS1.0/1.1 — those leak credentials.
        ssl_protocols             TLSv1.2 TLSv1.3;
        ssl_ciphers               TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;
        ssl_session_cache         shared:CRYSTAL:10m;
        ssl_session_timeout       1h;

        # Slow-loris defense at the proxy edge. Belt-and-braces with the
        # server-side ReceiveTimeout/SendTimeout we already set in
        # MirConnection.cs.
        proxy_connect_timeout     5s;
        proxy_timeout             30s;
    }
}

# Patch host — HTTPS for downloads + manifest.
http {
    server {
        listen 80;
        server_name patch.example.com;
        # Force everything to HTTPS.
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name patch.example.com;

        ssl_certificate           C:/nginx/ssl/fullchain.pem;
        ssl_certificate_key       C:/nginx/ssl/privkey.pem;
        ssl_protocols             TLSv1.2 TLSv1.3;

        root C:/crystal-patch-files;
        autoindex off;

        # Patch manifest is checked for SHA-256 + RSA signature client-side.
        # HTTPS here is defense in depth against MITM injection.
        location / {
            try_files $uri =404;
            add_header Cache-Control "public, max-age=300";
        }
    }
}
```

Start it:

```powershell
C:\nginx\nginx.exe -t       # config syntax check
Start-Service nginx         # or: C:\nginx\nginx.exe
```

## Step 5 — Point the client at the new endpoint

In `Mir2Test.ini` (or whatever the production client config is):

```ini
[Server]
IPAddress = your.public.host.name
Port      = 443

[Launcher]
Host      = https://patch.example.com/
```

The client's TCP code is protocol-agnostic — wrapping it in TLS at the proxy is transparent.

⚠️ The current Crystal client uses raw `TcpClient`, not `SslStream`. So pointing the client at TLS:443 will fail unless the client is also updated to use `SslStream` over the socket. **The reverse-proxy approach therefore requires a one-time client update**. Alternatives:

- Tunnel option: have the launcher establish a TLS connection itself (one `SslStream` wrap in `Network.cs`).
- Or: terminate TLS only on the **patch download** (`P_Host`) for now — that's purely HTTPS and works today without any client code change. The TCP game channel stays raw until you ship a client update.

## Client pinning (when you do update the client)

If the cert is from a public CA, the OS trust store is enough. If self-signed, pin the cert thumbprint:

```csharp
// in Client/Network/Network.cs where the socket is created:
using var stream = new SslStream(client.GetStream(), false,
    (sender, cert, chain, errors) =>
    {
        var pinned = "SHA256:AB:CD:..."; // your cert's SHA-256 fingerprint
        return cert.GetCertHashString(HashAlgorithmName.SHA256) ==
               pinned.Replace("SHA256:", "").Replace(":", "");
    });
stream.AuthenticateAsClient("your.public.host.name");
// then read/write through `stream` instead of `client.GetStream()`.
```

Pinning means a stolen wildcard cert from your CA does not let an attacker MITM your client. Recommended for production.

## Quick test

From a different machine, after standing this up:

```powershell
# Confirm the game port is NOT reachable directly.
Test-NetConnection your.public.host.name -Port 7000   # should fail (listener bound to 127.0.0.1)

# Confirm 443 accepts TLS.
Test-NetConnection your.public.host.name -Port 443    # succeeds
openssl s_client -connect your.public.host.name:443 -servername your.public.host.name | head
# you should see your cert chain back
```

## What this still doesn't protect against

- A **trojaned client** (a player running a modified binary) — TLS only protects the wire, not the endpoints. Mitigation: code-sign the binary (`signtool sign /f mycert.pfx Mir2.exe`), embed integrity checks (the asset SHA-256 + manifest signature we already added), and consider per-packet HMAC (deferred — see `#4 Packet HMAC` in the original threat list).
- A **compromised proxy host** — anyone with admin on the nginx box can MITM the loopback connection. Mitigation: same-host hardening from `SECURITY-OPS.md`.
- **CDN / patch-host compromise** — even with HTTPS, if your patch CDN is breached, signed manifests are your last line. Don't skip key generation.
