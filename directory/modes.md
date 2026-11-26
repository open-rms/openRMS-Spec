# Directory modes and compat mapping

Status: **draft v0** (2026-11-25). Implementation: `openRMS-Directory/src/modes.ts`.

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
| `data2g` | Data2G | — | no |
| `modem73` | modem73 | — | no |
| `ion-hf` | ION HF | — | no |
| `freedata` | FreeDATA | — | no |
| `other` | Other | free text | — |

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
| 9999 | other / owner override |

## How channels appear in the Winlink-compatible feed

Pat picks its transport from the `SupportedModes` text (`VARA FM…` → varafm,
`VARA…` → varahf, `ARDOP` → ardop, `Packet` → ax25, `Pactor` → pactor). An
unknown string gets no connect URL, and an **empty** string crashes Pat.

| Modem | SupportedModes | Mode | ServiceCode |
|---|---|---|---|
| Winlink-native | Winlink's own string (`VARA 2750`, `ARDOP 2000`…) | Winlink's code | channel's |
| Mercury | `VARA` | 9001 | forced to `MERCURY` |
| Data2G, modem73, ION HF, FreeDATA | — not in the feed (no client can dial it yet) | | |
| any, with `compatSupportedModes` set by the owner | the owner's string | modem's code or 9999 | channel's, or the forced code |

**Why `MERCURY` is forced:** Mercury speaks the VARA host protocol, so Pat's
VARA HF transport drives it. A real VARA client cannot connect to it. Putting it
under `MERCURY` (the convention Mercury sysops use on Winlink today, see
`mercury/docs/WINLINK.md`) means clients that ask only for `PUBLIC` never get a
gateway they cannot reach. Users opt in with `"service_codes": ["PUBLIC", "MERCURY"]`
in Pat.

Open modes get the same treatment as soon as a client can dial them. Each new
modem adds its own service code (e.g. `DATA2G`) and a compat rule here.
