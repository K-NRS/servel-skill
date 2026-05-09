# Signed Auto-Update

Servel ships a signed self-upgrade pipeline for the CLI binary, the server-side daemon, and configured remotes. Landed 2026-05-07 in commit `d3c4db3`. Notify-only by default — operators opt in to auto-apply per machine.

The pipeline:

1. Fetch signed manifest from `<base-url>/api/version?channel=<channel>` (default base `https://servel.dev`)
2. Compare current vs `latest.version`. For non-stable channels, fall back to BuildID compare (`YYYYMMDD.HHMMSS.commit`, lex-sortable) when SemVer matches
3. Refuse if release is **yanked** (`manifest.latest.yanked: true` is an emergency kill-switch)
4. Download platform binary into a temp file
5. Verify sha256 against `manifest.platforms[i].checksum`
6. Verify ed25519 signature (`servel-sig-v1:<base64>`) over canonical `(version, channel, os/arch, checksum)` payload, against keys embedded in the running binary at build time
7. `chmod +x`, atomic rename (`current → .prev`, `tmp → current`)

Either verification failure aborts before swap. The live binary is never touched until both checks pass.

## CLI commands

| Verb | Mutates | Purpose |
|------|---------|---------|
| `servel upgrade` | binary | Self-upgrade local CLI |
| `servel upgrade --check` | nothing | Print whether an update is available, exit |
| `servel upgrade --rollback` | binary | Restore `.prev` binary (offline-safe) |
| `servel upgrade --pin <ver \| off>` | `~/.servel/config.yaml` | Persist a version ceiling for auto-apply |
| `servel upgrade --set-mode <mode>` | `~/.servel/config.yaml` | Persist `notify` / `apply` / `off` mode |
| `servel upgrade --servers` | every configured remote | After local upgrade, also run `upgrade-servers` |
| `servel upgrade-servers` | each remote's binary | Roll the fleet to match client version |
| `servel upgrade-servers --rolling` | as above, sequentially | One-at-a-time with health gate between |

`--pin` and `--set-mode` short-circuit before any network call.

## Channels

| Channel | Use when |
|---------|----------|
| `stable` | Production. Stable line; falls back to alpha when no stable cut exists yet (manifest declares `fallback: true`, alpha BuildID drives the decision) |
| `alpha` | Daily-ish builds, not production-ready, deliberate opt-in |

The fallback-to-alpha case has a subtle gotcha: a `stable` client looking at a `fallback: true` manifest sees `latest.channel = "alpha"` but `requested_channel = "stable"`. `internal/autoupdate/autoupdate.go:HasUpdate` reads `rec.Latest.Channel` (where the artifact actually shipped), NOT `rec.Channel` (what the client asked for) — without that distinction, alpha snapshots that share a SemVer with the stable line would never be detected as "newer" by clients tracking stable.

Fixed live on KN's daemon (commit `dbf3a97`) which had been silently treating every alpha snapshot as "no update available" because the client requested stable and there was no stable release yet.

## Auto-apply gates

`internal/autoupdate/autoupdate.go:ShouldAutoApply` returns `Apply=true` only when ALL of:

1. `rec` advertises a newer version (HasUpdate)
2. Release is not yanked
3. **`version.HasSigningKeys()`** — at least one ed25519 pubkey is embedded. Unsigned dev builds refuse auto-apply unconditionally
4. Release is within `pinned_version` ceiling (if set)
5. Not a major-version bump (when `refuse_major: true`, the default). Major bumps require explicit `servel upgrade`

Each gate has a Reason string. Operators see exactly why an apply was refused.

## Daemon update notifier

The same pipeline runs on every server-side daemon, but **never auto-applies**. Surface only.

- Invoked from the pressure tick (`PressureCheckInterval`, default 5min)
- Network fetch gated to once per 24h (`updateCheckMinInterval`); cache at `/var/servel/cache/update.json`
- **Goroutine dispatch**: `m.checkServerUpdates()` calls `go m.runUpdateBody()` so a slow earlier step (constraint-drift remediation, overlay probe) cannot starve the update check. Mutex on `updateCheckState` serializes overlapping in-flight goroutines. Fix landed in `4b677e9` after live diagnosis on KN where `autoRepairMissingConstraints` was holding the tick goroutine for many minutes — `update.json` was never written
- Logs `servel update available` with current/latest/channel
- Fires `ConditionUpdateAvailable` info alert through every configured channel (Telegram, Slack, Discord, webhook). One alert per discovered version — re-discovering same version is a no-op
- **Read by `servel remote status`** — the recommendation surfaces inline. `internal/cli/cliremote/update_check.go:checkRemoteServelUpdate` reads `cat /var/servel/cache/update.json` on the remote, returns a `Recommendation{Level: "info", Message: "New servel release available: vX.Y.Z", Action: "servel remote upgrade"}`. Returns nil on any error — best-effort surface, never blocks status

## Rolling fleet upgrades

`servel upgrade-servers` walks every configured remote (or one named with `--server <name>`):

1. Probe each remote's current version (parallel reachability check)
2. For each remote needing upgrade:
   - Check `/var/servel/locks/{deploy,build}.*` — skip if a deploy is in flight (override with `--ignore-deploy-lock`)
   - SSH in, run `sudo servel upgrade --force --channel <auto-detected>`
   - Verify with `servel version --json`
   - **Restart `servel-daemon` if installed.** Without this, the binary swap doesn't take effect until reboot — fix landed after a node ran a stale daemon for two days post-upgrade and silently kept executing pre-upgrade code

Sequential execution avoids HTTP/2 stream exhaustion on the download server during fleet-wide upgrades.

`--rolling` adds a per-server health gate: after each upgrade, wait up to `--health-wait` (default 60s) for `servel version --json` to come back healthy before touching the next remote. Abort on first failure — bounds the blast radius of a bad release to one node.

## Signing keys + key rotation

Two pubkey slots are embedded at build time via `-ldflags`:

```
go build -ldflags "
  -X github.com/K-NRS/servel/internal/version.SigningPubKey0=<base64-ed25519-pub>
  -X github.com/K-NRS/servel/internal/version.SigningPubKey1=<base64-ed25519-pub>
"
```

Two slots make zero-flag-day key rotation possible:

1. Add new pubkey as `SigningPubKey1` while old signer keeps signing
2. Ship N-1 releases with both slots populated so the install base picks up the new key
3. Switch the release pipeline to sign with the new private key
4. In a later release, move the new pubkey into slot 0 and rotate slot 1 again (or leave empty until next rotation)

When BOTH slots are empty (unsigned dev builds), `version.HasSigningKeys()` returns false. Auto-apply refuses; operator-driven `servel upgrade` still works (signature verification skipped — there are no keys to verify against).

`version.IsDevBuild()` returns true for `-dev` / `-snapshot` / `-local` suffixes. Surfaced in `servel version` so operators recognize an untrusted local build.

## Release signing tool

`cmd/servel-sign/main.go` is the operator tool that produces signed manifest entries from a private key + binary. Not shipped in distributed binaries — used in the release pipeline only.

Canonical signed payload (`internal/upgrade/verify.go:canonicalManifestPayload`):

```
servel-manifest-v1
<version>
<channel>
<os>/<arch>
<sha256-lowercase>
```

Any change to this format must bump the `servel-sig-v1:` prefix in lockstep so old clients reject the new format cleanly rather than mis-validating it.

## Config

`~/.servel/config.yaml` — client side:

```yaml
auto_update:
  enabled: true              # master switch (default true)
  channel: stable            # stable | alpha (default stable)
  mode: notify               # notify | apply (default notify)
  check_interval: 24h        # min gap between network fetches
  pinned_version: ""         # if set, never auto-apply past this
  refuse_major: true         # block auto-apply across major bumps (default true)
```

Server side: no new config knobs. Cadence is `pressure_check_interval` (default 5min, floor 1min). The 24h fetch gate is a `const` in `internal/daemon/update_check.go`.

## Failure model

Every error path is non-fatal:

- Network fetch failure → return cached record (may be stale, better than nothing) and don't update the cache timestamp so we retry on the next check
- Cache read failure → re-fetch
- Cache write failure → ignored, just refetch sooner
- `panic` anywhere in the update path → `defer recover()` traps it. **A broken update check must NEVER prevent the user's command from succeeding**
- `atomic.Bool` so a chained subcommand (e.g. `servel deploy && servel ps`) only runs the check once per process
- Suppression: `last_notified_version` in cache means the same release won't re-notify until a newer one publishes. Delete `~/.servel/cache/update.json` to re-trigger.

## File layout

| Path | Owner | Purpose |
|------|-------|---------|
| `~/.servel/cache/update.json` | client | Last-fetched manifest snapshot + suppression cursor |
| `/var/servel/cache/update.json` | daemon | Same, server side. Read by `servel remote status` |
| `<binary>.prev` | client | Previously-installed binary, preserved by every `servel upgrade` |
| `<binary>.rollback-tmp` | client | Three-way swap intermediate, cleaned at end of rollback |
| `~/.servel/config.yaml` | client | `auto_update:` block |
| `/var/servel/locks/{deploy,build}.*` | daemon | In-flight deploy markers; checked by `upgrade-servers` |

## Cross-references

- [SWAP.md](SWAP.md) — zram swap and the elastic disk-tier resize loop landed in the same `d3c4db3` commit cycle as the auto-update system
- [AUTONOMOUS_REMEDIATION.md](AUTONOMOUS_REMEDIATION.md) — the daemon-side notifier follows the same Tier 1 (surface only) pattern; auto-apply on the daemon would be Tier 2 and is not implemented
- [DASHBOARD.md](DASHBOARD.md) — `servel dashboard` shows server-vs-client version drift inline
