# Atlas service codes

Status: **draft v0** (2026-11-26). Implementation: `openRMS-Atlas` (`service_code` table,
`src/data.ts` `publicDirectory`, `src/lib/codelimit.ts`).

A Winlink **service code** says who a gateway channel is for. A client such as Pat or
Winlink Express asks only for the codes it is set to (`"service_codes": ["PUBLIC"]` by
default) and never sees channels listed under other codes.

## Codes in use

| Code | What it is |
|---|---|
| `PUBLIC` | The default. Open to licensed amateurs (and maritime users). |
| `EMCOMM` | Emergency communications. Restricted, or only open during declared emergencies. A common client setting is "PUBLIC EMCOMM". |
| `MERCURY`, `XARA`, `DATA2G` | Community codes for open modes that Winlink clients dial as VARA (`modes.md`). |
| anything else | Group codes: MARS, SHARES, agencies, served groups, clubs. |

ARES, RACES and AUXCOMM are roles, not standard Winlink codes. Any codes those groups
use are their own. The Winlink team does not publish private group codes
("The Winlink Team does not distribute that information"), and SHARES codes are
deliberately unpublished.

## Custom codes

A sysop can list any channel under any code. A channel has a main `serviceCode` and up to
8 `extraServiceCodes`, so one frequency can be listed under `PUBLIC` and a group code at
once. Codes are 1–16 letters or digits, stored uppercase.

For Mercury, XARA and Data2G channels, the compat feed replaces `PUBLIC` and `EMCOMM`
with the mode's own code, so default clients never dial a modem they can't reach. Group
codes stay as typed: the group knows what it is dialing.

## Visibility

Every code is either **public** or **unlisted**.

| | public | unlisted |
|---|---|---|
| Site filter chips, `/api/v1/service-codes` | listed | never |
| `/api/v1/stats` (`byServiceCode` and every total) | counted | never |
| Dumps (JSON, GeoJSON, CSV) | included | never |
| `/api/v1/gateways`, `/api/v1/gateways/{id}` | shown | only when `?serviceCode=` names it exactly |
| `/api/v1/watch/callsigns` (watcher naming list) | shown | never |
| Compat feed `/gateway/status.json` | when requested | only when `ServiceCodes` names it exactly |

- `PUBLIC`, `EMCOMM`, `MERCURY`, `XARA` and `DATA2G` start public.
- A code nobody has used before is registered as **unlisted** the first time a sysop saves
  it. It works at once.
- Staff (`servicecode:edit`, admin and up) can make a code public or unlisted. `PUBLIC`
  and `EMCOMM` always stay public. Moderators can see every code (`servicecode:view`).
  Changes are audited.
- In public output, a channel's `serviceCodes` hold only the codes the request may see.
  A channel with none left is dropped, and so is a gateway left with no channels. Stored
  codes never appear in public output. Owners see their own stored codes on
  `/api/v1/me/gateways`.
- The Site shows the public codes as chips, plus a text box for any other code. It never
  suggests or autocompletes codes.

### Unlisted is not secret

Unlisted means "not enumerated". It doesn't mean hidden from someone who knows, or
guesses, the code. Anyone who types `MARS` sees every listing under `MARS`. A group
that needs its gateways kept secret should not list them in a public directory at all.

To make guessing slow:

- Each IP may look up a limited number of **distinct** non-public codes per hour
  (`UNLISTED_CODES_PER_HOUR`, default 60). Asking again for a code it already looked up
  is free, so a group member browsing with their code isn't charged twice. Over the limit,
  the answer is `429` with `Retry-After`. A single request may name at most 5 non-public
  codes (`400` otherwise), so one request can't test many guesses. Requests naming only
  public codes are never limited.
- The client IP comes from `TRUSTED_IP_HEADERS` (e.g. `cf-connecting-ip`), and only when
  the request arrives from the local reverse proxy on loopback. Otherwise it is the
  socket address.
- Answers to such requests are sent with `Cache-Control: private, no-store`, so a shared
  cache can't answer around the limit.
- An unknown code and an unlisted code with no listings take the same path and return
  the same empty answer.
- An IP that keeps asking for non-public codes that match nothing is logged every
  10 misses an hour. The log line holds counts only, never the codes.

### On-air sightings

Watcher sightings (`watch/feed.md`) can name any listed gateway by callsign, including one
listed only under an unlisted code, because the callsign was heard on the air. Sightings,
events and the live stream never carry service codes.

## Winlink import

The Atlas fetches only the codes in `WINLINK_SERVICE_CODES` that are **public** in the
table. Any other configured code is skipped, with a log line. Channels in a snapshot
under a non-public code are dropped. Never add MARS, SHARES or another private group's
code to the import.
