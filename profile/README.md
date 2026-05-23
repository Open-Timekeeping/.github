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

## Where the code lives

Everything that ships at v0 lives in one repository: [**`open-timekeeping`**](https://github.com/Open-Timekeeping/open-timekeeping). It is a single Cargo workspace containing the conceptual spec, the OTK protocol stack crates, port traits, adapters, the SDK, the reference producer, the runtime node, and the conformance suite.

For the conceptual model, terminology, contracts, and operating principles, start with [`spec/architecture.md`](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/spec/architecture.md). For workspace orientation (how to build, where each crate lives, development conventions) see the repo's [`AGENTS.md`](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/AGENTS.md) and top-level [`README.md`](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/README.md).

## Architecture: ports and adapters

The runtime follows the [ports-and-adapters (hexagonal) architecture](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/spec/architecture.md). Inside the single Cargo workspace, crates fall into the following layers:

```
spec/                       Conceptual standard (markdown, normative)

event-model/                Canonical event types and identifiers
otk-protocol/               Wire protocol DTOs: envelope, handshake, messages
frame-codec/                Frame encode/decode (length-prefix stream + COBS serial)
ingest-protocol/            Transport-agnostic server-side handshake / dispatch
port-in-ingest/             Inbound port trait: EventIngestPort, IngestSession
port-out-event-log/         Outbound port trait: EventLog, LogSubscription
otk-contracts/              Detector adapter and timebase trait contracts
timing-core/                Detection-to-crossing engine (a library, not a server)

adapter-ingest-tcp/         TCP transport binding
adapter-ingest-unix-socket/ Unix-socket transport binding (cfg(unix))
adapter-event-log-segment/  Append-only segment-file storage backend

otk-sdk/                    Producer + consumer SDK
producer-simulated/         Reference producer (binary: otk-simulator)

timing-node/                Timing Runtime Node (binary: otk-node)

conformance/                Test suite verifying implementations against the contracts
conformance-fixtures/       Test data corpus
```

Dependency rules (governing OTK-internal deps; third-party Cargo deps are unrestricted):

| Layer | May depend on (OTK crates) |
|---|---|
| Protocol-layer crates (`event-model`, `otk-protocol`, `frame-codec`) | only each other, in stack order |
| Port traits (`port-in-ingest`, `port-out-event-log`) | `event-model` only |
| Contract crate (`otk-contracts`) | `event-model` only |
| Adapter crates (`adapter-ingest-*`, `adapter-event-log-segment`) | the port trait they implement + protocol layer |
| `otk-sdk` | `event-model`, `otk-protocol` (producer feature), `otk-contracts` (producer feature) |
| `producer-simulated` | `otk-sdk` |
| `timing-node` | everything above; it is the composition root |
| `conformance` | contracts + protocol layer; never concrete adapters |

`timing-node` is the composition root for the runtime. Producer and consumer apps are their own composition roots. None of the three depend on each other at the trait level; `timing-node` Cargo-depends on the concrete adapters it instantiates, the pipeline logic inside it does not.

## The runtime

A **Timing Runtime Node** (binary `otk-node`, crate [`timing-node`](https://github.com/Open-Timekeeping/open-timekeeping/tree/main/timing-node)) is the central server process in an Open Timekeeping deployment. A real venue runs several things at once: one or more `otk-node` instances, each hosting one or more ingest listeners; the operator and spectator app for session management, scoring, results, and leaderboards; the diagnostics app for detector health and event-flow inspection; and standalone producer processes (using [`otk-sdk`](https://github.com/Open-Timekeeping/open-timekeeping/tree/main/otk-sdk)) on edge gateways or in firmware.

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
       |  (TCP, Unix socket; serial /          |
       |   USB CDC planned)                    |
       v                                       v
  +-------------------------+        +-------------------------+
  |  adapter-ingest-*       |        |  Plugin host (future)   |
  |  (framing + handshake   |        |  - load / start / stop  |
  |   encapsulated here;    |        |  - lifecycle + reload   |
  |   typed OtkEvent out)   |        |                         |
  +-----------+-------------+        +------------+------------+
              \                                  /
               \   typed OtkEvent               /
                v                              v
          +----------------------------------------+
          |   NodePipeline (port-in-ingest user)   |
          |   - sequence-gate                      |
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

The node ingests four kinds of first-class events, all defined in [`event-model`](https://github.com/Open-Timekeeping/open-timekeeping/tree/main/event-model):

- **Detection events** from any detector adapter.
- **Detector health events** that say whether a detector is producing trustworthy data.
- **Timebase status events** that say whether the clocks are locked, in holdover, drifting, or free-running.
- **Adapter metadata events** announcing identity and capabilities at startup and on config change.

### What it serves

- **Live subscribe** for apps and operator tools, streaming canonical events plus crossing results.
- **Range reads and replay** for any consumer that wants history.
- **Registry queries** for detector and timebase state.
- **Projection queries** computed by [`timing-core`](https://github.com/Open-Timekeeping/open-timekeeping/tree/main/timing-core).

### Honest provenance, immutable log, amendments

Every event carries the timestamping method, timebase reference, sync state at the moment of capture, and declared uncertainty. The log is append-only; corrections are emitted as explicit amendment events that supersede a prior offset, never as in-place edits.

### Deployment shapes

Six canonical deployment shapes are documented in [`spec/topologies.md`](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/spec/topologies.md):

- **Native producer.** Firmware on a device speaks OTK directly over a supported transport. No edge adapter required.
- **Edge adapter.** A standalone process on a Pi or mini-PC normalizes a raw device into canonical events and ships them to a node via `otk-sdk` (producer feature).
- **Hub plugin.** A raw device cables into the host running the node; the adapter is loaded as a plugin (future).
- **Local same-host adapters.** Adapter processes on the same host connect over a Unix socket binding.
- **One node per detector stack.** Each timing point has its own node; aggregation upstream is optional.
- **Central hub.** Many producers feed one central node that owns all storage and APIs.

### What a node does not do

- It does **not** parse vendor-specific raw device protocols. That is the adapter's job.
- It does **not** synthesize timestamps it didn't receive honestly.
- It does **not** apply policy to decide whether an event is "official." Policy is a separate layer above the runtime.
- It does **not** present a UI. Apps consume its APIs.

## The SDK

[`otk-sdk`](https://github.com/Open-Timekeeping/open-timekeeping/tree/main/otk-sdk) is the single SDK crate for producers and consumers. It has no dependency on server port contracts, adapters, or the timing-node binary; its only intra-workspace deps are the shared types `event-model` (always) and `otk-protocol` (producer feature only).

Inside the workspace, depend on it via a path dep and pick the feature set that matches the role:

```toml
# Consumer (default features include `client` for HTTP/SSE reads):
otk-sdk = { path = "../otk-sdk" }

# Producer (TCP) - exclude the client feature for producer-only processes:
otk-sdk = { path = "../otk-sdk", default-features = false, features = ["producer"] }
```

External consumers depending on a published version will substitute `path = "../otk-sdk"` with `version = "x.y"` once `otk-sdk` lands on crates.io.

Transport is runtime configuration, not a compile-time choice:

```rust
Producer::connect(Transport::Tcp("127.0.0.1:8463".parse()?), config).await?
```

The SDK re-exports `event-model` types so dependents never need a direct `event-model` dep.

## Networking models

A detector adapter is a logical role, not a deployment constraint. The same `DetectorAdapter` contract (in [`otk-contracts`](https://github.com/Open-Timekeeping/open-timekeeping/tree/main/otk-contracts), re-exported by `otk-sdk` producer feature) is satisfied by:

- **Native firmware** on a detector device that speaks OTK directly.
- **External adapter process** on an edge gateway using `otk-sdk` (producer feature).
- **Plugin** loaded directly into `timing-node` (future).
- **Simulator** or **CSV replay** for development and conformance.

## Operating principles

- **Honest provenance over false precision.** If timestamp resolution, uncertainty, or sync state is unknown, report it. Never pretend nanosecond representation means nanosecond accuracy.
- **Records are immutable. Corrections are amendments.** A late hit doesn't edit a crossing in place; it emits an amendment event that supersedes a prior offset.
- **Detectors observe. Timing-core interprets. Apps present.**
- **Mechanism vs policy.** The runtime records what happened and how trustworthy it is. Whether an event is acceptable as official is a separate policy decision.
- **AI is advisory, never authoritative.** Results, penalties, and flags are not AI-owned.
- **Domain neutrality at the timing layer.** Subjects, timing points, detections, crossings; not laps, drivers, races, or flags. Domain concepts live in the app layer.

## Status

**Early development.** Architecture is settling; protocol and schema details are explicitly tracked as open questions in [`spec/open-questions.md`](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/spec/open-questions.md). No released versions yet.

## Compatibility

An implementation claims Open Timekeeping compatibility by satisfying the relevant contracts and passing the [`conformance`](https://github.com/Open-Timekeeping/open-timekeeping/tree/main/conformance) suite. See [`spec/compatibility.md`](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/spec/compatibility.md).

## Get involved

- Read [`spec/architecture.md`](https://github.com/Open-Timekeeping/open-timekeeping/blob/main/spec/architecture.md) first.
- Each crate's `README.md` inside the workspace defines its boundary, dependencies, and what does and does not belong there.
- Issues and discussions are open on the [`open-timekeeping`](https://github.com/Open-Timekeeping/open-timekeeping) repo.
- Adapter, timebase, storage, and target ports are first-class extension points. Third-party detector adapters can compile against [`otk-contracts`](https://github.com/Open-Timekeeping/open-timekeeping/tree/main/otk-contracts) without inheriting the runtime's transport stack.

## License

Apache-2.0 across the workspace. Future hardware design files may ship under CERN-OHL-S or TAPR OHL depending on community feedback.

## Maintainer

The Open Timekeeping organization is maintained by Jeremy Willemse on behalf of **[Willemse & Company Technology Inc.](https://willemse.co)** Contributions are welcome.
