# Design Notes

## Architecture

Simple textfile exporter pattern:
1. Bash script polls Sharkey API endpoints
2. Writes Prometheus textfile format to a `.prom` file
3. Prometheus node_exporter or Grafana Alloy scrapes the textfile
4. Cron runs the script every minute

This avoids running a persistent HTTP server (like a typical Prometheus exporter would). For a small instance polled once per minute, this is simpler and uses fewer resources.

## Why not a "real" exporter?

A typical Prometheus exporter runs as an HTTP server that Prometheus scrapes directly. That's better for high-cardinality metrics or when you need precise scrape timing. For a small fedi instance with ~40 metrics updated every 60s, the textfile pattern is:

- Zero dependencies beyond bash/curl/jq
- No port to manage, no process to keep alive
- Works with any textfile-capable collector (node_exporter, Alloy, etc.)
- Easy to debug: just `cat` the `.prom` file

If this ever needs to scale (hundreds of metrics, sub-second resolution), rewrite as a proper exporter in Go/Python. Sharkey itself has no built-in Prometheus or OpenTelemetry support (as of 2025.4.x).

## Default vs optional metrics

The default set covers the metrics most useful for monitoring a typical small-to-medium instance: instance health, user activity, federation status, queue health, and DB growth. These are the things you want on a dashboard for a VPS or homelab.

Optional flags (`--charts-notes`, `--charts-users`, `--charts-drive`, `--extended-queue-stats`, `--delayed-hosts`) add more granular data that larger instances may want — note/user/drive deltas, all 11 internal queues, and per-domain federation delay breakdowns. These produce more metrics and API calls, so they're opt-in.

## API Endpoints Used

### Default: public (no auth)
- `POST /api/stats` — note counts, user counts, instance count, drive usage (rate limit: 3/s, cached 1h)
- `POST /api/get-online-users-count` — online users local + across network (cached 60s)
- `POST /api/charts/federation` — federation totals: sub/pub/pubsub/active/stalled/delivered/inbox (cached 1h)
- `POST /api/charts/active-users` — DAU-style breakdowns: read/write/readwrite (cached 1h)
- `POST /api/charts/ap-request` — AP delivery success/failure/inbox received (cached 1h)

### Default: admin (requires API token with moderator/admin role)
- `POST /api/admin/server-info` — CPU, memory, filesystem (requires moderator)
- `POST /api/admin/queue/stats` — 4 queues (deliver, inbox, db, objectStorage) × 5 states (requires moderator)
- `POST /api/admin/get-table-stats` — PostgreSQL table row counts and sizes (requires admin)

### Optional: public (opt-in flags)
- `POST /api/charts/notes` — note creation/deletion rates by local/remote (`--charts-notes`)
- `POST /api/charts/users` — user registration/deletion rates (`--charts-users`)
- `POST /api/charts/drive` — drive file count/size changes (`--charts-drive`)

### Optional: admin (opt-in flags, require token)
- `POST /api/admin/queue/queues` — all 11 queues with full state counts (`--extended-queue-stats`)
- `POST /api/admin/queue/deliver-delayed` — per-domain delayed outbound counts (`--delayed-hosts`)
- `POST /api/admin/queue/inbox-delayed` — per-domain delayed inbound counts (`--delayed-hosts`)

### Not used (future consideration)
- `POST /api/admin/federation/instances` — per-instance federation health with boolean filters (blocked, notResponding, suspended, silenced, federating). Would give accurate dead/suspended/active counts but requires pagination (limit 1-100). Could be added as an opt-in flag if `charts/federation` stalled count proves insufficient.
- `POST /api/admin/queue/queue-stats` — per-queue Redis internals (memory, fragmentation, connected clients). Overlaps with a dedicated Redis exporter.
- `POST /api/retention` — user retention cohort data. More analytics than monitoring.
- `POST /api/ping` — returns server timestamp. Could be used for latency measurement but adds little over `sharkey_up`.

## Known quirks

- `driveUsageLocal` and `driveUsageRemote` from `/api/stats` return 0 in some Sharkey versions — the fields exist but the aggregation may not be implemented.
- `charts/federation` fields like `sub`, `pub`, `pubsub` are "accumulated" type in the chart engine, meaning they represent running totals, not per-period deltas.
- `admin/server-info` returns `cpu` as a percentage but does not include `mem.used` — only `mem.total`. The non-admin `server-info` endpoint returns static hardware info only (no usage data) and requires `enableServerMachineStats` config.

## Deployment

Intended to run on the same host as Sharkey, hitting `http://127.0.0.1:3000` to avoid going through the reverse proxy. This also avoids any rate limiting.

## Token Security

The exporter supports multiple token sources (in priority order):
1. `--token <value>` — direct argument (visible in `/proc` and `ps`)
2. `--token-file <path>` — read from a file (recommended for cron)
3. `SHARKEY_TOKEN` environment variable
4. `SHARKEY_TOKEN_FILE` environment variable pointing to a file

For production use, prefer `--token-file` or `SHARKEY_TOKEN_FILE` to avoid exposing the token in process listings.

### Minimal token generation

`--create-token` prints the minimum permissions needed. These must be set manually in the Sharkey web UI (Settings > API > Generate Access Token) because Sharkey's `miauth/gen-token` API enforces `secure: true`, rejecting all API tokens and only accepting browser session auth.

| Permission | Used by | Purpose |
|-----------|---------|---------|
| `read:admin:server-info` | `admin/server-info` | CPU, memory, filesystem |
| `read:admin:queue` | `admin/queue/queues`, `admin/queue/*-delayed` | Queue stats, delayed hosts |
| `read:admin:emoji` | `admin/queue/stats` | Queue stats (upstream bug — this endpoint checks the wrong permission) |
| `read:admin:table-stats` | `admin/get-table-stats` | DB table sizes |

This avoids running cron with a full admin token. The generated token cannot modify any data.
