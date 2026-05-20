<img src="otk-logo.svg" alt="Open Timekeeping" height="64">

# Open Timekeeping

**Open, vendor-neutral full-stack timing, hardware, firmware, adapters, timebases, runtime, apps, and conformance.**

Open Timekeeping is not just an integration framework. It is a full-stack timing system and ecosystem spanning:

- **Physical detectors** and **reference hardware**
- **Firmware** that runs natively on detector devices
- **Detector adapter implementations** (firmware, standalone producers, edge sidecars, simulators, replays)
- **Timebase / clock sync** as a first-class concept (GNSS-PPS, PTP, NTP, local, and honest about their differences)
- **Timing Runtime Nodes**, the deployable server processes that ingest, persist, and serve
- **Timing Core**, the timing-domain engine for detections to crossings to laps to sectors to results
- **APIs and apps** for operators, spectators, and integrators
- **Conformance**: anyone claiming compatibility can prove it

The first motivating domain is motorsport. The same primitives serve athletics, cycling, rowing, karting, RC racing, point-to-point timing, industrial checkpoints, lab timing, and similar contexts.

## The standard

The conceptual model, terminology, contracts, and operating principles live in [**spec**](https://github.com/Open-Timekeeping/spec). Read [`spec/architecture.md`](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md) first.

## Architecture: ports and adapters

The server-side codebase follows the [ports-and-adapters (hexagonal) architecture](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md). Repos fall into four groups:

```
otk-core/        workspace: event-model, protocol, timing-core,
                            port-in-ingest, port-out-event-log

adapter-ingest-tcp/         implements port-in-ingest over TCP
adapter-event-log-segment/  implements port-out-event-log with segment files

timing-node/     reference server composition root (binary: otk-node)

otk-sdk/         producer + consumer SDK; no server deps

producer-simulated/   reference simulated producer (binary: otk-simulator)
```

Dependency rules:

| Layer | May depend on |
|---|---|
| `server/core/*` | nothing (no OTK deps) |
| `server/ports/*` | `server/core/*` only |
| `server/adapters/*` | `server/ports/*`, `server/core/*` |
| `server/app/*` | everything in `server/` |
| `sdk/otk-sdk` | `event-model`, `protocol` (optional) only |
| `producers/*` | `sdk/otk-sdk` only |
| `consumers/*` | `sdk/otk-sdk` only |

The timing node is the composition root for the server. Producer and consumer apps are their own composition roots. None of the three depend on each other.

See [repos.md](repos.md) for the full repo inventory and dependency map.

## The runtime

A **Timing Runtime Node** (binary `otk-node`, repo [`timing-node`](https://github.com/Open-Timekeeping/timing-node)) is the central server process in an Open Timekeeping deployment. A real venue runs several things at once: one or more `otk-node` instances, each hosting one or more ingest listeners; the operator and spectator app for session management, scoring, results, and leaderboards; the diagnostics app for detector health and event-flow inspection; and standalone producer processes (using [`otk-sdk`](https://github.com/Open-Timekeeping/otk-sdk)) on edge gateways or in firmware.

What makes the node the central one is that it owns the canonical event log, mediates between producers and consumers, holds the live registries, and is the addressable endpoint that apps and operator tools talk to.

### What the node does

```text
                       Timing Runtime Node (otk-node)
                       ================================

  external producers                    in-process
  (firmware, edge gateways,             plugins (future)
   simulators, replays)
       |                                       |
       |  OTK protocol over a transport        |  plugin-api calls
       |  (TCP, serial, USB CDC, ...)          |
       v                                       v
  +-------------------------+        +-------------------------+
  |  adapter-ingest-tcp     |        |  Plugin host (future)   |
  |  (framing + handshake   |        |  - load / start / stop  |
  |   encapsulated here;    |        |  - lifecycle + reload   |
  |   typed OtkEvent out)   |        |                         |
  +-----------+-------------+        +------------+------------+
              \                                  /
               \   typed OtkEvent               /
                v                              v
          +----------------------------------------+
          |   NodePipeline (port-in-ingest user)   |
          |   - appends to event log               |
          |   - feeds timing-core                  |
          +------------------+---------------------+
                             |
                             v
          +----------------------------------+
          |  adapter-event-log-segment       |
          |  (implements port-out-event-log) |
          +------------------+---------------+
                             |
          +------------------+------------------+
          |                  |                  |
          v                  v                  v
    timing-core          API server          Registries
    (crossings,          (REST/SSE;          (detector
     laps, results)       reads log)          health, ...)
```

The node ingests four kinds of first-class events, all defined in [`event-model`](https://github.com/Open-Timekeeping/otk-core/tree/main/event-model):

- **Detection events** from any detector adapter.
- **Detector health events** that say whether a detector is producing trustworthy data.
- **Timebase status events** that say whether the clocks are locked, in holdover, drifting, or free-running.
- **Adapter metadata events** announcing identity and capabilities at startup and on config change.

### What it serves

- **Live subscribe** for apps and operator tools, streaming canonical events plus crossing results.
- **Range reads and replay** for any consumer that wants history.
- **Registry queries** for detector and timebase state.
- **Projection queries** computed by [`timing-core`](https://github.com/Open-Timekeeping/otk-core/tree/main/timing-core).

### Honest provenance, immutable log, amendments

Every event carries the timestamping method, timebase reference, sync state at the moment of capture, and declared uncertainty. The log is append-only; corrections are emitted as explicit amendment events that supersede a prior offset, never as in-place edits.

### Deployment shapes

Six canonical deployment shapes are documented in [`spec/topologies.md`](https://github.com/Open-Timekeeping/spec/blob/main/topologies.md):

- **Native producer.** Firmware on a device speaks OTK directly over a supported transport. No edge adapter required.
- **Edge adapter.** A standalone process on a Pi or mini-PC normalizes a raw device into canonical events and ships them to a node via `otk-sdk` (producer feature).
- **Hub plugin.** A raw device cables into the host running the node; the adapter is loaded as a plugin (future).
- **Local same-host adapters.** Adapter processes on the same host connect over a Unix socket binding (future).
- **One node per detector stack.** Each timing point has its own node; aggregation upstream is optional.
- **Central hub.** Many producers feed one central node that owns all storage and APIs.

### What a node does not do

- It does **not** parse vendor-specific raw device protocols. That is the adapter's job.
- It does **not** synthesize timestamps it didn't receive honestly.
- It does **not** apply policy to decide whether an event is "official." Policy is a separate layer above the runtime.
- It does **not** present a UI. Apps consume its APIs.

## The SDK

[`otk-sdk`](https://github.com/Open-Timekeeping/otk-sdk) is the single SDK crate for producers and consumers. It has no server-side dependencies.

```toml
# Consumer app (default: client feature for HTTP/SSE reads):
otk-sdk = { git = "https://github.com/Open-Timekeeping/otk-sdk" }

# Producer (TCP):
otk-sdk = { git = "https://github.com/Open-Timekeeping/otk-sdk", default-features = false, features = ["producer"] }
```

Transport is runtime configuration, not a compile-time choice:

```rust
Producer::connect(Transport::Tcp("127.0.0.1:8463".parse()?), config).await?
```

The SDK re-exports `event-model` types so dependents never need a direct `event-model` dep.

## Networking models

A detector adapter is a logical role, not a deployment constraint. The same `DetectorAdapter` contract (in `otk-sdk` producer feature) is satisfied by:

- **Native firmware** on a detector device that speaks OTK directly.
- **External adapter process** on an edge gateway using `otk-sdk` (producer feature).
- **Plugin** loaded directly into `timing-node` (future `plugin-api`).
- **Simulator** or **CSV replay** for development and conformance.

## Operating principles

- **Honest provenance over false precision.** If timestamp resolution, uncertainty, or sync state is unknown, report it. Never pretend nanosecond representation means nanosecond accuracy.
- **Records are immutable. Corrections are amendments.** A late hit doesn't edit a crossing in place; it emits an amendment event that supersedes a prior offset.
- **Detectors observe. Timing-core interprets. Apps present.**
- **Mechanism vs policy.** The runtime records what happened and how trustworthy it is. Whether an event is acceptable as official is a separate policy decision.
- **AI is advisory, never authoritative.** Results, penalties, and flags are not AI-owned.
- **Domain neutrality at the timing layer.** Subjects, timing points, detections, crossings; not laps, drivers, races, or flags. Domain concepts live in the app layer.

## Status

**Early development.** Architecture is settling; protocol and schema details are explicitly tracked as open questions in [`spec/open-questions.md`](https://github.com/Open-Timekeeping/spec/blob/main/open-questions.md). No released versions yet.

## Compatibility

An implementation claims Open Timekeeping compatibility by satisfying the relevant contracts and passing [`conformance`](https://github.com/Open-Timekeeping/conformance) with the canonical [`conformance-fixtures`](https://github.com/Open-Timekeeping/conformance-fixtures). See [`spec/compatibility.md`](https://github.com/Open-Timekeeping/spec/blob/main/compatibility.md).

## Get involved

- Read [`spec/architecture.md`](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md) first.
- Each repo's `README.md` defines its boundary, dependencies, and what does and does not belong there.
- Issues and discussions are open on every repo.
- Adapter, timebase, storage, and target ports are first-class extension points. You don't need to host your implementation on this organization. What matters is the contract and a passing conformance run.

## License

Apache-2.0 across software repos. The reference-hardware repo's design-file license may shift to CERN-OHL-S or TAPR OHL depending on community feedback.

## Maintainer

The Open Timekeeping organization is maintained by Jeremy Willemse on behalf of **[Willemse & Company Technology Inc.](https://willemse.co)** Contributions are welcome.
