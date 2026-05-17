---
name: Feature request
about: Propose a new feature, schema, or protocol change
title: ""
labels: enhancement
---

## What problem are you solving

<!-- Use case or pain point. Concrete examples help; abstract "wouldn't it be nice" ideas get less traction than "I was trying to do X and hit Y." -->

## Proposed change

<!-- What you would like to see. Be specific about the surface: a new runtime capability, an event schema addition, a protocol change, an adapter behavior, a client SDK feature, etc. -->

## Which layer(s) does this affect

- [ ] Event Model (canonical event types, identifiers, provenance blocks)
- [ ] Wire Protocol (OTK message envelope, message types, sequencing, compatibility)
- [ ] Frame Codec (encoding/decoding, stream or resynchronizable framing)
- [ ] Transport Binding (TCP, serial, USB CDC, Unix socket, ...)
- [ ] Detector adapter contract
- [ ] Timebase contract
- [ ] Runtime node (ingest, registry, event log, plugin host, APIs)
- [ ] Storage backend
- [ ] App layer / downstream consumers
- [ ] Tooling, infrastructure, or CI

## Alternatives considered

<!-- What else you thought about, and why this approach. If you considered solving it in a downstream application instead of the timing layer, mention that. -->

## Compatibility

- [ ] Additive within the current protocol and schemas
- [ ] Requires a protocol change (describe the migration)
- [ ] Requires a schema change (describe the version bump and compatibility mode)
- [ ] Requires a new transport binding (which one, and why)
- [ ] I'm not sure, want input

## Additional context

<!-- Links to related issues, prior art, papers, vendor documentation, real-world incidents that motivate the change. -->
