# Changelog

All notable changes to this project will be documented in this file.

## [v0.3.0] - 2026-03-05

### Added
- **Rendezvous ProviderSummary integration** — `Agent::advertise_identity()` publishes a signed `ProviderSummary` to the rendezvous shard topic enabling global agent findability across gossip overlay partitions
- **`Agent::find_agent_rendezvous()`** — stage-3 lookup that subscribes to the rendezvous shard topic and waits for a matching `ProviderSummary`; addresses decoded from the `extensions` field
- **3-stage `find_agent()`** — upgraded from 2-stage to: cache hit → identity shard subscription (5s) → rendezvous (5s)
- **`rendezvous_shard_topic_for_agent()`** — deterministic `"x0x.rendezvous.shard.<u16>"` topic function
- **`RENDEZVOUS_SHARD_TOPIC_PREFIX`** constant
- **x0xd rendezvous config** — `rendezvous_enabled` (default `true`) and `rendezvous_validity_ms` (default 3,600,000 ms) config fields; initial advertisement at startup + background re-advertisement every `validity_ms / 2`
- **Identity heartbeat** — `Agent::start_identity_heartbeat()` re-announces identity at configurable interval (default 300s) so late-joining peers can discover earlier nodes
- **TTL filtering** — `presence()` and `discovered_agents()` filter entries older than `identity_ttl_secs` (default 900s); `discovered_agents_unfiltered()` returns all cache entries
- **Shard-based identity routing** — `shard_topic_for_agent()` returns `"x0x.identity.shard.<u16>"` derived via BLAKE3; `announce_identity()` dual-publishes to shard + legacy topics; 65,536-shard space
- **Human identity HTTP API** — `GET /users/:user_id/agents`, `GET /agent/user-id`; `?wait=true` query parameter on `GET /agents/discovered/:id` triggers active shard+rendezvous lookup
- **`Agent::find_agents_by_user()`** — discovers all agents in cache claiming a given `UserId`
- **`Agent::local_addr()`** — returns the bound socket address of the network node
- **`Agent::build_announcement()`** — public wrapper for building a signed `IdentityAnnouncement`
- **`AgentBuilder::with_heartbeat_interval()` / `with_identity_ttl()`** — configurable heartbeat and TTL
- **x0xd heartbeat/TTL config** — `heartbeat_interval_secs` and `identity_ttl_secs` fields
- **SKILL.md Discovery & Identity section** — full curl examples, human consent invariant, trust model, `x0x://user/<hex>` URI scheme

### Changed
- `find_agent()` timeout split: 5s for identity shard subscription + 5s for rendezvous (was 10s shard-only)
- `join_network()` now calls `announce_identity()` and `start_identity_heartbeat()` automatically

### Infrastructure
- Updated saorsa-gossip-* crates from 0.5.1 → 0.5.2 (adds `ProviderSummary.extensions`, `sign_raw`/`verify_raw`)
- Removed CI symlink workaround for ant-quic and saorsa-gossip from all 4 workflows (ci.yml, release.yml, build.yml, build-bootstrap.yml) — all deps now resolve from crates.io

## [v0.2.0] - 2026-02-01

### Added
- Signed identity announcements with machine-key attestation
- Contact trust store with `Blocked` / `Unknown` / `Known` / `Trusted` levels
- Trust-filtered pub/sub (blocked senders are dropped)
- Dual-stack IPv6 on all 6 bootstrap nodes
- Axum route improvements
- Production gossip integration
- `x0xd` daemon with full REST API

## [v0.1.0] - 2026-01-01

### Added
- Initial release
- `Agent` with machine + agent + user identity (three-layer model)
- CRDT collaborative task lists (OR-Set checkboxes, LWW-Register metadata, RGA ordering)
- MLS group encryption (ChaCha20-Poly1305)
- Gossip pub/sub via saorsa-gossip epidemic broadcast
- Bootstrap connection to 6 global nodes (NYC, SFO, Helsinki, Nuremberg, Singapore, Tokyo)
- Node.js bindings (napi-rs v3) and Python bindings (PyO3/maturin)
- GPG-signed SKILL.md for agent self-distribution
