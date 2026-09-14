# ADR-0001: Gossip-Based Mesh Architecture over HTTP/3 for Domain Endpoint Resolution

* Status: Proposed
* Date: 2026-09-12 (revised 2026-09-13, three times — see notes below)

> Revision note (2026-09-13, first pass): originally drafted and marked Accepted against the initial PRD; status was reopened to Proposed and this ADR revised in place (not superseded) because the PRD changed in ways that bear directly on this decision, before the decision was ever finalized: domains may now be registered by multiple concurrent sinks and must be merged rather than overwritten (PRD Req 1), a domain now carries a list of health-check URIs rather than one (PRD Req 5), and inter-node gossip transport must itself be encrypted (PRD Req 4). See §"Negative Consequences" and §"Pros and Cons" for what changed.
>
> Revision note (2026-09-13, second pass): the PRD was clarified further, resolving two things the first pass had left open or implicit: (1) a failing health-check now explicitly removes **only that URI** from a domain's health-check set — not the domain, not its listeners (PRD Req 1); (2) `/remove` is now explicitly required to be **idempotent** and to leave no trace of the domain, anywhere in the mesh, once eventual consistency is reached (PRD Req 4). Both are folded into "Negative Consequences" and "Pros and Cons" below.
>
> Revision note (2026-09-13, third pass — **reverses the second pass's eviction guidance**): the `/register` payload itself changed shape — from two parallel arrays (`listeners[]`, `healthchecks[]`) to a single `domain-uris` array of explicit `{listener, healthcheck}` pairs, with the PRD now stating a 1:1 relationship between a listener and its healthcheck. Consequently, the PRD now says that when a healthcheck fails, **its paired listener is removed too**, not just the healthcheck URI as the second pass concluded. That earlier guidance is superseded below, not left standing — see "Negative Consequences" and "Pros and Cons."

## Context and Problem Statement

Event sinks and event routers need to register the domains they can receive events for — as one or more explicit `{listener, healthcheck}` pairs, a 1:1 relationship per the PRD — and event sources need to resolve a domain to its current listener endpoints, against any node in a network of DER nodes. Multiple sinks can register the *same* domain concurrently, and the mesh must present a single merged view of that domain's listener/healthcheck pairs rather than one sink's registration clobbering another's. The mesh must stay fault-tolerant and horizontally scalable as nodes join and leave, must not depend on a persistent database (PRD §4), and must keep the registration/lookup/removal API and unhealthy-endpoint eviction working even as the routing table changes continuously across nodes. What architecture lets any node answer a lookup correctly, merge concurrent registrations correctly, and without a central source of truth?

## Decision Drivers

* No persistent database allowed (PRD §4) — routing state must live in memory and be reconstructable from gossip, not from a store
* Fault tolerance and horizontal scalability — no single point of failure; node membership changes shouldn't require a coordinator
* Concurrent registration of the same domain by two or more sinks must merge cleanly — union and de-duplicate their `{listener, healthcheck}` pairs (PRD Req 1; the payload is a single `domain-uris` array of explicit pairs, not two independent parallel arrays) — without cross-node locking or a designated "owner" for a domain
* A single failing health-check must remove its **entire pair** — both the healthcheck and its 1:1-paired listener — from the domain's set (PRD Req 1, revised this pass; supersedes this ADR's own prior conclusion that only the healthcheck URI should be removed) — and an explicit `/remove` must fully and idempotently clear a domain, leaving no trace anywhere in the mesh once eventual consistency is reached (PRD Req 4)
* Payload confidentiality in transit is a hard requirement for both the public API (PRD Req 4) *and*, as of this revision, inter-node gossip traffic itself, which must also terminate over TLS using the same CLI-supplied cert/key
* Low-latency, low-overhead wire format for both the public API and inter-node propagation, given lookups are on the hot path for every event send
* Rust-idiomatic implementation with strict error handling — no `.unwrap()` in production code (PRD §2) — deployable as a sidecar per PRD's Technical Brief context

## Considered Options

* Option A: Gossip mesh (HyParView membership + PlumbTree dissemination via `saorsa-gossip`) with an HTTP/3-first (fallback HTTP/2) API surface and MessagePack wire encoding
* Option B: Centralized registry service backed by a coordination store (e.g., Consul, etcd, or a SQL/KV database) that all nodes query
* Option C: Custom TCP protocol with persistent connections and JSON payloads, hand-rolled membership/failure-detection

## Decision Outcome

Chosen option: **Option A**, because it is the only option that satisfies the no-persistent-database constraint while still giving every node a locally-servable, eventually-consistent copy of the routing table, and it reuses well-understood gossip protocols (HyParView for partial-view membership, PlumbTree for efficient broadcast) instead of a bespoke failure-detection scheme. The PRD's revised merge requirement reinforces this choice: representing each domain's listeners and health-checks as a set that gossip mutations union together is a natural fit for a gossip/CRDT-style mesh, and would require bolting on explicit merge/transaction logic under either alternative.

### Positive Consequences

* No standing coordination service to operate, patch, or scale (rules out Option B's operational surface entirely)
* Any mesh node can serve `/register`, `/lookup`, and `/remove` independently — no request has to be proxied to a "primary"
* MessagePack keeps both the public API payloads and inter-node gossip messages small and fast to (de)serialize compared to JSON
* HTTP/3 (QUIC) gives encryption-in-transit (TLS 1.3) and connection migration for free, with HTTP/2 as a documented fallback for environments where UDP/QUIC is blocked; gossip transport now gets the same TLS treatment (PRD Req 4), so confidentiality is consistent across the public API and inter-node propagation rather than being a public-API-only guarantee
* Concurrent same-domain registrations (PRD Req 1) merge as a straightforward set-union of `{listener, healthcheck}` pairs propagated via gossip mutations — no cross-node coordination or locking needed to reconcile them
* The same mergeable-set model gives precise, per-pair eviction "for free": removing one failing `{listener, healthcheck}` pair (PRD Req 1, revised this pass) is just removing that one pair-element from the set, and a full `/remove` (PRD Req 4) is a domain-wide tombstone over every pair — both map directly onto gossip mutations already in place, with idempotent apply (§"Negative Consequences") satisfying the PRD's idempotency requirement without extra mechanism

### Negative Consequences (Instructions for AI)

* The mesh is only eventually consistent: a `/register` or health-check-triggered `/remove` on one node takes one or more gossip rounds to reach others. AI implementing lookups must not assume a lookup immediately after a register/remove on a *different* node will reflect that change — this is expected behavior, not a bug to "fix" with synchronous broadcast.
* `quiche`/`tokio-quiche` (HTTP/3) is a heavier, less mature dependency than a plain HTTP/1.1 or HTTP/2-only stack. The HTTP/2 fallback path must be implemented and tested explicitly, not assumed to be free — do not skip it as "unlikely to be hit."
* Because there is no persistent store, if every node in the mesh restarts simultaneously, the entire routing table is lost; recovery depends on event sinks re-registering. AI must not add local disk persistence as a "fix" for this — it's the explicit out-of-scope constraint in PRD §4, not an oversight.
* A domain entry is no longer a single owner's registration: it must be modeled as a **merged set of `{listener, healthcheck}` pairs**, unioned and de-duplicated across every sink that has registered that domain (PRD Req 1). AI must not implement registration as a last-writer-wins overwrite of the whole entry — a second sink registering the same domain adds its pairs to the existing set, it does not replace them. Note the payload shape itself: it is a single `domain-uris` array of paired objects, not two independently-indexed `listeners[]`/`healthchecks[]` arrays — there is no way to have a listener without its healthcheck, or vice versa.
* Because a domain can now carry multiple `{listener, healthcheck}` pairs (one or more per contributing sink), health-check polling and eviction must operate per pair, not once per domain (PRD Req 5 explicitly requires calling *every* health-check URI after a merge). **This pass reverses guidance the prior revision of this ADR gave here** — that earlier text said a failing health-check should remove *only* the healthcheck URI, leaving listeners untouched. The PRD has since made the 1:1 relationship explicit and states that removing a failing healthcheck must remove its *paired listener too*. AI must treat the pair as one atomic unit for eviction: on a health-check failure, remove both the healthcheck and its 1:1 listener together from the domain's pair-set, then gossip that removal through PlumbTree like any other mutation — do not implement the "healthcheck-only, listener untouched" behavior this ADR previously described.
* `/remove` (PRD Req 4) is explicitly required to be **idempotent** and to leave the domain with no trace anywhere in the mesh once eventual consistency is reached. AI must implement it as a full removal of every `{listener, healthcheck}` pair for that domain (broader in scope than single-pair health-check eviction above, which removes only the one failing pair), and must ensure a repeated `/remove` for an already-removed or never-registered domain is a safe no-op — not a `404`/error that a caller needs to avoid triggering twice.
* No `.unwrap()` in production code (PRD §2) is a hard constraint, not a style preference — this applies directly to gossip message (de)serialization, MessagePack encode/decode, and the now-multiple concurrent health-check HTTP calls per domain, all of which are realistic runtime failure points that must be handled explicitly (`Result`/`?`, not panics).
* Registration, lookup, and removal actions must be logged (PRD Req 6, via the newly added `tracing` dependency) with the specific fields the PRD lists (source IP/port, domain, listener/health-check URIs, success/failure). AI must treat this as a cross-cutting instrumentation requirement on every API handler, not an optional add-on.
* The mesh's management/introspection interface and bulk "retrieve all domain endpoints" capability are now explicitly out of scope (PRD §4, added in this revision). AI must not build either as part of implementing this architecture, even though the Technical Brief context describes a management interface — the PRD supersedes it here.

## Pros and Cons of the Options

### Option A: Gossip mesh (HyParView + PlumbTree) over HTTP/3, MessagePack ✅ Chosen

* Good, because it satisfies the "no persistent database" constraint natively — state lives in memory and self-heals via gossip
* Good, because HyParView bounds each node's connection fan-out (partial view), so the mesh scales without all-to-all connections
* Good, because HTTP/3/TLS 1.3 satisfies the encrypted-payload requirement without a separate encryption layer, and the same TLS approach extends naturally to encrypting gossip transport (PRD Req 4)
* Good, because representing a domain as a gossiped, mergeable set of `{listener, healthcheck}` pairs handles concurrent multi-sink registration (PRD Req 1) without extra locking or transaction logic
* Good, because the same set-based model gives precise single-*pair* health-check eviction (removing a failing healthcheck and its 1:1-paired listener together, PRD Req 1 revised) and idempotent, complete domain removal (PRD Req 4) as natural operations on the set — no bolt-on mechanism needed
* Bad, because eventual consistency means a lookup can briefly return stale or missing data right after a change elsewhere in the mesh
* Bad, because `quiche`/`tokio-quiche` and `saorsa-gossip` are less mature/battle-tested than mainstream alternatives, adding integration risk

### Option B: Centralized registry (Consul / etcd / database)

* Good, because it gives strong consistency and a simpler mental model (single source of truth)
* Good, because these are mature, well-documented systems with strong operational tooling
* Bad, because it violates the explicit "no persistent database" constraint (PRD §4)
* Bad, because it introduces a single point of failure / a standing service the DER mesh would depend on, undermining the fault-tolerance goal
* Bad, because merging concurrent multi-sink registrations of the same domain (PRD Req 1) would require explicit transactional read-modify-write logic against the store, rather than getting set-union merge for free from the replication mechanism itself

### Option C: Custom TCP + JSON, hand-rolled membership

* Good, because it avoids taking on `quiche`/`saorsa-gossip` as dependencies
* Bad, because it means reimplementing failure detection, membership views, and broadcast dissemination that HyParView/PlumbTree already solve
* Bad, because JSON is slower to (de)serialize and larger on the wire than MessagePack, and TCP alone doesn't provide encryption without layering TLS manually — including now for the additional inter-node gossip channel (PRD Req 4)
* Bad, because it directly contradicts the PRD's specified crate dependencies (`saorsa-gossip`, `quiche`, `rmp`, `tracing`)
* Bad, because the merge semantics for concurrent multi-sink registration (PRD Req 1) would have to be hand-built alongside everything else, with no gossip/CRDT substrate to lean on

## Links

* [Domain Endpoint Resolver Product Requirements Document (PRD)](../Domain%20Endpoint%20Resolver%20Product%20Requirements%20Document%20(PRD).md) — source of the architectural constraints and API surface driving this decision; see Req 1 (the `domain-uris: [{listener, healthcheck}]` payload shape, merge semantics, and paired eviction scope), Req 4 (gossip encryption *and* idempotent, complete `/remove`), Req 5 (multi-pair health-check), Req 6 (logging) for what changed across all three revisions
* [Technical Brief for Event Network Architecture](../context/Technical%20Brief%20for%20Event%20Network%20Architecture.md) — broader mesh/sidecar deployment context
* [Domain Endpoint Resolver System Design](../Domain%20Endpoint%20Resolver%20System%20Design.md) — companion design doc; still modeled around the second revision's "healthcheck-only eviction" and independent listener/healthcheck sets, both now superseded by this pass (paired `{listener, healthcheck}` model, paired eviction) — needs a follow-up revision
