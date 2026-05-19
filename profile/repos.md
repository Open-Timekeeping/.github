# Repo family

How the Open Timekeeping organization's repos fit together.

The conceptual model lives in [`spec`](https://github.com/Open-Timekeeping/spec), start at [`spec/architecture.md`](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md). This page is the deployment-and-dependency-side view: who depends on whom, and where to look for what.

The org has no `open-timekeeping-` or `otk-` repo-name prefix; the org namespace already provides that. Crate names, binaries, and CLI commands may use `otk-` (the runtime binary is `otk-node`).

The architecture is [ports-and-adapters (hexagonal)](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md).

---

## Dependency rules

| Layer | May depend on |
|---|---|
| `otk-core` members | path deps within the workspace only |
| `adapter-ingest-tcp` | `otk-core` (port-in-ingest, protocol, event-model) |
| `adapter-event-log-segment` | `otk-core` (port-out-event-log, event-model) |
| `otk-sdk` | `otk-core` (event-model, protocol optional) |
| `timing-node` | `otk-core` + adapters |
| `producer-simulated` | `otk-sdk` only |
| `consumers/*` | `otk-sdk` only |

---

## Standard and shared contracts

| Repo | Role |
|---|---|
| [spec](https://github.com/Open-Timekeeping/spec) | The Open Timekeeping standard and conceptual model. Terminology, architecture, topologies, compatibility, open questions. |

---

## otk-core workspace

A single Cargo workspace containing all shared core crates. Internal deps use path refs; no member references anything outside the workspace.

| Crate | Role |
|---|---|
| [event-model](https://github.com/Open-Timekeeping/otk-core/tree/main/event-model) | Canonical event types and identifiers. No OTK deps. |
| [protocol](https://github.com/Open-Timekeeping/otk-core/tree/main/protocol) | Wire DTOs: OtkEnvelope, handshake messages, MessageType. Depends on event-model. |
| [timing-core](https://github.com/Open-Timekeeping/otk-core/tree/main/timing-core) | Detection-to-crossing engine. Depends on event-model. |
| [port-in-ingest](https://github.com/Open-Timekeeping/otk-core/tree/main/port-in-ingest) | Inbound port: EventIngestPort, IngestSession. Depends on event-model. |
| [port-out-event-log](https://github.com/Open-Timekeeping/otk-core/tree/main/port-out-event-log) | Outbound port: EventLog, LogSubscription, Offset. Depends on event-model. |

Downstream crates reference members via:
```toml
event-model = { git = "https://github.com/Open-Timekeeping/otk-core", package = "event-model" }
```

---

## server adapters

Each adapter implements one port contract over a specific technology. One repo per adapter.

| Repo | Role |
|---|---|
| [adapter-ingest-tcp](https://github.com/Open-Timekeeping/adapter-ingest-tcp) | Implements `port-in-ingest` over TCP. Encapsulates framing and OTK handshake. |
| [adapter-event-log-segment](https://github.com/Open-Timekeeping/adapter-event-log-segment) | Implements `port-out-event-log` using append-only segment files on disk. |

Future: `adapter-ingest-serial`, `adapter-ingest-usb-cdc`, `adapter-ingest-unix-socket`.

---

## server app

| Repo | Role |
|---|---|
| [timing-node](https://github.com/Open-Timekeeping/timing-node) | Reference Timing Runtime Node. Binary: `otk-node`. Wires inbound adapters, storage, and timing-core together. |

---

## SDK

| Repo | Role |
|---|---|
| [otk-sdk](https://github.com/Open-Timekeeping/otk-sdk) | Single SDK crate for producers and consumers. `default=["client"]` for HTTP/SSE reads; `features=["producer"]` for TCP producers. Re-exports event-model. No server deps. |

---

## producers

| Repo | Role |
|---|---|
| [producer-simulated](https://github.com/Open-Timekeeping/producer-simulated) | Reference simulated producer. Uses `otk-sdk` (producer feature) only. Binary: `otk-simulator`. |

Future: `producer-csv-replay`, `producer-manual`.

---

## consumers (future)

Consumer applications depend on `otk-sdk` (client feature) only.

---

## Embedded toolkit

| Repo | Role |
|---|---|
| [embedded-core](https://github.com/Open-Timekeeping/embedded-core) | Portable embedded logic for native detector adapters (`no_std`). |
| [embedded-hal](https://github.com/Open-Timekeeping/embedded-hal) | HAL trait surface for chip / board ports. |
| [target-rp2040](https://github.com/Open-Timekeeping/target-rp2040) | RP2040 / Raspberry Pi Pico port. |
| [target-stm32](https://github.com/Open-Timekeeping/target-stm32) | STM32 port. |

---

## Reference hardware and firmware

| Repo | Role |
|---|---|
| [reference-hardware](https://github.com/Open-Timekeeping/reference-hardware) | Reference detector hardware design (schematics, PCB, BOM). |
| [reference-detector-firmware](https://github.com/Open-Timekeeping/reference-detector-firmware) | Complete firmware combining `embedded-*` and a `target-*` port. |

---

## APIs and apps (future)

| Repo | Role |
|---|---|
| [app-live-timing](https://github.com/Open-Timekeeping/app-live-timing) | Operator / spectator live timing app. |
| [app-diagnostics](https://github.com/Open-Timekeeping/app-diagnostics) | Operator diagnostics app (detector health, timebase status, latency). |

---

## Conformance and docs

| Repo | Role |
|---|---|
| [conformance](https://github.com/Open-Timekeeping/conformance) | Compatibility suite for adapters, producers, consumers, storage, runtime. |
| [conformance-fixtures](https://github.com/Open-Timekeeping/conformance-fixtures) | Shared test fixtures (sample streams, edge cases, expected outputs). |
| [docs-site](https://github.com/Open-Timekeeping/docs-site) | Optional unified documentation site. `spec` remains the canonical standards source. |

---

## Archived repos

These repos have been superseded and will be archived after `otk-core` is merged:

| Repo | Superseded by |
|---|---|
| `event-model` | `otk-core` workspace member |
| `timing-core` | `otk-core` workspace member |
| `protocol` (was `wire-protocol`) | `otk-core` workspace member |
| `port-in-ingest` (was `transport-api`) | `otk-core` workspace member |
| `port-out-event-log` (was `storage-api`) | `otk-core` workspace member |
| `frame-codec` | absorbed into `adapter-ingest-tcp` (server) and `otk-sdk` producer feature (client) |
| `detector-adapter-api` | absorbed into `otk-sdk` producer feature |
| `detector-adapter-common` | absorbed into `otk-sdk` producer feature |
| `otk-ingest-client` | replaced by `otk-sdk` producer feature |

---

## Dependency principles

- **Ports-and-adapters (hexagonal) architecture.** Core types flow inward; adapters implement outward contracts. The timing node (app) is the composition root.
- **Dependency rules are enforced by crate boundaries.** `timing-node` never imports `protocol` types. `otk-sdk` never imports server ports or adapters.
- **Producers and consumers have zero server-side dependencies.** They depend only on `otk-sdk`.
- **Transport is runtime configuration.** `Producer::connect(Transport::Tcp(addr))`.

## What's intentionally not here

- **No vendor-specific proprietary adapters** at this stage.
- **No race-control app** at this stage.
- **No vague "fabric repo."** Timing Fabric is a deployment concept; it is not a repo or a binary.
