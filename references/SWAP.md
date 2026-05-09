# Elastic Swap (zram + disk)

Servel manages a two-tier autonomous swap stack on every Linux node in the swarm. Landed in four phases between 2026-05-07 and 2026-05-09. Closes the failure mode that took down KN-MANAGER in Feb 2026 (98% disk-swap thrashing → reboot).

The daemon configures, monitors, and resizes both tiers automatically within per-node policy bounds. The CLI gives operators visibility (`servel node swap status`), policy overrides (`enable` / `disable`), and one-shot manual resize (`resize`). The advisor surfaces 7-day-trend recommendations but never auto-applies them.

## Architecture

| Tier | Backing | Priority | When | Sizing |
|------|---------|----------|------|--------|
| **zram** | RAM (lz4-compressed) | 100 (preferred) | Routine pressure absorption | Set once at install; default `min(RAM × 0.25, 4G)` |
| **swapfile** | Disk (`/swapfile`) | -2 (overflow) | Only when zram saturates | Elastic 2→8G; grows under sustained pressure |

The kernel always tries zram first (priority 100). When zram fills, pressure overflows to the disk tier. Most workloads never touch disk because lz4 gets ~3:1 compression — a 2G zram device absorbs ~6G of effective swap pressure in RAM, faster than disk I/O and without SSD wear.

**Why zram default-on:** ~10× faster than disk swap, no SSD wear, no disk-pressure conflict (servel already fights disk bleed via `df --growers`, registry retention, BuildKit GC). Modern Linux default since Fedora 33 (2020). The KN-MANAGER incident under same conditions: zram absorbs the ~6G of effective swap pressure in compressed RAM before the disk swapfile is even touched. No reboot.

## The four daemon loops

| Loop | Cadence | What it does | Tier |
|------|---------|--------------|------|
| **Swap ensure** | 1h | Idempotent: ensures every node has a zram unit installed and a swapfile of at least `MinGB` | Tier 2 (auto-applies) |
| **Elastic resize** | 5m | Grows disk swapfile when usage > 70% sustained over 15min, shrinks when < 20% | Tier 2 (auto-applies, gated) |
| **History tracker** | 5m record + 1h save | Per-host daily aggregates persisted to `/var/servel/cluster/swap_history.json` | data |
| **Advisor** | 6h | Rolls 7-day history into per-node policy recommendations | Tier 1 (surface only) |

All four are leader-only (only the swarm leader's daemon writes; workers don't run the daemon).

## Safety invariants

The daemon enforces these unconditionally; CLI `--force` is the only way to override.

1. **Never `swapoff -f`.** If pages are in use, abort and retry — never force eviction. Shrink failures are expected during pressure spikes.
2. **Never resize during a deploy.** Daemon checks the build queue; any in-flight deploy on the cluster blocks all elastic resizes.
3. **Never grow past disk pressure.** Refuses when `/` > 85% full (also surfaced in `servel df --growers`).
4. **Never silently absorb chronic pressure.** 24h at `MaxGB` fires `node_undersized` so operators provision more RAM, not bigger swap.
5. **Never online-resize zram.** Resizing zram requires `swapoff` + `reset` + reconfigure — too invasive online. Drift is reported via `zram_drift_detected`; reconfigure happens only at next operator-driven `servel node swap enable`.
6. **Never apply advisor recommendations.** The advisor writes to `/var/servel/cluster/swap_recommendations.json` and surfaces them in `node swap status`. Operators run `servel node swap enable …` to apply.

## Per-node policy

Each node has either a per-host override or inherits the cluster default. Persisted in `/var/servel/cluster/nodes.json` under `swap_policies[hostname]`.

| Field | Default | Purpose |
|-------|---------|---------|
| `Enabled` | true | Master switch — daemon ignores the node when false |
| `MinGB` | 2 | Disk-tier floor (also initial swapfile size) |
| `MaxGB` | 8 | Disk-tier ceiling (elastic loop refuses to grow past) |
| `StepGB` | 2 | Each elastic grow/shrink delta |
| `Zram` | true | Whether to install the zram unit |
| `ZramSizeGB` | 2 | zram device size (set once at install) |
| `ZramAlgo` | lz4 | Compression algorithm (kernel-fallback chain) |
| `Swappiness` | 10 | `vm.swappiness` applied at swapfile ensure |

Use `servel node swap enable <hostname> --max <GB>` etc. to write a per-node override. The merge preserves any field not explicitly set in the flags.

## Operator workflow

```bash
# Inspect cluster swap state
servel node swap status                   # cluster-wide table
servel node swap status kn-deployments    # detail view + advisor block

# First-time fleet rollout: apply cluster default to every node
servel node swap enable --all --apply-now --yes

# Memory-bound workload — raise the cap on one node
servel node swap enable kn-deployments --max 16G

# Disable on a node where the kernel module is missing (rare)
servel node swap enable worker-1 --no-zram

# Switch to higher-compression algo (workload that lz4 doesn't compress well)
servel node swap enable kn-deployments --zram-algo zstd

# Stop the daemon's elastic loop on a node (keeps existing swap state in place)
servel node swap disable worker-1

# Tear down zram + swapfile on a node entirely
servel node swap disable worker-1 --purge

# One-shot manual resize (when you can't wait for the elastic tick)
servel node swap resize kn-deployments --to 6G

# Bypass safety gates during maintenance window
servel node swap resize worker-1 --to 2G --force
```

## Advisor recommendations

After a node has accumulated 3+ days of observation history, the advisor produces zero or more `Suggestion` items per node, sorted by confidence:

| Kind | Trigger | Apply via |
|------|---------|-----------|
| `adjust-max` (grow) | At MaxGB ≥ 30% of last 7 days | `servel node swap enable <host> --max <bigger>` |
| `adjust-max` (shrink) | 7-day peak < 5% (rarely touched) | `servel node swap enable <host> --max <smaller>` |
| `adjust-zram-size` | zram peak fill ≥ 90% | `servel node swap enable <host> --zram-size <bigger>` |
| `adjust-zram-algo` | zram avg ratio < 2x with non-zstd algo | `servel node swap enable <host> --zram-algo zstd` |
| `enable` | Disk peak ≥ 30% with zram disabled | `servel node swap enable <host>` |

Confidence tiers (`high`, `medium`, `low`) reflect signal strength. High-confidence suggestions surface in `node swap status` colored as warnings; medium as info; low as muted.

The advisor caps `MaxGB` at `min(host RAM × 1, 32GB hard cap)` — beyond that, the right answer is more RAM, not more swap.

## Alert conditions

| Condition | Severity | When |
|-----------|----------|------|
| `swap_configured` | info | Initial swapfile created |
| `zram_configured` | info | zram unit installed |
| `zram_configure_failed` | warning | modprobe / mkswap / swapon failed |
| `zram_drift_detected` | warning | Live zram differs from policy |
| `swap_resized` | info | Daemon grew or shrank disk swap |
| `swap_resize_blocked` | warning | Resize attempt failed |
| `node_undersized` | warning | Held at MaxGB ≥ 24h (chronic shortage) |

## File layout

| Path | Owner | Purpose |
|------|-------|---------|
| `/etc/systemd/system/servel-zram.service` | daemon | Boot-time zram install (deterministic, hash-gated rewrite) |
| `/swapfile` | daemon | Disk-tier swap, registered in `/etc/fstab` |
| `/var/servel/cluster/nodes.json` | shared | Per-node `SwapPolicy` overrides under `swap_policies[hostname]` |
| `/var/servel/cluster/swap_history.json` | daemon | 8-day rolling daily aggregates per host |
| `/var/servel/cluster/swap_recommendations.json` | daemon | Advisor output read by `node swap status` |

## Config knobs

```yaml
# /var/servel/config.yaml
swap_ensure_enabled: true       # 1h ensure tick (Phase 1+2)
swap_ensure_interval: 1h
swap_min_gb: 2                  # Cluster default for swapfile floor

swap_elastic_enabled: true      # 5m elastic resize tick (Phase 3)
swap_elastic_interval: 5m

swap_advisor_enabled: true      # 6h advisor tick (Phase 4)
swap_advisor_interval: 6h
```

## Elastic decision actions

`internal/swap/elastic.go:Decide` is a pure function — same input always produces the same output. The decision is one of:

| Action | Meaning |
|--------|---------|
| `grow` | Sustained high pressure → bump swapfile by `StepGB` (capped at `MaxGB`) |
| `shrink` | Sustained low pressure AND current swap usage < 50% → reduce by `StepGB` (floored at `MinGB`) |
| `hold` | Mixed signals in the stability window — at least one sample inside the band |
| `skip:disabled` | Per-node `Enabled = false` |
| `skip:cooldown` | Last resize within `ResizeCooldown` (default 1h) |
| `skip:no-samples` | Fewer than `MinSamples` (3) within `StabilityWindow` (15min) |
| `skip:mem-pressure` | RAM > 95% — swapoff blip during resize would risk OOM |
| `skip:disk-full` | `/` > 85% — refuse grow (no headroom for bigger swapfile) |
| `skip:deploy-active` | Cluster-wide deploy in flight (any node) |
| `skip:shrink-busy` | Current swap_used > 50% of size — pages would have to migrate back |
| `skip:at-min` | Already at `MinGB`, can't shrink further |
| `skip:at-max` | Already at `MaxGB`, can't grow. Tracked separately — fires `node_undersized` after 24h |
| `skip:policy-mismatch` | Inspection returned size 0 — swap not configured, let `swap_ensure` create it first |

The "every sample agrees" rule: even one transient dip past the threshold breaks the streak. Better to wait another tick than to flap.

## Stopped-deployment display in `servel ps`

Shipped alongside the elastic-swap commit (2026-05-08). The replicas column now distinguishes four states; `servel ps --sort status` orders the worst first:

| Replicas | State | Sort priority | Color |
|----------|-------|---------------|-------|
| `0/N` (running = 0, desired > 0) | Crashed / scheduling failure | `0` (worst, top) | red |
| `M/N` (running < desired, M > 0) | Partial — some unhealthy | `1` (middle) | yellow |
| `N/N` | All healthy | `2` (after problems) | green |
| `0/0` | Intentionally stopped (`servel stop`) | `3` (after healthy) | dim |

The fourth bucket exists because `0/0` is a deliberate state, not a problem. Without it, every `servel stop`-ed deployment looked like a crash and noise-rotated to the top of `--sort status`. Code: `internal/cli/ps_helpers.go:getReplicaStatus`.

## Cross-references

- [AUTO_UPDATE.md](AUTO_UPDATE.md) — zram install and elastic resize ship together with signed auto-update
- [AUTONOMOUS_REMEDIATION.md](AUTONOMOUS_REMEDIATION.md) — same Tier 1 (advisory) / Tier 2 (auto-apply) pattern as rebalance and right-size
- [DASHBOARD.md](DASHBOARD.md) — `servel dashboard` includes node swap headroom in the subsystem view
- Design doc: `docs/internal-docs/features/node/SWAP_MANAGEMENT.md` in the servel repo
