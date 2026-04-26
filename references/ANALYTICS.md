# Analytics — visitor data from Traefik access logs

Per-deployment and cluster-wide request analytics. Opt-in; zero external dependencies (no Plausible, no GA, no Grafana stack).

## Enable

```bash
servel logs config --access-logs
```

This writes Traefik JSON access logs to `/var/servel/logs/traefik/access.log`, installs a daily logrotate config (7-day retention), and restarts Traefik. The daemon picks up the file automatically and starts per-minute rollups at `/var/servel/analytics/`.

Disable: `servel logs config --no-access-logs`.

## Use

```bash
# In a project dir — resolves from .servel/state.json
servel analytics

# Explicit deployment
servel analytics myapp --period 7d --paths --errors

# Infra with public routes
servel analytics @chatwoot

# Cluster-wide (opt-in)
servel analytics --cluster

# JSON output
servel analytics --json
```

**Default resolution** matches `servel logs`/`ps`/`secrets`: with no args, uses the deployment from `.servel/state.json`. Use `--cluster` for the aggregate view.

## Flags

| Flag | Effect |
|------|--------|
| `-p, --period` | `1h`, `6h`, `24h`, `7d`, `30d` (default `24h`) |
| `--cluster` | aggregate across all deployments |
| `--paths` | top 10 request paths |
| `--referrers` | top 10 origin referrers |
| `--user-agents` | browser/bot families |
| `--errors` | status-code breakdown |
| `--json` | machine-readable output |

## What's measured

- Requests, unique visitors (daily-rotating IP hash), 4xx/5xx counts and rate
- Bandwidth in/out
- Latency avg / p50 / p95 / p99 (milliseconds)
- Top paths, referrers (origin only), UA families, status codes

## Privacy posture (non-negotiable)

- **IPs hashed** with SHA-256 using a salt that rotates daily at `/var/servel/analytics/.salt`. Raw IPs dropped at ingest — never persisted. Same IP hashes to the same value within a day (enables unique-visitor count), different value next day (no cross-day tracking).
- **Referrers** truncated to origin (`https://example.com`) — path/query always stripped.
- **User-agents** reduced to family (`Chrome`, `Firefox`, `bot`, `other`) — full UA discarded.
- **No cookies, no fingerprinting, no client-side tracker.** Pure server-side log analytics.

Opt out per deployment: `analytics: false` in `servel.yaml`.

## Storage layout

```
/var/servel/analytics/
├── .salt                    # daily-rotated hash salt (0600)
├── .offsets.json            # daemon tailer resume state
└── {deployment}/
    ├── 2026-04-19.json      # today (1-min buckets)
    └── 2026-04-18.json      # compacted to hourly after 24h
```

- 1-min buckets retained for 24 hours, then rolled up to hourly
- Raw Traefik logs rotate daily (7-day retention)
- Rollups are source-of-truth for historical queries

## Agent heuristics

**When a user asks "how many visitors does my app get?" / "traffic numbers" / "who's visiting":**
1. Check if access logs enabled — `servel analytics myapp` will print a hint if not.
2. If not: `servel logs config --access-logs`, wait for traffic, try again.
3. If yes: `servel analytics [name] --period 24h` for quick view, add `--paths --errors` for detail.

**When a user asks "is there a traffic spike?" / "error rate up?":**
- `servel analytics <name> --period 1h --errors` vs `--period 24h` for comparison.
- Compare p95 latency between periods.

**When diagnosing 5xx issues:**
- `servel analytics <name> --errors --paths` — which paths are erroring.
- Cross-reference with `servel logs <name> -f` for the actual stack traces.

**Never** suggest installing a third-party analytics tool (Plausible, GA, Umami, etc.) for a servel-deployed app without first showing `servel analytics`. Dogfooding applies.

## Limits

- High-cardinality top-N (paths, referrers) capped at 50 per 1-min bucket — scalar totals (requests, errors, bytes) remain exact.
- Latency samples capped at 1024 per minute for percentile computation.
- Geo lookup not yet wired; countries bucket empty until MaxMind GeoLite2 integration lands.
