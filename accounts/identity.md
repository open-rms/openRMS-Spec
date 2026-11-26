# openRMS identity: Accounts → services and apps (draft v0)

> Status: **draft for review**, 2026-11-25; the Directory was renamed **Atlas** on 2026-11-26; 2FA, radios, profile visibility and avatars added 2026-11-26. Implemented by openRMS-Accounts. Consumed by
> openRMS-Atlas, openRMS-Site, openRMS-Rigs, openRMS-Watch, later openRMS-Gateway,
> openRMS-Exchange and third-party apps. Open questions are marked **Q**.

One account per person (or per callsign holder), one login, shared profile. openRMS-Accounts
is the only place that stores passwords, sessions, callsigns, clubs and staff roles. Every
other service is a **relying party**: it trusts tokens Accounts signs, and stores only the
account id (`sub`) next to its own data.

## 1. Issuer and hosts

| Host | What |
|---|---|
| `https://auth.openrms.dev` | **Issuer.** Login, sign-up, consent, device-link page, account settings, OIDC endpoints |
| `https://openrms.dev` | Home: project landing page (static, no login) |
| `https://atlas.openrms.dev` | Atlas (formerly the Directory): the Atlas API and its website (openRMS-Site) on the same origin |
| `https://directory.openrms.dev` | 301 redirect to `atlas.openrms.dev` (old name; not an audience) |
| `https://watch.openrms.dev` | Watch downloads and docs (static, no API) |
| `https://rigs.openrms.dev` | Rigs |

One subdomain per project; no project takes a generic `api.` name (user decision, 2026-11-25).

- Discovery: `https://auth.openrms.dev/.well-known/openid-configuration` (also
  `/.well-known/oauth-authorization-server`). Keys: the `jwks_uri` it advertises.
- **The issuer is configuration, never a constant.** Every service and app takes
  `OPENRMS_ISSUER` (default `https://auth.openrms.dev`), so a self-hosted stack runs its own
  Accounts. A verified callsign is trusted **per issuer**: a service accepts the
  `callsign_verified` claim only from issuers it is configured to trust.
- `.dev` is on the HSTS preload list: HTTPS everywhere, including staging. Local development
  uses `localhost` (Accounts 8789, Atlas 8787, Rigs API 8790 and site 5174).

## 2. How each kind of client signs in

| Client | Mechanism |
|---|---|
| First-party web app on `*.openrms.dev` (Site, Rigs UI) | Shared session cookie on `.openrms.dev`. The browser gets a short-lived JWT from Accounts and sends it to its service as `Authorization: Bearer` (§3a) |
| First-party server-rendered page | May forward the cookie to Accounts' session endpoint (§3b) |
| Third-party web or mobile app | OIDC authorization code + **PKCE**, registered client |
| Headless or desktop device (Watch, rigctlweb, hamstack) | OAuth device authorization grant (RFC 8628), public client id per app |
| Service-to-service (Atlas → Accounts lookups, webhooks) | OAuth client credentials |
| Machines that post data (gateway heartbeats, watcher uploads, rig agents) | **API keys minted by the service that owns the data** (§5), not by Accounts |
| Translator and Gateway **local** control UIs | **Never** a central login. Local auth only, so radios work with no internet. Linking an account is optional and only used for reporting to Atlas |

## 3. Tokens

Access tokens are JWTs signed by Accounts (asymmetric, published in JWKS). Services verify them
**offline** against the cached JWKS; no shared secret, no call to Accounts per request.

Every service checks: `iss` is a trusted issuer, `aud` is the service's own audience, `exp`,
and the signature. Audiences:

| Service | `aud` |
|---|---|
| Atlas | `https://atlas.openrms.dev` |
| Rigs | `https://rigs.openrms.dev` |

Clients ask for an audience with the `resource` parameter (RFC 8707). Checked in
`@better-auth/oauth-provider` 1.7.28: the **device grant** accepts `resource` (one or many) on
the device authorization request, binds it to the user's approval, and refuses any other
resource at the token endpoint (`invalid_target`). So Watch sends
`resource=https://atlas.openrms.dev` and gets a token with Atlas as `aud`.
**Q:** confirm the same for the authorization-code grant once the first third-party web
client exists.

### Device flow, step by step (as implemented)

1. Device: `POST {issuer}/api/auth/device/code` (form) `client_id=openrms-watch`,
   `scope=openid callsign clubs offline_access`, `resource=<atlas>` →
   `{device_code, user_code, verification_uri, verification_uri_complete, interval, expires_in}`.
2. Approval page (signed in): `GET {issuer}/api/auth/device?user_code=…` **claims** the code
   for the session, then `POST {issuer}/api/auth/device/approve {userCode}` (or `/device/deny`).
3. Device polls `POST {issuer}/api/auth/oauth2/token` (form)
   `grant_type=urn:ietf:params:oauth:grant-type:device_code&device_code=…&client_id=openrms-watch`
   no faster than `interval` (else `slow_down`) until it gets the token set (access token
   `aud=<atlas>`, refresh token).

A client may only request resources it is linked to; `openrms-watch` is linked to the
Atlas only.

Lifetimes: access token 15 min, refresh token 30 days (rotating, revoked on password change,
ban or sign-out everywhere), session cookie 7 days sliding.

### 3a. First-party browser flow

1. The user signs in at `auth.openrms.dev`. The session cookie is set for `.openrms.dev`.
2. The app calls `GET https://auth.openrms.dev/api/v1/token?aud=<service>` with credentials
   (CORS allows the trusted origins only) and receives
   `{token, token_type: "Bearer", expires_in: 900, expires_at}`. `aud` must be a configured
   first-party audience; anything else is `400 invalid_target`. The token carries the
   first-party scopes `openid profile email callsign clubs radios` and `role`, and no `client_id`.
3. The app calls its service with `Authorization: Bearer <jwt>` and refreshes before expiry.

### 3b. Server-side session check

`GET https://auth.openrms.dev/api/auth/get-session` with the forwarded cookie. Cache the
answer for at most 60 s. Use this only where 3a does not fit. There is **no** shared-secret
introspection endpoint. (OAuth token introspection, RFC 7662, is available to registered
confidential clients for opaque or refresh tokens.)

## 4. Claims and scopes

`sub` is the account id. It never changes. **Key your data on `sub`, never on the callsign**:
moderators can change an account's callsign.

| Scope | Claims |
|---|---|
| `openid` | `sub` |
| `profile` | `name`, `grid`, `picture` (URL of the avatar proxy, §6a; only when the avatar is set and public) |
| `email` | `email`, `email_verified` |
| `callsign` | `callsign` (bare, upper case, no SSID), `callsign_verified` (bool), `callsign_verified_method` (`moderator`, later `lotw`, `license-db`, `on-air`) |
| `clubs` | `clubs`: `[{"id","callsign","role":"owner"\|"admin"\|"member","verified":bool}]` |
| `radios` | `radios`: Hamlib model macros of the user's radios, e.g. `["RIG_MODEL_FTDX10","RIG_MODEL_IC7300"]`, primary first, at most 20 (agreed with openRMS-Rigs) |
| `offline_access` | refresh token |
| `accounts.read` | client credentials only: the service API (§6) |

First-party tokens also carry `role` (`user`, `moderator`, `admin`, `superadmin`). It says
**who the person is to openRMS**. What a role may do in a service is decided by that service's
own permission table (for example Atlas's `gateway:hide`). Third-party clients never
receive `role`.

### 4a. Profile visibility

Each profile field has a visibility: `public` (anyone), `members` (anyone signed in to openRMS)
or `private` (only the owner and openRMS staff tools). Defaults:

| Field | Default | Notes |
|---|---|---|
| `callsign` | public | fixed: the callsign is the handle and a public record |
| `name` | members | |
| `grid` | private | at `public`, shown to 4 characters at most (e.g. `CN87`), never 6 |
| `city` | private | free text, at most 64 characters |
| `bio` | public | at most 280 characters |
| `links` | public | at most 5 https URLs, each `{kind: "qrz"\|"website"\|"other", url}` |
| `avatar` | public | §6a |

A user can also hide the whole public profile (`hidden`); then §6a answers 404 to everyone but
the owner.

How visibility reaches tokens:
- **Third-party** tokens (any `client_id` that is not first-party): under `profile`, a `private`
  field is `null`, `grid` is cut to what its visibility allows, and `picture` is absent unless
  the avatar is `public`. The consent screen says "as you share them".
- **First-party** tokens and the service API (§6) carry full values, because services use them
  server-side (Atlas computes distances from `grid`). **Anything a service shows to other people
  must come from the public profile (§6a) or honour the `visibility` object (§6), never from a
  token's claims.**

### 4b. Staff second factor

Staff (`moderator`, `admin`, `superadmin`) must have a second factor: TOTP or a passkey. A staff
account without one is treated as `user`: first-party tokens carry `role: "user"` and Accounts'
admin API refuses it (`403 staff actions need 2FA`). Nobody can be given a staff role before
they have enrolled. Recovery when every factor is lost is a superadmin command on the Accounts
host (`reset-2fa`), audited, which also ends the account's sessions and refresh tokens.
Second factors do not change the device flow or §3a: both run inside an Accounts session that
has already passed them.

Claims in a token are as fresh as the token (at most 15 min). A service that needs the current
value for a sensitive action (deleting, transferring ownership) asks Accounts (§6).

## 5. Service-owned API keys

Keys that let a machine act on a service's data are minted and checked by that service, with
the owner's `sub` stored on the key. A key only works at the service that issued it.

| Prefix | Service | Purpose |
|---|---|---|
| `orms_` | Atlas | gateway heartbeats |
| `ormsw_` | Atlas | watcher uploads (`../watch/feed.md`) |
| `ormsr_` | Rigs | rig agents |

Rules: keys are stored hashed, the kind of key is fixed at creation (never read from
user-editable metadata), each has its own rate limit, and a user can list and revoke their
keys. A device flow token (§2) is used once to mint the key, then discarded.

**Migration:** keys issued before the split keep working unchanged: same hash, same owner
(the account id becomes `sub` with the same value), same watcher. Atlas copies them
into its own table in AC1; no deployed watcher or gateway needs re-linking.

**Watch linking after the split** (agreed with openRMS-Watch): the approval page on Accounts
knows nothing about watchers. The watcher app, holding the device token
(`aud=https://atlas.openrms.dev`), calls `GET /api/v1/me/watchers` to let the user pick, then
`POST /api/v1/watch/link {watcherId} | {name, gridsquare?}`, which creates the watcher if
needed, creates or rotates its key, and returns `{key, watcherId, name}`. Both accept the
Accounts JWT as `Authorization: Bearer`.

## 6. Accounts API for services (client credentials)

Get a token: `POST {issuer}/api/auth/oauth2/token`, HTTP Basic with the service's client id and
secret, form `grant_type=client_credentials&resource={issuer}`. The token's `sub` is the client
id and its scope is `accounts.read`. Call the endpoints below with `Authorization: Bearer`.
`GET /api/v1/users?updatedSince=<ISO>` lists changed accounts for reconcile. Every change bumps
the account: profile, role, verification, ban, and club changes (membership for that member;
club verification, update or deletion for all its members). Each user payload carries its
current `clubs`, so polling users is enough. `GET /api/v1/clubs` lists every existing club,
so a service can drop deleted ones.

- `GET /api/v1/users/{sub}`: `{sub, name, callsign, callsign_verified, callsign_verified_method, grid, role, banned, clubs, radios, visibility}`.
  `visibility` maps each §4a field to `public`/`members`/`private` and adds `hidden` (bool).
- `GET /api/v1/users/{sub}/radios`: `[{model: "RIG_MODEL_FT991", hamlib_model: 1035, label: "FT-991A", primary: bool}]`.
  `model` (the Hamlib macro) is the key; `hamlib_model` is the fallback should a macro be renamed;
  `label` is the user's own name for it (one macro can cover several products).
- `GET /api/v1/users?callsign=W1AW`: resolve a callsign to `sub` (for "add member", claims).
- `GET /api/v1/clubs/{id}` and `GET /api/v1/clubs/{id}/members`.

## 6a. Public profile (no client credentials)

- `GET {issuer}/api/v1/profiles/{callsign|sub}`: `{sub, callsign, callsign_verified, name?, grid?,
  city?, bio?, links?, picture?}` with only the fields the caller may see: anonymous callers get
  `public` fields; a first-party bearer token or an Accounts session adds `members` fields.
  `private` fields are never returned. Hidden, banned or unknown accounts: 404. Rate limited.
  The profile page is `{issuer}/u/{callsign}`.
- `GET {issuer}/api/v1/avatars/{sub}?s=64|128|256`: the avatar as WebP, or 404 (show a callsign
  monogram). Accounts fetches the source (Gravatar, opt-in, or a user's https URL) itself, so
  browsers never see the email hash and image hosts never see viewers. It checks the type by
  magic bytes (PNG, JPEG, WebP, GIF; never SVG), refuses private addresses and more than 2 MB,
  and re-encodes. Cached; honours `ETag`.

## 7. Events (webhooks)

Accounts posts signed JSON (`openrms-signature: t=<unix>,v1=<hmac-sha256>`, one secret per
subscriber) to each registered service, retried with backoff from an outbox table:

`user.updated` (incl. callsign, profile, visibility and radios changes), `user.callsign_verified`, `user.banned`,
`user.deleted`, `club.created`, `club.deleted`, `club.member_added`, `club.member_removed`,
`club.verified`.

Every `club.*` event carries the club's `id`, `callsign` and `name`; member events add `sub`
(and `role`). User events carry `sub`.

Services must handle each event idempotently (`id` field) and may also reconcile on a timer.

**Local account cache.** A service that needs facts about *other* accounts (not the caller)
keeps a cache table fed by these events plus a timed reconcile through §6: `sub`, `callsign`,
`callsign_verified`, `banned`, club memberships. Example: Atlas uses callsign
verification in SQL (gateway dedupe, club `ownerVerified`, sightings `confirmed`, contributor
list). Rules:
- **Writes check the cache, not only the token**, for club membership and bans. A token can be
  up to 15 min old; `club.member_removed` and `user.banned` must cut access at once.
- `user.banned` also disables that account's service-owned keys (§5) in every service.
Example: Atlas deletes a club's own listings and releases its Winlink claims on
`club.deleted` (its webhook handler).

## 8. Registered clients (initial)

| client_id | Type | Grants |
|---|---|---|
| `openrms-site` | first-party, public | cookie + JWT (3a) |
| `openrms-rigs` | first-party, confidential | cookie + JWT, client credentials |
| `openrms-atlas` | service, confidential | client credentials |
| `openrms-watch` | public | device code |
| `rigctlweb` | public | device code, linked to Rigs only; may ask for `radios` |
| `hamstack` | public | device code, linked to Rigs only; may ask for `radios` |

## 9. Shared verifier

`@openrms/service-auth` (TypeScript, in openRMS-Accounts) exports
`verifyAccessToken(token, {audience, issuers?}) → claims`. It returns the §4 claims (`sub`,
`callsign`, `callsign_verified`, `callsign_verified_method`, `clubs`, `role`, …) or throws, and
caches the JWKS. Atlas and Rigs use it. For Rust devices, `openrms-auth` (openRMS-Libs) does the device flow.

## 10. Security notes

- The session cookie goes to **every** `*.openrms.dev` host. Only openRMS-run services may
  live under `openrms.dev`. No user content, previews or third-party hosting on subdomains.
- Cookies are `Secure`, `HttpOnly`, `SameSite=Lax`. CSRF: Better Auth checks `Origin` on
  `/api/auth/*`; Accounts refuses cookie-carrying state changes from untrusted origins on its
  own routes (`/api/v1/me*`, `/api/v1/admin/*`). Services that accept the cookie directly
  (§3b) must do the same. Services that accept only bearer tokens are not exposed.
- A ban, and any password change or reset, revokes the account's refresh tokens at once.
  Access tokens already issued run out within 15 min; services cut access sooner from the
  `user.banned` event (§7).
- Staff need a second factor (§4b). Passkeys are bound to the issuer's host
  (`auth.openrms.dev`); a self-hosted issuer has its own.
- Each service has a different `aud`, so a token stolen from one service cannot be replayed
  at another.
