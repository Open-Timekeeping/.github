# Security policy

## Reporting a vulnerability

If you think you have found a security vulnerability in any Open Timekeeping repo, report it privately. **Do not open a public issue.**

**Email**: jeremy@willemse.co
**Subject prefix**: `[SECURITY]`

Include:

- The affected repo and version (commit, tag, or release).
- A description of the issue and its impact.
- A way to reproduce it. A minimal example is ideal; a partial walkthrough is fine.
- Any mitigations you have already identified.

You will get an acknowledgment within 72 hours and a substantive response within seven days. Disclosure timing is negotiated case-by-case; we will work with you in good faith and we will not go after researchers who follow this process.

## Scope

Every repository under <https://github.com/Open-Timekeeping>. Community-maintained adapter plugins hosted elsewhere are out of scope here — report those to their maintainers — but if the issue affects the fabric protocol itself or anything in `open-timing-fabric/specs/`, send it here too.

## What counts as a security issue

The categories that matter most for a timing fabric:

- **Authentication and authorization.** Anything that lets a caller act without proper credentials or escalate privileges.
- **Integrity of timing records.** Anything that allows silent mutation, undetected replay, or forging of timing events. This is the most critical class for us, because the whole project's value rests on records being trustworthy.
- **Schema validation bypass** at the producer or consumer edge, letting malformed events into the data plane unflagged.
- **Cluster consensus and metadata replication.** Paths that could induce split-brain, silent state divergence, or unauthorized writes to control-plane state.
- **Denial of service** against running nodes or against schema-registry / consumer-group coordinators.
- **Supply chain.** Compromise paths through dependencies, build pipelines, or release artifacts.

Functional bugs, performance regressions, and design disagreements are not security issues. File those as regular issues.

## Public advisories

When a fix ships, we publish a security advisory on the affected repo describing the issue, the fix, the affected versions, and a credit line for the reporter (unless they prefer anonymity).
