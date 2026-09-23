# CoreValley status page

A self-contained static status page for **status.corevalley.ai**. No build
step, no framework: `index.html` renders from `status.json`, and `brand/`
holds the two logo files it uses.

## Deploy

Upload the contents of this folder to the status host. Any static host works
(GitHub Pages, Cloudflare Pages, Netlify, an S3 bucket, a plain nginx root).
Keep the three parts together:

```
index.html     the page (all CSS and JS inline)
status.json    the data feed the page fetches on load
brand/         cv-brandmark.svg, cv-wordmark-white.svg, cv-wordmark-carbon.svg
```

Point the `status` DNS record at the host and enable HTTPS. The page's
canonical URL is `https://status.corevalley.ai/`; change the `<link
rel="canonical">` and the Open Graph URL in `index.html` if the hostname
differs. The main site links here from the header and footer
(`NEXT_PUBLIC_STATUS_URL`, default `https://status.corevalley.ai`).

### GitHub Pages, in this repository

Create a second Pages site from a `status` branch, or a separate repository
`CoreValleyAI/status`, add a `CNAME` file containing `status.corevalley.ai`,
and add a DNS `CNAME` record `status → corevalleyai.github.io`. The
`doc_cname_readme.txt` at the repo root walks through the same steps for the
docs host; they are identical apart from the name.

## Data feed

`status.json` is the only thing a monitor needs to write. Shape:

```jsonc
{
  "updatedAt": "2026-10-01T04:15:00Z",      // ISO 8601, UTC; null = no data yet
  "overall": "operational",                 // operational | degraded | partial | major | maintenance | unknown
  "groups": [
    { "name": "Platform · np-ktm-1", "components": [
      {
        "id": "api", "name": "Control-plane API", "note": "api · auth · metering",
        "status": "operational",              // operational | degraded | down | maintenance | unknown
        "uptime90": 99.97,                    // percent over the last 90 days, or null
        "history": [                          // newest first, one per day, max 90
          { "date": "2026-10-01", "status": "operational" },
          { "date": "2026-09-30", "status": "degraded" }
        ]
      }
    ]}
  ],
  "metrics": [
    { "id": "api-latency", "name": "API response time", "unit": "ms",
      "series": { "24h": [ { "t": "2026-10-01T04:00:00Z", "v": 142 } ], "7d": [], "30d": [] } }
  ],
  "maintenance": [
    { "title": "Firmware update, rack A", "components": ["pods"],
      "start": "2026-10-05T18:00:00Z", "end": "2026-10-05T20:00:00Z",
      "note": "Running pods are drained and rescheduled; new launches pause." }
  ],
  "incidents": [
    { "title": "Elevated API latency", "status": "resolved", "components": ["api"],
      "updates": [
        { "state": "investigating", "time": "2026-09-28T09:10:00Z", "text": "p95 latency above 800 ms on the control-plane API." },
        { "state": "identified",    "time": "2026-09-28T09:32:00Z", "text": "A metering batch job saturated the database pool." },
        { "state": "monitoring",    "time": "2026-09-28T09:48:00Z", "text": "Job rescheduled off-peak; latency back under 200 ms." },
        { "state": "resolved",      "time": "2026-09-28T10:30:00Z", "text": "Stable for 40 minutes. Pool sizing corrected." }
      ] }
  ]
}
```

Everything is optional except `groups`; missing fields render as "no data".
The placeholder shipped here has every status `unknown` and empty histories
on purpose, so the page never shows a made-up number.

## Connecting a monitor

Any of these produces `status.json`:

- **UptimeRobot / Better Stack / Checkly** — create one monitor per component
  (the `id`s above), then a small scheduled job (GitHub Action, cron on a VM)
  that calls the provider API, maps monitors to components, computes
  `uptime90` and the daily `history`, and uploads `status.json`. Incidents
  and maintenance can be typed into the JSON by hand or synced from the
  provider.
- **GitHub Action with curl** — for a first version, a workflow on a
  15-minute schedule that hits each public URL, records up/down in a JSON
  file committed to the status branch, and rebuilds `status.json`. Free, and
  enough for external HTTP checks.
- **Managed status page** (Statuspage, Instatus, Better Stack status pages) —
  if you would rather not host this page at all, keep the component list and
  point `NEXT_PUBLIC_STATUS_URL` at the provider's page.

Internal components (GPU pods, storage, tenant networking) need a probe that
runs inside np-ktm-1 and reports out; until that exists, leave them `unknown`
rather than marking them operational on the strength of the website being up.

## Editing

The page's copy and colours are in `index.html`. Colours follow the design
system tokens (Carbon, Ink, Hydro) and switch with the OS colour scheme. Keep
the file under ~40 KB and dependency-free so it stays independent of the main
site's build and survives a main-site outage, which is when it matters.
