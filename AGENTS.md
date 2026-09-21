# AGENTS.md

## What this is

Rust implementation of the **Domain Endpoint Resolver (DER)** — a distributed mesh node for Layer 9 Labs' Event Network Architecture (patent-pending US-20240129230-A1). Event sinks register their domain + endpoint + health-check URI with the mesh; event sources query mesh nodes to resolve domain endpoints; mesh nodes periodically health-check and evict stale entries.

## Source of truth (docs/)

Read these before making design decisions — the repo is currently docs-heavy, code-light:

- `docs/context/Technical Brief for Event Network Architecture.md` — authoritative architecture. DER mesh described in §9–10: HyParView gossip membership protocol, gRPC registration interface, distributed routing cache, sidecar deployment model.
- `docs/Domain Endpoint Resolver Product Requirements Document (PRD).md` — behavior & constraints.

## Hard constraints (from PRD §4)
- No frontend/UI, no persistent database, no authentication logic, no dynamic config — static environment variables only.
- Error handling per PRD §6: malformed payloads → structured warning + drop (never panic); downstream availability → bounded exponential backoff retry.

## Build/test

Rust binary crate. Before `cargo test`/`cargo build` works, a manifest must exist; create it if asked to build (package name: `domain-endpoint-resolver`).

## Conventions

- ADRs live in `docs/adrs/` using `docs/adrs/MADR Template.md` (MADR format, numbered `ADR-XXXX`, status line required). Record any significant design decision there.
- `*.log` files are gitignored — the stray `*.log` at repo root is scratch OpenCode/IDE output, not something to keep.
