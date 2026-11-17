# shoppingcart-status

Out-of-band **service status file** for the ShoppingCart app. It lives here — on GitHub Pages, **off** the
production box — so the apps can learn about planned maintenance or an outage *even when the box itself is
down and can't answer for itself*.

The mobile apps read `status.json`; it is **advisory, never authoritative**. It never stops the app from
calling the API (the one exception is the `minAppVersion` force-update gate). Only a successful API response
clears a failure state, so a stale `maintenance` flag can never keep a healthy app looking broken.

## URLs

- **Primary:** `https://status.gameexperts.net/status.json`
- **Fallback:** `https://raw.githubusercontent.com/dbayev/shoppingcart-status/main/status.json`

The fallback is used when the primary fails, so the apps don't depend on the `status.gameexperts.net` DNS /
Pages path. (Once a custom domain is set, `*.github.io` redirects to it, so the `github.io` URL is **not** a
valid fallback — use the `raw.githubusercontent.com` one above.)

## `status.json` schema (v1)

```json
{
  "schema": 1,
  "state": "ok",
  "updatedAt": "2026-11-15T00:00:00Z",
  "until": null,
  "minAppVersion": { "android": "1.0.0", "ios": "1.0.0" }
}
```

| Field | Type | Rules |
|---|---|---|
| `schema` | int | Currently `1`. Clients ignore unknown fields. If `schema` is higher than a client supports, it uses only `state` and `minAppVersion` when parseable; otherwise treats the file as unavailable. |
| `state` | `"ok"` \| `"maintenance"` \| `"outage"` | Any unknown value is treated as `"ok"`. The app turns the state into its **own** localized banner text (all supported languages) — there is **no `message` field**. |
| `updatedAt` | ISO-8601 UTC | Informational. |
| `until` | ISO-8601 UTC or `null` | Expected end; rendered in the device's local time when present. |
| `minAppVersion` | object per platform (semver) | The **off-box mirror** of the force-update floor. The app's primary source is the in-band `app_config.MIN_APP_VERSION_*` it reads from the API; this file is consulted when the API is unreachable. If the installed version is lower → the app shows a blocking "Update required" screen with a store link. Missing/invalid → no enforcement. **A mandatory bump means setting *both* the `app_config` floor and this mirror.** |

A filled-in maintenance example is in `status.example.maintenance.json`.

## Editing the status (manual, for now)

Edit `status.json` on `main` (GitHub web/app/`git`). Caching caveats:

- GitHub's CDN caches for **~5–10 minutes** — changes are **not** instant. The apps add a per-minute
  cache-buster (`?t=<floor(unix/60)>`), best effort.
- **Operational rule:** flip to `maintenance` **at least 15 minutes before** planned downtime. Flipping back
  late is harmless — the file never stops the app from retrying; the first successful API call clears the
  banner.

**States:** `ok` (normal), `maintenance` (planned window — set `until`), `outage` (unplanned —
set `state:"outage"`, investigate, flip back to `ok` once `/api/health` is healthy). The banner wording for
each state is built in to the app; you set only `state` (and optionally `until`).

## Hosting notes

- GitHub Pages is served from `main` root; `CNAME` holds the custom domain; "Enforce HTTPS" is on in Pages
  settings.
- The DNS `CNAME` record `status.gameexperts.net → dbayev.github.io` is managed at mydnsservice.com (the
  domain's DNS host), **not** on the production box — that independence is the whole point.
- Keep 2FA on the account that can edit this repo: whoever can edit this file controls what every installed
  app shows.
