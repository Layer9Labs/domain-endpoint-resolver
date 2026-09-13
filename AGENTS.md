# AGENTS.md

## What this is

Rust implementation of the **Domain Endpoint Resolver (DER)** — a distributed mesh node for domain endpoint discovery in Layer 9 Labs' Event Network Architecture. Event sinks register domains + listener endpoints + a health-check URI; the mesh gossips the routing table; event sources query any mesh node to resolve domain endpoints; nodes periodically health-check registered sinks and evict unhealthy entries.

## Source of truth (docs/)

- `docs/Domain Endpoint Resolver Product Requirements Document (PRD).md` — current behavior spec. API surface: `/register` (POST, list of domain/listeners/healthcheck), `/lookup` (GET by domain), `/remove` (DELETE by domain). Stacks: HTTP/3 w/ fallback HTTP/2 (`quiche`+`tokio-quiche`), HyParView+PlumbTree gossip via `saorsa-gossip`, MessagePack (`rmp`) for wire serialization, `clap` for CLI (TLS cert, health-check interval).
- `docs/context/Technical Brief for Event Network Architecture.md` — architecture context for the mesh (distributed cache, sidecar deployment, gossip semantics). Note: it describes a gRPC interface, but the PRD supersedes that with HTTP routes.

Health-check contract (PRD Req 5): healthy = HTTP 200 with body "healthy"; unhealthy = HTTP 503 with body "unhealthy" → remove the domain's endpoints.

## Hard constraints (PRD §4)

- Out of scope: Event Router / Event Sink implementations; no persistent database.

## Build/test

Rust binary crate. **Gotcha:** there is currently no `Cargo.toml` — only `src/main.rs` (hello world) and `Cargo.lock`. Before `cargo test`/`cargo build` works, a manifest must exist; create it if asked to build (package name: `domain-endpoint-resolver`).

## Conventions

- ADRs live in `docs/adrs/` using `docs/adrs/MADR Template.md` (MADR format, numbered `ADR-XXXX`, status line required). Record any significant design decision there.
- `*.log` files are gitignored — stray `*.log` files at repo root are scratch tool output, not something to keep.
