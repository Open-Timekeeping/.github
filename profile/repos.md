# Repo family

As of May 2026 the Open Timekeeping org has consolidated into **one repository**: [`open-timekeeping`](https://github.com/Open-Timekeeping/open-timekeeping), a single Cargo workspace containing the spec, the OTK protocol layer crates, port traits, adapters, SDK, reference producer, runtime node, and conformance suite.

The canonical inventory of crates and the dependency map inside the workspace lives in the consolidated repo's own [`AGENTS.md`](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/AGENTS.md) and [`README.md`](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/README.md). Start with [`spec/architecture.md`](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/spec/architecture.md) for the conceptual model.

The only other repo in the org is [`.github`](https://github.com/Open-Timekeeping/.github), which holds this org profile and community-health files (contributing, code of conduct, security policy, issue/PR templates).

## History

Before consolidation, the codebase lived across ten sibling repos (`otk-core`, `adapter-ingest-tcp`, `adapter-event-log-segment`, `adapter-ingest-unix-socket`, `otk-sdk`, `producer-simulated`, `timing-node`, `conformance`, `spec`, plus this `.github` repo). Cross-cutting changes required coordinated PRs across multiple repos; spec edits and the code that satisfied them lived in separate review threads. Folding everything into one workspace removed that friction. The pre-consolidation per-commit history is preserved inside the new repo's `git log` for the `otk-core` lineage; the other eight repos were squash-imported with the source SHA captured in each `consolidate: import <src>` commit message.

Future hardware, firmware, and frontend work that requires a different toolchain (TypeScript apps, embedded-Rust firmware with non-overlapping `Cargo.toml` constraints) will spin out as new sibling repos when work actually begins, not before.
