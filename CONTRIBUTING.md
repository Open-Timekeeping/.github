# Contributing

Thanks for the interest. This file covers what is true across every Open Timekeeping repo. Each repo's specific conventions and commands live in its own `AGENTS.md`.

## Where the truth lives

1. **The repo's `AGENTS.md`** — repo-specific orientation, conventions, commands. Read this first.
2. **The `AGENTS.md` at the root of [open-timing-fabric](https://github.com/Open-Timekeeping/open-timing-fabric)** — the canonical project-level design spec. Three architectural planes, event schemas, redundancy model, implementation phases. Read it before proposing protocol or schema changes.
3. **The project hub in Notion** — current work, refined as Card / Conversation / Confirmation. Ask the maintainer for access if you need it.

Treat these as normative. If your code conflicts with what an `AGENTS.md` says, update the `AGENTS.md` in the same PR or open a discussion before writing the code.

## Before you open a PR

- Fork, branch, and open the PR against `main`.
- One conceptual change per PR. Refactors and feature work get separate PRs.
- Run the repo's local checks. CI mirrors them; failing CI blocks merge.
- For the fabric and other Rust crates: `cargo fmt --all --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`, `cargo build --workspace`.
- New behavior needs a test. Bug fixes need a regression test that fails without your change.
- If you touch the protocol or schemas, update `specs/` in the same PR.

## Code conventions (fabric and adapter repos)

- **u64 nanosecond timestamps** go through the `Ns` wrapper from `otf-time`. Never raw `u64`. JSON wire encoding is a string; the wrapper enforces it, and the rule is non-negotiable.
- **Stream records are opaque bytes** to the runtime. Schema validation happens at producer and consumer edges via the schema registry, not in the log.
- **The fabric core is domain-neutral.** No motorsport-specific terminology in fabric crates. Laps, cars, drivers, flags belong in domain applications, never here.
- **Records are immutable; corrections are amendments.** Do not add mutation paths to existing event types. Add an `*.amended.v1` variant that supersedes a prior offset.

## Adapter plugins

Real clock and decoder adapters live in their own repos, one per profile or hardware family. You do not need to host your plugin on this organization — anywhere is fine. What matters is that it conforms to the contract in `open-timing-fabric/specs/` and passes the conformance harness in `open-timing-fabric/conformance/`. We're happy to link community-maintained plugins from the org profile.

## Reporting problems

- Bugs and feature requests: open an issue on the relevant repo.
- Conduct concerns: see [CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md).
- Security vulnerabilities: see [SECURITY.md](./SECURITY.md). Please do not file public issues for security.

## Licensing

Every repo is Apache-2.0. By submitting a PR you assert you have the right to license your work under Apache-2.0 and agree to do so.

## Maintainer

Jeremy Willemse, on behalf of [Willemse & Company Technology Inc.](https://willemse.co). Email: jeremy@willemse.co.
