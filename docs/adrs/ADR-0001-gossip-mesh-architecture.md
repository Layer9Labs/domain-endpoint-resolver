# ADR-0001: Gossip-Based Mesh Architecture over HTTP/3 for Domain Endpoint Resolution

* Status: Accepted
* Date: 2026-09-12

## Context and Problem Statement

Event sinks and event routers need to register the domains they can receive events for (with listener endpoints and a health-check URI), and event sources need to resolve a domain to its current listener endpoints, against any node in a network of DER nodes. The mesh must stay fault-tolerant and horizontally scalable as nodes join and leave, must not depend on a persistent database (PRD §4), and must keep the registration/lookup/removal API and unhealthy-endpoint eviction working even as the routing table changes continuously across nodes. What architecture lets any node answer a lookup correctly without a central source of truth?

## Decision Drivers

* No persistent database allowed (PRD §4) — routing state must live in memory and be reconstructable from gossip, not from a store
* Fault tolerance and horizontal scalability — no single point of failure; node membership changes shouldn't require a coordinator
* Payload confidentiality in transit is a hard requirement (PRD Req 4), ideally without hand-rolling a TLS layer
* Low-latency, low-overhead wire format for both the public API and inter-node propagation, given lookups are on the hot path for every event send
* Rust-idiomatic implementation with strict error handling, deployable as a sidecar per PRD's Technical Brief context

## Considered Options

* Option A: Gossip mesh (HyParView membership + PlumbTree dissemination via `saorsa-gossip`) with an HTTP/3-first (fallback HTTP/2) API surface and MessagePack wire encoding
* Option B: Centralized registry service backed by a coordination store (e.g., Consul, etcd, or a SQL/KV database) that all nodes query
* Option C: Custom TCP protocol with persistent connections and JSON payloads, hand-rolled membership/failure-detection

## Decision Outcome

Chosen option: **Option A**, because it is the only option that satisfies the no-persistent-database constraint while still giving every node a locally-servable, eventually-consistent copy of the routing table, and it reuses well-understood gossip protocols (HyParView for partial-view membership, PlumbTree for efficient broadcast) instead of a bespoke failure-detection scheme.

### Positive Consequences

* No standing coordination service to operate, patch, or scale (rules out Option B's operational surface entirely)
* Any mesh node can serve `/register`, `/lookup`, and `/remove` independently — no request has to be proxied to a "primary"
* MessagePack keeps both the public API payloads and inter-node gossip messages small and fast to (de)serialize compared to JSON
* HTTP/3 (QUIC) gives encryption-in-transit (TLS 1.3) and connection migration for free, with HTTP/2 as a documented fallback for environments where UDP/QUIC is blocked

### Negative Consequences (Instructions for AI)

* The mesh is only eventually consistent: a `/register` or health-check-triggered `/remove` on one node takes one or more gossip rounds to reach others. AI implementing lookups must not assume a lookup immediately after a register/remove on a *different* node will reflect that change — this is expected behavior, not a bug to "fix" with synchronous broadcast.
* `quiche`/`tokio-quiche` (HTTP/3) is a heavier, less mature dependency than a plain HTTP/1.1 or HTTP/2-only stack. The HTTP/2 fallback path must be implemented and tested explicitly, not assumed to be free — do not skip it as "unlikely to be hit."
* Because there is no persistent store, if every node in the mesh restarts simultaneously, the entire routing table is lost; recovery depends on event sinks re-registering. AI must not add local disk persistence as a "fix" for this — it's the explicit out-of-scope constraint in PRD §4, not an oversight.
* Health-check-driven eviction (PRD Req 5) is per-node (the node that owns the health-check call removes the entry locally, then gossips the removal); AI must ensure the removal itself is propagated through PlumbTree like any other mutation, not treated as purely local state.

## Pros and Cons of the Options

### Option A: Gossip mesh (HyParView + PlumbTree) over HTTP/3, MessagePack ✅ Chosen

* Good, because it satisfies the "no persistent database" constraint natively — state lives in memory and self-heals via gossip
* Good, because HyParView bounds each node's connection fan-out (partial view), so the mesh scales without all-to-all connections
* Good, because HTTP/3/TLS 1.3 satisfies the encrypted-payload requirement without a separate encryption layer
* Bad, because eventual consistency means a lookup can briefly return stale or missing data right after a change elsewhere in the mesh
* Bad, because `quiche`/`tokio-quiche` and `saorsa-gossip` are less mature/battle-tested than mainstream alternatives, adding integration risk

### Option B: Centralized registry (Consul / etcd / database)

* Good, because it gives strong consistency and a simpler mental model (single source of truth)
* Good, because these are mature, well-documented systems with strong operational tooling
* Bad, because it violates the explicit "no persistent database" constraint (PRD §4)
* Bad, because it introduces a single point of failure / a standing service the DER mesh would depend on, undermining the fault-tolerance goal

### Option C: Custom TCP + JSON, hand-rolled membership

* Good, because it avoids taking on `quiche`/`saorsa-gossip` as dependencies
* Bad, because it means reimplementing failure detection, membership views, and broadcast dissemination that HyParView/PlumbTree already solve
* Bad, because JSON is slower to (de)serialize and larger on the wire than MessagePack, and TCP alone doesn't provide encryption without layering TLS manually
* Bad, because it directly contradicts the PRD's specified crate dependencies (`saorsa-gossip`, `quiche`, `rmp`)

## Links

* [Domain Endpoint Resolver Product Requirements Document (PRD)](../Domain%20Endpoint%20Resolver%20Product%20Requirements%20Document%20(PRD).md) — source of the architectural constraints and API surface driving this decision
* [Technical Brief for Event Network Architecture](../context/Technical%20Brief%20for%20Event%20Network%20Architecture.md) — broader mesh/sidecar deployment context
