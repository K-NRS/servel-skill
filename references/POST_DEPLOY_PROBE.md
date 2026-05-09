# Post-Deploy Probe

After Swarm reports task convergence, every `servel deploy` and `servel rollback` probes the public URL to verify the service is actually routable. Convergence proves containers started healthy. It does **not** prove Traefik picked up the new endpoint. The 2026-04-24 agentkarma incident was 7 minutes of Cloudflare 504s while the deploy reported success and `servel inspect` showed `Status: running` — a `docker service update --force` recovered it instantly. The probe + auto-respawn closes that class of outage automatically.

## Contract

1. After convergence, Servel issues a single HTTPS GET to `<primary-domain><probe-path>`.
2. If the response status is in the accept list (default 200–399), the deploy/rollback completes normally.
3. If not, Servel calls `docker service update --force` once. This rebuilds the VIP cache and overlay attachments — the same primitive `servel restart` uses.
4. Servel re-probes the URL.
5. If the second probe passes, the deploy completes (audit log records the auto-respawn).
6. If the second probe fails, the deployment is marked `degraded`:
   - The new image **stays running** — image is not rolled back automatically.
   - The CLI exits non-zero so CI catches it.
   - Two audit entries are written: `deploy.probe` (failed) and `deploy.respawn` (still unreachable).

## Defaults

| Setting | Default | Purpose |
|---|---|---|
| Probe path | `/` | Root of the primary HTTP route |
| Total budget | 30s | Bounded — covers one auto-respawn easily |
| Per-request timeout | 5s | Single attempt before retry |
| Backoff between attempts | 2s | Cheap retry within the budget |
| Accept | 200–399 | Treats redirects as routable |
| Auto-respawn | enabled | Single attempt only |
| TLS verification | disabled | Common to mix Cloudflare and origin certs |

## Configuration

**Per deployment in `servel.yaml`:**

```yaml
post_deploy:
  probe:
    path: /healthz       # default '/'
    timeout: 45s         # default 30s
    accept: [200, 401]   # default 200-399 — useful for auth-protected '/'
    auto_respawn: false  # default true
    verify_tls: true     # default false
    skip: false          # default false (true disables probe entirely)
```

**Per invocation via flags** (work on both `servel deploy` and `servel rollback`):

```bash
--probe-path /healthz
--probe-timeout 60s
--no-auto-respawn
--skip-probe
```

CLI flags win over `servel.yaml`; `servel.yaml` wins over hardcoded defaults.

## When to skip vs. configure

| Situation | Recommendation |
|---|---|
| App with auth-protected `/` | `accept: [200, 401]` — keeps probe meaningful without breaking on auth |
| Long cold start | `timeout: 60s` (probes retry within the budget) |
| Background worker with no HTTP route | Probe auto-skips — no config needed |
| TCP-only service | Probe auto-skips — no config needed |
| Flaky upstream during deploy window | `--no-auto-respawn` — log probe result without auto-healing |
| Nothing at root and no health endpoint | `skip: true` (last resort — you lose the safety net) |

## Diagnosing `degraded` status

`degraded` means: the new image is running, healthy at the container level, but the public URL is not reachable. Check in this order:

1. **`servel logs traefik`** — does Traefik even know about the service? Look for "service `<name>` not found" or stale VIP errors.
2. **`servel verify routing <name>`** — re-runs the routing checks manually (DNS, HTTPS reach, redirect, middleware chain).
3. **`servel restart <name>`** — same primitive the auto-respawn used. If the second cycle works, the first cleared a transient state.
4. **Cloudflare side** — if you see `Host: Error` in the 504 page, Cloudflare can't reach origin (Traefik issue). If you see `Origin returned 504`, your app returned the 504 (app issue).

## Architecture notes

- The probe runs **server-side** (where `servel deploy` already runs in server mode). This means it traverses the same Traefik / overlay path Cloudflare would, exercising every layer that can fail.
- The `internal/postdeploy/` package owns the logic. It reuses `internal/verify/HTTPClient.CheckURL` so probe semantics stay consistent with `servel verify routing`.
- The auto-respawn primitive is `docker.Client.ForceUpdateService`, which increments `TaskTemplate.ForceUpdate++` — guaranteeing Swarm reschedules tasks even when the image string is unchanged. The same primitive backs `servel restart`.
- Rollback uses identical primitives (`UpdateServiceImage` with `ForceUpdate++` + same probe). This was a deliberate Phase B unification — pre-2026-04-30, rollback shelled out raw `docker service update --image` over SSH and could no-op when image digests collided.

## Audit trail

Each probe writes exactly one audit entry (`deploy.probe`) recording URL, status code, duration, attempts, reachable bool, and degraded flag. If a force-respawn fires, a second entry (`deploy.respawn`) records whether recovery succeeded. Query with:

```bash
servel audit list --action deploy.probe --limit 20
servel audit list --action deploy.respawn --limit 20
```

Use `deploy.respawn` count as a leading indicator of overlay/Traefik instability — if it spikes, investigate the swarm before assuming individual deploys are flaky.
