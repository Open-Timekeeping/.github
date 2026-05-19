# Repo family

How the Open Timekeeping organization's repos fit together.

The conceptual model lives in [`spec`](https://github.com/Open-Timekeeping/spec), start at [`spec/architecture.md`](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md). This page is the deployment-and-dependency-side view: who depends on whom, and where to look for what.

The org has no `open-timekeeping-` or `otk-` repo-name prefix; the org namespace already provides that. Crate names, binaries, and CLI commands may use `otk-` (the runtime binary is `otk-node`).

The architecture is [ports-and-adapters (hexagonal)](https://github.com/Open-Timekeeping/spec/blob/main/architecture.md). Repos fall into four groups: `server/core`, `server/ports`, `server/adapters`, `server/app` (the timing node), `sdk`, and `producers`/`consumers`.

---

## Dependency rules

| Layer | May depend on |
|---|---|
| `server/core/*` | nothing (no OTK deps) |
| `server/ports/*` | `server/core/*` only |
| `server/adapters/*` | `server/ports/*`, `server/core/*` |
| `server/app/*` | everything in `server/` |
| `sdk/otk-sdk` | `event-model`, `protocol` (optional) only |
| `producers/*` | `sdk/otk-sdk` only |
| `consumers/*` | `sdk/otk-sdk` only |

---

## Standard and shared contracts

| Repo | Role |
|---|---|
| [spec](https://github.com/Open-Timekeeping/spec) | The Open Timekeeping standard and conceptual model. Terminology, architecture, topologies, compatibility, open questions. |

---

## server/core: domain types and logic

No external OTK dependencies. The bedrock of the server.

| Repo | Role |
|---|---|
| [event-model](https://github.com/Open-Timekeeping/event-model) | Canonical event types and identifiers, transport-independent. Shared across server and SDK. |
| [protocol](https://github.com/Open-Timekeeping/protocol) | Wire DTOs: `OtkEnvelope`, message types, handshake messages. Shared between `adapter-ingest-tcp` (decode) and `otk-sdk` producer feature (encode). |
| [timing-core](https://github.com/Open-Timekeeping/timing-core) | Timing-domain engine: detections to crossings to laps to results. Depends on `event-model` only. |

---

## server/ports: typed boundary contracts

Each port is an interface that the timing node uses, implemented by one or more adapters. Ports depend only on `server/core`.

| Repo | Role |
|---|---|
| [port-in-ingest](https://github.com/Open-Timekeeping/port-in-ingest) | Inbound port: `EventIngestPort` and `IngestSession`. The timing node receives typed `OtkEvent` values; all framing and handshake is the adapter's concern. |
| [port-out-event-log](https://github.com/Open-Timekeeping/port-out-event-log) | Outbound port: `EventLog` and `LogSubscription`. Storage is pluggable by design. |

---

## server/adapters: port implementations

Each adapter implements one port contract over a specific technology.

| Repo | Role |
|---|---|
| [adapter-ingest-tcp](https://github.com/Open-Timekeeping/adapter-ingest-tcp) | Implements `port-in-ingest` over TCP. Encapsulates framing (4-byte length prefix), CBOR decoding, and the OTK handshake. |
| [adapter-event-log-segment](https://github.com/Open-Timekeeping/adapter-event-log-segment) | Implements `port-out-event-log` using append-only segment files on disk. The v0 storage backend. |

Future adapters (not yet created): `adapter-ingest-serial`, `adapter-ingest-usb-cdc`, `adapter-ingest-unix-socket`.

---

## server/app: composition root

| Repo | Role |
|---|---|
| [timing-node](https://github.com/Open-Timekeeping/timing-node) | The deployable Timing Runtime Node. Binary: `otk-node`. Wires inbound adapters, storage adapters, and timing-core together. Never imports wire-protocol types directly. |

---

## SDK

| Repo | Role |
|---|---|
| [otk-sdk](https://github.com/Open-Timekeeping/otk-sdk) | Single SDK crate for all producers and consumers. `default=["client"]` for HTTP/SSE consumers; `features=["producer"]` for TCP producers. Re-exports `event-model` types. No server-side dependencies. |

The `producer` feature absorbs what was previously `otk-ingest-client`, `detector-adapter-api`, `detector-adapter-common`, and `timebase-api`. The `client` feature is the query-API client (Phase 2).

---

## producers

| Repo | Role |
|---|---|
| [adapter-simulator](https://github.com/Open-Timekeeping/adapter-simulator) | Synthetic detector producer. Uses `otk-sdk` (producer feature) only. Zero server-side dependencies. |

Future producers: `adapter-csv-replay`, `adapter-manual`.

---

## consumers (future)

Consumer applications will depend on `otk-sdk` (client feature) only. They read from the timing node's REST/SSE API. None yet created.

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

These repos have been superseded by the hexagonal refactor:

| Former repo | Superseded by |
|---|---|
| `wire-protocol` | `protocol` (renamed) |
| `transport-api` | `port-in-ingest` (redesigned) |
| `transport-tcp` | `adapter-ingest-tcp` (renamed, expanded) |
| `storage-api` | `port-out-event-log` (renamed) |
| `storage-segment-log` | `adapter-event-log-segment` (renamed) |
| `frame-codec` | absorbed into `adapter-ingest-tcp` (server) and `otk-sdk` producer feature (client) |
| `detector-adapter-api` | absorbed into `otk-sdk` producer feature |
| `detector-adapter-common` | absorbed into `otk-sdk` producer feature |
| `timebase-api` | absorbed into `otk-sdk` producer feature |
| `otk-ingest-client` | replaced by `otk-sdk` producer feature |

---

## Dependency principles

- **Ports-and-adapters (hexagonal) architecture.** Core types flow inward; adapters implement outward contracts. The timing node (app) is the composition root.
- **Dependency rules are enforced by crate boundaries.** `timing-node` does not import `protocol` types; framing knowledge stays in `adapter-ingest-tcp`. `otk-sdk` does not import server port or adapter crates.
- **Producers and consumers have zero server-side dependencies.** They depend only on `otk-sdk`.
- **Transport is runtime configuration, not a compile-time feature.** `Producer::connect(Transport::Tcp(addr))` vs `Transport::Serial { port, baud }`.

## What's intentionally not here

- **No vendor-specific proprietary adapters** at this stage.
- **No race-control app** at this stage.
- **No vague "fabric repo."** Timing Fabric is a deployment concept; it is not a repo or a binary.
