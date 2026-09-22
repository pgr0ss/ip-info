# ip-info

A single-page static site that shows your current public IP and continuously
tells you whether your device's internet connection is actually working. Built
for checking a phone's mobile data: open the page and glance at the ONLINE /
OFFLINE banner.

Live site: https://ip.pgrs.net

## What it does

- Polls `https://api.ipify.org?format=json` every 10 seconds as the
  connectivity probe and source of the current public IP. ipify documents its
  API as usable without limit.
- Fetches network details (hostname, city, region, country, org/ASN,
  coordinates, postal, timezone) from `https://ipinfo.io/json` only when the
  probe reports a new IP. A failed details fetch is retried on the next
  successful probe. Details errors are shown inside the details card and never
  affect ONLINE/OFFLINE.
- Shows a big color-coded status banner: **ONLINE** (green), **OFFLINE**
  (red), or **CHECKING…** (grey).
- Status is **recency-based**: ONLINE requires the last fetch to have both
  succeeded *and* landed within the freshness window (`STALE_AFTER`). A once-
  successful but stale state reads OFFLINE.
- **Resume rechecks instead of flashing OFFLINE.** Timers don't run while a
  phone suspends the page, so on return every reading is stale. When the tab
  becomes visible, or the 1-second tick sees no attempt within `STALE_AFTER`
  (polling stopped for any reason), the page aborts any in-flight probe, shows
  CHECKING… with "last confirmed …", and probes immediately. A probe whose
  result arrives more than `STALE_AFTER` after it started spanned a suspension
  and is discarded the same way.
- Shows the last-known IP and details even while OFFLINE.
- On failure, a red "Why it's failing" card surfaces the categorized error
  (timeout / HTTP status / unexpected response body / generic network failure)
  and a plain-English hint. A 200 whose body isn't JSON with an `ip` string
  counts as a failure, since something other than the endpoint answered.
- Shows last-success time, last-attempt time, and current time, each with live
  relative ages.
- Pauses polling while the tab is hidden and refreshes immediately when it
  returns to the foreground (`visibilitychange`).

## Architecture

Everything lives in a **single file: `index.html`** — markup, inline CSS, and
inline vanilla JavaScript. No build step, no dependencies, no framework. It can
be opened directly from disk or served by any static host.

Key JS constants (top of the `<script>` block):

- `PROBE_ENDPOINT` — `https://api.ipify.org?format=json` (IPv4-only, like
  ipinfo.io, so both report the same address family)
- `DETAILS_ENDPOINT` — `https://ipinfo.io/json`
- `POLL_MS` — 10000 (fetch cadence)
- `TICK_MS` — 1000 (re-render cadence for live relative times + staleness)
- `TIMEOUT_MS` — 4000 (per-request `AbortController` timeout)
- `STALE_AFTER` — 22000 (ONLINE freshness window; keep it above
  `POLL_MS + TIMEOUT_MS` or ONLINE will falsely blip to OFFLINE between polls)

State is a handful of module-level variables: probe state (`probeIp`,
`lastSuccessAt`, `lastAttemptAt`, `lastAttemptOk`, `lastError`, `probeCtrl`,
`rechecking`,
`pollTimer`) and details state (`lastData`, `detailsIp`, `detailsError`,
`detailsInFlight`). Status is *derived* at render time from the probe state,
never stored.

## Deployment

Hosted on GitHub Pages via `.github/workflows/pages.yml`, which deploys the
repo root on every push to `main`. `.nojekyll` disables Jekyll processing so
files are served as-is. Pages source must be set to "GitHub Actions" in repo
settings (one-time).

## Notes for future work (important gotchas)

- **ipinfo.io's unauthenticated limit is 1,000 requests/day, shared by every
  client behind the same public IP** (mobile carriers commonly put many users
  behind one CGNAT address). Polling ipinfo every 10s used up that quota in
  under 3 hours, which is why ipinfo is now only called when the probe IP
  changes. Don't move ipinfo back onto the poll loop. While details fetches
  keep failing, they are retried on every probe (every `POLL_MS`).

- **The ipinfo.io 429 rate-limit response is unreadable from the browser.**
  When rate-limited, ipinfo returns HTTP 429 **without** an
  `Access-Control-Allow-Origin` header. On a cross-origin `fetch` the browser
  therefore rejects the request as a generic `TypeError: Load failed` and never
  exposes the status or JSON body to JavaScript. That means the actual
  `{"status":429,...}` payload **cannot** be displayed. The code has a branch to
  parse an ipinfo error body (`e.apiError`), but it is effectively unreachable
  cross-origin and only works same-origin or if ipinfo ever adds CORS to error
  responses. The details error hint calls out 429 as the likely cause
  instead. To actually read the error / raise the limit, use an ipinfo token
  (`?token=...`); token'd error responses do send CORS headers.

- **All iOS browsers use WebKit**, so Safari vs Firefox differences are not
  engine/CORS differences. Failures that appear only in Firefox iOS are content
  blockers or the same rate-limiting surfacing as `TypeError: Load failed`.

- **A blocked/failed request is indistinguishable from a real outage** at the JS
  level. This is acceptable here — both mean "data not working right now" — but
  it's why the diagnostics can only *guess* at the cause.

- **No fallback endpoint** is currently implemented (considered and declined).
  If ipify is blocked, the page reads OFFLINE even when the connection works.

- After editing, sanity-check the inline JS with:
  `sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' | node --check /dev/stdin`

## Files

- `index.html` — the entire app.
- `.github/workflows/pages.yml` — GitHub Pages deploy workflow.
- `.nojekyll` — disable Jekyll on Pages.
