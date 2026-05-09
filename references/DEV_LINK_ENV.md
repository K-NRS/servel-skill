# Dev — Link Production Env (`--link-env` / `--link-infra`)

`servel dev` can pull env (and optionally decrypted secrets) from a real
deployment or one or more `@infra` instances and inject them into the dev
container. Eliminates the "copy `.env` from prod, paste into local, hope
it's still current" loop and gives the dev process the same configuration
shape production runs with — without leaking values to shell history or
disk in plaintext beyond a tightly-scoped cache.

## Flag matrix

All flags are on `servel dev` (and `servel dev start`).

| Flag | Type | Purpose |
|------|------|---------|
| `--link-env <deployment>` | string | Pull plaintext env from a deployment by name. Wired through new server query handler `dev.pull-env`. |
| `--link-infra @name[,@name...]` | string slice | Inject `DATABASE_URL`/`REDIS_URL`/etc. from one or more `@infra`. Reuses `clideploy.ResolveInfraLinks` — same code path as `servel deploy --link-infra`. |
| `--secrets` | bool | Also pull decrypted secret values for `--link-env`. ACL-gated (`dev:env:pull`) unless caller is the owner. |
| `--only KEY[,KEY...]` | string slice | Whitelist filter applied server-side to env + secrets. |
| `--exclude KEY[,KEY...]` | string slice | Blacklist filter, applied after `--only`. |
| `--no-tunnel` | bool | Disable auto-tunneling of internal Docker hosts. Each detected host surfaces a warning instead. |

Flags declared at `internal/cli/clidev/dev.go:57-64` and registered at
`internal/cli/clidev/dev.go:242-247`.

## Source priority

When linked env merges with the user's existing dev env, priority is
(highest first):

1. User-supplied dev env — `servel.yaml` `dev.env`, `--env` flag.
2. `--link-env <deployment>` env (and secrets if `--secrets`).
3. `--link-infra` connection bundles.

Within steps 2-3, `ResolveLinkedEnv` applies fill-gaps semantics so
deployment values pre-empt infra values (rationale: an app may override
pool params on top of the raw infra `DATABASE_URL`; the app's value is
what runtime expects). The caller in `dev_session.go:281-306` applies
layer 1 on top with the same fill-gaps behavior. Implementation:
`internal/cli/clidev/link_env.go:97-161`.

When `--secrets` is requested and a key appears in both `Env` and
`Secrets` of a deployment spec (legal but unusual), the secret value
wins and a warning is surfaced — same convention as the deploy pipeline
at `deploy/swarm.go:148`.

## Authorization

Server-side check in `internal/server/query/handlers/dev_env.go:94-115`
(`authorizeDevEnvPull`). In order of precedence:

1. **Empty `SERVEL_GATE_USER`** → allow. Direct SSH means the caller has
   shell access to `/var/servel/*` already; gating here adds no real
   security.
2. **Gate user equals `deployment.DeployedBy`** → allow (owner bypass).
   `DeployedBy` is the trusted gate user recorded at deploy time — see
   `types/deployment.go:205`.
3. **Gate user has `PermDevEnvPull` ("dev:env:pull") in any scope** →
   allow. `scopeGrantsDevEnvPull` at `dev_env.go:125-143` walks the
   user's scopes (across all servers) and checks for the explicit
   permission.
4. **Empty `DeployedBy` for gate users** → DENY. Originally bypassed in
   the design but caught in review: leaking env from every legacy
   deployment to gate-enrolled users would be a regression.
5. Otherwise → DENY with explicit `deny_reason`.

Role-based grants (`admin`, `super_admin`) deliberately do NOT include
`PermDevEnvPull`. Pulling prod secrets to a laptop is opt-in per user,
never inherited from a broad role.

**Grant command:**

```bash
servel access scope add <user> --server <name> --permissions dev:env:pull
```

`PermDevEnvPull` is registered in `AllPermissions()` at
`types/access.go:81`; description at `types/access.go:117`.

## Auto-tunnel

`servel dev` detects internal Docker hostnames in linked env values
(`servel-infra-*`, `servel-system-*`) and starts one `portfwd.Manager` per
unique `(host, port)` pair, then rewrites the value to `localhost:N`
before injection.

Detection regex (anchored at URL boundaries to avoid false positives in
free-text values):

```
(?:^|[@/])(servel-(?:infra|system)-[a-z0-9][a-z0-9-]*)(?::(\d{1,5}))?(?:[/?#]|$)
```

Defined twice intentionally:

- Server-side: `dev_env.go:276` (`internalHostRE`) — covers
  deployment env+secrets returned by the RPC.
- Client-side: `tunnel_rewrite.go:166` (`scanInternalHostsRE`) — covers
  `--link-infra` values resolved client-side via
  `clideploy.ResolveInfraLinks` (server never sees them).

`mergeInternalHosts` at `tunnel_rewrite.go:215` deduplicates the union of
both sources before establishing tunnels.

Substring replacement is intentional — env values come in many shapes
(`postgres://u:p@host:port/db`, `redis://host:port`, `host=H port=P`,
bare `host:port`). All collapse to "find `host:port`, replace with
`localhost:N`". Rewrite list is sorted longest-first so a more-specific
`host:port` rewrite wins over a less-specific bare-host substring match.

Edge cases:

- **Bare host, no port** (e.g. `DB_HOST=servel-infra-pg`): cannot tunnel
  — no port to forward. Value passes through unchanged with a warning.
- **Cross-server hosts**: the tunnel uses the dev session's SSH endpoint.
  If the linked deployment / infra is on a different swarm, `portfwd`
  errors out, the tunnel start fails, and the raw value is passed
  through with a warning. See "Cross-server limitation" below.
- **`--no-tunnel`**: skip tunneling entirely; warn per detected
  internal host with a hint (`tunnel_rewrite.go:165-175`).

Tunnels are tracked on `LinkResolution.Tunnels` and torn down via
`StopTunnels()` deferred from `dev_session.go:274`. Idempotent — safe to
call multiple times.

## Cache file

Per-session cache at `~/.servel/dev/sessions/<sid>/env.cache`. Owner:
`internal/cli/clidev/env_cache.go`.

Properties:

| Property | Value | Why |
|----------|-------|-----|
| File mode | `0600` | Owner-readable only. Refused on read if perms differ — defends against tampering. |
| Dir mode | `0700` | Even processes in the same group can't enumerate other sessions. |
| TTL | 1 hour from `pulled_at` | Long enough to survive normal restart-within-window; short enough that rotated secrets don't linger. |
| Schema version | 1 | Bumped on incompatible changes. New fields use `omitempty`. |
| Atomic write | tmp + rename | Crash between create and rename can't leak. |
| Auto-delete on exit | Yes | `DeleteEnvCache` at `dev_session.go:417`. |
| Auto-evict on next start | Yes | `EvictExpired` at `dev_session.go:259`. Walks the sessions root, removes files past TTL by mtime first then payload TTL field. |

Payload shape (`EnvCachePayload` at `env_cache.go:41-51`):

```json
{
  "schema_version": 1,
  "session_id": "...",
  "deployment": "myapp-prod",
  "infra": ["@postgres-prod"],
  "pulled_at": "2026-05-07T12:34:56Z",
  "ttl_seconds": 3600,
  "sources": ["myapp-prod (env)", "@postgres-prod"],
  "env": { "...": "..." },
  "tunnels": [{"host":"servel-infra-pg","port":5432,"local_port":54320}]
}
```

Env values in the cache are **post-tunnel-rewrite** — a debugger can see
exactly what the dev container received without re-deriving the
substitutions.

## Audit

Every `dev.pull-env` invocation — allow OR deny — emits a single audit
entry. Action constant: `AuditActionDevEnvPull` (`"dev.env.pull"`) at
`types/audit.go:93`.

Metadata schema (`dev_env.go:147-167`):

| Field | Allowed | Denied |
|-------|---------|--------|
| `deployment` | yes | yes |
| `include_secrets` | yes | yes |
| `only_filter` | yes | yes |
| `exclude_filter` | yes | yes |
| `keys` | yes (env key names) | — |
| `secret_keys` | yes (secret key names) | — |
| `tunnels_hint` | yes (count) | — |
| `deny_reason` | — | yes (e.g. "deployment has no recorded owner and gate user X has no dev:env:pull scope") |

**Values never appear** — only key names. Audit emission is non-fatal
(`emitDevEnvPullAudit` at `dev_env.go:169-193`); a failed write logs to
stderr but doesn't break the user's dev session. Inspect:

```bash
servel audit list --action dev.env.pull --limit 20
```

## Cross-server limitation

The implementation assumes the linked deployment / `@infra` lives on the
same Docker swarm as the dev session's SSH endpoint. Cross-server
tunnels are not supported. Symptom: `portfwd.getContainerID` fails, the
tunnel doesn't start, the raw internal-host value passes through with a
warning, and the dev container gets a connection error.

Workarounds today: connect dev to the right server (`servel dev --remote
<server>`), or extract values manually. A `--link-server <name>` flag
that takes a second SSH endpoint is on the roadmap (project memory:
`dev_link_env_cross_server_tunnels.md`).

## Component map

| Layer | File | Role |
|-------|------|------|
| CLI flags | `internal/cli/clidev/dev.go:57-64,242-247` | Declare + register `--link-*`, `--secrets`, `--only`, `--exclude`, `--no-tunnel` |
| Client orchestration | `internal/cli/clidev/link_env.go` | `LinkOptions`, `LinkResolution`, `ResolveLinkedEnv`, `StopTunnels` |
| Auto-tunnel + rewrite | `internal/cli/clidev/tunnel_rewrite.go` | `establishTunnels`, `applyRewrites`, `scanInternalHosts`, `mergeInternalHosts` |
| Cache lifecycle | `internal/cli/clidev/env_cache.go` | `WriteEnvCache`, `ReadEnvCache`, `DeleteEnvCache`, `EvictExpired` |
| Dev session wiring | `internal/cli/clidev/dev_session.go:244-307` | Resolve link → tunnel → merge env → defer cleanup; `:413-421` cache write/delete |
| Server query handler | `internal/server/query/handlers/dev_env.go` | `dev.pull-env` RPC: ACL, decrypt, filter, host scan, audit |
| Permission constant | `internal/types/access.go:55` | `PermDevEnvPull = "dev:env:pull"` |
| Audit action | `internal/types/audit.go:93` | `AuditActionDevEnvPull = "dev.env.pull"` |
| Wire types | `internal/types/dev.go` | `DevEnvPullRequest`, `DevEnvPullResponse`, `InternalHost` |

## What ships in v1 vs deferred

Shipped (P1-P6, 2026-05-07):

- `--link-env` (deployment) and `--link-infra` (@infra) with merging
- `--secrets` opt-in, ACL-gated, audited
- `--only` / `--exclude` exact-match key filters
- Auto-tunnel via `portfwd.Manager` with substring rewrite
- `--no-tunnel` escape hatch with per-host warning
- Session-scoped cache with TTL + 0600 mode + auto-evict + auto-delete

Deferred to v2+:

- Glob filters (`STRIPE_*`)
- Mid-session `--refresh-env`
- `servel.yaml` `dev: { link_env: ... }` defaults
- Per-key ACL (`alice can pull DB_URL but not STRIPE_SECRET_KEY`)
- `servel dev env-diff` view (compare local vs prod env without
  injecting)
- `--link-server <name>` cross-server tunneling

## Cookbook

```bash
# Most common: prod-shaped local dev, no secrets
servel dev --link-env myapp-prod

# Real DB locally, no other prod env touching dev
servel dev --link-infra @postgres-staging

# Multi-infra with key whitelist
servel dev --link-infra @postgres-prod,@redis-prod \
  --only DATABASE_URL,REDIS_URL

# Owner pulls secrets for one-off debugging
servel dev --link-env myapp-prod --secrets

# Non-owner needs explicit grant first
servel access scope add bob --server KN --permissions dev:env:pull
# Then bob can:
servel dev --link-env myapp-prod --secrets

# Inspect what a session pulled
cat ~/.servel/dev/sessions/<sid>/env.cache | jq '.sources, .env | keys'

# Audit who pulled what
servel audit list --action dev.env.pull --limit 50
servel audit list --action dev.env.pull --user bob

# Disable auto-tunnel for a value you'll rewrite by hand
servel dev --link-infra @postgres-prod --no-tunnel \
  --env DATABASE_URL=postgres://localhost:55432/app
```
