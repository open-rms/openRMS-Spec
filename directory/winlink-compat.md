# Winlink-compatible feed

Status: **draft v0** (2026-11-25). Implementation:
`openRMS-Directory/src/compat/winlink-status.ts`.

`GET|POST /gateway/status.json` returns the shape of
`https://api.winlink.org/gateway/status.json`, so software that reads Winlink's
list reads ours unchanged.

## Format rules (each one breaks a real client if wrong)

- PascalCase keys: `Gateways[]` → `Timestamp, Callsign, BaseCallsign, Latitude,
  Longitude, HoursSinceStatus, LastStatus, Comments, RequestedMode,
  GatewayChannels[]` → `Timestamp, SupportedModes, Mode, Gridsquare, Frequency,
  OperatingHours, Baud, RadioRange, Antenna, ServiceCode`; then `ResponseStatus: {}`.
- `Frequency` is an integer in **Hz**, the centre frequency. Pat subtracts
  1500 Hz for the dial on ARDOP/VARA/PACTOR HF.
- `LastStatus` is Go's `time.RFC1123` layout with zone `UTC`:
  `Sun, 01 Nov 2026 07:53:00 UTC`. Pat refuses the **whole list** on one bad value.
- `Timestamp` is `"\/Date(ms)\/"`.
- `SupportedModes` is never empty (Pat indexes its first word).
- `Latitude`/`Longitude` are always present; EmComm Tools places gateways by them.
- `Baud` and `RadioRange` are strings.

## Parameters

Same as Winlink: `Mode` (AnyAll, Packet, Pactor, RobustPacket, AllHf),
`HistoryHours` (default 48), `ServiceCodes` (repeat or comma-separated; default
`PUBLIC`), `key` (accepted, ignored). openRMS extensions: `source=openrms|winlink`,
`extensions=1` (adds an `OpenRMS` object with the real modem).

## What reads this today (checked 2026-11-25)

| Client | How it gets the list | Works with openRMS by |
|---|---|---|
| Pat v1.0.1 | POSTs to a hard-coded `api.winlink.org`; caches as `$XDG_DATA_HOME/pat/rmslist[-CODES].json` and reads the cache when present | writing our feed to that cache file (`pat-rmslist-sync.sh`); later an upstream option for the list URL |
| EmComm Tools Community | `et-winlink-data` runs jq over Pat's `rmslist.json` (`Callsign, BaseCallsign, Latitude, Longitude, SupportedModes, Mode, Frequency`; keeps Mode 0, 41, 43, 50–54). It ships a prebuilt `rmslist.csv`, so `et-winlink-data` must be re-run after the cache changes | the same cache file (verified: ETC's own jq filter reads our feed); or point its script at `/gateway/status.json` |
| 73 Linux (PatMenu3) | `getardoplist` runs `pat rmslist … --force-download` and parses the text | **not yet**: `--force-download` skips the cache and re-fetches from Winlink, overwriting our file. Needs a PatMenu change (drop `--force-download` after a sync, or call our feed) or the upstream Pat option |

Test: `openRMS-Directory/test/compat.test.ts` re-serializes a 2,125-gateway
Winlink sample and checks that the real `pat rmslist` prints byte-identical
output for ours and for Winlink's.
