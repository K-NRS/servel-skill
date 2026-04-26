# Distributed Storage Substrate

Servel supports replicated volumes via LINSTOR + DRBD. Replicated
volumes survive node loss without operator intervention and turn cross-node
migration into a 3-8 second volume-promotion operation regardless of data
size.

## When to enable

| Cluster shape | Recommended |
|---------------|-------------|
| 1 node | Skip — replication needs ≥2 nodes |
| 2 nodes (no tiebreaker) | Skip — split-brain risk |
| 2 nodes + diskless tiebreaker on manager | Enable for HA-critical infra only |
| 3+ nodes | Enable for any DB/storage that migrates frequently |

## Commands

```bash
servel storage status        # health report
servel storage enable        # provision LINSTOR + DRBD across cluster
servel storage doctor        # deep diagnostics (volume list, sync, pool usage)
```

## Operator prerequisites

Not handled by `servel storage enable`:

- LVM VG named `linstor-vg` on each node with free thin-pool capacity
- Kernel: Ubuntu 22.04/24.04 or Debian 12
- Outbound TCP 3366-3368 between nodes (LINSTOR + DRBD control)

The `enable` command surfaces these as a remediation hint when prereqs are missing.

## Per-infra opt-in

```bash
servel infra add postgres mydb --replicated
servel infra add postgres mydb --replicated --storage-replicas 3
```

Flags:
- `--replicated` — store volumes on the substrate
- `--storage-replicas N` — synchronous replica count (default 2 with tiebreaker, 3 without)

## Preflight gates

`--replicated` deploys are refused with clear remediation when:

| Condition | Error guidance |
|-----------|----------------|
| Substrate not enabled | "run `servel storage enable` first" |
| Cluster has <3 satellite nodes | "minimum 3 nodes for full quorum" |
| Requested replicas > available nodes | "add more nodes or reduce --storage-replicas" |
| `--storage-replicas 2` without tiebreaker | "needs diskless arbitrator to avoid split-brain" |

## Migration semantics

Once a service is on replicated volumes, `servel move` automatically takes the
replicated path. No flag needed:

```bash
servel move @mydb --to worker-2
# 1. Verify target is a synced replica (else refuse with linstor resource create hint)
# 2. Stop service on source
# 3. Swap Swarm constraint to target
# 4. Start — LINSTOR plugin auto-promotes local replica
# Total downtime: 3-8s for any volume size
```

Falls back to snapshot/fullcopy if substrate becomes unavailable mid-migration.

## Failure modes

| Scenario | Behavior |
|----------|----------|
| Target isn't a replica | Refuses with `linstor resource create <vol> <node>` hint |
| Target replica out-of-sync | Refuses (DRBD won't promote stale data anyway) |
| Substrate unavailable | Falls back to snapshot/fullcopy automatically |
| Pool >85% full | Block new replicated deploys (Phase 4.1 — pending) |
| Split-brain (2-replica without tiebreaker) | Preflight refuses at deploy time |

## Files (for implementers)

```
src/internal/storage/
├── storage.go                    # Manager + Capabilities + IsDistributedVolume
├── linstor/{client,installer}.go # LINSTOR API wrapper + provisioning
└── drbd/{installer,status}.go    # DRBD kernel module + sync state

src/internal/infra/
├── migration_replicated.go       # migrateReplicated, canReplicated, ensureReplicaOnNode
└── replicated_preflight.go       # ReplicatedPreflight check at add-time

src/internal/cli/
├── clistorage/storage.go         # servel storage {enable, status, doctor}
└── add.go                        # --replicated, --storage-replicas flags
```

## Phase 4.1 follow-ups (not yet shipped)

- Live `servel storage enable` install (gated on LVM VG provisioning + kernel matrix validation)
- Diskless tiebreaker auto-provisioning (currently manual)
- `linstor resource create` helper to auto-add replicas during migration
- Per-replica sync-state inspection (currently relies on DRBD's own pre-promotion safety)
- Pool exhaustion monitoring (>85% block new replicated deploys)
