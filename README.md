# openRMS-Spec

**Provides:** the language-neutral contracts openRMS projects and third-party apps implement:
Atlas API (OpenAPI 3.1), Atlas modes and the Winlink-compatible feed, the watcher
feed (openRMS-Watch → Atlas), identity and single sign-on (openRMS-Accounts), and rig
settings profiles and their API (openRMS-Rigs). · **Consumes:** nothing. · **Status:** spec
(atlas drafted; others draft). · **License:** CC-BY-4.0 (prose) / MPL-2.0 (schemas and
OpenAPI), see below.

| Folder | What | Used by |
|---|---|---|
| `atlas/` | `openapi.yaml` (Atlas API), `modes.md` (modem ids, compat mapping), `winlink-compat.md` | [openRMS-Atlas](https://github.com/open-rms/openRMS-Atlas), [openRMS-Site](https://github.com/open-rms/openRMS-Site), Pat and other clients |
| `watch/` | `feed.md`: watcher uploads, sightings, live stream, privacy | openRMS-Watch, openRMS-Atlas |
| `accounts/` | `identity.md`: issuer, tokens and claims, service keys, webhooks | every openRMS service |
| `rigs/` | `profile.md`, `profile.schema.json`, `openapi.yaml`, `examples/` | openRMS-Rigs, rigctlWeb, hamstack |

`PLAN.md` lists the planned specs (Translator control API, config schema, modem model, host
personalities, mode timing table) and the rules: semantic versioning per spec, OpenAPI as the
source of truth, conformance tests in openRMS-Testbed.

## License

- **MPL-2.0** ([LICENSE-MPL-2.0](LICENSE-MPL-2.0)): machine-readable contracts and code
  generated from them: `atlas/openapi.yaml`, `rigs/openapi.yaml`,
  `rigs/profile.schema.json`, `rigs/examples/*`. Apps under any license can implement them.
- **CC-BY-4.0** ([LICENSE-CC-BY-4.0](LICENSE-CC-BY-4.0)): the prose specifications and docs
  (all `*.md` files).

Part of [openRMS](https://openrms.dev).
