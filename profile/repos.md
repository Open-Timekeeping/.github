# Repo family

How the Open Timekeeping organization's repos fit together.

The conceptual model lives in [`spec`](https://github.com/Open-Timekeeping/spec), start at [`spec/architecture.md`](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md). This page is the deployment-and-dependency-side view: who depends on whom, and where to look for what.

The org has no `open-timekeeping-` or `otk-` repo-name prefix; the org namespace already provides that. Crate names, binaries, and CLI commands may use `otk-` (the runtime binary is `otk-node`).

---

## Standard and shared contracts

| Repo | Role |
|---|---|
| [spec](https://github.com/Open-Timekeeping/spec) | The Open Timekeeping standard and conceptual model. Terminology, architecture, topologies, compatibility, open questions. |

## OTK Protocol stack

The OTK Protocol is the transport-independent wire protocol used by Open Timekeeping components to exchange canonical timing messages. It is split into four layers, each in its own repo (or set of repos).

| Layer | Repo | Role |
|---|---|---|
| Event Model | [event-model](https://github.com/Open-Timekeeping/event-model) | Canonical event types and identifiers, transport-independent. |
| Wire Protocol | [wire-protocol](https://github.com/Open-Timekeeping/wire-protocol) | OTK message envelope: versioning, message types, source identity, sequence numbers, acks, errors, compatibility rules. |
| Frame Codec | [frame-codec](https://github.com/Open-Timekeeping/frame-codec) | Encode/decode of OTK messages into byte frames (stream-oriented and resynchronizable). |
| Frame Codec (firmware) | [embedded-wire](https://github.com/Open-Timekeeping/embedded-wire) | `no_std`-friendly OTK encoder/decoder for firmware. |
| Transport Binding (contract) | [transport-api](https://github.com/Open-Timekeeping/transport-api) | Common listener/client abstraction across all transport bindings. |
| Transport Binding (TCP) | [transport-tcp](https://github.com/Open-Timekeeping/transport-tcp) | OTK frames over persistent TCP streams. Default for IP-capable producers. |
| Transport Binding (serial) | [transport-serial](https://github.com/Open-Timekeeping/transport-serial) | OTK frames over UART / RS-232 byte streams. |
| Transport Binding (USB CDC) | [transport-usb-cdc](https://github.com/Open-Timekeeping/transport-usb-cdc) | OTK frames over USB CDC serial devices. |
| Transport Binding (Unix socket) | [transport-unix-socket](https://github.com/Open-Timekeeping/transport-unix-socket) | OTK frames over Unix domain sockets for same-host adapters. |

## Detector contract and implementations

| Repo | Role |
|---|---|
| [detector-adapter-api](https://github.com/Open-Timekeeping/detector-adapter-api) | The universal detector boundary. Every detector event source implements this. |
| [detector-adapter-common](https://github.com/Open-Timekeeping/detector-adapter-common) | Shared helpers for detector adapter authors. |
| [adapter-csv-replay](https://github.com/Open-Timekeeping/adapter-csv-replay) | Detector adapter for replaying detections from CSV / structured files. |
| [adapter-manual](https://github.com/Open-Timekeeping/adapter-manual) | Detector adapter for manual timing input (button / keyboard / tablet). |
| [adapter-simulator](https://github.com/Open-Timekeeping/adapter-simulator) | Detector adapter wrapping `detector-simulator` for runtime use. |
| [detector-simulator](https://github.com/Open-Timekeeping/detector-simulator) | The simulation engine: realistic detection streams, fault injection, fixtures. |

## Timebase contract and implementations

| Repo | Role |
|---|---|
| [timebase-api](https://github.com/Open-Timekeeping/timebase-api) | First-class clock / timebase contract. |
| [timebase-common](https://github.com/Open-Timekeeping/timebase-common) | Shared helpers for timebase authors and consumers. |
| [timebase-gnss](https://github.com/Open-Timekeeping/timebase-gnss) | GNSS / GPS / PPS timebase. |
| [timebase-ptp](https://github.com/Open-Timekeeping/timebase-ptp) | PTP timebase (software, hardware-PHC, White-Rabbit-class). |
| [timebase-ntp](https://github.com/Open-Timekeeping/timebase-ntp) | NTP timebase. |
| [timebase-local](https://github.com/Open-Timekeeping/timebase-local) | Local-clock / monotonic timebase, honestly degraded. |

## Ingest client and plugin transport

| Repo | Role |
|---|---|
| [otk-ingest-client](https://github.com/Open-Timekeeping/otk-ingest-client) | Convenience client library for external producers that publish OTK frames into a runtime node. Not a required architectural component. |
| [plugin-api](https://github.com/Open-Timekeeping/plugin-api) | Contract for in-process modules loaded by `timing-node`. |

## Runtime

| Repo | Role |
|---|---|
| [timing-core](https://github.com/Open-Timekeeping/timing-core) | Timing-domain engine library, detections to crossings to laps to results. |
| [timing-node](https://github.com/Open-Timekeeping/timing-node) | The deployable Timing Runtime Node. Binary: `otk-node`. |

## Storage

| Repo | Role |
|---|---|
| [storage-api](https://github.com/Open-Timekeeping/storage-api) | Persistence trait. Storage is pluggable on principle. |
| [storage-segment-log](https://github.com/Open-Timekeeping/storage-segment-log) | Custom segment-file event log. The v0 backend. |

## APIs and apps

| Repo | Role |
|---|---|
| [api-model](https://github.com/Open-Timekeeping/api-model) | DTOs / schemas for the runtime node's outward APIs. |
| [app-live-timing](https://github.com/Open-Timekeeping/app-live-timing) | Operator / spectator live timing app. |
| [app-diagnostics](https://github.com/Open-Timekeeping/app-diagnostics) | Operator diagnostics app (detector health, timebase status, latency). |

## Embedded toolkit

| Repo | Role |
|---|---|
| [embedded-core](https://github.com/Open-Timekeeping/embedded-core) | Portable embedded logic for native detector adapters (`no_std`). |
| [embedded-hal](https://github.com/Open-Timekeeping/embedded-hal) | HAL trait surface for chip / board ports. |
| [target-rp2040](https://github.com/Open-Timekeeping/target-rp2040) | RP2040 / Raspberry Pi Pico port. |
| [target-stm32](https://github.com/Open-Timekeeping/target-stm32) | STM32 port. |

## Reference hardware and firmware

| Repo | Role |
|---|---|
| [reference-hardware](https://github.com/Open-Timekeeping/reference-hardware) | Reference detector hardware design (schematics, PCB, BOM). |
| [reference-detector-firmware](https://github.com/Open-Timekeeping/reference-detector-firmware) | Complete firmware combining `embedded-*` and a `target-*` port. |

## Conformance and docs

| Repo | Role |
|---|---|
| [conformance](https://github.com/Open-Timekeeping/conformance) | Compatibility suite for adapters, timebases, producers, plugins, storage, runtime. |
| [conformance-fixtures](https://github.com/Open-Timekeeping/conformance-fixtures) | Shared test fixtures (sample streams, edge cases, expected outputs). |
| [docs-site](https://github.com/Open-Timekeeping/docs-site) | Optional unified documentation site. `spec` remains the canonical standards source. |

---

## Dependency principles

- **Granular repos by design.** Each contract, implementation, and shared library is its own repo. No mega-monorepo.
- **Local Rust development uses sibling-relative path dependencies.** E.g. `event-model = { path = "../event-model" }`. Crates remain independently publishable when work matures.
- **Shared code belongs in shared library repos.** If two adapters need the same helper, it goes in `detector-adapter-common`. If two timebases need the same helper, it goes in `timebase-common`. Never copy-paste.

## What's intentionally not here

- **No vendor-specific proprietary adapters** (MYLAPS, RaceResult, etc.) at this stage. They may be considered later, or never, depending on protocol, licensing, and access.
- **No race-control app** at this stage.
- **No native-embedded-bypass path.** Embedded firmware is one packaging of the detector adapter contract, not an alternative to it.
- **No vague "fabric repo."** *Timing Fabric* is a deployment concept; it is not a repo or a binary.
