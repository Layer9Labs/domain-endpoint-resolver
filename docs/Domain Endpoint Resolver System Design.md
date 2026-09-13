# System Design: Domain Endpoint Resolver (DER)

Status: Draft — companion to the [PRD](Domain%20Endpoint%20Resolver%20Product%20Requirements%20Document%20(PRD).md), the [Technical Brief](context/Technical%20Brief%20for%20Event%20Network%20Architecture.md), and [ADR-0001](adrs/ADR-0001-gossip-mesh-architecture.md) (which this document assumes and builds on rather than re-litigates).

## 1. Requirements

### 1.1 Functional

- `POST /register` — an event sink/router registers one or more `{domain, healthcheck, listeners[]}` entries with whichever DER node it talks to.
- `GET /lookup?domain=...` — an event source/router resolves a domain to its current listener endpoints, from *any* DER node, not just the one the domain was registered with.
- `DELETE /remove?domain=...` — explicit removal of a domain's entry, called by a sink during orderly shutdown.
- Periodic health-check — the mesh itself assigns responsibility for polling each domain's health-check URI (see §3.4), based purely on the information gossiped at registration time, independent of which node originally accepted the `/register` call; a `503`/non-`"healthy"` response evicts that domain's endpoints, locally and mesh-wide.
- Mesh propagation — every register/remove mutation (explicit or health-check-triggered) must reach every other DER node so that a lookup against *any* node reflects it, eventually.
- A management/introspection surface for the local routing table (Technical Brief §"Dynamic Discovery" calls this out explicitly, even though the PRD doesn't spec its shape — see open question in §6).

### 1.2 Non-functional

- No persistent database (PRD §4) — all routing state is in-memory, reconstructed via gossip and re-registration.
- Fault tolerant / no single point of failure — any node answers any lookup; HyParView tolerates high churn (brief cites up to 90% node failure) without partitioning the mesh.
- Horizontally scalable — adding DER nodes should not require reconfiguring existing ones or a coordinator.
- Encrypted transport (PRD Req 4) via HTTP/3 (QUIC/TLS 1.3), HTTP/2 fallback where QUIC/UDP is unavailable.
- Low latency on the lookup path — lookups are synchronous, in the hot path of every event send from an event source.
- Rust, idiomatic error handling, `clap`-driven CLI configuration (cert path, health-check interval).

### 1.3 Constraints

- Fixed dependency set from the PRD: `saorsa-gossip` (HyParView + PlumbTree), `quiche` + `tokio-quiche` (HTTP/3), `rmp` (MessagePack), `clap`.
- Out of scope: implementing Event Router / Event Sink behavior; persistent storage of any kind.
- Current repo state: no `Cargo.toml` yet, `src/main.rs` is still the `cargo new` stub — this document describes the target architecture, not what's implemented.

## 2. High-Level Design

### 2.1 Component diagram (single DER node)

```
                    +------------------------------------------------------+
                    |                    DER Mesh Node                     |
                    |                                                      |
 HTTP/3 (QUIC)      |   +-----------+        +----------------------+      |
 fallback HTTP/2 -->|   |API Layer  |------->| Domain Routing Table |      |
 TLS 1.3, MsgPack   |   |/register  |<-------|  in-memory, keyed by |      |
 payloads           |   |/lookup    |        |  domain string       |      |
                    |   |/remove    |        +----------+-----------+      |
                    |   +-----+-----+                   |                  |
                    |         |                         v                  |
                    |         |                 +-----------------+        |
                    |         |                 |  Gossip Engine  |<------+---> peer DER nodes
                    |         |                 | HyParView +     |        |   (gossip protocol,
                    |         |                 | PlumbTree       |        |    MessagePack, via
                    |         |                 | (saorsa-gossip) |        |    saorsa-gossip's own
                    |         |                 +--------+--------+        |    transport)
                    |         |                          |                 |
                    |   +-----v----------+               |                 |
                    |   | Health-Check   |<---------------+                |
                    |   | Scheduler      |----> sinks the mesh's assignment|
                    |   | (ticker, only  |      function currently maps to |
                    |   |  mesh-assigned |      this node (GET healthcheck |
                    |   |  domains)      |      URI)                       |
                    |   +--------+-------+                                 |
                    |            |                                         |
                    |   +--------v--------+                                |
                    |   |  CLI Config     |  (clap: TLS cert path,         |
                    |   |  (clap)         |   health-check interval)       |
                    |   +-----------------+                                |
                    +------------------------------------------------------+
```

Five components per node:

1. **API layer** — terminates HTTP/3 (fallback HTTP/2), decodes/encodes MessagePack, exposes the three PRD routes.
2. **Domain routing table** — the in-memory distributed cache: `domain -> {healthcheck, listeners[], version}`.
3. **Gossip engine** — wraps `saorsa-gossip`: HyParView maintains this node's active/passive peer views, PlumbTree disseminates mutations over the tree built on top of that view.
4. **Health-check scheduler** — a ticker (interval from CLI) that walks whichever domains the mesh's assignment function currently maps to this node (§3.4) and calls their health-check URI. This is *not* tied to which node originally accepted the `/register` call.
5. **CLI config** — `clap`-parsed: TLS cert path, health-check interval, bind address, seed peers for mesh join.

### 2.2 Data flow

**Register**

```
sink --POST /register (MsgPack)--> Node A: API layer
                                         | validate + URL-decode domain
                                         v
                                   Node A: routing table (upsert)
                                         |
                                         v
                                   Node A: gossip engine -- PlumbTree broadcast --> Node B, C, D...
                                                                                     | apply mutation
                                                                                     | (mesh recomputes health-check
                                                                                     |  assignment for this domain, §3.4)
sink <--200 OK (MsgPack ack)-- Node A
```

**Lookup** (read-only, no gossip involved)

```
source --GET /lookup?domain=...--> any Node N: API layer
                                         | URL-decode domain, read routing table
                                         v
source <--{domain, endpoints[]} (MsgPack), or 404--
```

**Remove** (explicit, or health-check-triggered)

```
[sink shutdown]  DELETE /remove?domain=...       -+
                                                   +-> Node A: routing table (delete)
[health-check fails on the node the mesh         -+        |
 currently assigns to check this domain]                   v
                                              Node A: gossip engine -- PlumbTree broadcast --> rest of mesh
```

### 2.3 API contracts

Routes and payload shapes are fixed by the PRD; restated here for the design's internal reference:

| Route | Method | Request | Response |
|---|---|---|---|
| `/register` | POST | MessagePack list of `{domain, healthcheck, listeners: [url]}` | `200` ack, or `4xx` with a validation error, MessagePack-encoded |
| `/lookup` | GET | query param `domain` (URL-encoded) | MessagePack `{domain, endpoints: [url]}`, or `404` if unknown |
| `/remove` | DELETE | query param `domain` (URL-encoded) | `200`/`204` ack, or `404` if unknown |

`domain` values contain `@` and `.` (e.g. `starfoods.quality@v1`) — both routes must percent-encode/decode this correctly (PRD calls this out explicitly for `/lookup` and `/remove`).

### 2.4 Storage

No database, by constraint. The routing table is an in-memory concurrent map (e.g. `DashMap<String, DomainEntry>` or an `RwLock<HashMap<...>>` behind the gossip engine's apply path) — it *is* the distributed cache; durability comes from re-registration after a restart, not from disk.

## 3. Deep Dive

### 3.1 Data model

```
struct DomainEntry {
    healthcheck: String,       // e.g. "127.0.0.1:5076"
    listeners: Vec<String>,    // e.g. ["https://domain1:5050", ...]
    version: (u64, NodeId),    // logical clock + node that produced this version,
                                // used only for LWW tie-breaking (§3.2) — not related
                                // to health-check responsibility (§3.4)
}
```

Keyed by the full domain string (`"starfoods.quality@v1"`) — no normalization beyond what URL-decoding already does, since the domain string is opaque to the DER (schema/attribute semantics belong to the Event Router, not the resolver).

Note there is no `owner` field: which node originally accepted a domain's `/register` call has no bearing on anything past that request completing — not conflict resolution's node id (which just identifies whoever last wrote a version, for tie-breaking) and not health-check responsibility (§3.4), which the mesh assigns independently.

### 3.2 Conflict resolution (gap the PRD leaves open)

The PRD doesn't specify what happens if the same domain is registered at two different nodes concurrently, or if a register and a remove for the same domain race across the mesh. Recommended: **last-writer-wins per domain**, using a `(logical_clock, node_id)` pair as the version — monotonic per writing node, compared lexicographically on conflict. This is simple, requires no cross-node coordination, and matches the PRD's implicit model of "one sink owns one domain's registration at a time." Document this choice as a follow-up ADR once implementation starts (flagged as a decision this design *assumes*, not one the PRD makes).

### 3.3 Gossip message types

Two mutation types flow over PlumbTree, both MessagePack-encoded:

- `DomainUpserted { domain, healthcheck, listeners, version }`
- `DomainRemoved { domain, version }`

Both carry the version from §3.2 so receiving nodes can discard stale/duplicate mutations (PlumbTree's eager/lazy push can redeliver). A joining node performs anti-entropy against its initial HyParView peers to bulk-sync the current table rather than waiting to receive every historical mutation.

### 3.4 Health-check assignment

There is no "associated event sink" concept tied to a particular DER node. The node that happened to accept a domain's `/register` call has no ongoing role once that mutation has been gossiped — the health-check URI is just another field on the entry, replicated to the whole mesh like everything else.

Instead, **the mesh itself decides who checks what**, purely from the information carried in the entry: every node runs the same deterministic assignment function (e.g., rendezvous/consistent hashing of the domain string over the current set of live mesh members, as seen in that node's gossip membership view) to work out which node(s) currently own the duty of polling a given domain's health-check URI. Each node's health-check scheduler (§2.1) simply walks the domains the assignment function currently maps to *it* and polls them on the configured interval; a `503`/non-`"healthy"` response (or timeout) evicts the domain locally and gossips the removal, exactly as an explicit `/remove` does.

Because assignment is recomputed from membership rather than fixed at registration time, it self-heals: when a node fails, every remaining node's assignment function naturally remaps that node's domains onto surviving members on the next membership-view update — there is no dependency on the original registering node staying alive, and no permanently-stale entry to work around.

**Consequence to flag**: during a membership-view change, different nodes may briefly disagree on who owns a given domain's check (a short window where a domain is checked twice, or not at all, until views converge). This is a transient duplication/gap, not a correctness problem — the idempotent apply in §3.5 absorbs duplicate evictions, and the next tick closes any gap. Redundancy (assigning more than one node per domain) can trade a bit of duplicate-check overhead for a tighter gap window; start with a single assignee and revisit if the gap window proves too wide in practice (§5, §6).

### 3.5 Error handling & retries

- **Malformed `/register` payload** (missing field, bad URL) → `4xx`, no mutation applied, no gossip emitted.
- **Health-check timeout or connection refused** → treated identically to an explicit `503`/`"unhealthy"` response: evict.
- **Gossip message loss** → tolerated by design: PlumbTree's lazy-push graft/prune recovers missing branches, and anti-entropy on join/rejoin catches anything missed. No node-to-node retry logic needed above what `saorsa-gossip` already provides.
- **Duplicate mutations** (same version arriving twice) → idempotent apply, keyed by `(domain, version)`.

### 3.6 Security

- HTTP/3 (QUIC) gives TLS 1.3 by construction; the CLI-supplied certificate (PRD Req 4) configures this. HTTP/2 fallback must also be terminated over TLS — don't silently drop to plaintext HTTP/2 when QUIC is unavailable.
- The Technical Brief's access-control-list concept ("an event sink will only accept a message from a known event source") is an Event Router/Sink concern, out of scope for the DER itself (PRD §4).
- **Open question**: the PRD doesn't specify whether inter-node gossip traffic must also be encrypted. Given the DER mesh carries the same sensitive routing data the public API protects, recommend terminating gossip transport over TLS too, consistent with the "encrypted payload" requirement's spirit — flag this for an ADR before implementation rather than assuming either way.

## 4. Scale & Reliability

### 4.1 Load assumptions (explicit, since the PRD gives none)

Stated as assumptions, not requirements — revisit once real usage is known:

- O(10²–10³) DER nodes in a mesh, O(10⁴) registered domains.
- Lookups are the dominant traffic (every event source resolves before every send, or caches per Technical Brief's "Dynamic Discovery"); registers/removes are comparatively rare (sink lifecycle events).
- Health-check fan-out per node is bounded by however many domains the mesh's assignment function (§3.4) currently maps to it — on average roughly total-domains / mesh-size, not tied to how many were registered through that particular node.

### 4.2 Horizontal scaling

HyParView bounds each node's active view (small, fixed fan-out) regardless of mesh size, so per-node connection/memory cost doesn't grow linearly with total node count. Gossip convergence latency grows with the diameter of the overlay (~O(log N) hops for typical HyParView/PlumbTree configurations), not with N directly — new nodes join by contacting seed peers (CLI-configured) and populate their view/table via anti-entropy. Joining also causes health-check assignment (§3.4) to rebalance some domains onto the new node.

### 4.3 Failover & redundancy

- **DER node failure**: HyParView's partial-view membership is explicitly designed to tolerate high churn; surviving nodes repair their active views from the passive view without manual intervention. Lookups against surviving nodes are unaffected; health-check responsibility for domains the assignment function had mapped to the failed node is picked up by another node automatically on the next membership-view update (§3.4), with no stale-forever entries to manage.
- **Sink failure**: handled by the health-check → evict → gossip-remove path, independent of DER node failure.
- **Network partition**: gossip continues within each partition; on heal, anti-entropy reconciles using the version in §3.2. A registration made only inside a minority partition is visible there until the partition heals.

### 4.4 Monitoring

The Technical Brief explicitly calls for a management interface to introspect the routing table ("real-time rendering of the event network topology"). Suggested metrics to expose there, since the PRD doesn't specify observability:

- Local routing table size
- Count of domains currently assigned to this node for health-checking (§3.4), vs. total table size
- Health-check failure rate and eviction count, scoped to domains this node is currently assigned to check
- Gossip view size (active/passive), message send/receive rates, anti-entropy sync duration on join
- HTTP/3-vs-HTTP/2 request ratio, to catch unexpected QUIC-blocked environments early

## 5. Trade-off Analysis

Builds on [ADR-0001](adrs/ADR-0001-gossip-mesh-architecture.md); additional trade-offs specific to this design:

| Decision | Trade-off |
|---|---|
| Last-writer-wins version (§3.2) vs. vector clocks | LWW is simple and sufficient for "one sink owns its own domain," but would under-merge if the same domain were legitimately co-owned by multiple sinks concurrently — not a case the PRD describes, so accepted for now |
| Mesh-assigned health-checking via membership hashing (§3.4) vs. registration-owner health-checking | Assignment-based checking removes the dependency on the original registering node staying alive — a failed assignee's domains are picked up by another node on the next view change, with no permanent staleness. The cost is a brief duplicate-or-gap window during membership churn (§3.4), and the assignment function itself is a new piece of logic every node must agree on |
| Mesh-assigned (single-assignee) health-checking vs. full N-way redundant checking | A single assignee per domain keeps polling load roughly even and bounded (§4.1) instead of O(N) redundant checks per domain; the trade-off is that a domain briefly goes unchecked only during the assignee's own failure-to-reassignment window, versus always-redundant coverage. Revisit toward small-redundancy-factor assignment if that window proves too wide in practice |
| In-memory-only state (mandated) vs. any durability | Rules out crash recovery of the table without re-registration; this is an explicit PRD constraint, not a choice, but worth naming as the reason full-mesh-restart is a real risk (see ADR-0001's negative consequences) |
| Single flat domain namespace vs. sharding across sub-meshes | Flat namespace matches the PRD's API surface exactly (`/lookup?domain=`) and is simplest to reason about; sharding would only be worth revisiting if a single mesh's gossip fan-out become a bottleneck at real scale |

## 6. Open Questions / Revisit as the System Grows

- Should inter-node gossip traffic be encrypted, and if so, with what cert (same one from `/register`'s TLS config, or a separate mesh-internal cert)? (§3.6)
- What hashing/assignment scheme should determine which node(s) health-check a given domain (rendezvous hashing, consistent hashing, something else), and should it assign more than one node per domain for redundancy during membership churn? (§3.4)
- What does the "management interface" for routing-table introspection look like as an API — is it in scope for this crate at all, or a separate tool that reads exposed metrics? (§4.4)
- Concurrent registration semantics (§3.2) should get its own ADR once real implementation surfaces a concrete need, rather than being assumed here.
