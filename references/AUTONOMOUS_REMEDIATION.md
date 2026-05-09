# Autonomous Remediation

Servel's daemon ships a tiered autonomous-remediation surface (landed 2026-05-07). The design philosophy: **Tier 1 always on, Tier 2 opt-in**.

- **Tier 1** — diagnostic/advisory. Runs unconditionally when `rebalance_enabled: true`. Surfaces findings inline in `servel cap` and emits informational alerts. Operator decides whether to act.
- **Tier 2** — auto-execute. Default off. Only acts when every safety gate passes. Reuses existing operator-driven code paths via subprocess (so `servel move`, `docker service update --constraint-add`, etc. are the single source of truth — no duplicated logic in the daemon).

Operators get visibility before the daemon acts. The daemon only acts when each gate has been justified by a specific failure mode.

This document covers all six remediations shipped on 2026-05-07. Cross-links to [MIGRATION.md](MIGRATION.md), [DASHBOARD.md](DASHBOARD.md), [STORAGE.md](STORAGE.md) where relevant.

---

## 1. Auto-Rebalance: memory→tasks fallthrough

**What it does.** The daemon's stateless rebalance planner now defaults to `rebalance_strategy: auto`. Auto runs the **memory** planner first (safer — has gap-narrowing check). If it produces zero migrations AND the stateless task spread exceeds `rebalance_task_delta`, retries with the **tasks** planner. Either result feeds the same migration execution path.

**Why.** Pre-2026-05-07 the daemon was hard-coded to memory strategy. Memory simulation only adjusts `UsedMem` per move; CPU/units stay fixed in the sim. So when combined-pressure was dominated by CPU (e.g. KN-MANAGER 102% CPU, 60% memory), the planner kept rejecting moves via `DecisionSkipGapNotNarrow` and fired `rebalance.blocked` alerts forever — operator had to run `servel node balance --strategy tasks` by hand.

**Config knobs:**

| Field | Default | Clamp | Notes |
|-------|---------|-------|-------|
| `rebalance_strategy` | `"auto"` | `auto` / `memory` / `tasks` (anything else clamps to `auto`) | Picks the planning dimension |
| `rebalance_task_delta` | `5` | 2 – 100 | Max-min stateless task gap before auto falls through to tasks |

**Observability.** Daemon log line `auto-rebalancing cluster strategy=tasks` indicates the fallthrough fired. `Result.Strategy` is overwritten with the strategy that actually planned the moves (so `--dry-run` output is honest about what happened).

**Manual override.** `servel node balance --strategy memory|tasks` for one-shot operator planning. CLI defaults remain unchanged (memory) — only the daemon defaults to auto.

**Code:** `internal/daemon/config.go` (defaults + clamps), `internal/daemon/rebalance.go` (passes Strategy + TaskDelta into Rebalancer), `internal/rebalance/types.go` (`StrategyAuto` constant), `internal/rebalance/plan.go` (`planByStrategy` pure helper).

---

## 2. Right-size advisor (Tier 1 + Tier 2)

**What it does.** Every rebalance tick (10m default), the daemon analyzes per-service Docker reservations vs observed usage. Three classes:

| Class | Daemon action | Failure mode |
|-------|---------------|--------------|
| **Over-reserved** | Tier 2: auto-trim via `docker service update --reserve-cpu/--reserve-memory` | Bloats unit ledger, blocks deploys for memory/CPU never touched |
| **Under-reserved** | Tier 1 advisory only | Adding a reservation can break running workloads — operator decision |
| **Over-limit** | Tier 1 advisory only | Container near `--limit-*` — operator decides raise limit vs shrink workload |

**Tier 2 safety bounds (over-reserved auto-apply):**

- ✅ Stateless services only — DBs, queues, storage skipped (suffix-matched via `infra.IsStatefulByName`)
- ✅ High confidence required (≥24 samples; `≥12 = medium`, `≥6 = low`, below 6 → skipped)
- ✅ Bounded delta: ≤25% of current reservation per cycle (large over-reservations converge over multiple ticks, never thrash)
- ✅ Per-service per-dimension cooldown: 24h
- ✅ Cap of 5 mutations per cycle (bounded blast radius across the swarm)
- ✅ Defers when a stateless rebalance migration is in progress (avoids concurrent rolling task updates)

**Suggested values:** `max(EMA × 2, peak × 1.2)`, floored at **50 MiB memory** / **0.05 CPU**.

**Stale gate:** skip if `LastUpdated > 1h ago`.

**Tier 1 surface (`servel cap`):**

```
Reservation Health
  Over-reserved:  4 service-axis pair(s) reclaimable: ~1.8 GiB mem · ~1.40 cpu cores
  Under-reserved: 2 service(s) consume resources but have no reservation
  Over-limit:     1 service(s) approaching configured limit
  top over-reserved:
    · brokriq-api    512 MiB → 192 MiB  (peak 158 MiB)
    · agentkarma-web 1.00 cores → 0.30 cores  (peak 0.22 cores)
```

Hidden when there's nothing actionable.

**Why no separate `servel rightsize` verb?** First pass added one. Operator feedback: "having different commands gets servel complexier than aimed philosophy 'just works'." Diagnostics fold into the operator's existing entry point (`servel cap`); remediation runs autonomously. No new verbs to learn.

**Manual override.** `docker service update --reserve-memory <new> <service>` directly, or set `rebalance_enabled: false` to disable the master switch (also disables stateless rebalance, stateful rebalance, and constraint auto-repair).

**Code:** `internal/rightsize/` (Recommendation, Options, Confidence, Kind, Dimension types + `Analyze` pure function + `CollectServiceResources` / `CollectObservedUsage`), `internal/daemon/rightsize.go` (`checkRightsize`, `rightsizePlan` pure, `applyRightsize`), `internal/cli/capacity/capacity.go` (`RightsizeSummary` + `displayRightsizeSummary`).

---

## 3. Placement load-awareness fix

**What it does.** `infra.selectBestNode` now scores by FREE memory/CPU from `/var/servel/cluster/capacity.json` rather than total hardware. Local + manager bonuses dropped dramatically; unit-load penalty above 50% utilization is **uncapped**.

**Pre-fix:**

| Factor | Points |
|--------|--------|
| Local node | +100 |
| Manager role | +50 |
| Memory (TOTAL) | +0–30 |
| CPU cores (TOTAL) | +0–20 |
| Service count | −5 each, **capped at −50** |

**Post-fix (2026-05-07):**

| Factor | Points |
|--------|--------|
| Local node | +20 (was +100) |
| Manager role | +10 (was +50) |
| Free memory % | +0–50 (`(total − reserved) / total` from capacity snapshot) |
| Free CPU % | +0–30 (mirror) |
| Unit-load penalty | −2 per pp above 50% utilization, **uncapped** |
| Fallback (no snapshot) | total-mem +0–25, total-cpu +0–15, service count −3 each, uncapped |

**Why.** Pre-fix, a `servel add` from the laptop produced +150 baseline (local + manager) before any load consideration. Memory/CPU scoring was independent of actual usage. Service-count penalty cap (−50) couldn't overcome the +150 lead even when 25+ Supabase stacks were already piled on. Result: every `servel add` landed on KN-MANAGER, building up a 141% unit ledger that auto-rebalance couldn't fix (stateful infra, can't auto-move).

**Verified by regression test:** `TestPlacementBugRegression` reproduces the KN scenario — heavily-loaded local manager vs idle remote worker. Pre-fix: manager wins by +150. Post-fix: remote wins by ~140 points (manager penalty for 132% unit utilization is −164; that single penalty alone overrides the local+manager +30 bonus).

**Capacity snapshot dependency.** New scoring reads `/var/servel/cluster/capacity.json` (written by daemon every 5min). On a fresh cluster where the daemon hasn't ticked yet, scoring falls back to the legacy total-hardware path with adjusted weights (still better than before — service-count penalty now uncapped).

**Code:** `internal/infra/placement.go` (refactored `selectBestNode` to delegate to scoring helpers; reads capacity snapshot best-effort), `internal/infra/placement_score.go` (pure scoring helpers), `internal/infra/placement_score_test.go` (regression test).

---

## 4. Capacity uses Reservations only (CRITICAL FIX)

**What it does.** `cluster/capacity_collector.go` populates per-node `ReservedMemoryMB` / `ReservedCPUCores` / `AvailableMemoryMB` / `AvailableCPUCores` from `Spec.TaskTemplate.Resources.Reservations` ONLY. When a service has Limits but no Reservations, it contributes ZERO to the ledger — same as Docker Swarm's actual scheduler.

**Why this is critical.** Pre-fix code used Limits when Reservations were nil ("fall back to limits if no reservations"). Logic seemed conservative (over-count rather than under-count) but it didn't match Docker scheduling reality. Most templates (Supabase, Chatwoot, etc.) ship with `--limit-memory 1G --limit-cpu 1.0` and NO Reservation, because the operator-intent is a cgroup cap, not a scheduling commitment. Summing Limits inflated the ledger to 141% / -7.5-cores-available even when Docker would happily schedule.

**Symptom (KN, 2026-05-07):** `servel deploy agentkarma` failed with `insufficient CPU: need 0.25 cores, have -7.50 available` while real CPU on KN-MANAGER was 78%. Plenty of room. Preflight rejected because it was counting cgroup caps as scheduling commitments.

**Mechanism.** Docker Swarm scheduler refuses to place a task if no node has enough free Reservation budget. Limits are runtime cgroup caps applied AFTER scheduling — they don't influence "where does this task go?" decisions. Servel's preflight must mirror Docker's view.

**Separation of concerns:**

```go
type ResourceReservation struct {
    MemoryMB      int64   // sum of Reservations.MemoryBytes — feeds ledger
    CPUCores      float64 // sum of Reservations.NanoCPUs    — feeds ledger
    LimitMemoryMB int64   // sum of Limits.MemoryBytes        — display only (future overcommit warning)
    LimitCPUCores float64 // sum of Limits.NanoCPUs           — display only
    TaskCount     int
}
```

`NodeCapacity.AvailableMemoryMB = TotalMemoryMB − reservation.MemoryMB`. NEVER include `LimitMemoryMB` in this math.

**Test fixture rule.** When testing capacity collection, always set `Reservations` if the test asserts `ReservedMemoryMB > 0`. Setting only Limits used to work (because of the buggy fallback) but now correctly reports 0 reserved.

**Code:** `internal/cluster/capacity_collector.go`, `internal/cluster/capacity.go`.

---

## 5. Constraint drift auto-repair (Class 1) + Stale binding auto-prune (Class 2)

### Class 1: missing_constraint auto-repair

**What it does.** When the daemon's placement-drift check finds a `missing_constraint` (a service should pin to a node but isn't) AND the target node is alive in the swarm AND `rebalance_enabled = true`, it auto-applies `docker service update --constraint-add` (per-service 30min cooldown).

**What it explicitly does NOT do.** `wrong_node` drifts (service running on a different node than expected) and drifts targeting a vanished node are NEVER auto-repaired — those carry data-loss risk and stay surfaced as alerts for operator decision.

**Code:** `internal/daemon/constraint_check.go` (`checkConstraintDrift`, `autoRepairMissingConstraints`).

### Class 2: stale binding auto-prune

**What it does.** When the data-heal loop hits `errNoServices` ("no docker services discovered for binding") for **6 consecutive cycles** (~1h at default `data_heal_check_interval: 10m`), the daemon auto-prunes the stale binding:

- Clears `placement_state.data_node`, `placement_state.data_node_hostname`, `placement_state.bound_at`
- Flips `meta.json` status from `running` to `stopped` (only this transition; `failed`/`degraded` left alone)
- **Volumes + meta.json preserved** — operator can `servel start` to revive, or `servel rm` to fully delete

**TOCTOU guards:** spec re-read right before mutating; if `DataNode` no longer matches the stale ID, returns `ErrTOCTOUSkip`. Never deletes files — only zeros placement_state fields and flips status. All changes are reversible by editing meta.json.

**Alert.** `auto-pruned stale data binding` (warn-level), structured payload identifies the infra/deploy, old node ID, cycles-to-prune count, and old → new status.

**Code:** `internal/daemon/dataheal.go` (cycle counter + threshold), `internal/dataheal/prune.go` (`PruneStaleBinding`, `pruneInfraBinding`, `pruneDeployBinding`).

---

## 6. Stateful autonomous rebalance (the big one)

The stateless rebalancer never touches DBs, queues, storage. As of 2026-05-07 the daemon ships a two-tier path for the stateful side:

### Tier 1 (always on when `rebalance_enabled: true`)

`rebalance.DetectStatefulConcentration` runs every rebalance tick (10m default). Pure function, no SSH, no I/O — caller assembles `NodeLoad` + `InfraOnNode` slices.

**Detection algorithm:**

1. Cluster avg = mean tasks per **populated** node (zero-task nodes excluded so cold spares don't drag the average down)
2. Hot node = `tasks / avg ≥ rebalance_stateful_min_task_ratio` (default 3.0)
3. Dominant stateful = largest stateful infra on the hot node, must be ≥25% of node's tasks
4. Target = node with highest combined free-resource score `(memFree% + cpuFree%) / 2`, excluding nodes with real CPU > 80% or real memory > 85%

**Surface in `servel cap`:**

```
Stateful Concentration
  1 node(s) over the 3×-cluster-average task threshold; recommended moves below.
  · servel move @brokriq-db --to kn-deployments
    KN-MANAGER holds 24 tasks (3.4×cluster avg 7.0); brokriq-db contributes 8 (33% of node), target kn-deployments has most free real resources
```

Top 3 by impact (largest infra-task count first). Plus `SeverityInfo` alert.

### Tier 2 (opt-in via `rebalance_stateful_auto: true`)

When Tier 1 detects something AND every gate passes, the daemon auto-executes via local subprocess: `/usr/local/bin/servel move @<infra> --to <target> --yes`.

**Why subprocess (not in-process).** Daemon runs on the manager so the binary is local; no SSH overhead. Subprocess execution keeps every existing `servel move` safety check (`infra/manager_migration.go`) as the single source of truth — daemon orchestrates "when" and trusts `servel move` for "how".

**Tier 2 gates (all must pass):**

| # | Gate | Failure mode it prevents |
|---|------|--------------------------|
| 1 | `rebalance_enabled = true` | Operator's master kill switch — disables every autonomous remediation in one place |
| 2 | `rebalance_stateful_auto = true` | Default-off opt-in — stateful moves should only happen with explicit operator consent |
| 3 | Migration size ≤ `rebalance_stateful_max_bytes` | Large infras carry multi-minute downtime — operator should drive those |
| 4 | ≥24h cluster-wide cooldown | Don't thrash the cluster with back-to-back stateful moves |
| 5 | ≥7-day per-infra cooldown | Prevents ping-ponging the same DB between two nearly-balanced nodes |
| 6 | `/var/servel/build-queue` empty | Avoids contending with operator-driven deploys |
| 7 | Cluster topology stable for ≥1h (proxy: daemon `StartedAt`) | Don't migrate while nodes are joining/leaving |
| 8 | Target real CPU ≤70% AND real memory ≤80% | Don't move onto a hot node — would just shift the problem |
| 9 | Target free disk ≥2× migration size | Snapshot strategy needs headroom for baseline + delta |

When any gate fails, recommendation stays in Tier 1 only — `servel cap` keeps showing it, operator runs `servel move` when ready.

**Config knobs:**

| Field | Default | Clamp |
|-------|---------|-------|
| `rebalance_stateful_auto` | `false` | — |
| `rebalance_stateful_max_bytes` | `1073741824` (1 GiB) | 100 MiB – 10 GiB |
| `rebalance_stateful_min_task_ratio` | `3.0` | 1.5 – 10 |

**Observability:**

- **`servel cap`** — "Stateful Concentration" section, top 3 with exact commands
- **Alerts** — `ConditionClusterRebalanceBlocked` (Tier 1 detection), `ConditionClusterRebalanced` (Tier 2 success), `SeverityCritical` (Tier 2 failure)
- **Daemon logs** — `stateful concentration detected` (info), `stateful auto-move executing/completed/failed` (warn/error)
- **State** — `state.StatefulRebalance.Records` (capped 50): `FromNode`, `ToNode`, `BytesMoved`, `DurationSec`, `Status`
- **Migration history** — every Tier 2 attempt also lands in `/var/servel/migrations/history.jsonl` (visible via `servel move history`)

**Manual override.** `servel move @<infra> --to <node>` works exactly as before. Daemon doesn't compete — if an operator-driven move is in flight (build queue non-empty), Tier 2 defers (gate 6).

**Code:** `internal/rebalance/stateful.go` (pure detector + target selector), `internal/rebalance/stateful_collect.go` (SSH-side gather), `internal/daemon/stateful_rebalance.go` (`checkStatefulRebalance` + `evaluateStatefulGates`), `internal/daemon/state.go` (`StatefulRebalanceState`, `StatefulMoveRecord`).

---

## Master switch + override paths

Every autonomous remediation gates on `rebalance_enabled` (default `true`). Setting it to `false` disables in one shot:

- Stateless auto-rebalance (memory + tasks fallthrough)
- Right-size advisor auto-trim
- Constraint drift Class 1 auto-repair
- Stale binding Class 2 auto-prune (still detects + alerts; just doesn't prune)
- Stateful auto-rebalance (both Tier 1 and Tier 2)

Tier 1 surfaces in `servel cap` come back the moment `rebalance_enabled` flips back on; daemon doesn't need a restart for that.

To disable just one feature without killing the rest:

| Feature | Disable knob |
|---------|--------------|
| Stateless rebalance auto-execute | Set `rebalance_strategy: memory` and `rebalance_threshold` higher than achievable spread |
| Right-size auto-trim | (no per-feature knob — flip `rebalance_enabled` or override reservations directly) |
| Constraint drift auto-repair | Same — bound to master switch |
| Stale binding prune | Set `data_heal_enabled: false` |
| Stateful Tier 2 | Set `rebalance_stateful_auto: false` (Tier 1 stays on) |

---

## Cross-references

- [MIGRATION.md](MIGRATION.md) — `servel move` is what stateful Tier 2 invokes via subprocess
- [DASHBOARD.md](DASHBOARD.md) — `servel dashboard` shows daemon status + last migration row
- [STORAGE.md](STORAGE.md) — replicated volumes change the migration-strategy decision tree (but not the autonomous-remediation gates)
- [SWAP.md](SWAP.md) — elastic swap (zram + disk) follows the same Tier 1/Tier 2 split: the resize loop auto-applies within policy bounds; the advisor only surfaces recommendations

## Design philosophy

**Tier 1 always-on diagnostics + Tier 2 opt-in execution** is the pattern the entire 2026-05-07 release follows. Operator visibility before the daemon acts. Every safety gate is justified by a specific failure mode that would happen without it (the table above isn't paranoid — each row corresponds to a real incident class). Subprocess execution wherever possible keeps existing operator-driven verbs as the single source of truth — no parallel daemon-only migration logic to drift from `servel move`.
