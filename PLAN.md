# openRMS-Spec

**Role:** the language-neutral contracts every other project implements. This gets written
first, so projects can be built in parallel and other apps can integrate without reading our
code.
**License:** CC-BY-4.0 for documents, MPL-2.0 for schemas and generated code.

## Contents

| Spec | Used by | Format |
|---|---|---|
| **Identity** | Accounts, every service and app | Markdown (`accounts/identity.md`): issuer, flows, token audiences, claims, scopes, service-owned keys, webhooks |
| **Atlas API** | Atlas, Site, Gateway, third-party apps | OpenAPI 3.1 (`atlas/openapi.yaml`) |
| **Rigs profiles + API** | Rigs, rigctlWeb, hamstack, Translator (later), third parties | JSON Schema 2020-12 (`rigs/profile.schema.json`, guide `rigs/profile.md`, examples `rigs/examples/`) and OpenAPI 3.1 (`rigs/openapi.yaml`). Hamlib names and units, structured apply steps, no keying |
| **Translator control API** | Gateway, UIs, third parties | Protobuf/gRPC + JSON-over-socket mapping (or OpenAPI for local HTTP; decide in Phase 0) |
| **Config schema** | Translator, Gateway, UIs | JSON Schema for the TOML config. Every key has default, range, unit, help and **tier** (standard/advanced/expert) and whether it's a safety guard |
| **Modem model** | Libs, Translator | Markdown: stream vs frame modems, events (PTT, BUSY, CONNECTED, DISCONNECTED), states |
| **Host personalities** | Translator, Libs | What we implement of the ARDOP host, VARA-compatible host, KISS and AGWPE protocols, with the quirks found in the wild (e.g. Mercury `TNC.md` notes on BPQ32, varim) |
| **Mode timing table** | Translator scanner | Per modem and version: longest call frame, minimum dwell, bandwidths |
| **Atlas dump format** | Atlas, offline clients | JSON/CSV/GeoJSON/SQLite layouts, signature scheme |

## Rules

- Semantic versioning per spec. Breaking changes need a new major version and a migration note.
- Each spec has conformance tests in openRMS-Testbed.
- OpenAPI is the source of truth: server stubs and clients are generated from it.

## Phases

- **P0:** config schema v0, modem model, control API draft, Atlas OpenAPI draft, mode
  timing table seeded from Mercury's docs.
- **P1–P2:** host-personality spec refined from the compatibility matrix.
- **P3:** Atlas API v1 frozen.
- **P4+:** extended as projects need.
