<img src="otk-logo.svg" alt="Open Timekeeping" height="64">

# Open Timekeeping

**Open, vendor-neutral full-stack timing, hardware, firmware, adapters, timebases, runtime, apps, and conformance.**

Open Timekeeping is not just an integration framework. It is a full-stack timing system / ecosystem spanning:

- **Physical detectors** and **reference hardware**
- **Firmware** that runs natively on detector devices
- **Detector adapter implementations** (firmware, standalone producers, edge sidecars, runtime-node plugins, simulators, replay)
- **Timebase / clock sync** as a first-class concept (GNSS-PPS, PTP, NTP, local, and honest about their differences)
- **Timing Runtime Nodes**, the deployable server processes that ingest, persist, and serve
- **Timing Core**, the timing-domain engine for detections → crossings → laps → sectors → results
- **APIs and apps** for operators, spectators, and integrators
- **Conformance**, anyone claiming compatibility can prove it

The first motivating domain is motorsport. The same primitives serve athletics, cycling, rowing, karting, RC racing, point-to-point timing, industrial checkpoints, lab timing, and similar contexts.

## The standard

The conceptual model, terminology, contracts, and operating principles, is in [**spec**](https://github.com/Open-Timekeeping/spec). Read [`spec/architecture.md`](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md) first.

## The contracts every implementation meets

The **OTK Protocol** is the transport-independent wire protocol used by Open Timekeeping components to exchange canonical timing messages. It is split into four layers, each with its own contract and repo, plus the role-level contracts on top.

| Contract | Defined in | Implemented by |
|---|---|---|
| **Event Model** (canonical events) | [event-model](https://github.com/Open-Timekeeping/event-model) | Every event in the system |
| **Wire Protocol** (OTK message envelope) | [wire-protocol](https://github.com/Open-Timekeeping/wire-protocol) | Producers, runtime nodes |
| **Frame Codec** (encode/decode) | [frame-codec](https://github.com/Open-Timekeeping/frame-codec), [embedded-wire](https://github.com/Open-Timekeeping/embedded-wire) (firmware) | Anything that sends or receives OTK frames |
| **Transport Binding** (link-specific) | [transport-api](https://github.com/Open-Timekeeping/transport-api) + [transport-tcp](https://github.com/Open-Timekeeping/transport-tcp), [transport-serial](https://github.com/Open-Timekeeping/transport-serial), [transport-usb-cdc](https://github.com/Open-Timekeeping/transport-usb-cdc), [transport-unix-socket](https://github.com/Open-Timekeeping/transport-unix-socket) | Producers and runtime nodes, per chosen link |
| **Detector adapter** | [detector-adapter-api](https://github.com/Open-Timekeeping/detector-adapter-api) | All detector event sources (firmware, external processes, plugins, simulators, replays) |
| **Timebase** | [timebase-api](https://github.com/Open-Timekeeping/timebase-api) | Every clock-source implementation |

Plus [`storage-api`](https://github.com/Open-Timekeeping/storage-api) for persistence backends and [`plugin-api`](https://github.com/Open-Timekeeping/plugin-api) for in-process modules loaded by a runtime node. [`otk-ingest-client`](https://github.com/Open-Timekeeping/otk-ingest-client) is a convenience library for external producers, sitting on top of `wire-protocol` + `frame-codec` + a transport binding. It is not a required runtime process.

## The runtime

A **Timing Runtime Node** (binary `otk-node`, repo [`timing-node`](https://github.com/Open-Timekeeping/timing-node)) is the central server process in an Open Timekeeping deployment. A real venue runs several things at once: one or more `otk-node` instances, each hosting one or more ingest listeners (TCP, USB CDC, Unix socket, ...); the operator and spectator app ([`app-live-timing`](https://github.com/Open-Timekeeping/app-live-timing)) for session management, scoring, results, leaderboards, and displays; the diagnostics app ([`app-diagnostics`](https://github.com/Open-Timekeeping/app-diagnostics)) for detector health, timebase status, and event-flow inspection; possibly standalone detector adapter processes (using [`otk-ingest-client`](https://github.com/Open-Timekeeping/otk-ingest-client) for convenience, or building directly on the OTK Protocol stack) on edge gateways; and firmware running on native detector devices that emit OTK frames directly.

What makes the node the *central* one is that it owns the canonical event log, mediates between producers and consumers, holds the live registries, and is the addressable endpoint that the apps and operator tools talk to. The contracts (`event-model`, `wire-protocol`, `detector-adapter-api`, `timebase-api`, `plugin-api`, `storage-api`, `api-model`) and shared libraries (`detector-adapter-common`, `timebase-common`, `timing-core`) are not separate processes; they are compiled into whichever process they belong in. In deployment shape the node is broker-like: it is the addressable hub that everything else either feeds into or reads from. (We avoid calling it "the broker" in normative docs; the term is here only as a comparison.)

### What the node does

```text
                       Timing Runtime Node (otk-node)
                       =================================

  external producers                    in-process
  (firmware speaking OTK,               plugins
   edge gateways, simulators,           (adapters / timebases /
   replays)                              storage / exports)
       |                                       |
       |  OTK frames over a chosen             |  plugin-api calls
       |  transport binding                    |
       v                                       v
  +-------------------------+        +-------------------------+
  |  Ingest listener(s)     |        |  Plugin host            |
  |  - TCP, USB CDC, Unix   |        |  - load / start / stop  |
  |    socket, ...          |        |  - lifecycle + reload   |
  |  - handshake + register |        |  - in-process channels  |
  |  - schema validation    |        |                         |
  +-----------+-------------+        +------------+------------+
              \                                  /
               \   (canonical events,           /
                \   detector health,           /
                 \  timebase status,          /
                  \ adapter metadata)        /
                   v                        v
                +-----------------------------+
                |    Event log / stream       |
                |    (storage-api backed)     |
                +--------------+--------------+
                               |
            +------------------+------------------+--------------+
            |                  |                  |              |
            v                  v                  v              v
      Detector             Timebase           timing-core      API server
      registry             registry +         projections      (HTTP/WS/SSE)
      (live state +        clock monitor      (laps, sectors,  reading
       health)             (sync state +      gaps, results)   from log +
                            uncertainty)                       projections
                                                                    |
                                                                    v
                                                            apps, operator
                                                            tools, integrations
```

The node ingests four kinds of first-class events, all defined in [`event-model`](https://github.com/Open-Timekeeping/event-model):

- **Canonical detector events** (detections, hits, crossings) from any detector adapter.
- **Detector health events** that say whether a detector is producing trustworthy data.
- **Timebase status events** that say whether the clocks are locked, in holdover, drifting, or free-running.
- **Adapter metadata events** announcing identity and capabilities at startup and on config change.

### Two ingest paths, one contract

```text
  Path A: external producer                        Path B: in-process plugin
  -------------------------------                  -------------------------
  detector adapter logic                           detector adapter logic
        |                                                |
        v   wire-protocol envelope                       v   plugin-api call
        v   + frame-codec                                v   direct in-process
        v   + a transport binding                        v
        |   (TCP / USB CDC / Unix socket / ...)          |
        v                                                v
  ingest listener                                  plugin host
        |                                                |
        +----------------------+-------------------------+
                               v
                          event log
```

Both paths produce identical canonical events on the log. The choice between them is a packaging decision, not a contract decision. A well-built detector adapter can be built either way with minimal code change.

### What it serves

- **Live subscribe** for apps and operator tools, streaming canonical events plus projection deltas.
- **Range reads and replay** for any consumer that wants history.
- **Registry queries** for "which detectors are connected and healthy?" / "which timebase is in use and what's its current state?"
- **Projection queries** computed by [`timing-core`](https://github.com/Open-Timekeeping/timing-core) (current leaderboard, lap and sector results, gaps, session state).
- **Diagnostics** for [`app-diagnostics`](https://github.com/Open-Timekeeping/app-diagnostics): event-flow latency, ingest rate, log lag, detector and timebase health.

### Honest provenance, immutable log, amendments

Every event carries the timestamping method, timebase reference, sync state at the moment of capture, and declared uncertainty. The log is append-only; corrections (late hits, operator corrections, regrouping replays) are emitted as explicit amendment events that supersede a prior offset, never as in-place edits. The original is always there.

### Deployment shapes

A node is sized for its role. Five canonical deployment shapes are documented in [`spec/topologies.md`](https://github.com/Open-Timekeeping/spec/blob/main/topologies.md):

- **Native producer.** Detector adapter runs in firmware on the device; the device speaks OTK directly over a transport binding it supports (USB CDC, TCP, UART, ...). No node-side adapter required.
- **Edge adapter.** A standalone process on a Pi or mini-PC normalizes a raw / legacy device into canonical events and ships OTK frames to a node, typically over TCP.
- **Hub plugin.** A raw device cables into the host running the node; the adapter is loaded as a plugin and never leaves the process.
- **Local same-host adapters.** Adapter processes on the same host as the node connect over a Unix socket binding.
- **One node per detector stack.** Each timing point has its own node, persisting locally; aggregation upstream is optional.
- **Central hub.** Many producers feed one central node that owns all storage and APIs, with multiple ingest listeners (TCP + USB CDC + Unix socket + ...).

These combine freely. A real venue might run firmware producers for some detectors, an edge gateway for legacy hardware, manual-entry and CSV-replay as in-process plugins, and a central node ingesting everything.

### What a node does *not* do

- It does **not** parse vendor-specific raw device protocols. That is the detector adapter's job.
- It does **not** synthesize timestamps it didn't receive honestly. It records `appended_at` for storage time, never as event time.
- It does **not** apply policy to decide whether an event is "official." Policy is a separate layer above the runtime; mechanism and policy are kept apart.
- It does **not** present a UI. Apps consume its APIs.

### Timing Fabric

A **Timing Fabric** is the deployed topology around one or more runtime nodes: nodes plus their connected producers, adapters, timebase sources, apps, storage, and operator tools. "Fabric" describes a *deployment*. It is not a synonym for any single repo or process, and there is no repo called "fabric." When you read "the fabric" in Open Timekeeping docs, it always means a running system in a venue.

## Networking models

A detector adapter is a logical role, not a deployment constraint. The same `detector-adapter-api` contract is satisfied by:

- **Native firmware** on a detector device that speaks OTK directly ([`embedded-core`](https://github.com/Open-Timekeeping/embedded-core) + [`embedded-hal`](https://github.com/Open-Timekeeping/embedded-hal) + a `target-*` port + [`embedded-wire`](https://github.com/Open-Timekeeping/embedded-wire)). A native detector device is not required to run a TCP stack; any supported transport binding suffices.
- **External adapter process** on an edge gateway (using [`otk-ingest-client`](https://github.com/Open-Timekeeping/otk-ingest-client) for convenience, or building directly on [`wire-protocol`](https://github.com/Open-Timekeeping/wire-protocol) + [`frame-codec`](https://github.com/Open-Timekeeping/frame-codec) + a transport binding).
- **Plugin** loaded directly into `timing-node` ([`plugin-api`](https://github.com/Open-Timekeeping/plugin-api)).
- **Simulator** or **CSV replay** for development and conformance.

See [`spec/topologies.md`](https://github.com/Open-Timekeeping/spec/blob/main/topologies.md) for the five canonical deployment models and how they combine.

## Operating principles

- **Honest provenance over false precision.** If timestamp resolution, uncertainty, or sync state is unknown, report it. Never pretend nanosecond representation means nanosecond accuracy.
- **Records are immutable. Corrections are amendments.** A late hit doesn't edit a crossing in place, it emits an amendment event that supersedes a prior offset.
- **Detectors observe. Timing-core interprets. Apps present.**
- **Mechanism vs policy.** The runtime records what happened and how trustworthy it is. Whether an event is acceptable as official is a separate policy decision.
- **AI is advisory, never authoritative.** Results, penalties, and flags are not AI-owned.
- **Domain neutrality at the timing layer.** Subjects, timing points, detections, crossings, not laps, drivers, races, or flags. Domain concepts live in the app layer.

## Status

**Early development.** Architecture is settling; protocol and schema details are explicitly tracked as open questions in [`spec/open-questions.md`](https://github.com/Open-Timekeeping/spec/blob/main/open-questions.md). No released versions yet.

## Compatibility

An implementation claims Open Timekeeping compatibility by satisfying the relevant contracts and passing [`conformance`](https://github.com/Open-Timekeeping/conformance) with the canonical [`conformance-fixtures`](https://github.com/Open-Timekeeping/conformance-fixtures). See [`spec/compatibility.md`](https://github.com/Open-Timekeeping/spec/blob/main/compatibility.md).

## Get involved

- Read [`spec/architecture.md`](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md) first.
- Each repo's `README.md` defines its boundary, dependencies, and what does and does not belong there.
- Issues and discussions are open on every repo.
- Adapter, timebase, storage, and target ports are first-class plugins. You don't need to host your plugin on this organization, anywhere is fine. What matters is the contract and a passing conformance run.

## License

Apache-2.0 across software repos. The reference-hardware repo's design-file license may shift to CERN-OHL-S or TAPR OHL depending on community feedback.

## Maintainer

The Open Timekeeping organization is maintained by Jeremy Willemse on behalf of **[Willemse & Company Technology Inc.](https://willemse.co)** Contributions are welcome.
