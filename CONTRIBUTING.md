# Contributing

Thanks for the interest. This file covers what is true across every Open Timekeeping repo. Each repo's specific conventions and commands live in its own `AGENTS.md`.

## Where the truth lives

1. **The repo's `README.md` (and `AGENTS.md` if present)**, repo-specific orientation, conventions, commands. Read this first.
2. **The design spec in [spec/](https://github.com/Open-Timekeeping/spec)**, start with [`spec/architecture.md`](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md). It covers the conceptual model, the OTK Protocol stack (Event Model, Wire Protocol, Frame Codec, Transport Binding), topologies, compatibility, and the operating principles. Read the relevant `spec/` file before proposing protocol, schema, or transport changes.
3. **The project hub in Notion**, current work, refined as Card / Conversation / Confirmation. Ask the maintainer for access if you need it.

Treat these as normative. If your code conflicts with what an `AGENTS.md` says, update the `AGENTS.md` in the same PR or open a discussion before writing the code.

## Before you open a PR

- Fork, branch, and open the PR against `main`.
- One conceptual change per PR. Refactors and feature work get separate PRs.
- Run the repo's local checks. CI mirrors them; failing CI blocks merge.
- For Rust crates: `cargo fmt --all --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`, `cargo build --workspace`.
- New behavior needs a test. Bug fixes need a regression test that fails without your change.
- If you touch the protocol, schemas, or transport bindings, update [`spec/`](https://github.com/Open-Timekeeping/spec) and the relevant layer repo in the same PR.

## Code conventions

- **Honest provenance over false precision.** Every event carries timestamping method, timebase reference, sync state, and uncertainty. If something is unknown, say so. Never fake precision.
- **Records are immutable; corrections are amendments.** Do not add mutation paths to existing event types. Add an `*.amended.v1` variant that supersedes a prior offset.
- **Domain neutrality in the timing layer.** The runtime node, [`timing-core`](https://github.com/Open-Timekeeping/timing-core), and the adapter / timebase contracts know about subjects, timing points, detections, and crossings. Laps, cars, drivers, flags belong in the app layer, never in the timing layer.
- **OTK Protocol is transport-independent.** Do not say detectors must have TCP stacks; do not say OTK equals TCP; do not say HTTP is the detector ingest data plane. See [`spec/architecture.md`](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md).

## Adapter and timebase implementations

Real detector adapters and timebases live in their own repos, one per profile or hardware family. You do not need to host your implementation on this organization, anywhere is fine. What matters is that it conforms to the relevant contract ([`detector-adapter-api`](https://github.com/Open-Timekeeping/detector-adapter-api), [`timebase-api`](https://github.com/Open-Timekeeping/timebase-api), or [`plugin-api`](https://github.com/Open-Timekeeping/plugin-api)) and passes the [`conformance`](https://github.com/Open-Timekeeping/conformance) suite. We're happy to link community-maintained implementations from the org profile.

## Reporting problems

- Bugs and feature requests: open an issue on the relevant repo.
- Conduct concerns: see [CODE_OF_CONDUCT.md](./CODE_OF_CONDUCT.md).
- Security vulnerabilities: see [SECURITY.md](./SECURITY.md). Please do not file public issues for security.

## Licensing

Every repo is Apache-2.0. By submitting a PR you assert you have the right to license your work under Apache-2.0 and agree to do so.

## Maintainer

Jeremy Willemse, on behalf of [Willemse & Company Technology Inc.](https://willemse.co). Email: jeremy@willemse.co.
