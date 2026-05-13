---
name: Bug report
about: Something is broken or behaves unexpectedly
title: ""
labels: bug
---

## What happened

<!-- Plain description of the bug. Don't speculate about the cause yet; describe what you observed. -->

## What you expected

<!-- What you thought should have happened, and why. -->

## How to reproduce

1.
2.
3.

<!-- Minimal repro is gold. If the bug needs a multi-node cluster, simulated adapters, or a particular clock profile to surface, say so. -->

## Environment

- Repo and version (commit, tag, or release):
- OS and architecture:
- Rust toolchain (if applicable):
- Deployment topology (single-node, hub, distributed nodes, replicated):
- Clock profile in use (e.g. `simulated`, `ntp`, `gnss_pps`, `ptp_hardware`):
- Decoder adapter in use (e.g. `simulated`, real hardware family):
- Anything else relevant:

## Event excerpts and logs

<!--
Paste the relevant logs and any event records involved. For event records,
include the FULL object with all provenance blocks (timestamping, clock,
observation, broker) — that's often where the bug becomes visible.
Redact secrets, credentials, and any private subject identifiers.
-->

```text

```

## Anything else

<!-- Hypotheses about cause, related issues, things you've already ruled out. -->
