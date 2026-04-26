# Cross-Node Migration

Servel moves stateful services between nodes with bounded downtime and zero silent data loss. This reference covers the verbs, strategies, and decision rules for picking them.

## Canonical verb: `servel move`

```bash
servel move <target> --to <node> [flags]
```

Aliases: `mv`. Replaces the prior foot-gun where `servel infra update --node` silently moved metadata without the volume data.

### Target prefixes

| Prefix | Meaning |
|--------|---------|
| `@name` | Infrastructure (DB, cache, storage, etc.) |
| `name`  | Deployment (stateless app) |
| `~name` | System service (not supported) |

## Decision tree

```
Is the service stateful? (has data binding OR legacy Stateful.Enabled)
├── No  → metadata + constraint swap. Fast path. No confirmation.
└── Yes
    ├── --pointer-only set → metadata-only swap (UNSAFE, recovery only)
    └── Otherwise → full migration
        ├── restic available on both nodes AND single-service → snapshot strategy
        ├── restic available on both nodes AND compose → parallel snapshot per volume
        └── Otherwise → fullcopy (tar + rsync)
```

## Strategies

### `snapshot` (default when capable)

Pre-copy live migration via restic Alpine sidecars:

1. Baseline snapshot of source volume(s) **while service is UP**
2. Stream repo to target (SFTP, default) or rsync after local materialization (`--repo-location local`)
3. Optional pre-copy loop (`--iterations N`, 1-5) to shrink cutover delta
4. **Cutover window:** stop service, final delta snapshot, restore on target
5. Swap Swarm constraint, start on target, cleanup temp repos

**Downtime:** `final_delta_bytes / ssh_bandwidth + stop + start`. Seconds for typical DB write loads.

**Requirements:** restic installed on both source and target nodes.

### `fullcopy`

Stop, full tar+rsync of all volumes, restart. Conservative path.

**Downtime:** `total_data_size / ssh_bandwidth + stop + start`. Minutes for multi-GB databases.

Use when:
- restic isn't available on one or both nodes
- Bind-mount data (not Docker named volumes)
- Debugging a snapshot migration regression

### `replicated` (Phase 4)

Auto-selected when the infra was deployed with `--replicated` AND the substrate is healthy. Volume already lives on every replica via DRBD; migration is a metadata + scheduling op:

1. Stop service on source (replica on target is already in sync via DRBD Protocol C)
2. Swap Swarm constraint to target
3. Start on target — LINSTOR's Docker plugin auto-promotes local replica to Primary

**Downtime: 3-8 seconds regardless of volume size.** Dramatically faster than even snapshot for multi-GB DBs.

Provisioning:

```bash
servel storage enable                              # one-time substrate setup
servel infra add postgres mydb --replicated        # opt-in per-infra
servel move @mydb --to worker-2                    # auto-takes replicated path
```

Falls back to snapshot/fullcopy if substrate becomes unavailable, target isn't a replica, or replica is out-of-sync. See [STORAGE.md](STORAGE.md) for substrate operations.

## Repo location (snapshot only)

| Value | Behavior | When to use |
|-------|----------|-------------|
| `target-sftp` (default) | restic streams direct to target via SFTP | Almost always |
| `local` | Materialize repo on source, then rsync to target | Air-gapped targets, debugging |

`--repo-location target-sftp` requires source-side SSH key access to target (`/root/.ssh/id_rsa` mounted into sidecar). Standard for servel-provisioned nodes.

## Pre-copy loop (snapshot only)

`--iterations N` (1-5) runs N delta passes WHILE the service is UP before cutover. Each iteration snapshots+ships the delta since the previous pass.

Early exit rules:
- **Converged** — iteration shipped in under 10s. Safe to cut over.
- **Not converging** — iteration elapsed >= previous. Write pressure exceeds sync throughput; cutting over now.
- **Budget exhausted** — hit N iterations without converging.

Default: `1` (single baseline + single delta, matches Phase 2 behavior). Higher values trade total migration wall time for shorter cutover downtime.

## Compose stacks

Multi-service stacks (Supabase, Chatwoot, OpenReplay) use **pre-copy live** parallel per-volume snapshot sidecars:

- **Phase 1 (services UP)** — baseline snapshot of every volume. Concurrency capped at **4**. Happens *while the stack is serving*.
- **Phase 2 (cutover = downtime window)** — reverse-dependency-order stop, then small *delta* snapshot per volume (only the bytes written since the baseline). Restore runs in parallel on the target.
- **Phase 3 (services UP)** — dependency-order start on the target node.

One-shot password per volume (failure isolation). Reserved temp path `/var/servel/cache/migrate/<infra>-<vol>/`.

**Downtime scales with the slowest volume's delta**, not the total stack size. For a 50GB Supabase under typical write load: tens of seconds instead of minutes.

Example:

```bash
servel move @my-supabase --to worker-2 --fast
# → parallel restic sidecars for db-data, storage-data, analytics-data, …
# → baseline runs while the stack serves; cutover ships only the delta
```

Falls back to stop-then-snapshot when restic isn't available on both nodes.

## Safety model

- **Confirmation prompt** on stateful migrations unless `--yes` or `--pointer-only`
- **Pre-flight plan** shown before the prompt: source, target, volume count + size, network path, strategy, ETA
- **Rollback on failure** — for snapshot, source data is untouched through the cutover phase. If any step fails before target is ready, the service restarts on source.
- **One-shot repo passwords** — fresh random per migration, never persisted
- **Reserved migration path** — `/var/servel/cache/migrate/<infra>[-<vol>]`. Cleaned on success. Orphans after catastrophic failure need manual `rm -rf`.

## `--pointer-only` escape hatch

When data has been moved manually (disaster recovery, out-of-band rsync), `--pointer-only` updates only servel's metadata + Docker constraint:

```bash
# DANGER: strands volume data on source if data is still there.
# Use ONLY when data is already on target.
servel move @postgres --to worker-2 --pointer-only
```

Also available on the legacy `servel infra update --node` for operators with existing scripts.

## Node drain with evacuation

```bash
servel node drain worker-2 --evacuate-data
```

Enumerates stateful infra bound to `worker-2`, picks healthy targets, runs `servel move @<name> --to <target> --yes` sequentially, then executes the Swarm drain.

Rationale: Swarm's native drain only reschedules stateless tasks. Stateful services stay pinned and become unreachable once the node leaves.

## Validation

`servel doctor migration --target <host>` runs an end-to-end self-test against an ephemeral `servel-migration-probe` Postgres fixture. Exit codes:

- `0` all good
- `1` data mismatch after migration (regression)
- `2` migration itself failed
- `3` fixture teardown failed (leak)

Safe to run unattended. Reserved fixture name won't collide with operator infra. Use for:
- Pre-release smoke tests
- Cron on staging
- CI gate before production rollout

## Bench: prove the strategy crossover

`servel doctor migration` proves correctness; `servel bench migration` proves *speed*. Use it when you want measurable evidence the snapshot fast path actually wins at your data size — instead of trusting docs.

```bash
servel bench migration --target worker-2 --size 1GB
# → deploys ephemeral servel-bench-probe postgres
# → populates with ~1GB of data
# → migrates back-and-forth under each --strategies entry
# → prints comparison table: Strategy | Rows | Volume | Cutover | Total
```

Flags:

| Flag | Default | Description |
|------|---------|-------------|
| `--target <hostname>` | (required) | Target node |
| `--size <size>` | `100MB` | Approximate dataset size |
| `--rows <n>` | `0` | Explicit row count (overrides `--size`) |
| `--strategies <list>` | `fullcopy,snapshot` | Comma-separated strategies |
| `--keep` | `false` | Don't clean up the fixture (debugging) |

Reserved fixture name `servel-bench-probe`, distinct from `servel-migration-probe`. Operators safely run both back-to-back without collision.

## History: audit log of past migrations

Every migration appends one record to `/var/servel/migrations/history.jsonl` — timestamp, name, source, target, strategy, observed downtime, success/failure. Best-effort write: a failed log never blocks the migration itself.

```bash
servel move history                  # last 20 across all infra
servel move history @postgres        # filter to one infra
servel move history --limit 100      # extend the count
```

The dashboard's Migration section also surfaces the newest record inline (`Last migration: postgres → kn-deployments (snapshot, 3s, ok)`). For richer queries than the table view, `tail -f` and `jq` work directly on the JSONL file.

## Readiness probes

Templates can define an app-level probe that runs after migration:

```yaml
readiness:
  command: ["pg_isready", "-U", "postgres"]
  timeout: 60s
  interval: 2s
```

Catches the gap between "Docker says healthy" and "app is actually serving"
(WAL replay, cluster rejoin, RDB load). Probe failures warn but don't roll
back — target is already migrated.

## Graceful shutdowns

Services with `traefik.enable=true` get `stop_grace_period=30s` automatically
so in-flight HTTP requests drain before SIGKILL. Internal services keep
Docker's default 10s unless overridden in compose.

## Legacy commands (still supported)

| Command | Status | Behavior |
|---------|--------|----------|
| `servel data migrate @<name> --node <host>` | Deprecated tip | Still works; prints pointer to `servel move` |
| `servel infra update @<name> --node <host>` | Safe-by-default | Auto-delegates to full migration for stateful, with confirmation |

## When NOT to use `servel move`

- **Across swarms** — `servel move` is intra-swarm only. Cross-swarm requires `servel infra backup` + `servel infra restore` on the new swarm.
- **For renames** — use `servel rename` (no longer aliased as `mv`/`move`).
- **Expanding replicas** — use `servel infra update --replicas N`, not move.

## Typical operator recipes

### Migrate a Postgres under active writes

```bash
# Preview
servel move @prod-db --to worker-2 --plan

# Execute with 3 pre-copy iterations (minimizes cutover delta)
servel move @prod-db --to worker-2 --fast --iterations 3
```

### Evacuate a node for maintenance

```bash
servel node drain worker-2 --evacuate-data --dry-run  # review plan
servel node drain worker-2 --evacuate-data            # execute
# After maintenance:
servel node activate worker-2
```

### Migrate a compose stack (Supabase, Chatwoot)

```bash
servel move @my-supabase --to worker-2 --fast
# Parallel per-volume snapshots; reverse-dep stop → parallel restore → dep-order start
```

### Recovery after manual data move

```bash
# You've manually rsync'd /var/lib/docker/volumes/<name>/ source → target
servel move @postgres --to worker-2 --pointer-only --yes
# Metadata + constraint only. No data movement.
```

### Scheduled regression test

```bash
# Cron on staging: catch migration regressions after servel upgrades
0 3 * * * servel doctor migration --target staging-worker-2 || alert-on-call
```

## Files (for implementers / debugging)

Go source tree:

```
src/internal/infra/
├── data_binding.go               # RebindToNodeOpts, RebindOptions
├── manager_migration.go          # MigrateStateful dispatch, MigrateOptions
├── migration_snapshot.go         # migrateSingleSnapshot, restic sidecars
├── migration_snapshot_compose.go # composeSnapshotSync (parallel per-vol)
├── migration_precopy.go          # shouldStopPrecopy, resolveMaxIterations
├── migration_repo.go             # resolveRepoLocation, buildRepoURL
└── migration_estimate.go         # EstimateMigration (plan output)

src/internal/cli/
├── climove/move.go               # servel move
├── cliinfra/update.go            # --node dispatch w/ safety prompt
├── node/drain.go                 # --evacuate-data
├── node/evacuate.go              # planStatefulEvacuation, executeEvacuation
└── doctor_migration.go           # servel doctor migration
```

## History

- **Earlier behavior** — `servel infra update --node` silently stranded data. Use `servel data migrate` manually.
- **Current** — `servel move` canonical. `infra update --node` auto-delegates to full migration with confirmation. Snapshot strategy default when restic available. Compose stacks supported via parallel sidecars.
