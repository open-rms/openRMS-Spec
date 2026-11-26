# Atlas modes and compat mapping

Status: **draft v0** (2026-11-25). Implementation: `openRMS-Atlas/src/modes.ts`.

## Modem ids

A channel records its **real over-the-air waveform** as a modem id plus an
optional variant (bandwidth or speed label).

| id | Label | Variants | Winlink lists it? |
|---|---|---|---|
| `packet` | AX.25 Packet | 1200, 2400, 4800, 9600, 19200, 38400 | yes |
| `robust-packet` | Robust Packet | — | yes |
| `pactor` | PACTOR | 1 · 1,2 · 1,2,3 · 2 · 2,3 · 3 · 1,2,3,4 · 2,3,4 · 3,4 · 4 | yes |
| `ardop` | ARDOP | 200, 500, 1000, 2000, 2000 FM | yes |
| `vara-hf` | VARA HF | 500, 2300, 2750 | yes |
| `vara-fm` | VARA FM | NARROW, WIDE | yes |
| `mercury` | Mercury | — | as VARA + service code `MERCURY` |
| `xara` | XARA | 500, 2300, 2750, FM NARROW, FM WIDE | as VARA + service code `XARA` |
| `data2g` | Data2G | 500, 1200, 2400 (bandwidth cap, Hz) | as VARA + service code `DATA2G` |
| `modem73` | modem73 | — | no |
| `ion-hf` | ION HF | — | no |
| `freedata` | FreeDATA | — | no |
| `other` | Other | free text | — |

### Real waveforms of the modes Winlink carries under their own service codes

- **Mercury** (Rhizomatica, GPL): its own waveform. It speaks the VARA host protocol,
  so it cannot reach a VARA station on the air.
- **XARA** ([NR5P/xara](https://github.com/NR5P/xara), MIT, Rust): an open
  VARA-compatible modem. It has HF modes 500, 2300 and 2750 and FM modes narrow and wide.
  It speaks the VARA host protocol (8300/8301) and KISS on 8100. Its authors test it
  against VARA HF 4.9.0 and FM 4.4.0, but only over audio loopback, so on-air interop
  with real VARA is a claim, not yet established. Until it is, the feed keeps XARA under
  its own code.
- **Data2G** ([arodland/Data2G](https://github.com/arodland/Data2G), Artistic-2.0): its
  own adaptive waveform. It is mostly OFDM (PSK/QAM) with FSK modes for weak signals, at
  200 to 2400 Hz, and runs on HF plus a separate FM build. Its README says "A Data2G
  station can't connect to a VARA station." It exposes a VARA-compatible host interface
  (8300/8301), so Pat drives it as VARA HF.

## openRMS `Mode` codes

Winlink's codes run 0–54 (one stray `1200` in the wild). openRMS codes start at
9000, so a client that maps Winlink codes to names never mistakes an open mode
for a Winlink one. For example, EmComm Tools keeps only codes 0, 41, 43, 50–54.

| Code | Modem |
|---|---|
| 9001 | Mercury |
| 9002 | Data2G |
| 9003 | modem73 |
| 9004 | ION HF |
| 9005 | FreeDATA |
| 9006 | XARA |
| 9999 | other / owner override |

## How channels appear in the Winlink-compatible feed

Pat picks its transport from the `SupportedModes` text (`VARA FM…` → varafm,
`VARA…` → varahf, `ARDOP` → ardop, `Packet` → ax25, `Pactor` → pactor). An
unknown string gets no connect URL, and an **empty** string crashes Pat.

| Modem | SupportedModes | Mode | ServiceCode |
|---|---|---|---|
| Winlink-native | Winlink's own string (`VARA 2750`, `ARDOP 2000`…) | Winlink's code | channel's |
| Mercury | `VARA` | 9001 | forced to `MERCURY` |
| XARA | `VARA`, `VARA 500`, `VARA 2750`, `VARA FM`, `VARA FM WIDE` (by variant) | 9006 | forced to `XARA` |
| Data2G | `VARA 500` for the 500 Hz cap, else `VARA` | 9002 | forced to `DATA2G` |
| modem73, ION HF, FreeDATA | — not in the feed (no client can dial it yet) | | |
| any, with `compatSupportedModes` set by the owner | the owner's string | modem's code or 9999 | channel's, or the forced code |

**Why `MERCURY` is forced:** Mercury speaks the VARA host protocol, so Pat's
VARA HF transport drives it. A real VARA client cannot connect to it. Putting it
under `MERCURY` (the convention Mercury sysops use on Winlink today, see
`mercury/docs/WINLINK.md`) means clients that ask only for `PUBLIC` never get a
gateway they cannot reach. Users opt in with `"service_codes": ["PUBLIC", "MERCURY"]`
in Pat.

XARA and Data2G get the same treatment, under `XARA` and `DATA2G`, the codes their
sysops use on Winlink (e.g. Winlink Express "PUBLIC XARA MERCURY DATA2G"). Other open
modes get it as soon as a client can dial them. Each new modem adds its own service code
and a compat rule here.

## Winlink import

The Atlas asks Winlink for the service codes in `WINLINK_SERVICE_CODES` (default
`PUBLIC,MERCURY,XARA,DATA2G,EMCOMM`). A VARA channel listed under `MERCURY`, `XARA` or
`DATA2G` is stored as that modem. Data2G's `VARA 500` maps to the 500 cap; other VARA
strings map to 2400. Winlink-listed channels go back out in the compat feed with
Winlink's own `SupportedModes`, `Mode` and `ServiceCode`, unchanged.
