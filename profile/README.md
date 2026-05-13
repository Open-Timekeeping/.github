# Open Timekeeping

**Open, vendor-neutral infrastructure for synchronized real-world timing events.**

Open Timekeeping develops open-source infrastructure, specifications, and reference implementations for **observing subjects at analog detection points** — with precise hardware-grade timestamping when the components allow it, and honest software timestamping (clearly marked as such) when they don't. The resulting normalized event streams are stored, replayed, and interpreted by downstream domain applications.

Motorsport is the first motivating domain. The same primitives are designed to serve athletics, cycling, rowing, karting, RC racing, point-to-point timing, industrial checkpoints, lab timing, and any other context with the same shape: physical event at the edge → trustworthy timestamp → replayable stream → interpretation in a domain application.

The fabric is designed for serious, production-grade timing — redundant decoders with hot failover, replicated metadata, honest cross-node clock provenance, and stream-based replay are first-class capabilities of the architecture, not bolt-ons.

## Projects

| Project | What it is |
|---|---|
| **[open-timing-fabric](https://github.com/Open-Timekeeping/open-timing-fabric)** | Core infrastructure. A Kafka-shaped, timing-specific event fabric. Defines the runtime, event schemas, decoder/clock adapter contracts, and the conformance harness. Domain-neutral. |
| **open-timing-client-\*** | Per-language backend client SDKs (Rust first, then Node/Go/Python). Browsers never speak the fabric protocol directly — they talk to an app server that holds the client. |
| **open-timing-clock-\*** | Clock adapter plugins, one repo per profile (`gnss-pps`, `ptp-hardware`, `white-rabbit`, etc.). Platform- and hardware-specific sync code stays out of the fabric core. |
| **open-timing-decoder-\*** | Decoder adapter plugins, one repo per hardware family (`loop`, `rfid`, `beam-gate`, etc.). Vendor-specific code is isolated and licensed independently when needed. |
| **open-timekeeper** | Reference timing application that consumes fabric streams — app server, projection engine, frontend, AI insights. |
| **open-race-ops** | Motorsport-specific application built on the fabric — race control, scoring, marshal and steward workflows. |

The fabric is built first. Everything else is created on demand as the work to fill it lands.

## Three architectural planes

```text
Time Plane         clocks, sync, per-event provenance, cross-node skew
                   → makes timestamps trustworthy

Control Plane      cluster membership, stream ownership, schema registry,
                   consumer groups, Raft-replicated metadata, leadership
                   → tells everyone what the system is

Data Plane         append, store, replay, subscribe; single-writer logs;
                   format-agnostic bytes; HTTP / SSE / WebSocket
                   → moves and replays timing events
```

The planes are independent at the protocol level. A time-plane clock-status update never blocks a data-plane append; a control-plane failover decision surfaces inline in data-plane subscriptions as a control event.

## Design principles

- **Observe at the edge. Timestamp honestly.** The decoder adapter assigns the event timestamp as close to the physical event as the hardware allows. The broker never invents official timing truth.
- **Honest provenance over false precision.** Every event carries clock state and observation quality. If the hardware can't deliver a capability, the field is marked `null`, `unknown`, or degraded — never silently faked.
- **Single-writer streams; redundancy via logical streams.** Each physical decoder writes its own append-only log. Redundancy is a logical-stream alias over one or more physical streams with hot-failover. There is no per-event consensus.
- **Records are immutable. Corrections are amendments.** A late hit doesn't edit a passing in place — it emits a `*.amended.v1` event that supersedes a prior offset. The original record always survives.
- **Nanosecond precision is non-negotiable.** `u64` ns timestamps go on the JSON wire as strings, native as `u64` in binary formats. No 53-bit truncation, ever.
- **Interpret in the app layer.** The fabric core knows about subjects, timing points, observations, and quality. It does not know about laps, cars, drivers, races, or flags. Those concepts live in domain applications, not in the fabric.
- **AI is advisory and auditable, never authoritative.** AI may summarize, flag anomalies, draft reports. It may not be the authority for results, penalties, or flags.

The full design spec lives in [`AGENTS.md` at the root of open-timing-fabric](https://github.com/Open-Timekeeping/open-timing-fabric/blob/main/AGENTS.md).

## Status

Early development. The architecture is complete on paper; implementation phases are additive within the same protocol — no breaking changes between phases.

| Phase | Description |
|---|---|
| **1** | Single-node fabric with full protocol surface *(in progress)* |
| 2 | Multi-node cluster with Raft-replicated metadata |
| 3 | Redundant decoders + hot failover |
| 4 | Consumer groups |
| 5 | Replication followers + binary formats |
| 6 | First real hardware adapter |

Protocol and schemas are stabilizing. No released version yet.

## Get involved

- Each repo's `AGENTS.md` is the orientation document for both human contributors and AI coding assistants. Read it first.
- Issues and discussions are open on every repo.
- **Adapter plugins are first-class.** If you want to bring a new clock profile or decoder hardware family into the ecosystem, that's a new `open-timing-{clock,decoder}-<name>` repo conforming to the contract defined in `open-timing-fabric/specs/`. The fabric repo ships the conformance harness; your plugin lives anywhere — on this org or your own.

## License

Apache-2.0 across all repos. The patent grant matters for protocol-level work.

## Maintainer

The Open Timekeeping organization is maintained by Jeremy Willemse on behalf of **[Willemse & Company Technology Inc.](https://willemse.co)** Contributions are welcome from anyone; see [`CONTRIBUTING.md`](https://github.com/Open-Timekeeping/.github/blob/main/CONTRIBUTING.md).

---

```text
Observe at the edge.
Timestamp honestly.
Store as an ordered stream.
Replay deterministically.
Fail over without losing the thread.
Interpret in the app layer.
Explain with AI, but do not let AI own truth.
```
