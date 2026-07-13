# ip-info

A single-page static site that shows your current public IP and continuously
tells you whether your device's internet connection is actually working. Built
for checking a phone's mobile data: open the page and glance at the ONLINE /
OFFLINE banner.

Live site: https://pgr0ss.github.io/ip-info/

## What it does

- Polls `https://ipinfo.io/json` every 10 seconds for the public IP plus
  network details (hostname, city, region, country, org/ASN, coordinates,
  postal, timezone).
- Shows a big color-coded status banner: **ONLINE** (green), **OFFLINE**
  (red), or **CHECKING…** (grey).
- Status is **recency-based**: ONLINE requires the last fetch to have both
  succeeded *and* landed within the freshness window (`STALE_AFTER`). A once-
  successful but stale state reads OFFLINE. A 1-second render tick means the
  display flips to OFFLINE on its own if polls silently stop, even with no
  explicit fetch error.
- Shows the last-known IP and details even while OFFLINE.
- On failure, a red "Why it's failing" card surfaces the categorized error
  (timeout / HTTP status / generic network failure) and a plain-English hint.
- Shows last-success time, last-attempt time, and current time, each with live
  relative ages.
- Refreshes immediately when the tab returns to the foreground
  (`visibilitychange`).

## Architecture

Everything lives in a **single file: `index.html`** — markup, inline CSS, and
inline vanilla JavaScript. No build step, no dependencies, no framework. It can
be opened directly from disk or served by any static host.

Key JS constants (top of the `<script>` block):

- `ENDPOINT` — `https://ipinfo.io/json`
- `POLL_MS` — 10000 (fetch cadence)
- `TICK_MS` — 1000 (re-render cadence for live relative times + staleness)
- `TIMEOUT_MS` — 4000 (per-request `AbortController` timeout)
- `STALE_AFTER` — 22000 (ONLINE freshness window; keep it above
  `POLL_MS + TIMEOUT_MS` or ONLINE will falsely blip to OFFLINE between polls)

State is a handful of module-level variables (`lastData`, `lastSuccessAt`,
`lastAttemptAt`, `lastAttemptOk`, `lastError`, `inFlight`). Status is *derived*
at render time from these, never stored.

## Deployment

Hosted on GitHub Pages via `.github/workflows/pages.yml`, which deploys the
repo root on every push to `main`. `.nojekyll` disables Jekyll processing so
files are served as-is. Pages source must be set to "GitHub Actions" in repo
settings (one-time).

## Notes for future work (important gotchas)

- **The ipinfo.io 429 rate-limit response is unreadable from the browser.**
  When rate-limited, ipinfo returns HTTP 429 **without** an
  `Access-Control-Allow-Origin` header. On a cross-origin `fetch` the browser
  therefore rejects the request as a generic `TypeError: Load failed` and never
  exposes the status or JSON body to JavaScript. That means the actual
  `{"status":429,...}` payload **cannot** be displayed. The code has a branch to
  parse an ipinfo error body (`e.apiError`), but it is effectively unreachable
  cross-origin and only works same-origin or if ipinfo ever adds CORS to error
  responses. The OFFLINE diagnostics hint calls out 429 as the likely cause
  instead. To actually read the error / raise the limit, use an ipinfo token
  (`?token=...`); token'd error responses do send CORS headers.

- **All iOS browsers use WebKit**, so Safari vs Firefox differences are not
  engine/CORS differences. Failures that appear only in Firefox iOS are content
  blockers or the same rate-limiting surfacing as `TypeError: Load failed`.

- **A blocked/failed request is indistinguishable from a real outage** at the JS
  level. This is acceptable here — both mean "data not working right now" — but
  it's why the diagnostics can only *guess* at the cause.

- **No fallback endpoint** is currently implemented (considered and declined).
  If reliability across blockers matters, the intended approach is to try
  several IP endpoints and use whichever responds first, keeping ipinfo.io
  primary for its richer data.

- After editing, sanity-check the inline JS with:
  `sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' | node --check /dev/stdin`

## Files

- `index.html` — the entire app.
- `.github/workflows/pages.yml` — GitHub Pages deploy workflow.
- `.nojekyll` — disable Jekyll on Pages.
