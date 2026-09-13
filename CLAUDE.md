# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Rust implementation of the **Domain Endpoint Resolver (DER)** — a distributed mesh node for domain endpoint discovery in Layer 9 Labs' Event Network Architecture. Event sinks register domains + listener endpoints + a health-check URI; the mesh gossips the routing table; event sources query any mesh node to resolve domain endpoints; nodes periodically health-check registered sinks and evict unhealthy entries.

Current state: this is a brand-new crate. `src/main.rs` is still the `cargo new` hello-world stub — none of the functional requirements below are implemented yet.

## Source of truth (docs/)

- [`docs/Domain Endpoint Resolver Product Requirements Document (PRD).md`](docs/Domain%20Endpoint%20Resolver%20Product%20Requirements%20Document%20(PRD).md) — the current behavior spec, and the first thing to (re-)read before implementing any route or protocol behavior. Key points:
  - API surface: `POST /register` (JSON list of `{domain, healthcheck, listeners[]}`), `GET /lookup?domain=...` (returns `{domain, endpoints[]}`), `DELETE /remove?domain=...`.
  - Stack: HTTP/3 with fallback to HTTP/2 (`quiche` + `tokio-quiche`), HyParView + PlumbTree gossip via `saorsa-gossip`, MessagePack (`rmp`) for wire serialization, `clap` for CLI args (TLS cert path, health-check interval).
  - Health-check contract (Req 5): healthy = HTTP 200 with body `"healthy"`; unhealthy = HTTP 503 with body `"unhealthy"` → mesh node removes that domain's endpoints.
  - Out of scope (§4): Event Router / Event Sink implementations; no persistent database — this service only routes data, all state is in-memory/gossiped.
- [`docs/context/Technical Brief for Event Network Architecture.md`](docs/context/Technical%20Brief%20for%20Event%20Network%20Architecture.md) — broader architecture context for the mesh (distributed cache, sidecar deployment, gossip semantics). Note: it describes a gRPC interface; the PRD supersedes that with the HTTP routes above where the two disagree.

## Core domain entities (PRD §5)

- **Event Sink** — listens on an IP/port for incoming event data; registers itself with the mesh.
- **Event Router** — an event sink whose received events are processed against a ruleset.
- **Event Source** — generates event data and sends it to a resolved sink/router endpoint.

The DER mesh node itself is neither of these — it only stores and gossips the domain → endpoint routing table.

## Build/test

Rust binary crate, but **there is currently no `Cargo.toml`** — only `src/main.rs` and `Cargo.lock`. `cargo build`/`cargo test`/`cargo run` will not work until a manifest exists. If asked to build or add dependencies, create `Cargo.toml` first (package name: `domain-endpoint-resolver`, matching `Cargo.lock`).

Once a manifest exists, standard Cargo workflow applies:

```bash
cargo build
cargo test
cargo test <test_name>   # run a single test
cargo clippy
cargo fmt
```

## Conventions

- **ADRs**: record any significant design decision (e.g. HTTP/3 framework choice, gossip topology tuning, eviction policy) in `docs/adrs/` as `ADR-XXXX-*.md`, using [`docs/adrs/MADR Template.md`](docs/adrs/MADR%20Template.md) (MADR format — status line required: proposed/rejected/accepted/deprecated/superseded).
- `*.log` files are gitignored — stray `*.log` files at repo root are scratch tool output, not something to preserve or commit.

## Agent skills

### Issue tracker

Issues and specs live as GitHub issues in `Layer9Labs/domain-endpoint-resolver`, via the `gh` CLI. See [`docs/agents/issue-tracker.md`](docs/agents/issue-tracker.md).

### Domain docs

Single-context layout: `CONTEXT.md` at the repo root (not yet created) + `docs/adrs/` for ADRs. See [`docs/agents/domain.md`](docs/agents/domain.md).
