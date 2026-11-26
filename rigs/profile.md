# openRMS Rigs profile format (v1)

> Status: **draft**, 2026-11-26. Contract: `profile.schema.json` (JSON Schema 2020-12) and
> `cat-allowlist.json`.
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
| `cat` | rigctld `send_cmd_rx <cmd> 0` for sets, `send_cmd` for reads | For menu items Hamlib can't reach. `ascii` (Kenwood/Yaesu/Elecraft) or `civ` (Icom CI-V body in hex, framed by the client). `read` + `reply` regex (group 1 = value) + `set` template (`{value}`, `{value:03}`). Always reviewed by a moderator. |
| `manual` | the person, on the radio | `manual.menu` is the menu path, `manual.value` the setting. Clients show it as a checklist item. |

Common fields:

- `check_only: true` means verify the value and warn, but never set it. Example: AGC must not be OFF.
- `expect` gives the values a check accepts: `min`, `max`, `one_of`, or `not` (values that are
  a problem). Without it, a check wants `value` exactly, within one Hamlib step for levels
  (see "Checks").
- `safety: tx` marks a step that changes transmit behaviour (power, mic gain, compressor,
  modulation source). Clients ask the operator before applying it. The server raises a step to
  `tx` by the rule in "Safety".
- `menu` is where the setting lives in the radio's own menus, outermost first
  (`["OPERATION SETTING", "TX AUDIO", "PRMTRC EQ1"]`). Any step can have it. Clients show it
  as the manual fallback when a step can't be applied over CAT. In the apply-plan, `menu` falls
  back to `manual.menu` split on `>`. Front-panel controls count too (`["FUNCTION", "NB"]`).
  The server warns (never rejects) when a `level`, `func`, `parm` or `cat` step has no `menu`,
  because people who set the radio up by hand need it.

## Safety

The server raises a step to `safety: tx` (in the stored document and in the apply-plan) when:

- it is a `level` named RFPOWER, MICGAIN, COMP, VOXGAIN, VOXDELAY, ANTIVOX, KEYSPD, BKINDL,
  BKIN_DLYMS, USB_AF_INPUT or TXEQ;
- it is a `func` named COMP;
- it is a `cat` step that sets a value (not `check_only`) and its `set` command, with each
  placeholder filled with zeros, contains a command from a `safety: tx` family in
  `cat-allowlist.json`.

The author can mark any other step `tx`; the server never lowers one. The apply-plan's
`safety` is final, so clients don't have to repeat this rule.

## Checks

`POST /check` (and any client that checks locally) compares each step with the value read from
the radio, keyed by step id:

- `manual` steps are always `manual`. A missing or null reading is `unknown`.
- If `expect` has any field, it replaces `value` entirely: the reading must not equal any `not`
  entry, must equal a `one_of` entry when given, and must be within `min`/`max` (inclusive, no
  step tolerance).
- Otherwise, with no `value`, the step is `ok`.
- Numbers: `|reading − value| ≤ max(t, 1e-6)`. For `level` steps `t` is the step of that
  level's **get** range in `rigctl --dump-caps` (`RFPOWER(0.05..1/0.003922)` gives 0.003922;
  0 when the range is `0..0` or absent), taken from Rigs' catalog and given in the apply-plan
  as `tolerance`. Local checks should use the plan's `tolerance`, not their own Hamlib's, so
  they agree with `/check`. For every other kind `t` is 0. Readings may be numbers
  or numeric strings (`"050"` equals 50).
- Booleans (`func`): the reading counts as true only for `true`, `1`, `"1"` or `"true"`
  (any case).
- Strings: equal ignoring case.
- `mode` steps compare the **mode name only**. Send what rigctld's `m` returns on its first line
  (`PKTUSB`), not the passband; `passband_hz` is applied but never checked.
- `cat` steps: send capture group 1 of `reply` as a string.

`examples/check-vectors.json` has conformance cases for these rules and for the CAT allow-list.
openRMS-Rigs runs them in its tests.
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
- raw CAT outside the allow-list in `cat-allowlist.json` (see "Raw CAT allow-list");
- non-numeric level values (other than the catalog's named choices) and CAT values that aren't
  plain letters and digits. Commands can't contain spaces or line breaks. The check runs on
  the command as rendered with the value;
- PTT and vfo_op TUNE, which can't be written in the format at all.

Clients should still enforce their own rules. rigctlWeb refuses any plan that breaks them.

## Raw CAT allow-list

`cat-allowlist.json` is the machine-readable list of the raw CAT that a profile may contain
and that a client may send from one. openRMS-Rigs loads it at startup instead of hard-coding
it, and rigctlWeb uses the same file, so both enforce the same rules. It has, per protocol
(`ascii` for Kenwood, Yaesu and Elecraft; `civ` for Icom):

- `families`: allowed command prefixes, each saying whether it may `read` and/or `set`, and its
  `safety`. When several prefixes match, the longest decides (CI-V `140A`, RF power, is `tx`
  inside `14`, levels).
- `denied`: prefixes refused even if a family matches, with a reason: transmit, MOX, VOX,
  break-in, tuner, sending CW or messages, memory writes, frequency (use rigctld), power.
  Anything that matches no family is refused too; the denied list makes the intent explicit
  and gives test material.
- `command_pattern` and the rules: check every command as sent (the read command, the set
  template filled with zeros, and the set template rendered with the value); split ascii on `;`
  and check each part; CI-V is the hex body only, without preamble, addresses or FD. Values
  must match `value_pattern`.

### Step rules: read, set and reply are bound together

ASCII CAT can't tell a read from a set by syntax: `PC100;` *sets* power to 100 although `PC`
allows reads. Clients send `read` before the operator confirms anything (to snapshot the
current value), so a `read` that is really a set would change the radio unasked. The server
therefore rejects a `cat` step unless (`step_rules` in `cat-allowlist.json`):

- `set` has exactly one `{value}` or `{value:NN}`, at the end (just before `;` for ascii), and
  ascii `set` is a single command;
- `read` equals `set` with the placeholder removed: `set: "EX010413{value:03};"` means
  `read: "EX010413;"`. This applies to `check_only` steps too: they carry a `set` template
  only to check the read against, and it is never sent. A step without `set` may have a `read`
  only when the read's family is read-only (`set: false`, e.g. meters);
- `reply` is given whenever `read` is, and is exactly `^` + the read without its `;` + one
  capture group + `;$` (CI-V: no `;`): `^EX010413(\d{3});$`. The group may use only letters,
  digits and `\ [ ] { } , + * ? -`.

Moderators: check that the menu item number in `set` is the setting the step names, from the
radio's CAT reference, and that `read` is the bare query for the same item. Clients that check
locally should enforce the same rules; `examples/check-vectors.json` has cases (`steps`).

With rigctld, send sets with `send_cmd_rx <cmd> 0` (no reply expected). Plain `send_cmd`
waits for a reply a set never sends, about 2 s per command on the FT-DX10 backend.

Menu items (Yaesu `EX`, Icom `1A05`) are allowed as a family. Some of them change transmit
behaviour or assign keying to a serial line, so moderators review every `cat` step, and authors
should mark such steps `safety: tx`. Changes to the file follow the Spec's versioning: adding a
denied prefix is a patch, allowing a new family is a minor version.

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

To send a person to a profile, link `https://rigs.openrms.dev/p/{id}` (the deployment's site
URL + `/p/{id}`). That URL is canonical and stays the same across revisions. Unknown or
non-public ids return 404, and the page's `<title>` is the profile title.

For offline use (field, EmComm), bundle `GET /dump.json` and use ETags to refresh it. Each
entry carries its `plan` (the same as the apply-plan, with `readable`, `safety`, `tolerance` and
`menu`), so an offline client needs no catalog.
