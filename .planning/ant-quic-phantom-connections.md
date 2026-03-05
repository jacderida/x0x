# ant-quic: Phantom One-Sided Connections (Asymmetric Connection State)

## Summary

When multiple bootstrap nodes connect to each other simultaneously on startup, some QUIC connections become "phantom" — one side (the initiator) believes the connection is established while the other side has no record of it. These phantom connections persist indefinitely and cannot be recovered by reconnection attempts.

## Reproduction

**Environment:** 6 VPS nodes across 4 providers (DigitalOcean NYC/SFO, Hetzner Helsinki/Nuremberg, Vultr Singapore/Tokyo), all binding on `0.0.0.0:12000` (UDP/QUIC), all using ML-DSA-65 raw public keys (development mode).

**Steps:**
1. Start all 6 nodes simultaneously (within ~2s of each other)
2. Each node calls `connect_addr()` to all 5 other nodes in parallel
3. Wait for all handshakes to complete

**Expected:** Each node reports 5 connected peers. Full mesh = 30/30 connections.

**Actual:** Consistently 29/30. One specific pair (Singapore→NYC) creates a phantom connection where:
- NYC (`142.93.199.50`) reports **5 peers** (including Singapore)
- Singapore (`149.28.156.231`) reports **4 peers** (NYC missing)

The phantom connection on NYC's side shows `observed_address=None` in the `add_connection` log, which appears to be the telltale sign:

```
# NYC logs - phantom connection from Singapore (note: observed_address=None on later attempts)
{"message":"add_connection: peer PeerId([...]) at 149.28.156.231:12000 observed_address=None"}

# Singapore logs - NYC never appears in add_connection at all
# (grep for 142.93.199.50 in add_connection logs returns nothing)
```

## Key Observations

### 1. Phantom connections have `observed_address=None`

When a connection is healthy, it logs:
```
add_connection: peer PeerId([...]) at X.X.X.X:12000 observed_address=Some(Y.Y.Y.Y:12000)
```

Phantom connections log:
```
add_connection: peer PeerId([...]) at X.X.X.X:12000 observed_address=None
```

### 2. The initiator's `connect_addr()` returns `Ok` but the acceptor never registers it

Singapore's reconnect loop calls `connect_addr(142.93.199.50:12000)` — the call starts (`"Connecting directly to 142.93.199.50:12000"`) but **never completes**. No success message, no error message. It hangs for ~76 seconds (much longer than the 30s connection timeout configured in the endpoint) and then silently falls through.

### 3. Simultaneous bidirectional connection is the trigger

When Node A calls `connect_addr(B)` at the same time Node B calls `connect_addr(A)`, both using the same `addr:port` pair (`*:12000`), a race condition occurs. The QUIC handshake from one side "wins" and the other side's ClientHello either:
- Gets demuxed to the existing connection instead of creating a new one
- Completes the handshake on one side but the accept loop never sees it on the other

### 4. Reconnection cannot recover the state

Once a phantom connection exists, subsequent `connect_addr()` calls from Singapore→NYC hang indefinitely. This suggests the endpoint's connection table has a stale entry that prevents a new connection to the same address, but the entry doesn't represent a real working connection.

### 5. Connection timeout is not respected

The configured `connection_timeout: 30s` is ignored. The `connect_addr()` call hangs for ~76s before the sequential reconnect moves on. This suggests the timeout is not propagated to the underlying QUIC `connect()` call, or there's an internal retry in `p2p_endpoint` that multiplies the effective timeout.

## Suggested Investigation Areas

### A. Simultaneous open / connection deduplication

When both sides initiate a QUIC connection to each other simultaneously on the same `addr:port` pair:
- How does the endpoint demux incoming packets that match an outgoing connection?
- Is there a tiebreaker (e.g. higher CID wins) when both ClientHellos arrive?
- Does `connect_addr()` check if an existing connection to that address already exists?

### B. `observed_address=None` path

What causes `observed_address` to be `None` in `add_connection`? This seems to correlate with phantom connections. Possible causes:
- Connection was registered via the accept path but the handshake didn't complete address discovery
- Connection was registered optimistically before the handshake finished

### C. Connection timeout enforcement

`connect_addr()` should respect the configured timeout. If `p2p_endpoint` has an internal retry, it should be bounded by the overall timeout, not multiplied.

### D. Stale connection cleanup

If `connect_addr(X)` is called and there's already a dead/phantom connection to address X, the old connection should be detected and cleaned up before attempting a new one. Currently it appears to silently block.

## Proposed Fix Directions

1. **Add simultaneous-open tiebreaker**: When both sides detect a simultaneous connection, use a deterministic tiebreaker (e.g. compare peer IDs) to decide which side's connection wins. The other side should drop its attempt and accept the incoming one.

2. **Connection liveness check before dedup**: Before deduplicating a `connect_addr()` against an existing connection, verify the existing connection is actually alive (e.g. send a ping, check for recent frames).

3. **Enforce timeout on `connect_addr()`**: Wrap the internal connect with `tokio::time::timeout(configured_timeout, ...)` at the `p2p_endpoint` level.

4. **Expose `observed_address` in connection status**: Allow callers to check if a connection has a valid observed address, which could be used as a health signal.

## Log Evidence

Full log sequence from Singapore during a reconnect cycle:

```
17:27:51 [INFO] Mesh incomplete: 4/5 peers connected, reconnecting...
17:27:51 [INFO] Connecting directly to 142.93.199.50:12000     ← NYC attempt starts
                                                                ← 76 seconds of silence
17:29:07 [INFO] Connecting directly to 147.182.234.192:12000   ← moves to SFO (NYC timed out)
17:29:07 [INFO] add_connection: peer ... at 147.182.234.192:12000 observed_address=None
17:29:07 [INFO] add_connection: now have 5 connections          ← temporarily 5
17:29:07 [INFO] Reconnected to bootstrap peer: 147.182.234.192:12000
17:29:07 [INFO] Connecting directly to 65.21.157.229:12000     ← Helsinki
17:29:07 [INFO] add_connection: peer ... at 65.21.157.229:12000 observed_address=None
17:29:07 [INFO] add_connection: now have 5 connections          ← still 5 (SFO phantom dropped?)
17:29:07 [INFO] Connecting directly to 116.203.101.172:12000   ← Nuremberg
                                                                ← 30s timeout
17:29:37 [INFO] Connecting directly to 45.77.176.184:12000     ← Tokyo
17:29:37 [INFO] add_connection: peer ... at 45.77.176.184:12000 observed_address=None
17:29:37 [INFO] add_connection: now have 5 connections
17:29:37 [INFO] Reconnect cycle complete: 4/5 peers connected  ← back to 4 (phantoms died)
17:30:37 [INFO] Mesh incomplete: 4/5 peers connected, reconnecting...
17:30:37 [INFO] Connecting directly to 142.93.199.50:12000     ← NYC again, will fail again
```

Note: Reconnect briefly reaches 5 connections during the cycle (from phantom connections to already-connected peers), but settles back to 4 because the phantoms die and NYC never connects.
