# openRMS Rigs profile format (v1)

> Status: **draft**, 2026-11-25. Contract: `profile.schema.json` (JSON Schema 2020-12).
> API: `openapi.yaml`. Implemented by openRMS-Rigs. Consumed by rigctlWeb, hamstack, and anyone
> who reads or writes profiles. Licence for profile content: **CC-BY-SA-4.0**.

A profile describes how to set up **one radio model** for **one operation**. YAML and JSON are
two ways of writing the same document, and the API converts between them (`POST /convert`).

## Names and units

- Radios are identified by their Hamlib **macro** (`RIG_MODEL_IC7300`). That name stays the same
  across Hamlib releases. `hamlib_model` (the number passed to `rigctld -m`), `mfg` and `name`
  are copied from `rigctl --dump-caps` so clients can match on any of them.
- Levels, functions, parameters and modes use Hamlib's names **exactly** as dump_caps prints
  them, in Hamlib's units. `RFPOWER` is a 0–1 fraction, not watts; use `display: "~35 W"` for
  people. `GET /rigs/{macro}` lists each level's range and unit.
- Version ranges are space-separated comparators that must all hold: `">=4.6 <4.8"`. Use
  `hamlib` for the profile as a whole (what it was tested with) or for one step (what that
  step needs). Use `firmware` for firmware or Yaesu menu-table versions.

## Step kinds

| kind | Set through | Notes |
|---|---|---|
| `mode` | `M <mode> <passband>` | `value` is a Hamlib mode (`PKTUSB`). `passband_hz` 0 means the radio's default. |
| `level` | `L <name> <value>` | Validated against the catalog's range for that radio. |
| `func` | `U <name> 0/1` | Allow-list only (the same as rigctlWeb's v1). Keying functions are rejected. |
| `parm` | `P <name> <value>` | |
| `cat` | rigctld `send_cmd` (`w`) | For menu items Hamlib can't reach. `ascii` (Kenwood/Yaesu/Elecraft) or `civ` (Icom CI-V body in hex, framed by the client). `read` + `reply` regex (group 1 = value) + `set` template (`{value}`, `{value:03}`). Always reviewed by a moderator. |
| `manual` | the person, on the radio | `manual.menu` is the menu path, `manual.value` the setting. Clients show it as a checklist item. |

Common fields:

- `check_only: true` means verify the value and warn, but never set it. Example: AGC must not be OFF.
- `expect` gives the values a check accepts: `min`, `max`, `one_of`, or `not` (values that are
  a problem). Without it, a check wants `value` exactly, within one Hamlib step for levels.
- `safety: tx` marks a step that changes transmit behaviour (power, mic gain, compressor,
  modulation source). Clients ask the operator before applying it. The server raises a step to
  `tx` when it knows the name changes transmit behaviour.
- Audio targets (`audio.rx_target_dbfs`) let a client measure a level and give advice when the
  level can't be set over CAT.

## Sources and confidence

`sources` lists where the settings come from: `{id, title, author?, url?, date?, note?}`, with
ids `S1`, `S2`, …. Steps cite them by id (`sources: [S1, S3]`), so citations survive reordering and
clones. The server rejects a citation of an id that isn't listed. A step can also carry
`confidence`:
- `high`: the manual, or three or more sources, agree;
- `medium`: some sources support it;
- `low`: thin support.

Profiles are CC-BY-SA-4.0. Record settings as facts with a link and credit; don't copy text
from manuals or articles. Keep an `unverified` tag until someone reports the profile working on
air. Sources stay in the document but are left out of the apply-plan.

## Never allowed

A profile can't contain anything that **keys the transmitter**. The server rejects it with a 422:

- the functions VOX, SBKIN, FBKIN, TBURST, TUNER and anything starting with SEND_;
- raw CAT outside an **allow-list** of command families that only read or change settings.
  ascii (Kenwood, Yaesu, Elecraft) allows EX, MN, MP, MD, DT, GT, AG, RG, SQ, RA, PA, IS, SH, BW,
  FW, NA, NB, NL, NR, BC, CO, AN, AP, RL, ML, PC, MG, PR, KS, KP, SM and RM, with every
  `;`-separated command checked. CI-V allows first bytes 04, 06, 11, 14, 15, 16 and 1A, but not
  1646 (VOX) or 1647 (break-in). So TX, MX, VX, BI, AC, KY, PB, SW*, TT, ZZ* and Icom
  1C, 17 and 28 are all refused;
- non-numeric level values (other than the catalog's named choices) and CAT values that aren't
  plain letters and digits. Commands can't contain spaces or line breaks. The check runs on
  the command as rendered with the value;
- PTT and vfo_op TUNE, which can't be written in the format at all.

Clients should still enforce their own rules. rigctlWeb refuses any plan that breaks them.

## Lifecycle

Each edit is a new, immutable revision. Stars and "worked for me" reports point at revisions.

- Profiles from verified callsigns and staff publish at once.
- Others wait for a moderator. Any profile with `cat` steps always waits, even one from staff,
  so approving it is a separate, audited action.
- While an edit waits, the previously published revision stays public.

## Cloning ("forked from")

To start from someone else's settings, clone the profile, change only what's different and
submit with `?forked_from=<id>&forked_revision=<n>`. The new profile is a separate profile with its own
author, revisions and reports, and it records the exact parent revision. Readers see "forked
from" with a link back, a list of changes from that parent revision, and on the parent a fork
count and list. Lineage lives on the server, not in the document, so a downloaded YAML file
stays a plain profile.

## Client flow (apply-plan)

1. `GET /profiles/{id}/apply-plan` returns structured steps (kind, name, value, safety,
   readable), with rigctld lines as a convenience.
2. Read every readable step from the radio, then `POST /profiles/{id}/check` with
   `{"current": {"<step id>": value}}`. The answer gives each step `ok`, `drift`, `unknown` or
   `manual`, plus `noop`.
3. Show the diff. Ask before `safety: tx` steps. Mark steps with `readable: false` as "can't
   restore automatically".
4. Snapshot the current values, apply, and offer to restore.

For offline use (field, EmComm), bundle `GET /dump.json` and use ETags to refresh it.
