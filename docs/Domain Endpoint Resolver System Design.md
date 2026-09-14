# System Design: Domain Endpoint Resolver (DER)

Status: Draft — companion to the [PRD](Domain%20Endpoint%20Resolver%20Product%20Requirements%20Document%20(PRD).md), the [Technical Brief](context/Technical%20Brief%20for%20Event%20Network%20Architecture.md), and [ADR-0001](adrs/ADR-0001-gossip-mesh-architecture.md) (which this document assumes and builds on rather than re-litigates).

> Revision note (first pass): reassessed against the PRD/ADR-0001 revision that added multi-sink merge semantics (Req 1), multiple health-check URIs per domain (Req 5), mandatory gossip-transport encryption (Req 4), structured logging (Req 6), and narrowed scope (management interface and bulk domain retrieval now explicitly out of scope, PRD §4). §3.1–§3.4 changed the most: the conflict-resolution model moves from last-writer-wins to an add-wins merged set, per ADR-0001.
>
> Revision note (second pass — **reverses part of the first pass**): the `/register` payload itself changed shape, from two independent parallel arrays (`listeners[]`, `healthchecks[]`) to a single `domain-uris` array of explicit `{listener, healthcheck}` pairs, with an explicit 1:1 relationship between the two. This resolves what the first pass had flagged as an open question (§3.4/§6, then §5's "whole-domain vs. per-sink-scoped" row): a failing healthcheck now evicts its **paired listener too**, not just the healthcheck URI. The data model (§3.1) moves from two independent OR-Sets to one OR-Set of pairs — see ADR-0001's third revision for the source of this change.

## 1. Requirements

### 1.1 Functional

- `POST /register` — an event sink/router registers a domain as a `domain-uris` array of one or more explicit `{listener, healthcheck}` pairs with whichever DER node it talks to. There is a 1:1 relationship between a listener and its healthcheck (PRD Req 1) — the payload is a single array of pairs, not two independent `listeners[]`/`healthchecks[]` arrays. If a domain is already registered (by this or another sink), the mesh **merges** the new pairs into the existing set — union, de-duplicated — rather than overwriting it.
- `GET /lookup?domain=...` — an event source/router resolves a domain to its current (merged) listener endpoints, from *any* DER node, not just the one the domain was registered with.
- `DELETE /remove?domain=...` — explicit removal of a domain's entire entry (every pair), called by a sink during orderly shutdown. Unlike health-check eviction, this is domain-wide, not scoped to one sink's contribution (the PRD's `/remove` request carries only a domain, no per-sink identity).
- Periodic health-check — the mesh itself assigns responsibility for polling each domain's health-check URIs (see §3.4), based purely on the information gossiped at registration time, independent of which node originally accepted the `/register` call. Every health-check URI in a domain's merged set must be called (PRD Req 5, since a domain can now carry more than one pair, contributed by different sinks); a `503`/non-`"healthy"` response (or timeout) evicts **the whole pair** — the failing healthcheck *and* its 1:1-paired listener together (PRD Req 1, §3.4).
- Mesh propagation — every register/remove/eviction mutation must reach every other DER node so that a lookup against *any* node reflects it, eventually.
- Structured logging (PRD Req 6, via `tracing`) on every `/register`, `/lookup`, and `/remove` call, with the specific fields the PRD lists (§3.7).
- **Out of scope, per PRD §4 (as of this revision)**: a management/introspection interface into the mesh, and any "retrieve all domain endpoints" bulk API. Earlier drafts of this document treated the former as an open question (Technical Brief mentions one) — the PRD now settles it: don't build it.

### 1.2 Non-functional

- No persistent database (PRD §4) — all routing state is in-memory, reconstructed via gossip and re-registration.
- Fault tolerant / no single point of failure — any node answers any lookup; HyParView tolerates high churn (brief cites up to 90% node failure) without partitioning the mesh.
- Horizontally scalable — adding DER nodes should not require reconfiguring existing ones or a coordinator.
- Encrypted transport (PRD Req 4) via HTTP/3 (QUIC/TLS 1.3) for the public API, HTTP/2 fallback where QUIC/UDP is unavailable — **and, as of this revision, inter-node gossip transport must also terminate over TLS using the same CLI-supplied cert/key.** This resolves what earlier drafts of this document listed as an open question (§3.6, §6): it's no longer open.
- Low latency on the lookup path — lookups are synchronous, in the hot path of every event send from an event source.
- Rust, idiomatic error handling — **no `.unwrap()` in production code** (PRD §2, added this revision) — `clap`-driven CLI configuration (cert path, health-check interval).

### 1.3 Constraints

- Fixed dependency set from the PRD: `saorsa-gossip` (HyParView + PlumbTree), `quiche` + `tokio-quiche` (HTTP/3), `rmp` (MessagePack), `clap`, `tracing` (added this revision, for Req 6 logging).
- Out of scope: implementing Event Router / Event Sink behavior; persistent storage of any kind; a management/introspection interface into the mesh; bulk retrieval of all domain endpoints (the last two added to the PRD's out-of-scope list this revision).
- Current repo state: no `Cargo.toml` yet, `src/main.rs` is still the `cargo new` stub — this document describes the target architecture, not what's implemented.

## 2. High-Level Design

### 2.1 Component diagram (single DER node)

```
                    +-----------------------------------------------------+
                    |                    DER Mesh Node                    |
                    |                                                     |
 HTTP/3 (QUIC)      |   +-----------+        +------------------------+   |
 fallback HTTP/2 -->|   |API Layer  |------->| Domain Routing Table   |   |
 TLS 1.3, MsgPack   |   |/register  |<-------|  in-memory, keyed by   |   |
 payloads           |   |/lookup    |        |  domain string; each   |   |
                    |   |/remove    |        |  entry an add-wins set |   |
                    |   +-----+-----+        |  of {listener,         |   |
                    |         |              |  healthcheck} pairs    |   |
                    |         |              |  (§3.1)                |   |
                    |         |              +----------+-------------+   |
                    |         |  (tracing:               |                |
                    |         |   Register/Lookup/Remove v                |
                    |         |   events, PRD Req 6) +-----------------+  |
                    |         |                 |  Gossip Engine  |<------+--> peer DER nodes
                    |         |                 | HyParView +     |       |   (gossip protocol,
                    |         |                 | PlumbTree       |       |    MessagePack, over
                    |         |                 | (saorsa-gossip) |       |    TLS -- PRD Req 4)
                    |         |                 +--------+--------+       |
                    |         |                          |                |
                    |   +-----v----------+               |                |
                    |   | Health-Check   |<--------------+                |
                    |   | Scheduler      |----> for each domain assigned  |
                    |   | (ticker, only  |      to this node, GET *every* |
                    |   |  mesh-assigned |      pair's healthcheck URI in |
                    |   |  domains)      |      its merged set (Req 5);   |
                    |   |                |      on failure, evict that    |
                    |   |                |      WHOLE pair (Req 1)        |
                    |   +--------+-------+                                |
                    |            |                                        |
                    |   +--------v--------+                               |
                    |   |  CLI Config     |  (clap: TLS cert path,        |
                    |   |  (clap)         |   health-check interval)      |
                    |   +-----------------+                               |
                    +-----------------------------------------------------+
```

Five components per node, plus cross-cutting logging:

1. **API layer** — terminates HTTP/3 (fallback HTTP/2), decodes/encodes MessagePack, exposes the three PRD routes, merges concurrent registrations rather than overwriting (§3.2), and emits a `tracing` event for every call (§3.7).
2. **Domain routing table** — the in-memory distributed cache: `domain -> OrSet<{listener, healthcheck}>` (an add-wins merged set of *pairs*, not two independent per-field sets — §3.1).
3. **Gossip engine** — wraps `saorsa-gossip`: HyParView maintains this node's active/passive peer views, PlumbTree disseminates mutations over the tree built on top of that view, now over a TLS-terminated transport (PRD Req 4).
4. **Health-check scheduler** — a ticker (interval from CLI) that walks whichever domains the mesh's assignment function currently maps to this node (§3.4) and calls *every* pair's health-check URI currently in that domain's merged set — not just one. On failure, evicts the whole pair (listener + healthcheck together). This is *not* tied to which node originally accepted the `/register` call.
5. **CLI config** — `clap`-parsed: TLS cert path (now used for both the public API *and* gossip transport), health-check interval, bind address, seed peers for mesh join.

### 2.2 Data flow

**Register**

```
sink --POST /register (MsgPack)--> Node A: API layer
                                         | validate + URL-decode domain
                                         | log "Registration": src ip/port, domain,
                                         |     listener+healthcheck pairs (PRD Req 6)
                                         v
                                   Node A: routing table (merge: union new {listener,
                                         |   healthcheck} pairs into the domain's
                                         |   existing OR-Set, tagged (node_id, clock),
                                         |   de-duplicated by pair value -- §3.2)
                                         v
                                   Node A: gossip engine -- PlumbTree broadcast (TLS) --> Node B, C, D...
                                                                                     | apply merge
                                                                                     | (mesh recomputes health-check
                                                                                     |  assignment for this domain, §3.4)
sink <--200 OK (MsgPack ack)-- Node A
```

**Lookup** (read-only, no gossip involved)

```
source --GET /lookup?domain=...--> any Node N: API layer
                                         | URL-decode domain, read merged routing-table entry
                                         | log "Lookup": domain, success/failure + listener URIs (PRD Req 6)
                                         v
source <--{domain, endpoints[]} (MsgPack), or 404--
```

**Remove** — two distinct triggers with different scope, per this revision:

```
[sink shutdown]  DELETE /remove?domain=...  --> Node A: routing table (tombstone the WHOLE
                                                 |   domain -- PRD's /remove carries only a
                                                 |   domain, no per-sink identity, so this
                                                 |   removes every contributor's entries -- §3.2)
                                                 | log "Remove": domain (PRD Req 6)
                                                 v
                                          Node A: gossip engine -- PlumbTree broadcast (TLS) --> rest of mesh

[health-check on the node the mesh assigns  --> that node: routing table (remove the WHOLE
 to check this domain fails for ONE of its       {listener, healthcheck} pair -- the failing
 pairs' healthcheck URIs, §3.4]                   healthcheck AND its 1:1-paired listener,
                                                   together, from the OR-Set -- PRD Req 1, §3.4)
                                                 v
                                          that node: gossip engine -- PlumbTree broadcast (TLS) --> rest of mesh
```

### 2.3 API contracts

Routes and payload shapes are fixed by the PRD; restated here for the design's internal reference:

| Route | Method | Request | Response |
|---|---|---|---|
| `/register` | POST | MessagePack list of `{domain-name, domain-uris: [{listener, healthcheck}]}` — each pair merged into any existing entry for that domain, not overwritten | `200` ack, or `4xx` with a validation error, MessagePack-encoded |
| `/lookup` | GET | query param `domain` (URL-encoded) | MessagePack `{domain, endpoints: [url]}` (the merged listener set), or `404` if unknown |
| `/remove` | DELETE | query param `domain` (URL-encoded) | `200`/`204` ack (removes the whole domain, all contributors), or `404` if unknown |

`domain` values contain `@` and `.` (e.g. `starfoods.quality@v1`) — both routes must percent-encode/decode this correctly (PRD calls this out explicitly for `/lookup` and `/remove`).

### 2.4 Storage

No database, by constraint. The routing table is an in-memory concurrent map (e.g. `DashMap<String, DomainEntry>` or an `RwLock<HashMap<...>>` behind the gossip engine's apply path) — it *is* the distributed cache; durability comes from re-registration after a restart, not from disk. Each `DomainEntry` is now a merged set (§3.1), not a single overwritable value.

## 3. Deep Dive

### 3.1 Data model

```
struct ListenerHealthcheckPair {
    listener: Url,
    healthcheck: Url,
    // 1:1 relationship (PRD Req 1): a listener never exists in the routing
    // table without its healthcheck, and vice versa. The pair, not either
    // field alone, is the unit of registration, merge, and eviction.
}

struct DomainEntry {
    // Add-wins observed-remove set (OR-Set) of PAIRS, not two independent
    // per-field sets: each element is a (ListenerHealthcheckPair) value
    // tagged with (node_id, logical_clock) identifying the registration
    // that added it. Merge = union of all (pair, tag) values seen; a pair
    // is "live" if it has at least one add-tag not cancelled by a matching
    // remove-tag. Eviction removes a pair's tags as a single atomic unit --
    // there is no operation that touches a listener or healthcheck alone.
    pairs: OrSet<ListenerHealthcheckPair>,
}
```

Keyed by the full domain string (`"starfoods.quality@v1"`) — no normalization beyond what URL-decoding already does, since the domain string is opaque to the DER (schema/attribute semantics belong to the Event Router, not the resolver).

This is the second structural change to this model. The first revision of this document moved from a single-value-with-version model (one sink owns one domain's registration) to two independent OR-Sets — `listeners` and `healthchecks` — merged separately, because PRD Req 1 required concurrent multi-sink registration to merge rather than clobber. This revision merges those two sets into **one OR-Set of `{listener, healthcheck}` pairs**, because the PRD went further: it specified an explicit 1:1 relationship between a listener and its healthcheck (the `/register` payload itself changed to a single `domain-uris` array of paired objects), and — critically — that evicting a failing healthcheck must remove its paired listener too. Two independent sets can't express that pairing or enforce that a listener and its healthcheck rise and fall together; one set of pair-values can, for free, which is why ADR-0001's third revision treats this as a further reinforcement of the gossip-mesh/mergeable-set choice rather than a complication of it.

**New subtlety this model surfaces**: dedup is now by *pair* value, not by listener or healthcheck individually. If two sinks register the same listener URL with two different healthcheck URLs, both pairs survive as distinct set members — the listener isn't deduplicated across pairs. Evicting one of those pairs (its healthcheck fails) removes only that pair; the same listener URL remains reachable via the other, still-healthy pair. This is a natural consequence of pair-level atomicity, not a bug, but worth calling out since a naive reader might expect "the listener" to be a single dedup key (see §6).

### 3.2 Merge semantics (was: conflict resolution)

Three distinct operations touch a domain's entry, with different scope:

- **Register** (`/register`, PRD Req 1) — **add-only**: the `{listener, healthcheck}` pairs in the request are unioned into the domain's existing OR-Set, each tagged with `(node_id, logical_clock)` of the registering node. Two sinks registering the same domain concurrently both succeed; their contributions merge, de-duplicated by *pair* value, per the PRD's literal wording ("removing duplicates from both the 'healthcheck' array and the 'listener' entries").
- **Health-check-triggered eviction** (§3.4, PRD Req 1/5) — **single-pair removal**: when one healthcheck URI fails, the mesh removes that entire `{listener, healthcheck}` pair — both fields together — from the OR-Set. This was an open question in the prior revision of this document (would a failing healthcheck evict only itself, or its listener too?); the PRD has since settled it explicitly in favor of removing the whole pair.
- **Remove** (`/remove`, PRD Req 4) — **whole-domain tombstone**: since the request carries only a domain name and no per-sink identity, this removes every current contributor's pairs for that domain, not just one sink's. Unlike the prior revision of this document, this is no longer a deliberate asymmetry worth flagging as unresolved — both `/remove` (all pairs) and health-check eviction (one pair) now operate on the same unit (a pair, or the full set of pairs), just at different scope. The PRD additionally requires this call to be idempotent, leaving no trace of the domain once eventual consistency is reached.

Document the OR-Set tagging scheme itself (exact tag structure, tombstone garbage-collection policy) as a follow-up ADR once implementation starts — this design assumes add-wins OR-Set semantics satisfy the PRD's merge requirement, but the PRD doesn't mandate a specific CRDT, so treat this as this design's recommendation, not a PRD requirement.

### 3.3 Gossip message types

Mutation types flow over PlumbTree, MessagePack-encoded, now over a TLS-terminated gossip transport (PRD Req 4):

- `DomainRegistered { domain, pairs: [{listener, healthcheck}], tag }` — adds these `{listener, healthcheck}` pairs to the domain's OR-Set; applied as a **union-merge**, never an overwrite.
- `PairEvicted { domain, listener, healthcheck, tag }` — removes one specific unhealthy `{listener, healthcheck}` pair from the OR-Set, in its entirety. (Named `HealthCheckEvicted` in the prior revision of this document, when it was assumed to touch only the healthcheck field — renamed to make the paired-removal explicit.)
- `DomainRemoved { domain }` — the explicit `/remove` path: tombstones the whole domain, every pair from every contributor, independent of per-tag OR-Set bookkeeping. Idempotent — applying it twice (or replaying it via anti-entropy) is a no-op the second time.

All three carry enough identity (`tag`, or the domain name for a full tombstone) for receiving nodes to apply them idempotently if PlumbTree redelivers them. A joining node performs anti-entropy against its initial HyParView peers to bulk-sync the current OR-Set state (adds *and* tombstones) rather than waiting to receive every historical mutation.

### 3.4 Health-check assignment

There is no "associated event sink" concept tied to a particular DER node. The node that happened to accept a domain's `/register` call has no ongoing role once that mutation has been gossiped — health-check URIs are just more values in the domain's gossiped OR-Set, replicated to the whole mesh like everything else.

Instead, **the mesh itself decides who checks what**, purely from the information carried in the entry: every node runs the same deterministic assignment function (e.g., rendezvous/consistent hashing of the domain string over the current set of live mesh members, as seen in that node's gossip membership view) to work out which node currently owns the duty of polling a given domain's health-check URIs. Each node's health-check scheduler (§2.1) walks the domains the assignment function currently maps to *it*, and polls **every** pair's health-check URI in that domain's merged set (PRD Req 5: "if a domain has two different healthcheck URI's (after a merge) then the mesh should call both healthcheck URI's"), not just one.

Because assignment is recomputed from membership rather than fixed at registration time, it self-heals: when a node fails, every remaining node's assignment function naturally remaps that node's domains onto surviving members on the next membership-view update.

**Eviction scope, resolved (this was an open question in the prior two revisions of this document)**: when one of several health-check URIs for a domain goes unhealthy, the mesh removes **the entire `{listener, healthcheck}` pair** — the failing healthcheck *and* its 1:1-paired listener, together — from the domain's OR-Set (§3.1/§3.2). The PRD now states this explicitly: "there is a 1:1 relationship between the listener URI and its healthcheck endpoint. When the healthcheck is removed because of a failure, so should the listener entry for that healthcheck." Other pairs for the same domain (other sinks' still-healthy listener/healthcheck pairs) are untouched. Do not implement healthcheck-only eviction (leaving the paired listener in place) — that was this document's prior recommendation and is now superseded.

**Consequence to flag** (unchanged from the prior revision): during a membership-view change, different nodes may briefly disagree on who owns a given domain's check (a short window where a domain is checked twice, or not at all, until views converge). This is transient, not a correctness problem — idempotent apply (§3.5) absorbs duplicate evictions, and the next tick closes any gap.

### 3.5 Error handling & retries

- **Malformed `/register` payload** (missing field, bad URL) → `4xx`, no mutation applied, no gossip emitted.
- **Health-check timeout or connection refused, for any individual pair's healthcheck URI** → treated identically to an explicit `503`/`"unhealthy"` response for that URI: evict the whole `{listener, healthcheck}` pair (§3.4).
- **Gossip message loss** → tolerated by design: PlumbTree's lazy-push graft/prune recovers missing branches, and anti-entropy on join/rejoin catches anything missed. No node-to-node retry logic needed above what `saorsa-gossip` already provides.
- **Duplicate mutations** (same tag arriving twice) → idempotent apply, keyed by `(domain, tag)`.
- **Concurrent register and remove racing for the same domain** → the OR-Set's add-wins semantics (§3.1/§3.2) mean a register that a node hasn't yet seen the corresponding remove for still merges in cleanly; if the remove was a full-domain tombstone (§3.2) issued *after* that register was observed, standard OR-Set causal tracking removes it as expected. No special-cased locking is needed.
- **No `.unwrap()` in production code** (PRD §2, added this revision) — applies directly to gossip message (de)serialization, MessagePack encode/decode, and the now-multiple concurrent health-check HTTP calls per domain: all realistic runtime failure points that must return `Result`/propagate with `?`, not panic.

### 3.6 Security

- HTTP/3 (QUIC) gives TLS 1.3 by construction; the CLI-supplied certificate (PRD Req 4) configures this for the public API. **As of this revision, the same cert/key also terminates inter-node gossip transport over TLS** (PRD Req 4) — this was an open question in the prior revision of this document; the PRD now settles it explicitly, so gossip traffic is no longer treated as a separately-decided concern.
- HTTP/2 fallback must also be terminated over TLS — don't silently drop to plaintext HTTP/2 when QUIC is unavailable.
- The Technical Brief's access-control-list concept ("an event sink will only accept a message from a known event source") is an Event Router/Sink concern, out of scope for the DER itself (PRD §4).

### 3.7 Logging & Observability (PRD Req 6, new this revision)

Every API handler emits a structured `tracing` event:

| Action | Required fields |
|---|---|
| `/register` | `"Registration"`, source IP/port, domain, listener URIs, health-check URIs |
| `/lookup` | `"Lookup"`, domain, success/failure (listener URIs returned, or not-found) |
| `/remove` | `"Remove"`, domain |

This is instrumentation on the existing handlers, not a queryable interface — the PRD explicitly puts a mesh management/introspection interface and bulk domain-listing out of scope (§4, added this revision). Any future operational need to inspect mesh state should go through whatever `tracing` subscriber/exporter is wired up (e.g. shipping spans to a log aggregator), not a bespoke API on the DER node itself.

## 4. Scale & Reliability

### 4.1 Load assumptions (explicit, since the PRD gives none)

Stated as assumptions, not requirements — revisit once real usage is known:

- O(10²–10³) DER nodes in a mesh, O(10⁴) registered domains.
- Lookups are the dominant traffic (every event source resolves before every send, or caches per Technical Brief's "Dynamic Discovery"); registers/removes are comparatively rare (sink lifecycle events).
- A domain may now have more than one `{listener, healthcheck}` pair (one or more per contributing sink, PRD Req 1/5) — health-check fan-out per node is bounded by the total number of pairs across the domains the mesh's assignment function (§3.4) currently maps to it, not by node count directly.

### 4.2 Horizontal scaling

HyParView bounds each node's active view (small, fixed fan-out) regardless of mesh size, so per-node connection/memory cost doesn't grow linearly with total node count. Gossip convergence latency grows with the diameter of the overlay (~O(log N) hops for typical HyParView/PlumbTree configurations), not with N directly — new nodes join by contacting seed peers (CLI-configured) and populate their view/table (including OR-Set tombstones, §3.3) via anti-entropy. Joining also causes health-check assignment (§3.4) to rebalance some domains onto the new node.

### 4.3 Failover & redundancy

- **DER node failure**: HyParView's partial-view membership is explicitly designed to tolerate high churn; surviving nodes repair their active views from the passive view without manual intervention. Lookups against surviving nodes are unaffected; health-check responsibility for domains the assignment function had mapped to the failed node is picked up by another node automatically on the next membership-view update (§3.4).
- **Sink failure**: handled by the health-check → evict → gossip-remove path, independent of DER node failure. With the pair model (§3.1/§3.2), one sink's failure evicts exactly its `{listener, healthcheck}` pair(s) from a shared domain, leaving other sinks' pairs intact — this is now settled behavior, not the open granularity question earlier revisions of this document flagged in §3.4.
- **Network partition**: gossip continues within each partition; on heal, anti-entropy reconciles OR-Set state (adds and tombstones, §3.3). A registration made only inside a minority partition is visible there until the partition heals.

### 4.4 Monitoring

The PRD explicitly puts a management/introspection interface out of scope this revision (§4) — so, unlike the prior revision of this document (which suggested exposing metrics via such an interface), any operational visibility into mesh health must ride on the `tracing` instrumentation already required by Req 6 (§3.7), e.g. a `tracing` subscriber exporting to logs or a metrics backend, not a bespoke query API on the DER node. Worth instrumenting via that path:

- Health-check failure rate and eviction count, scoped to domains/URIs this node is currently assigned to check
- Gossip view size (active/passive), message send/receive rates, anti-entropy sync duration on join
- HTTP/3-vs-HTTP/2 request ratio, to catch unexpected QUIC-blocked environments early

## 5. Trade-off Analysis

Builds on [ADR-0001](adrs/ADR-0001-gossip-mesh-architecture.md); additional trade-offs specific to this design:

| Decision | Trade-off |
|---|---|
| Add-wins OR-Set merge per domain (§3.1/§3.2) vs. last-writer-wins overwrite | An OR-Set correctly supports the PRD's mandated union/de-dup merge for concurrent multi-sink registrations (Req 1) — a plain LWW register (this design's original approach) cannot express "keep both sinks' contributions." The cost is more bookkeeping (per-value add/remove tags instead of one version per domain) |
| One OR-Set of `{listener, healthcheck}` pairs (§3.1) vs. two independent OR-Sets (this design's prior approach) | A single pair-set correctly enforces the PRD's 1:1 relationship and paired eviction (Req 1) — two independent sets can't express "these two values rise and fall together" without extra correlation logic. The cost: a listener can now appear in more than one live pair (if paired with different healthchecks by different sinks) rather than being a single dedup key — a subtlety worth remembering (§3.1) |
| Whole-domain tombstone on `/remove` vs. per-pair-scoped removal (§3.2) | Matches the PRD's actual `/remove` request shape (domain only, no sink identity) exactly; no longer an unresolved asymmetry with health-check eviction now that both operate on the same unit (a pair) at different scope — this row previously flagged that asymmetry as an open question, now resolved |
| Mesh-assigned health-checking via membership hashing (§3.4) vs. registration-owner health-checking | Assignment-based checking removes the dependency on the original registering node staying alive — a failed assignee's domains are picked up by another node on the next view change. The cost is a brief duplicate-or-gap window during membership churn, and now also multiple HTTP calls per assigned domain instead of one |
| Reusing the public API's TLS cert/key for gossip transport (PRD Req 4) vs. a separate mesh-internal cert | The PRD mandates reuse of the same CLI-supplied cert/key, which is simpler to configure but couples gossip-channel cert rotation to the public API's — accepted as a given constraint, not a genuine trade-off this design is free to weigh |
| In-memory-only state (mandated) vs. any durability | Rules out crash recovery of the table without re-registration; this is an explicit PRD constraint, not a choice, but worth naming as the reason full-mesh-restart is a real risk (see ADR-0001's negative consequences) |
| Single flat domain namespace vs. sharding across sub-meshes | Flat namespace matches the PRD's API surface exactly (`/lookup?domain=`) and is simplest to reason about; sharding would only be worth revisiting if a single mesh's gossip fan-out become a bottleneck at real scale |

## 6. Open Questions / Revisit as the System Grows

- **Resolved, kept here for history**: ~~should inter-node gossip traffic be encrypted?~~ — yes, PRD Req 4 mandates it, reusing the public API's cert/key (§3.6). ~~Should the mesh expose a management/introspection interface?~~ — no, PRD §4 explicitly puts it out of scope (§3.7, §4.4). ~~Does health-check-triggered eviction remove only the failing URI, or its listener too, or the whole domain?~~ — resolved this revision: it removes the whole `{listener, healthcheck}` pair, nothing more and nothing less (§3.4).
- **New subtlety this revision surfaces, not fully a question but worth flagging**: because dedup is by pair value (§3.1), the same listener URL can appear in more than one live pair if different sinks register it with different healthcheck URLs. Is that actually desirable (redundant listener reachability via multiple healthchecks), or should the mesh instead reject/flag a listener URL that's already paired with a *different* healthcheck elsewhere in the domain? The PRD doesn't address this case.
- What hashing/assignment scheme should determine which node(s) health-check a given domain (rendezvous hashing, consistent hashing, something else), and should it assign more than one node per domain for redundancy during membership churn? (§3.4)
- The OR-Set tagging scheme (§3.1/§3.2) — exact tag structure, and when/how tombstones can be garbage-collected without breaking causal merge correctness — should get its own ADR once real implementation surfaces a concrete need, rather than being assumed here.
