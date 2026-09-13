# Domain Docs

How the engineering skills should consume this repo's domain documentation when exploring the codebase.

Layout: **single-context** (one crate, no monorepo/workspace signals).

## Before exploring, read these

- **`CONTEXT.md`** at the repo root, if it exists.
- **`docs/adrs/`**: read ADRs that touch the area you're about to work in.

Note: this repo's existing convention (see `AGENTS.md`/`CLAUDE.md`) is `docs/adrs/` (plural), using `docs/adrs/MADR Template.md`, numbered `ADR-XXXX`, with a required status line. Use that path and template, not the generic `docs/adr/` singular default.

If `CONTEXT.md` or any ADRs don't exist yet, **proceed silently**. Don't flag their absence; don't suggest creating them upfront. The `/domain-modeling` skill (reached via `/grill-with-docs` and `/improve-codebase-architecture`) creates them lazily when terms or decisions actually get resolved.

## File structure

```
/
├── CONTEXT.md              ← not yet created
├── docs/
│   └── adrs/
│       └── MADR Template.md
└── src/
```

## Use the glossary's vocabulary

When your output names a domain concept (in an issue title, a refactor proposal, a hypothesis, a test name), use the term as defined in `CONTEXT.md`. Don't drift to synonyms the glossary explicitly avoids.

If the concept you need isn't in the glossary yet, that's a signal: either you're inventing language the project doesn't use (reconsider) or there's a real gap (note it for `/domain-modeling`).

## Flag ADR conflicts

If your output contradicts an existing ADR, surface it explicitly rather than silently overriding:

> _Contradicts ADR-0007 (event-sourced orders), but worth reopening because…_
