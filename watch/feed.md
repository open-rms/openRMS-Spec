# Watch feed: community watchers → Atlas (draft v0)

> Status: **draft for review** between openRMS-Watch and openRMS-Atlas, 2026-11-25.
> Nothing here is implemented yet. Open questions are marked **Q**.

A *watcher* is an `openrms-watch` install that hears Mercury (later other modems) connect
frames on the air. Contributing is opt-in. A watcher always keeps its own local database and
web page; contributing adds an upload of the same events to Atlas, which shows a live
feed, "heard on air" evidence on gateway pages, and a contributor list.

## 1. Accounts, watchers and tokens

- A contributor signs up on the site like any user (callsign required, one account per callsign).
- `POST /api/v1/me/watchers` (session) registers a watcher: `name`, `grid` (4 or 6 char),
  `locationPrecision` (`grid4` default | `grid6` | `hidden`), `public` (show in the contributor
  list, default true). Returns `{id, ...}`. `GET|PATCH|DELETE /api/v1/me/watchers/{id}`.
- **Watcher key (since 2026-11-26, identity.md §5):** an openRMS API key (`ormk_…`) minted
  at openRMS-Accounts for Atlas with scope `watch:ingest`, renewable. Atlas mints nothing.
  1. The watcher runs the device flow at Accounts (client `openrms-watch`). With an access
     token for Atlas it may list the account's watchers (`GET /api/v1/me/watchers`); with one
     for Accounts (`resource` = issuer, scope `keys`) it mints the key
     (`POST {issuer}/api/v1/keys`, identity.md §5.3).
  2. It binds the key once: `POST /api/v1/watch/link {watcherId} | {name, gridsquare?}` with
     `Authorization: Bearer ormk_…`. Atlas records `jti` → watcher; relinking moves the key.
     The tokens are then discarded.
  3. It renews the key before expiry (`POST {issuer}/api/v1/keys/renew`). The new key carries
     `replaces: <old jti>` and takes over the old key's watcher on its first upload, whether or
     not Atlas has heard of the old key's revocation yet.
- An unlinked key gets `403` (`code: not_linked`), a key without `watch:ingest` `403`, a revoked
  or expired key `401`. About 400 uploads per hour per key.
- Stopping uploads: revoke the key at Accounts (`{issuer}/keys`), which stops it everywhere
  within seconds (webhook) or 5 minutes (pull). `DELETE /api/v1/me/watchers/{id}/key` only
  unlinks keys from that watcher. Either way the watcher shows "token rejected" and keeps
  working locally.
- Retired: the Atlas-minted `ormsw_` keys, `POST /api/v1/me/watchers/{id}/key` (now `410`) and
  the Better Auth key configurations. Copy-paste still works: a person can make a
  `watch:ingest` key at `{issuer}/keys` and the watcher links it with step 2.

## 2. Upload: `POST /api/v1/watch/ingest`

`Authorization: Bearer ormk_...` or `x-api-key: ormk_...` (a linked `watch:ingest` key, §1), `content-type: application/x-ndjson`, at most 5000 lines or 5 MB. **Watchers batch: one POST every 10 minutes** (decided 2026-11-26),
sooner only when the queue nears the line limit. The live feed therefore lags real time by up
to about 10 minutes; timestamps are the watcher's, so sightings are unaffected.
One JSON object per line. Every event has a watcher-generated `id` (ULID), so retries are
idempotent: the server ignores ids it has already stored (unique per watcher). The watcher
queues events locally and retries with backoff while offline, so nothing heard is lost.

Response: `200 {"accepted": n, "duplicate": m, "rejected": [{"id":..., "error":...}]}`;
`401` bad or revoked key; `413` too big; `429` slow down (honour `retry-after`).

### Event types (`type` field)

```jsonc
// A decoded Mercury frame (from the monitor's MONITOR line)
{"type":"frame","id":"01J...","ts":"2026-11-25T21:36:12.465Z",
 "modem":"mercury","modemVersion":"2.0.x",
 "dialHz":7101340,            // reported dial = centre - 1500, ppm-corrected
 "freqSource":"scan",         // scan (measured from spectrum) | channel (fixed, configured)
 "freqAccuracyHz":10,         // the watcher's own estimate (calibrated or not)
 "mode":"DATAC16","kind":"CALL",   // CALL ACCEPT CQ DISCONNECT STATION BCAST_CTRL BCAST_DATA
 "from":"K7TEST",             // CALL/CQ: callsign heard in clear
 "toCrc":"3DEF",              // CALL: CRC16 of the called station
 "callerCrc":"C492","calledCrc":"3DEF",   // ACCEPT
 "sid":127,"bw":2300,"snrDb":24.0,
 "raw":"MONITOR DATAC16 CALL FROM=K7TEST TO_CRC=3DEF BW=2300 SID=127 SNR=24.0"}

// Optional: one connection attempt, folded on the watcher from repeated CALLs and the
// ACCEPT/DISCONNECT that followed (what the local app already shows). References its frames.
{"type":"attempt","id":"01J...","start":"2026-11-25T21:36:12.465Z","end":"2026-11-25T21:36:31.020Z",
 "modem":"mercury","dialHz":7101340,"from":"K7TEST","calledCrc":"3DEF","callerCrc":"C492",
 "accepted":true,"calls":3,"bestSnrDb":24.4,"frames":["01J...","01J..."],
 "hint":{"callsign":"W1RMS-10","match":"inferred","candidates":["W1RMS-10"]}}

// Listening coverage: what the watcher was able to hear, and for how long
{"type":"coverage","id":"01J...","start":"...","end":"...",
 "ranges":[[7000000,7300000],[10100000,10150000]],   // Hz, scanned or fixed (lo=hi)
 "receiver":"hackrf"}

// Every 5 min: health for the contributor list and debugging
{"type":"status","id":"01J...","ts":"...","software":"openrms-watch 0.2.0",
 "os":"linux-aarch64","sdr":"HackRF One","decoders":8,"cpuPct":41.0,
 "calibrated":true,"ppm":-24.06,"uptimeS":86400}
```

### Division of work (decided 2026-11-25)

- **Watcher:** decodes audio (Mercury already drops frames that fail their CRC), folds repeats
  into attempts, filters obvious junk, and attaches its own naming guess as a `hint`.
- **Atlas:** validates every upload anyway (it can't trust a client's own judgment), resolves
  names against the current list, combines evidence across watchers (`confirmed`), and decides
  what the public sees (client-callsign toggle, opt-outs).

Watchers **should** send `frame` events: they are the raw evidence, and they let Atlas
re-resolve old sightings when listings change. `attempt` events are **optional** summaries; the
Atlas stores them and uses them for display, and derives its own when only frames arrive.

`hint` (on `frame` and `attempt`, optional) is the watcher's local resolution:
`{"callsign": "W1RMS-10" | null, "match": "heard" | "inferred" | "ambiguous" | "unknown",
"candidates": [...]}`. Atlas stores it and may show it until its own resolution runs,
but never treats it as authoritative and never uses it for `confirmed`.

## 3. Callsign list for local naming: `GET /api/v1/watch/callsigns`

Public, ETag'd, about 100 KB: `[{"callsign":"W1RMS-10","crc":"3DEF","gatewayId":"...","modems":["mercury"],"dialHz":[7101500]}]`
for every listed gateway callsign (all SSIDs as listed, never brute-forced). `dialHz` lists
the dial frequency (centre − 1500) of **every** listed Mercury channel of that gateway, not just
the first, so the local app can narrow candidates by frequency. Modem ids are the
Atlas's (`modes.md`). Winlink-sourced rows are left out when Atlas runs with
`WINLINK_PUBLISH=false`. The watcher refreshes it daily, so a local-only user gets names too,
without a hand-made calls file.

**CRC collisions are certain, not rare.** A 16-bit CRC over the ~2,100 callsigns (with SSID)
on today's list gives about 34 colliding pairs (birthday bound), and the count grows with the
square of the list. So a CRC maps to a *list* of
candidates. Resolution, on the server and in the local app alike, narrows by (1) a listed
Mercury channel within ±200 Hz of the heard dial, then (2) Mercury-capable gateways over others,
then (3) callsigns heard in clear in CALL/CQ frames. A match that is still ambiguous is shown as
"one of: A, B", never as a single name.

## 4. What Atlas derives (server side)

- **Sightings:** per (gateway CRC, dial ± 80 Hz): first/last heard, heard by N watchers, best SNR,
  clients count. *Heard* means the station itself transmitted (ACCEPT or CQ), as in the local app;
  a CALL alone is "called", which is weaker evidence.
- **Gateway page / API:** `heardOnAir: {lastHeard, dialHz, watchers, confidence}`. It stays
  separate from `status` (heartbeat or Winlink) and **never edits a listing**. It is independent
  evidence that a listed Mercury channel is really on the air, and where.
- **Unlisted stations:** CRCs heard as RMS (ACCEPT/CQ) that match no listing, shown as "unknown
  station, CRC 3DEF, 7.101.3 MHz". A CRC proves nothing about ownership, so these are not
  claimable. When the sysop lists the gateway, the CRC and frequency match it automatically.
- **Confidence:** one watcher = `single`; two or more independent watchers within 10 min = `confirmed`.

## 5. Live feed for the site and third parties

- `GET /api/v1/watch/stream`: **Server-Sent Events**. Each `event: frame` message is the public
  projection of an accepted frame (Section 6), plus `watcher` (public name and grid, or
  "anonymous") and the resolved `gateway` if any. Supports `Last-Event-ID` resume (last 15 min).
  Filters: `?band=40m&modem=mercury&gateway=<id>`.
- `GET /api/v1/watch/events?since=<ts>&limit=` gives the same projection, paged, for history.
- `GET /api/v1/watch/sightings`: the derived table, filterable like `/gateways`.

## 6. Privacy and trust

- **Client callsigns** (the users connecting to an RMS) are amateur transmissions in the clear,
  like PSKReporter spots. **Decided (maintainer, 2026-11-25):** stored, but shown to admins
  only at first. A site setting (admin toggle, default off) makes them public in the live feed,
  events API and gateway pages; while off, the public projection says "a client". Stations
  can ask to join an opt-out list, which hides their callsign everywhere public even when the
  toggle is on.
- Watcher location is shown at the precision the contributor chose; exact coordinates are never
  uploaded.
- **Abuse:** events are attributable to an account; moderators can disable a watcher and purge
  its events (new RBAC permissions `watcher: ["disable", "purge"]` for moderator and up, see
  `openRMS-Atlas/src/rbac.ts`; actions go to the audit log). Sanity checks drop impossible events (future timestamps, frequencies outside amateur
  allocations, more than ~1 frame/s sustained). A single watcher can create a sighting but not
  `confirmed`.
- **Decided (maintainer, 2026-11-25):** any account may contribute; only watchers of
  callsign-verified accounts count towards `confirmed`.

## 7. Contributor recognition

`GET /api/v1/watch/contributors?days=7`: public watchers active within `days` (default 7,
max 90): callsign, watcher name, grid (as allowed), bands covered, listening hours, frames,
first contribution date, `live` (an upload within 25 min, allowing for the 10-minute batches). The site shows it as a ranked
list with a live indicator. Accounts with `public=false` watchers are counted in totals but not
named.

## 8. Atlas implementation notes

- Hono plus Postgres. Ingest parses NDJSON line by line and inserts with
  `ON CONFLICT (watcher_id, event_id) DO NOTHING`, so duplicates count without errors.
- `watch_event` grows fast: partition by month and keep raw frames for 90 days. Sightings are
  kept indefinitely.
- SSE through Hono's `streamSSE` on the Node server. With more than one API process, fan out
  through Postgres `LISTEN/NOTIFY`. The reverse proxy must not buffer `text/event-stream`.
- Frequencies: Atlas stores the **centre**; `dialHz` here is centre − 1500, the same
  convention Pat uses for HF ARQ modes.

## 9. Versioning

The ingest body carries no version field; new event fields are additive and ignored by older
servers. A breaking change gets `/api/v2/watch/ingest`. The OpenAPI paths go into
`atlas/openapi.yaml` once this draft is agreed.

---
**Review changes by Atlas session (2026-11-25):** watcher keys use a separate
Better Auth key configuration (`configId: "watcher"`) instead of a metadata scope, and are
minted server-side; the device flow returns a session token that is traded for a key; CRC
resolution is multi-candidate with frequency disambiguation; unlisted CRCs aren't claimable;
added RBAC permissions for watcher moderation; added §8 Atlas implementation notes and
renumbered Versioning to §9. Later the same day, by user decision: the optional `attempt`
event, the `hint` field, and the "Division of work" section.
