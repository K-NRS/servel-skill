# Servel Dashboard

`servel dashboard` (alias: `dash`) is the at-a-glance read-only view of every optional subsystem on a server. Use it as the **first command** when picking up an unfamiliar (or your own neglected) server: one screen tells you what's enabled, what's idle, and the exact command to enable each idle feature.

## When to invoke

| Scenario | Why |
|----------|-----|
| Auditing a server you haven't touched in months | Surface every feature that's off + ready to enable |
| After provisioning a new server | Verify daemon, master key, error pages, alerts are wired |
| Before requesting "is feature X on?" loops with the user | Faster than running 5 separate `servel ...` commands |
| Detecting silent regressions (Traefik down, daemon inactive, restic missing on a worker) | One probe per subsystem, all on one screen |
| Health-check before a risky operation (migration, drain, prune) | Reveals warnings (`!`) you'd otherwise miss |

## Output format

| Icon | Meaning |
|------|---------|
| `✓` | Enabled — short summary |
| `○` | Available but off — exact enabling command shown |
| `!` | Warning — needs attention (disk ≥ 80%, missing master key, partial restic install, …) |
| `ⓘ` | Informational — count or value, no action |

Sections (in order): **Cluster, Migration, Routing, Observability, Automation, Security, Resources**.

## Sample row interpretation

```
Migration
  ✓  Snapshot strategy (restic)       on all 5 node(s) — snapshot strategy active
```

Means restic is on every swarm node, so `servel move --fast` will use the snapshot strategy (delta-bounded downtime) instead of falling back to fullcopy.

```
  !  Snapshot strategy (restic)       only on 3/5 nodes (KN-MANAGER, kn-deployments, norastech) — falls back to fullcopy for cross-node migrations involving norasdev, premarketcap-noras-systems
```

Partial install — snapshot fast path silently falls back to fullcopy for the listed nodes. Run `apt install restic` on the missing nodes via `servel ssh <node> -- apt install -y restic`.

## Cluster-wide restic probe (Migration section)

Critical detail: the restic row probes **every swarm node**, not just the manager. Earlier dashboard versions probed only the SSH'd manager and reported "installed" while workers were missing it. The current implementation:

1. `docker node ls --format '{{.Hostname}} {{.Status}}'` — enumerate ready nodes
2. For the manager (where SSH terminates) — probe `which restic` locally (managers can't SSH to themselves)
3. For workers — resolve hostname → IP via `docker node inspect` (worker hostnames aren't DNS-resolvable from manager) and `ssh root@<ip> 'which restic'`

This matters because the snapshot migration code checks restic on BOTH source and target — partial installs degrade to fullcopy without warning unless the dashboard surfaces it.

## "Last migration" row

The Migration section's `Last migration` row reads the newest record from `/var/servel/migrations/history.jsonl` (the same audit log surfaced by `servel move history`). Format: `<name> → <target> (<strategy>, <downtime>, <status>)`. Renders `—` when the log is empty (no migrations have happened on this server yet).

Use it as the "did the last migration actually finish, and how long did it take?" answer without leaving the dashboard. For more than the newest record, run `servel move history`.

## Disk-grower investigation

When the dashboard reports `! Manager disk 87% used` (≥80% threshold), the canonical follow-up is `servel df --growers` — top disk consumers across `/var/servel/`, `/var/lib/docker/`, `/tmp` and a few others, sorted by size. Pairs naturally with `servel dashboard --watch 5` in a second terminal during cleanup so you can watch the disk row tick down as you `servel prune --all` / clear the migration cache.

## What dashboard probes vs ignores

**Probes** (filesystem markers + a few shell-outs):
- `/var/servel/{secrets/master.key, metrics/snapshots.json, alerts/{channels,state.json}, audit/, ci/jobs/, dev/, version, routes/, access/{policies,users.json}, bans/server.json, infrastructure/*/spec.json}`
- `docker service ls` for `servel-system-{traefik,error-pages,bastion}`
- `docker node ls`, `df -h /`, `free -m`, `uptime`, `hostname`, `systemctl is-active servel-daemon`
- `grep` against `/var/servel/config.yaml`, `/etc/ssh/sshd_config`, `/etc/traefik/traefik.yml`, `/var/servel/traefik/dynamic/servel-middlewares.yml`

**Ignored** (out of scope):
- Backup contents, infra credential validity, network egress
- Per-deployment health (use `servel ps`, `servel verify health <name>`)
- Per-infra status (use `servel infra check`, `servel infra status`)

## Targeting

```bash
servel dashboard                  # default remote
servel dashboard --remote KN      # specific server
servel dashboard --watch 5        # in-place refresh every 5 seconds
```

Multi-server sweeps: loop the command, one server at a time. Output is human-formatted — for scripts use the underlying subcommand JSON (`servel capacity --json`, `servel alerts status --json`, `servel storage status`).

### `--watch` for incident monitoring

`--watch <N>` redraws the dashboard in place every N seconds (Ctrl-C to exit). Use during:

- **Migrations** — watch `Last migration` flip + `Snapshot strategy (restic)` recover after a worker rejoins
- **Disk pressure** — watch `Manager disk` tick down after `servel prune --all` completes
- **Capacity events** — watch `Active alerts` clear after a rebalance lands
- **Server upgrades** — watch `Server version` flip from `!` (drift) to `ⓘ` (matches client)

Pair with `servel df --growers` in a second terminal when chasing disk-grower incidents — dashboard for high-level state, `df --growers` for the offending paths.

## Behavior under failure

Each section is best-effort — a failed probe omits that one row, doesn't blank the screen. If you see a missing row that should be there, the underlying shell-out failed (e.g., `docker` daemon down, file permission issue). Diagnose by running the probe directly via `servel ssh <server>`.

## File map

```
src/internal/cli/dashboard.go          # all sections + probeResticAcrossCluster + helpers
src/internal/cli/cliutil/{icons,colors}.go  # ✓/○/!/ⓘ rendering
```

## Common follow-ups by row

| Dashboard row says | Run next |
|--------------------|----------|
| `! Manager disk 87% used` | `servel df --nodes`, `servel prune --all`, `servel remote gc` |
| `! Snapshot strategy (restic) — only on 3/5 nodes` | `servel ssh <missing-node> -- apt install -y restic` |
| `! Middleware config missing` | `servel doctor --fix` or `servel remote fix-middlewares` |
| `! Master key missing` | `servel auth init` (CRITICAL — secrets unreadable) |
| `! Active alerts: N firing` | `servel alerts status` |
| `○ Visitor analytics off` | `servel analytics enable <app>` per app |
| `○ Capacity forecasting off` | Daemon not running — `systemctl start servel-daemon` |
| `○ Alert channels off` | `servel alerts setup telegram\|slack\|discord\|webhook` |
| `○ Scheduled backups off` | `servel infra backup <name> --schedule '0 3 * * *'` |
| `○ Replicated volumes off` | `servel storage enable` (see [STORAGE.md](STORAGE.md)) |
| `○ Auto-rebalance disabled in config` | Edit `/var/servel/config.yaml` → `rebalance_enabled: true` |
| `○ Access ACL — single-operator mode` | `servel access invite --role deployer` |
| `○ SSH gate (ForceCommand) off` | `servel access setup` |
| `○ Bastion off` | `servel bastion deploy` |
| `○ Access 2FA (TOTP) off` | `servel access mfa setup <user>` per user |

## Versioning

Shipped alongside the cross-node migration work. Layout may change between versions — treat the dashboard as human output, not a stable API.
