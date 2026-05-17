# Language choices

How Open Timekeeping picks a language per layer, and why.

The contracts between layers, canonical events and the wire protocol, are language-agnostic, so each layer can pick the best-fit language without forcing the others. The first reference stack is mostly Rust because the edge / runtime constraints argue for it; layers above that are conventional web work.

---

## Per layer

| Layer / repo | Language | Runs where |
|---|---|---|
| `event-model`, `wire-protocol`, `frame-codec`, `transport-api`, `transport-tcp`, `transport-serial`, `transport-usb-cdc`, `transport-unix-socket`, `detector-adapter-api`, `detector-adapter-common`, `timebase-api`, `timebase-common`, `storage-api`, `plugin-api`, `otk-ingest-client`, `timing-core`, `api-model`, `conformance`, `conformance-fixtures` | **Rust** (first reference impl) | libraries; cross-target |
| Detector adapter implementations (`adapter-*`, `detector-simulator`) | Rust | edge gateway, runtime-node host, simulator |
| Timebase implementations (`timebase-gnss`, `timebase-ptp`, `timebase-ntp`, `timebase-local`) | Rust default; native bindings as needed | runtime-node host, edge gateway |
| `timing-node` (binary `otk-node`) | Rust | runtime node host |
| `storage-segment-log` | Rust | runtime node host |
| `app-live-timing`, `app-diagnostics` | TypeScript (framework TBD) | browsers, talking to runtime-node APIs |
| `embedded-core`, `embedded-hal`, `embedded-wire`, `target-rp2040`, `target-stm32`, `reference-detector-firmware` | Rust (`#![no_std]`) | native detector devices |
| `docs-site` | TypeScript / framework TBD | static site / build artifact |

Bindings or alternative-language clients (Node, Go, Python, JVM, .NET producer libraries) are not in scope at this stage and will be added under appropriate names *if* real downstream consumers ask for them. The OTK Protocol stack (Event Model + Wire Protocol + Frame Codec + Transport Binding) is hand-rollable in any backend that wants to consume or produce events; `otk-ingest-client` is just convenience.

---

## Why Rust on the edge and in the runtime

The runtime node and embedded targets need predictable latency and small static binaries:

- No GC pauses between `recv()` and `clock_gettime()`.
- Single static binaries cross-compile cleanly to `aarch64-unknown-linux-gnu`, `armv7-unknown-linux-gnueabihf`, and `thumbv6m-none-eabi` / `thumbv7em-none-eabihf`.
- Small resident footprint (single-digit to low-tens of MB on the edge; flash-budget-friendly on MCUs).
- Direct access to the OS / hardware APIs that matter for honest timestamping: `SO_TIMESTAMPING`, PHC (`/dev/ptp*`), `clock_gettime(CLOCK_TAI)`, GPIO interrupts via `gpio-cdev`, USB-CDC framing, MCU timer-capture peripherals.
- The type system makes the provenance discipline this project insists on enforceable at compile time, "every event must include timebase provenance" becomes a struct field, not a code-review convention, and `Option<T>` makes the `null` / `unknown` / `degraded` distinction explicit.

## Why not Rust for the apps

Live-timing and diagnostics apps are conventional web frontends. Forcing Rust on them costs velocity for no latency benefit (the timing-criticality is over by the time data reaches them). TypeScript + a chosen framework is the default.

## Why a Rust-only first reference stack

The first reference path, the OTK Protocol stack (`event-model` → `wire-protocol` → `frame-codec` → `transport-api` + a transport binding) plus `detector-adapter-api`, `otk-ingest-client`, and `timing-node`, is fastest to bring up coherently in one language. Bindings for other languages can follow once the contracts have settled.
