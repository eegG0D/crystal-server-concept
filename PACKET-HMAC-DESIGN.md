# Packet HMAC — Deferred Implementation Plan

**Status:** Designed, not implemented. This is the breaking-protocol change you wanted to hold for a release window.

**Threat model:** A trojaned client or a MITM that has already breached the transport (e.g., compromised proxy host, downgraded TLS) injecting forged packets. With TLS terminated at the reverse proxy (`docs/TLS-PROXY.md`), MITM on the wire is already mitigated; packet HMAC's remaining value is **defense in depth at the packet-handler layer** and **forcing every packet through a single auth gate** so a future packet handler bug can't be reached by skipping the check.

---

## Wire format change

Current Crystal packet:

```
[uint16 size]   ── total bytes including this header
[uint16 packetId]
[bytes body]
```

New:

```
[uint16 size]
[uint16 packetId]
[bytes body]
[32 bytes HMAC-SHA256 over (packetId || body || counter)]
```

A monotonically-incrementing 8-byte counter is appended to the HMAC input (not the packet) and tracked on both ends. Replays of an earlier packet fail because the counter, and therefore the HMAC, differs. The counter does not need to be transmitted because the receiver maintains its own copy of the expected counter for that direction.

If a packet's HMAC fails: increment a per-connection bad-packet counter; on 3+ bad packets in 10s, disconnect with reason "packet auth fail".

## Key establishment

Two ways. Pick one:

### Option A — Simple: server-generated session key over TLS

If TLS termination is in place, the channel is already confidential. Server generates a 32-byte random key on `Connected`, stuffs it in the existing `S.Connected` packet (currently empty payload), client uses it. **Implementation cost: ~30 lines.**

### Option B — Better: ECDH handshake

Defends against the case where TLS is compromised (e.g., a malicious proxy with a stolen wildcard cert). Uses X25519:

1. Client and server each generate an ephemeral X25519 keypair on connect.
2. Each sends its public key in a new `KeyExchange` packet (no HMAC required for these — they're the bootstrap).
3. Both compute the shared secret with `ECDiffieHellman.DeriveKeyMaterial`.
4. HMAC key = `HKDF(shared_secret, salt=session_id, info="Crystal-HMAC-v1", length=32)`.

**Implementation cost: ~100 lines + dependency on `System.Security.Cryptography` (already in stdlib).**

Both should also include forward secrecy: regenerate the HMAC key every N packets or every M minutes via `HKDF(prev_key, "Crystal-HMAC-rotate")`. Optional, only needed if you fear long-term key compromise.

## Code touchpoints

### Server (`Server\MirNetwork\MirConnection.cs`)

```csharp
// New per-connection state
private byte[] _hmacKey;                // 32 bytes, set after handshake
private ulong _recvCounter, _sendCounter;
private int _badPacketCount;

// In ReceivePacket, after assembling the packet bytes:
if (_hmacKey != null)
{
    if (packetBytes.Length < 32) { /* fail */ }
    var (body, tag) = SplitTrailing(packetBytes, 32);
    var expected = ComputeHmac(_hmacKey, body, _recvCounter);
    if (!CryptographicOperations.FixedTimeEquals(expected, tag))
    {
        _badPacketCount++;
        if (_badPacketCount >= 3) { SendDisconnect(99); return; }
        return; // drop this packet
    }
    _recvCounter++;
}

// In SendPacket, before writing to the socket:
if (_hmacKey != null)
{
    var tag = ComputeHmac(_hmacKey, packetBytes, _sendCounter);
    socket.Write(packetBytes);
    socket.Write(tag);
    _sendCounter++;
}
```

`ComputeHmac` is a thin wrapper around `HMACSHA256` that mixes the counter into the input.

### Client (`Client\Network\Network.cs`)

Mirror image of the server side. Same key, opposite counters.

### Protocol version

Bump the client/server version constant in `Envir.cs` (`public const int Version = ...`) so older clients can't connect after the change rolls out.

## Backward compatibility

There isn't any. Once enabled, every client must be updated. Rollout plan:

1. Implement on a feature branch.
2. Push a beta client build to a separate patch channel.
3. Tell players to use the beta for a week, watch for issues.
4. Merge to main, bump `Version`, force a launcher update via the existing patcher.

## Why this is *deferred*, not done

- TLS termination at the reverse proxy already provides the strongest threat-model improvement (no plaintext on the wire, no MITM injection).
- A trojaned client running on a player's PC can comply with the HMAC perfectly — the player owns the key. So HMAC does not stop cheats originating from modified clients; it only stops *injection from outside* the legitimate client.
- The breaking-protocol cost is significant: every existing connected client breaks, every dev/test setup needs updating in lockstep.

If you later see attacks that specifically inject packets from outside the client (e.g., a proxy-based exploit kit), prioritize this. Until then, the value-to-cost ratio is lower than the other items already shipped.

## Estimated effort

- Option A (server-generated key over TLS): **~half a day** including testing.
- Option B (ECDH handshake): **~1 day** including testing.
- Force-update rollout: **1 release window** (whatever your normal cadence is).
