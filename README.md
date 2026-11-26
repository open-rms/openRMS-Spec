# openRMS-Spec

**Provides:** the language-neutral contracts openRMS projects and third-party apps implement:
the Directory API (OpenAPI 3.1), directory modes and the Winlink-compatible feed, the watcher
feed (openRMS-Watch → Directory), identity and single sign-on (openRMS-Accounts), and rig
settings profiles and their API (openRMS-Rigs). · **Consumes:** nothing. · **Status:** spec
(directory drafted; others draft). · **License:** CC-BY-4.0 (prose) / MPL-2.0 (schemas and
OpenAPI), see below.

| Folder | What | Used by |
|---|---|---|
| `directory/` | `openapi.yaml` (Directory API), `modes.md` (modem ids, compat mapping), `winlink-compat.md` | [openRMS-Directory](https://github.com/open-rms/openRMS-Directory), [openRMS-Site](https://github.com/open-rms/openRMS-Site), Pat and other clients |
| `watch/` | `feed.md`: watcher uploads, sightings, live stream, privacy | openRMS-Watch, openRMS-Directory |
| `accounts/` | `identity.md`: issuer, tokens and claims, service keys, webhooks | every openRMS service |
| `rigs/` | `profile.md`, `profile.schema.json`, `openapi.yaml`, `examples/` | openRMS-Rigs, rigctlWeb, hamstack |

`PLAN.md` lists the planned specs (Translator control API, config schema, modem model, host
personalities, mode timing table) and the rules: semantic versioning per spec, OpenAPI as the
source of truth, conformance tests in openRMS-Testbed.

## License

- **MPL-2.0** ([LICENSE-MPL-2.0](LICENSE-MPL-2.0)): machine-readable contracts and code
  generated from them: `directory/openapi.yaml`, `rigs/openapi.yaml`,
  `rigs/profile.schema.json`, `rigs/examples/*`. Apps under any license can implement them.
- **CC-BY-4.0** ([LICENSE-CC-BY-4.0](LICENSE-CC-BY-4.0)): the prose specifications and docs
  (all `*.md` files).

Part of [openRMS](https://openrms.dev).
