---
name: servel
description: Self-hosted deployment platform. Use when (a) deploying or operating apps and infrastructure on a remote server; (b) working in a repo that contains `.servel/` or `servel.yaml` and the user references the app, the deployment, production, the live container, or the env/logs/secrets/routing of the running version; (c) debugging a deployed app — "why is prod 500", "show me the logs", "tail the logs", "what env did the container actually get", "the live app is crashing", "production is slow", "exec into the container", "my deployment is down", "the deployed version misbehaves"; (d) any infrastructure task — postgres, redis, supabase, mongodb, libsql, mysql, mongodb-ha, postgres-ha, backup, restore, registry (ghcr / gitlab / self-hosted), domains, SSL, traefik routing, secrets, alerts, capacity, analytics, visitors, rebalance, node management, access control, bastion, tunnel, port-forward, swap / zram / OOM, dev mode (link-env / link-infra / pull prod env to dev), CI/CD, auto-update / rolling upgrade, cross-node migration (servel move), volumes, audit, IP bans, security checkpoint, visitor IP / X-Real-Ip / CF-Connecting-IP. Enforces dogfooding: ALWAYS prefer servel commands over raw docker / ssh / rsync / docker-compose for any operation servel can perform.
---

# Servel

Deploy applications and infrastructure to Docker Swarm with Vercel-like simplicity. Auto-detects project type, provisions SSL, zero-downtime rolling updates.

## ⚡ FIRST CHECK — Is there a `.servel/` directory?

**Before answering any question about a project, check for `.servel/` in the working directory (or any ancestor). If it exists, this project is already deployed via servel and you have a direct line to the live production deployment — use it.**

```bash
ls .servel/ 2>/dev/null && echo "deployed via servel"
```

If `.servel/state.json` exists, **every servel command auto-resolves to this project's deployment — no name argument needed**. Before reading code, guessing, or asking the user what's wrong in production, run:

```bash
servel logs -f          # live tail of production
servel logs --tail 200  # what just happened
servel logs --since 10m # recent window
servel logs --build     # last deploy's build output
servel env vars         # what env the running container actually sees
servel inspect          # image, replicas, health, command
servel exec sh          # shell into the live container
servel verify           # routing + SSL + DNS sanity
```

**`servel logs` is the single most important command when a `.servel/` directory is present.** It is the ground truth for "what is production doing right now" — cheaper, more accurate, and more current than reading the codebase and guessing. Reach for it **before** static analysis whenever the user mentions prod, the deployment, the live app, an error, a 5xx, slowness, or "why is X happening".

This applies to **every** service-targeting command: `logs`, `exec`, `inspect`, `env`, `restart`, `stop`, `start`, `scale`, `redeploy`, `rollback`, `rm`, `deploy`, `verify`, `secrets`, `domains`, `routes`, `analytics`. None of them need the deployment name when run from inside a `.servel/`-tracked project.

Multi-env: if `.servel/state.staging.json` exists too, add `--env staging`. Linked infra: `servel logs @<infra> -f` and `servel exec @<infra> --service db sh`.

## Always Use Servel Commands

**Native servel commands are safer than ssh+docker.** Every native verb:
- Reads/writes `/var/servel/` state atomically (no half-applied changes on crash)
- Records to the audit log (`servel audit list` — who/what/when, forensics-grade)
- Is multi-node-correct (SSH-hops to the node hosting the container; raw docker on a manager misses worker containers)
- Validates inputs and preserves invariants (Traefik labels, networks, constraints, healthchecks — easy to drop with raw `docker service update`)
- Triggers downstream reconciliation (drift detection, post-deploy probes, rollback on failure, alert dispatch)
- Survives upgrades — raw docker commands codify today's swarm shape; servel commands are forward-compatible
- Is idempotent and rerunnable — raw `docker service update` flag-bombs are not

Raw `ssh user@host -- docker ...` skips all of the above. State, audit, multi-node correctness, invariants — all gone. Even when the immediate effect looks identical, you've taken on silent risk. **Default: native command. Fallback to ssh+docker only when no native verb exists, and say so out loud.**

| Instead of... | Use servel |
|---|---|
| `ssh user@server` | `servel ssh <server>` |
| `docker exec` | `servel exec <name> sh` or `servel exec <name> -- cmd` |
| `docker logs` | `servel logs <name> -f` |
| `docker service create/update/rm` | `servel deploy`, `servel rm` |
| `docker service scale X=N` | `servel scale <name> N` |
| `docker service scale X=0` (stop) | `servel stop <name>` (or `servel scale <name> 0`) |
| `docker service scale X=1` (start after stop) | `servel start <name>` (or `servel scale <name> 1`) |
| `docker service update --force` | `servel restart <name>` |
| `docker stack/compose` | `servel deploy` (compose auto-detected) |
| `docker ps` / `docker service ls` | `servel ps`, `servel node ps` |
| Manual DB setup | `servel add postgres --name mydb` |
| `rsync` / `scp` to server | `servel dev` |
| Manual backups / `pg_dump` | `servel infra backup <name>` |
| `docker exec psql < file.sql` | `servel infra sql @mydb file.sql` |
| Raw `curl` health checks | `servel verify health <name>` |
| Manual `iptables -A INPUT -s X -j DROP` | `servel ban <ip>` |
| Per-app IP blocking via raw Traefik labels | `servel ban <name> <ip>` |
| Manual `rsync` of Docker volumes between nodes | `servel move @<infra> --to <node> --fast` |
| `docker service update --constraint-add node.hostname==` for stateful | `servel move @<infra> --to <node>` (moves data too) |
| Manually checking 5 subsystems with separate commands | `servel dashboard` (one screen — every optional subsystem) |
| Eyeballing benchmarks to guess if the fast path is actually fast at your data size | `servel bench migration --target <node> --size 1GB` (deploys ephemeral postgres, measures both strategies, prints comparison table) |
| Grepping `journalctl` to figure out what migrated, when, and how long it took | `servel move history` (audit trail at `/var/servel/migrations/history.jsonl`) |
| `du -sh /var/servel/* /var/lib/docker/*` to find what's eating disk | `servel df --growers` (curated 16-path scan + drills into volumes, classifies as orphan/active/system/archive/migrate/backup) |

### 🚫 Banned escape hatch: `servel ssh <host> -- docker service ...`

**Before emitting any `servel ssh <host> -- "docker service <verb> ..."` command, STOP.** This is the most common dogfooding leak — using `servel ssh` as a transport for raw docker bypasses state tracking, audit logs, drift detection, and rollback safety. Rewrite to the native verb:

| If you typed... | Use instead |
|---|---|
| `servel ssh H -- "docker service scale X=N"` | `servel scale <addr> N` |
| `servel ssh H -- "docker service scale X=0"` | `servel stop <addr>` |
| `servel ssh H -- "docker service scale X=1"` | `servel start <addr>` |
| `servel ssh H -- "docker service update --force X"` | `servel restart <addr>` |
| `servel ssh H -- "docker service rm X"` | `servel rm <addr>` |
| `servel ssh H -- "docker service update --image Y X"` | `servel redeploy <addr>` or `servel rollback <addr>` |
| `servel ssh H -- "docker service ls"` | `servel ps --remote H` |
| `servel ssh H -- "docker service logs X"` | `servel logs <addr> -f` |
| `servel ssh H -- "docker exec ..."` | `servel exec <addr> -- ...` |

`<addr>` = symbol-prefixed name: `myapp` (deployment), `@mydb` (infra), `~traefik` (system).

**Service name doesn't fit `servel-{name,infra-name,system-name}-*`?** That service was created outside servel's addressing contract. Options, in order:
1. **Rename to fit the taxonomy** so it becomes addressable (`servel-system-<name>` for a daemon you control, redeploy via `servel deploy` for an app).
2. **Resolve manually with `servel ps --remote H --all`** to find the closest servel-managed equivalent.
3. **Last resort only:** fall back to `servel ssh H -- docker ...` *and explain why no native verb fits in the same response* — never silently. If this happens, it's a gap in servel — file it.

`servel ssh` is for interactive shells and one-off diagnostic commands that have no servel equivalent (kernel checks, `journalctl`, `ip`, `iptables` reads, package queries). It is **not** a docker-CLI proxy.

## Complete Capability Map — scan this BEFORE reaching for ssh+docker

The full top-level command set (from `servel --help`, current as of 2026-05-17). Every row is a native verb; if your task fits a row, **use it** — do not shell out. Subcommands listed after `:` are the most common; run `servel <cmd> --help` for the full subtree.

### Lifecycle (apps + infra)
| Command | What it does |
|---|---|
| `deploy` | Build + push + create/update service + post-deploy probe + auto-heal. Final summary derives status/replicas from current Swarm tasks (`?/N` or `?/global` when unavailable). Compose / Dockerfile / preset / nixpacks auto-detected. |
| `redeploy` | Re-apply stored spec without source code (good for env-only changes; respects `servel.yaml` edits since 2026-05-15 — use `--no-refresh` to disable). |
| `rollback` | Roll to previous image; same convergence + probe contract as deploy. **Bare `rollback` = `live_version − 1`, where "live" is the running generation, NOT the newest record.** A killed/failed deploy leaves a higher-versioned dead record that a bare rollback skips past — servel warns when one exists. To restore a known-good version after a failed deploy, pass it explicitly (`servel rollback <app> <version>`) rather than bare rollback. |
| `restart` | Force-update service (same as `docker service update --force`, but tracked). |
| `scale <name> <N>` | Replicas → N. `0` = stop, ≥1 = start. Works with deployment / `@infra` / `~system`. |
| `stop <name>` / `start <name>` | Aliases for `scale 0` / `scale 1`. |
| `rm` / `remove` | Tear down (deployment, infra, system service). Preserves volumes by default. |
| `rename` | Rename a deployment (state-aware). |
| `promote <src> <tgt>` | Transfer env + domains (swap / merge / cleanup-source / rebuild flags). |
| `preview` | Create a TTL'd preview deployment. |
| `attach` / `watch` | Watch in-flight build (raw log stream vs. phase TUI). |
| `abort` | Cancel in-progress deploy. |
| `resume` | Continue failed deploy from last successful step. |

### Inspect / observe
| Command | What it does |
|---|---|
| `ps` / `ls` / `list` | List deployments (`--all-servers`, `--tree`, `--all`). |
| `inspect` / `info` | Full deployment detail (image, replicas, health, command). |
| `logs <addr> [-f]` | Logs for deployment / `@infra` / `~system`. Per-task streaming since 2026-04-13. |
| `events` | Deployment events timeline. |
| `history` / `versions` | Past deploys + available images. |
| `find <name>` | Cross-server search (works for `@infra` too). |
| `stats` | Container resource usage (CPU/mem/net/io). |
| `url` / `open` | Resolve / open primary URL in browser. |
| `dashboard` | One-screen view of every optional subsystem. |
| `deck` | Interactive TUI dashboard. |
| `verify` | Health + SSL + DNS + routing sanity. |
| `validate` | Validate `servel.yaml` before deploy. |
| `detect` | Show what auto-detection would do for current project. |
| `doctor` | Client/server connectivity + cluster-wide health (`migration` subcommand for end-to-end migration self-test). |
| `df` | Disk usage (`--growers` for curated 16-path scan). |
| `queue` | Build queue status. |

### Exec / shell / tunnel
| Command | What it does |
|---|---|
| `exec <addr> [sh|-- cmd]` | Cross-node-correct (SSH-hops to node hosting the container). |
| `ssh <server>` | Server SSH. **Diagnostic shells only — not a docker proxy.** |
| `attach` | Stream live build log (Ctrl+C detaches, build continues). |
| `debug-shell <app>` | Shell into rootfs snapshot of last crashed task. |
| `port-forward <target> [local-port]` | SSH tunnel to a remote service. `list`, `stop`. |
| `tunnel start <local-port>` | Public dev tunnels. `list`, `logs`, `stop`. |
| `connect <name>` | Print infra connection details (URL, user, password). `--open` launches the web dashboard in the browser, logged in automatically for basic-auth dashboards (e.g. Supabase Studio). |

### Infrastructure (`servel infra ...` + `servel add`)
| Subcommand | What it does |
|---|---|
| `add <type> [name]` | Add infra from hub template (46 types: postgres, redis, mongodb, supabase, chatwoot, openreplay, ...). Supports `--ha`, `--replicated`, `--prefix` bundles. |
| `infra` (no args) | List all infra. |
| `infra check [--all-nodes]` | Diagnose orphans / conflicts / stuck services / **Postgres connection saturation** (≥90% of `max_connections` = critical, SQLSTATE 53300 — catches the "auth/DB connection error" class on Supabase/postgres stacks). Also **ClickHouse system-log bloat** (G.2): any `clickhouse`/`openreplay`/`highlight` infra whose own `system.*` tables hold ≥20% of their volume (≥40% = error, 2 GiB floor). |
| `infra repair @<name>` | Auto-rsync bind sources, alert on drift. |
| `infra backup / restore` | Per-infra backup. |
| `infra sql @<name> [file|dir]` | Run SQL (file, directory of migrations, or interactive shell). Discovers container via `docker service ls`. Directory input is tracked by default (`--no-track` to opt out); `--baseline` adopts tracking on a DB whose schema already exists. |
| `infra logs @<name> [--service X]` | Multi-service infra logs. |
| `infra inspect / status / vars / labels / domains / customize / archives` | Read views. |
| `infra start / stop / restart / remove / rename / update / upgrade / rotate` | Lifecycle. |
| `infra run <name> <action>` / `infra run-hooks` | Lifecycle hooks declared in template. |
| `infra version <name>` | Stack version. |
| `infra cert status [@name] [--json]` | Read-only: cert expiry for self-TLS infra (poste.io-class — terminates own TLS behind Traefik passthrough). |
| `infra cert renew @<name> [--force] [--staging] [--dns-wait <dur>]` | Issue/renew a publicly-trusted Let's Encrypt cert via **DNS-01** (Cloudflare token) and install it in-container (`/data/ssl/server.{crt,key}` for poste). The only ACME path that works behind `traefik-tcp-passthrough` (HTTP-01 is blocked by Traefik's `acme-http@internal` + :80 host-publish collision). Daemon auto-renews <14d-to-expiry (budget+breaker bounded). Needs `servel cf token set`. |
| `link <infra>` / `unlink <infra>` | Wire infra → app (auto-injects env vars, internal DNS since 2026-05-09). |
| `templates` / `hub` | Manage templates, browse Hub registry. |

### Import / migration from other platforms
| Command | What it does |
|---|---|
| `import vercel [slug...]` | Pull Vercel projects (decrypted envs + domains + framework + build overrides) into a staging dir at `~/servel-imports/<scope>/<slug>/`. Read-only against your FS by default; emits `servel.yaml`, `IMPORT.md` (recipe + DNS deltas + storage hints), `secrets.env` (0600). `--all` for whole-team imports, `--team`, `--token` / `$VERCEL_TOKEN`, `--target {production,preview,development}`, `--workspace`, `--concurrency`, `--overwrite`. `--clone` flag is reserved (returns error) — clone manually then `cp servel.yaml` into the repo. Storage envs (POSTGRES_URL, KV_URL, BLOB_READ_WRITE_TOKEN, EDGE_CONFIG) become recommendations, never auto-provisioned. |

### Data / state / volumes / migration
| Command | What it does |
|---|---|
| `data bind / unbind / heal / migrate / status / check / volumes` | Per-service data binding (which node holds the data). |
| `move @<infra> --to <node>` | Cross-node migration (auto-picks `replicated` / `snapshot` / `fullcopy`). `--plan`, `--fast`, `--pointer-only`. |
| `move @<infra> --to-remote <r> --to <n>` | **Cross-remote migration** (different operator-managed remote). `--plan` always safe; execution gated behind `SERVEL_EXPERIMENTAL_CROSS_REMOTE=1`. Single + multi-service supported. |
| `move @<infra> --to-remote <r> --abort` | Cross-remote: idempotent cleanup of partial state on both remotes after a failed migration. Restarts source if stopped. |
| `move @<infra> --to-remote <r> --commit-source-cleanup` | Cross-remote: drop source after successful migration. Refuses if target isn't serving. |
| `move history` | Audit trail of past migrations (intra-swarm AND cross-remote). |
| `volumes [list|inspect|rm]` | Docker volume management. |
| `storage status / enable / doctor` | DRBD/LINSTOR distributed storage substrate. |
| `bench migration --target <node> --size 1GB` | Measure migration strategies head-to-head. |

### Config / env / secrets
| Command | What it does |
|---|---|
| `config [set|get|list|wizard|validate]` | Reflection-based config edit (client `~/.servel/config.yaml` or server `/var/servel/config.yaml`). Incl. `user.name`/`user.email` operator identity (`git config user.*` parallel; gates destructive commands — see Configuration knobs). |
| `env [list|set|copy]` | Plaintext env vars. No rebuild for `set`. |
| `set-env-file` | Wire `env_file` into `servel.yaml`. |
| `secrets [list|get|set|delete|rotate|rotate-all|pull|import|export|copy|reconcile|scan|migrate]` | Age-encrypted secrets. `scan` finds hardcoded secrets in code. |

### Domains / SSL / routing
| Command | What it does |
|---|---|
| `domains [list|add|remove|update|status|redirect|list-redirects|remove-redirect]` | Domain + SSL + redirect management. |
| `domains claim [deployment]` | Claim (or retry) the **magic subdomain** for a deployment. Domainless deploys auto-claim `https://<app>-<hash6>.<magic-domain>` (hash6 = derived from the server's identity key); a failed claim NEVER fails the deploy — it prints `subdomain pending — retry: servel domains claim <name>` and the daemon retries in the background (budget 3/day per marker). This verb is the manual retry; idempotent for the owning server. Auto-detects the deployment from `.servel/state.json`; `--env` for env-scoped deployments. Requires a server binary with subdomain support (`servel upgrade-servers` if it errors with "unknown command"). |
| `dns [verify|cleanup]` | DNS record verification + cleanup. |
| `traefik [status|logs|restart|routes|certs|test|debug|pin|unpin|where]` | Traefik introspection + control. |
| `routes [debug]` | Route-level debugging. |
| `verify [cf-ssl|...]` | SSL/DNS/health sanity checks. |
| `verify cf-ssl` | Cloudflare SSL-mode probe (Full strict / Full / Flexible / Off). |
| `cf token {set|unset|verify}` | Save / clear / verify a CF API token. Stored Age-encrypted server-side at `/var/servel/secrets/cf_token.age`. Required permissions: `Zone:Zone:Read`, `Zone:Zone Settings:Edit`, `Zone:SSL and Certificates:Edit`, `Zone:DNS:Edit` (last one optional). Use `--stdin` to pipe and avoid shell history capture. |
| `cf zones [--json]` | List zones the token can see. |
| `cf ssl [<mode>] [zone] [--all] [--json]` | Read or write per-zone SSL mode. Modes: `off`, `flexible`, `full`, `strict` (= UI "Full (strict)"). No args = read every zone. |
| `cf ssl snapshot [-o file]` | Capture every zone's SSL mode to JSON (default `/var/servel/cf-ssl-snapshot-<ts>.json`). |
| `cf ssl restore <file> [--dry-run]` | Re-apply a snapshot. |
| `cf cert issue <domain ...> [--wildcard] [--days N] [--skip-root-install]` | Issue a CF Origin CA cert + install via Traefik file provider at `/var/servel/traefik/dynamic/origin-ca/`. `--wildcard <apex>` covers apex + `*.apex` (recommended). Default validity 5475 days (15y). First issue per server also installs the CF Origin CA root into `/usr/local/share/ca-certificates/` + `/etc/docker/certs.d/registry.srvl.app/ca.crt` so Docker push + servel preflight trust Origin CA-signed certs. |
| `cf cert list [--json]` | Inventory installed Origin CA certs (parses PEMs on disk, shows CN/SANs/expiry). |
| `cf cert revoke <safe-name>` | Remove a locally-installed cert + rebuild dynamic config. Use the `safe-name` from `cf cert list`. |
| `cf dns ls [zone] [--json]` | List DNS records for a zone. |
| `cf dns set <name> <type> <value> [--proxied] [--ttl N]` | Upsert a DNS record (PATCH if exists, POST if new). Zone inferred from name. |
| `cf dns rm <name> [--type X]` | Delete one or all record types for a name. |
| `cf status` | Overview: token + verify, zones with SSL mode, installed Origin CA certs (with expiry warnings), CA-root install state. |

**Cert-rate-limit incident replay (2026-05-19 pattern).** After a Let's Encrypt rate-limit (5 certs/week per identifier) or a Traefik cert wipe, recovery is six lines instead of an hours-long firefight:

```bash
servel cf token set <token>                # one-time per server
servel cf ssl snapshot -o /tmp/pre.json    # capture before
servel cf ssl flexible --all               # emergency CF→origin plaintext
servel cf cert issue --wildcard srvl.app   # Origin CA + auto-root install
# (repeat per affected zone)
servel cf ssl restore /tmp/pre.json        # back to strict / full
```

Self-signed bridge certs **do not work** through CF "Full" mode in 2024+ (CF tightened enforcement, docs are stale). Use Flexible mode as the emergency stop, then issue Origin CA wildcards for the permanent fix. Origin CA is CF-trusted at the edge, browser-untrusted (fine — browser sees CF's edge cert).

### Servers / nodes / cluster
| Command | What it does |
|---|---|
| `remote [add|remove|list|use|status|provision|env|...]` | Server registry + provisioning. Subcommands: `dns`, `domain`, `tunnel-domain`, `keys`, `registry`, `gc`, `prune`, `cleanup`, `verify-domain`, `diagnose`, `install-nixpacks`, `migrate-traefik`, `update-traefik`, `fix-middlewares`, `refresh-managers`, `setup-granularban`, `rename`. **`provision` now also installs + starts the `servel-daemon` systemd unit with `--local` and idempotently rewrites the unit on every run.** This is the canonical fix for "UNITS column shows `?` for every deployment" or "daemon crash-loop with `NRestarts > 10`" — both are symptoms of a legacy unit missing `--local` (the daemon tries to dial a non-existent SSH remote, exits 1, systemd respawns it forever, no stats are ever collected). Just re-run `servel remote provision` against the affected remote. |
| `servers [check]` | Multi-server dashboard. |
| `node [ls|ps|specs|capacity|health|add|remove|forget|rejoin|promote|demote|drain|activate|schedule|balance|alias|rename-all|label|prune|swap|events|install|install-events|upgrade]` | Swarm node management. **Never use raw `docker node ...`.** `node add worker --provider hetzner [--type cx42] [--region nbg1]` (BYOC) creates the VM on a connected cloud provider first — ssh-target is omitted; falls into the same join flow, stamps provider metadata for teardown. |
| `cloud [connect|status|disconnect]` | BYOC provider tokens. `cloud connect hetzner` stores the API token age-encrypted client-side (`~/.servel/cloud/`), live-verified before save; `status` shows token validity + `managed-by=servel` VM count. Provider: `hetzner` (cx32/fsn1/ubuntu-24.04 defaults). Token via hidden prompt (preferred) or stdin pipe — neither reaches shell history or argv. `--token` is a
scripting-only escape hatch: it lands in shell history and is visible in `ps`; passing it from an interactive
TTY prints a stderr warning (piped/scripted use stays silent). Never logged. Destroy paths refuse VMs missing the `managed-by=servel` label. |
| `capacity` / `cap` / `forecast` | Capacity forecast + per-node health verdicts + node recommendations. Prints a cluster headline (`cluster healthy` / `cluster busy but within capacity` / `N node(s) strained` / `N node(s) need attention`) immediately after the title. Per-node reason lines follow the Current Capacity table for any non-healthy node. LOAD column is always dim — high loadavg alone never makes a node strained or critical. Unit Capacity table columns: `RESERVED \| ALLOC \| ACTUAL \| MAX \| UTIL` + a one-line legend under the table. Reservation Health top-3 (both over- and under-reserved) rank by biggest offender in unit currency, not raw signed reclaim. Entries display servel identities — `myapp` / `@infra (svc)` / `~system` — never raw docker service names (raw only when the infra can't be resolved), and each class carries a `fix:` hint (over-reserved → daemon auto-trim + `servel stop <app>` when unneeded; phantom load → auto-reserve on next `servel deploy`; over-limit → raise `resources.*` in servel.yaml). |
| `units` | Unit-based capacity overview (declared limits + live estimates — the ALLOC currency). |
| `rebalance` | Auto-redistribute services (memory / tasks strategies; planner simulates live CPU as well as memory). |
| `reconcile` | Discover unlabeled services + missing state. |
| `migrate` | Migrate `/var/servel/` filesystem layout to latest version. |
| `prune` | Clean up Docker build artifacts. |
| `cleanup` | Remove expired environments. |

### Security / access / audit
| Command | What it does |
|---|---|
| `auth [enable|disable|update|status|login|logout|whoami|registry]` | BasicAuth on deployments + CLI identity + Docker registry creds. |
| `access [user|role|scope|permissions|setup|invite|join|leave|token|request|audit|request-hint]` | Multi-user team access + ACL. |
| `ban <ip|name>` / `unban` / `ban [ls|clear|sync]` | Server-wide and per-deployment IP bans (Traefik denyip plugin). |
| `bastion [init|install|uninstall|start|stop|restart|status|sessions|kick|session [list|play|info|commands]]` | SSH bastion with session recording. |
| `audit [list|stats|export|rotate]` | Audit log (forensics-grade who/what/when). Carries `user` (authenticated principal) + `operator`/`operator_email` (self-declared identity from `config set user.*`); destructive commands require the latter. |

### CI / CD / jobs / deps
| Command | What it does |
|---|---|
| `ci [init|setup|pipelines|pipeline-init|run|retry|cancel|status|logs|recent|artifacts|webhook|keys|server|list]` | CI/CD pipelines (GitHub Actions / GitLab CI / built-in). |
| `job [add|ls|rm|pause|resume|run|history|logs\|doctor]` | Cron-style scheduled jobs. App-linked jobs display as `app/name` (e.g. `agentkarma/rescore`); pass `app/name` OR the bare name (bare resolves when unique across apps, else qualify it) to `run`/`rm`/`logs`/`history`/`pause`/`resume`. App-linked jobs pull the app image with registry auth (run on any node) + track redeploys. `job history` shows a REASON column for failures (the only place a pre-container failure cause appears). `job doctor` = end-to-end self-test. |
| `deps [app]` | Show app dependencies + status. |

#### `servel job` — detail

```bash
# Add a job linked to an app (inherits image + env)
servel job add cleanup --schedule "0 3 * * *" --command "npm run cleanup" --app myapp

# Add a standalone job with an explicit image
servel job add sync --schedule "*/30 * * * *" --command "python sync.py" --image python:3.12

# Add-time flags: --timeout 2h, --skip-running (default true), --timezone Europe/Istanbul,
#   --retries 3, --retry-delay 5m, --constraint "node.hostname==worker-2", --env KEY=VALUE,
#   --dry-run (preview systemd units without creating)

# At least one of --app or --image is required; --app rejected if app has no runnable image

# List jobs — app-linked jobs show as "app/name"
servel job ls
servel job ls --app myapp

# Trigger, inspect, manage (pass app/name form for linked jobs)
servel job run agentkarma/rescore
servel job logs agentkarma/rescore [--tail 50]
servel job history agentkarma/rescore [--limit 10]
servel job pause agentkarma/rescore
servel job resume agentkarma/rescore
servel job rm agentkarma/rescore

# Self-test the full subsystem end-to-end
servel job doctor          # exit 0 = healthy, non-zero = a check failed (failing layer named in output)
servel job doctor --keep   # leave probe on failure for debugging
```

**Key behaviors:**
- Jobs run as one-shot Docker Swarm service tasks fired by systemd timers.
- App-linked jobs resolve their image from the app's live Swarm service at exec time — cross-node safe even if the registering node has no local deployment record.
- Fail-loud: unresolvable image or non-zero exit records FAILED in `job ls` STATUS and `job history`; no silent no-ops.
- Weekday ranges (`1-5`) and lists (`1,3,5`) in cron work correctly.
- `job exec` (hidden, systemd-internal) now dispatches to the server when called from a client — no more misleading "job not found".

### Monitoring / analytics / alerts
| Command | What it does |
|---|---|
| `status [enable|disable|list]` | Public status page (gatus-backed). `enable` enumerates every public Traefik route (file-provider + label-routed deployments), renders a gatus config.yaml, deploys `servel add gatus <name>`, seeds the config as a Docker config object mounted at `/config/config.yaml` (node-independent across a multi-node swarm), and writes the Traefik route. `disable` removes infra + route files (scoped to the infra's own gatus backend) + config objects. `list` (alias for bare `servel status`) shows URL, auth on/off, monitored count, and live up/down from the gatus API. Flags: `--domain`, `--auth` (generate basic auth, printed once), `--node`, `--name` (default `status`), `--title`. On-cluster caveat: a full ingress outage takes the page down too — complement with external monitoring. |
| `alerts [enable|disable|setup|add|remove|test|pause|resume|history|config|monitored|status]` | Telegram/Slack/Discord/webhook alerts with pressure detection. |
| `analytics` | Visitor analytics from Traefik logs (`--cluster` for cluster view). |
| `telemetry [status|enable|disable]` | Anonymous telemetry settings. |

### Backups
| Command | What it does |
|---|---|
| `backup [server|@infra]` | Backup archive of server or infra. Takes a SERVER NAME, not a subcommand — there is no `backup ls` (use `restic ls`); an unknown name fails immediately listing configured remotes, with no passphrase prompt. |
| `restore <file>` | Restore a backup archive. |
| `restic [backup|restore|ls|status|config|install|schedule|repos|rclone]` | Restic-based incremental backups (the modern path). |
| `infra backup <name>` / `infra restore <name>` | Per-infra (pg_dump-aware for postgres, supabase). |

### Dev mode / local
| Command | What it does |
|---|---|
| `dev [start|stop|list|logs|prune]` | Remote dev sandbox (bidirectional sync; `--link-env`, `--link-infra`, `--secrets`, `--team`). `--remote <name>` = server selection (alias of `--server`). Container gets RAW internal-host env (resolves on overlay); `localhost:N` tunnel variants printed for laptop tools only. Prod-env/`--secrets` links require confirmation (`--yes` skips; non-interactive without it fails loud); non-interactive + link + public URL forces basicauth (password printed once to stderr). |
| `init` | Initialize a new `servel.yaml`. |
| `set-env-file` | Wire env_file into `servel.yaml`. |
| `run <action>` | Run predefined action in deployed container (defined in `servel.yaml`). |

### Registry / images
| Command | What it does |
|---|---|
| `registry [info|ls|tags|du|rm|retain|migrate|decommission]` | Self-hosted + GHCR + GitLab + custom. |
| `cache [...]` | Build cache management. |
| `ports [list|show|stats|release]` | TCP/UDP port allocation tracking. |
| `tags` / `tag` / `untag` | Tag deployments + infra. |

### Self-management
| Command | What it does |
|---|---|
| `upgrade` | Upgrade local servel binary. Twin-serving by default (stage→smoke→swap→sentinel probation→auto-revert); `--no-twin` for direct swap. |
| `selfcheck` | Offline binary smoke (config parse + state-dir + docker ping). No network. Used by the upgrade flow against the staged candidate. |
| `upgrade-servers` | Bump all configured servers to match client version. **Rolling by default** (one node at a time, health-gated, auto-revert); `--no-rolling` opts out. `--rolling` deprecated (no-op). Gate = binary answers `version --json` AND daemon heartbeat fresh on the NEW build id (daemonless servers fall back to binary-only). On gate failure: remote auto-revert (`sudo servel upgrade --rollback` + daemon restart over SSH), budgeted 3/24h per `server:<name>`; if the server-local sentinel already reverted (old build fresh) it's classified `reverted (server-local sentinel)` with NO second rollback. Remotes joined via `servel access join` (non-empty `role`) are **skipped before probing** — ACL-scoped identities can't swap a server binary; naming one with `--server` errors. A binary too old to have `servel upgrade` is **reinstalled over SSH** (local binary first, servel.dev download as fallback) then upgraded normally — confirmation required, auto-confirmed by `--force`/`--yes`, fails safe non-interactively. First failure aborts the rest (breaker). |
| `check-versions` | Audit server versions for compatibility. |
| `daemon` | Auto-failover daemon controls (server side). Subcommands: `start`, `stop`, `restart`, `status`, `install`, `uninstall`, **`config {list,get,set}`** (added 2026-05-20 — reflection-driven get/set on the daemon Config block in `/var/servel/daemon/daemon-state.json`; sibling of `servel config set` which targets ServerConfig). Most keys take effect on next tick; `*_interval` / `*_cooldown` need `--restart-daemon`. Example: `servel daemon config set routing_traefik_repair_budget=8`. |
| `ai [question]` | Throwaway AI assistant session with full server context. Auto-detects agent (Claude Code → Codex → opencode); `--agent claude\|codex\|opencode`, `--remote <server>`, `--config <path>` (default `~/.servel/ai.yaml`). One-shot: `servel ai "why is myapp down?"`; interactive: bare `servel ai`. Spawns the agent wired to servel's MCP server over SSH (destructive tools enabled, gated by the agent's confirm prompt). |
| `ai install <claude\|codex\|opencode>` | **Persistent** MCP registration — writes servel's MCP server into the agent's own config so any session can manage servers. Idempotent (re-run updates in place); merges are surgical (siblings preserved). `--remote a,b` (one entry per remote: `servel-a`, `servel-b`; default = configured default remote), `--global` (else project-local: `.mcp.json` / `.codex/config.toml` / `opencode.json`), `--read-only`, `--allow-destructive`, `--binary <path>`. **Secure default: read+write registered, destructive (rm/rollback/prune) OMITTED unless `--allow-destructive`.** `servel ai uninstall <agent>` reverses it. |
| `mcp-server --remote <server>` | **Hidden.** Starts servel's MCP server on stdio for any MCP-compatible client (used by `servel ai` + `servel ai install`). Exposes up to 25 tools (14 read / 8 write / 3 destructive) that run `servel` over SSH. **Posture flags: `--read-only` (read tools only), `--allow-destructive` (include rm/rollback/prune). Default = read+write, destructive omitted** — a tool that isn't registered can't be called. Permission tiers (`~/.servel/ai.yaml`): read=auto, write=confirm, destructive=confirm; `allowed_remotes` allowlist + per-tool overrides (`deny`/`confirm`/`auto`). Tool behaviors: read/destructive **annotations** (hints for host UIs); **typed errors** `{error:{code,message,remediation}}` (codes: transient=retry-ok, not_found/denied/needs_confirmation=don't-retry, invalid_input, internal); destructive tools require **out-of-band elicitation** approval (LLM can't self-confirm); `servel_deploy` streams **progress** when given a `progressToken`; `servel_ps`/`servel_infra` accept `limit`+`cursor` **pagination** (returns `total`+`next_cursor`). |

### Diagnostic-only / read-only escape hatches (use freely)
`servel ssh <server>` for interactive shells, `journalctl`, `ip`, `iptables` reads, `dmesg`, package queries, kernel introspection, `top`/`htop`, `df -h /`, raw `docker info` / `docker version` / `docker node ls` read-only commands. **Reads = fine. Writes (`docker service ...`, `docker volume rm`, `docker network rm`) = banned.**

### Decision: "is there a native verb for what I'm about to do?"
1. Scan the categories above. If a row matches → use it.
2. Not sure? `servel <closest-category> --help` lists every subcommand for that domain.
3. Run `servel --help` to see all 80+ root commands.
4. Genuine gap (no native verb)? Tell the user explicitly: "no native servel verb exists for X; falling back to `servel ssh ... -- <command>`". This is the *only* acceptable path to ssh+docker — and it's a signal to file the gap as a servel improvement.

## Decision Tree

```
Task -> What are you trying to do?
|
+- Deploy app -> servel deploy --verbose
|   +- Need database? -> servel add postgres --name db && servel deploy --verbose --link-infra db
|   +- Preview/PR? -> servel deploy --verbose --preview --ttl 24h
|   +- Multi-env? -> servel deploy --verbose --env production
|
+- Add infrastructure -> servel add <type> --name <name>
|   +- Bundle? -> servel add redis,postgres --prefix app
|   +- High-availability? -> servel add postgres --name db --ha
|   +- Link to app? -> servel link db  (from the app's project dir; saved to servel.yaml)
|
+- Debug/inspect -> servel logs <name> -f | servel exec <name> sh
|   +- Infra? -> Use @ prefix: servel logs @mydb -f | servel exec @mydb --service rails sh
|
+- Dev mode -> servel dev
|   +- Team sync? -> servel dev --team
|
+- Find resource -> servel find <name>
|   +- Infra only? -> servel find @<name> --type postgres
|
+- Manage server -> servel remote status | servel ssh <server>
|   +- Capacity? -> servel capacity
|   +- Visitor analytics? -> servel analytics (resolves from .servel/state.json) | --cluster for cluster view
|
+- Block bad IPs -> servel ban <ip> (server-wide) | servel ban <name> <ip> (per-deployment)
|   +- Per-deployment first time? -> servel remote setup-granularban (one-time plugin install)
|   +- Multiple nodes drifted? -> servel ban sync
|
+- Routing issues -> servel traefik status | servel verify dns <domain>
|   +- Debug route? -> servel traefik debug <deployment>
|
+- Run DB migrations -> servel infra sql @<name> ./migrations
|   +- Single file? -> servel infra sql @<name> schema.sql
|   +- Supabase? -> servel infra sql @<name> ./supabase/migrations --service db
|   +- Preview first? -> servel infra sql @<name> ./migrations --dry-run
|   +- ORM migration? -> servel infra run <name> migrate (if action defined)
|
+- Backup/restore -> servel infra backup <name> | servel infra restore <name> <file>
|
+- Drift / silent state issues -> servel infra check --all-nodes (cluster sweep) | servel infra repair @<name> (auto-rsync bind sources, alert on spec/state drift)
|   +- Bind-mount source missing on worker (post-rejoin / OS-reinstall)? -> auto-fixed by daemon on schedule + on node-rejoin events
|   +- Manual `docker service update --mount-add` polluted live spec? -> alert-only; never auto-reverted
|   +- meta.json says running but 0 replicas, or stale target_node_id? -> alert-only; operator decides
|
+- Volumes -> servel volumes | servel volumes inspect <name>
|
+- Where do my images go? -> autodetected from git origin
|   +- ghcr.io/<owner>/<repo>          (GitHub origin)
|   +- registry.gitlab.com/<grp>/<prj> (GitLab origin)
|   +- self-hosted                     (no git remote)
|   +- override: servel.yaml `registry: <url|self-hosted|named>`
|
+- Audit -> servel audit list | servel audit export --format csv -o audit.csv
```

## Project Context (Auto-Detection)

**When inside a project directory with `.servel/state.json`, you don't need to pass the deployment name.** Servel auto-detects the project from local state:

```bash
# Inside a project with .servel/state.json:
servel logs -f              # No name needed — uses project context
servel inspect              # Same
servel env vars             # Same
servel deploy --verbose     # Deploys current project to its known server

# Outside a project (or targeting a different deployment):
servel logs myapp -f        # Explicit name required
```

This applies to: `logs`, `exec`, `inspect`, `env`, `restart`, `stop`, `start`, `scale`, `redeploy`, `rollback`, `rm`, `deploy`, and most service-targeting commands.

### Agent Workflow: Live Introspection Inside a Project

When you (the agent) are working in a repo with `.servel/state.json`, you have direct read access to the live production deployment for that project — no SSH, no credentials, no manual server-name resolution. Use this **before guessing** what's happening in production:

| Goal | Command (run from project root) |
|---|---|
| See what production is doing right now | `servel logs -f` |
| Diagnose a recent crash / 5xx | `servel logs --tail 200` or `servel logs --since 10m` |
| Check build output of the last deploy | `servel logs --build` |
| HTTP request flow (status, latency, client IP) | `servel logs --http --tail 50` (after `servel logs config --access-logs`) |
| What env the container actually sees | `servel env vars` |
| What's encrypted vs plaintext | `servel secrets ls` / `servel env vars --show-source` |
| Service health, replicas, image, command | `servel inspect` |
| Open a shell in the live container | `servel exec sh` |
| Run a one-off in production (migration, REPL, etc.) | `servel exec -- <cmd>` |
| Verify routing + SSL + DNS | `servel verify <name>` (or omit name) |
| Linked infra logs (e.g. postgres slow query log) | `servel logs @<infra-name> -f` |
| Linked infra shell (run psql, redis-cli, etc.) | `servel exec @<infra-name> --service db sh` |
| What domains/routes are live | `servel routes` / `servel domains ls` |

**Heuristic (load-bearing):** if `.servel/state.json` exists, **never speculate about prod behavior — ask servel first**. `servel logs` is the default opening move for any production question; `servel env vars` is the default for any "what config is the container running with" question. Both are cheaper, more accurate, and more current than reading the codebase + guessing. Treat the presence of `.servel/` as a standing invitation to run these directly — no need to confirm with the user, no need to pass a deployment name.

**Multi-environment projects:** if `.servel/state.staging.json` also exists, target it with `--env staging` on any of the above (e.g. `servel logs -f --env staging`).

**No `.servel/` directory?** The project hasn't been deployed yet — these commands won't auto-resolve. Either deploy first (`servel deploy --verbose`) or pass an explicit name (`servel logs myapp -f`).

## Service Addressing (Symbol Prefixes)

Most commands that target a service (exec, logs, inspect, stats, restart, stop, start, scale, env, remove) support symbol prefixes:

| Prefix | Type | Resolves to | Example |
|--------|------|-------------|---------|
| `name` | Deployment | `servel-name-*` | `servel logs myapp -f` |
| `@name` | Infrastructure | `servel-infra-name-*` | `servel logs @mydb -f` |
| `~name` | System | `servel-system-name` | `servel logs ~traefik` |

For multi-service infrastructure (chatwoot, supabase, etc.), use `--service`:
```bash
servel exec @chatwoot --service rails sh
servel logs @supabase --service postgres -f
servel restart @chatwoot --service sidekiq
```

**Off-taxonomy service names** (e.g. `servel-daemon-X`, `servel-foo`, hand-rolled docker services): the symbol prefixes won't resolve them because `DiscoverServices` matches `servel-{<name>,infra-<name>,system-<name>}-*` only. Do **not** reach for `servel ssh H -- "docker service ..."` as the workaround — see the banned escape hatch above. Correct moves:
- If the service is a system daemon you control → rename it to `servel-system-<name>` so `~<name>` works.
- If it's an app → redeploy via `servel deploy` so it lands at `servel-<name>-*` and becomes addressable.
- Diagnostic-only? `servel ps --remote <h> --all` to list, then pick the right addressable equivalent. If none exists, you've found a servel gap — surface it.

## Server Targeting (`--remote`)

The `--remote` flag is a **global flag** available on ALL commands. It targets a specific server instead of the default.

```bash
servel ps --remote KN              # List deployments on KN server
servel logs myapp --remote KN      # View logs on specific server
servel infra --remote tominance    # List infra on tominance
servel exec @mydb sh --remote KN   # Shell into infra on specific server
servel deploy --remote staging-srv # Deploy to non-default server
```

**When to use `--remote`:**
- The project has no `.servel/state.json` (not yet deployed, so no default server context)
- Targeting a server different from the default (`servel remote use <name>`)
- Running cross-server commands like `servel find` or `servel ps`
- Managing infrastructure on a specific server

**When NOT needed:**
- Inside a project directory with `.servel/state.json` — servel auto-detects the server
- After running `servel remote use <name>` to set a default
- Commands that already specify the server (e.g., `servel ssh KN`)

**Tip:** Use `servel remote list` to see available remotes, `servel remote use <name>` to change default.

### ⚠️ Remotes that name a swarm WORKER node

`servel remote list` shows a CLUSTER column like `KN (worker)` when a remote is a node inside another remote's swarm. Such a remote is a **machine**, not a control endpoint — every connection to it is redirected to one of the swarm's managers, because only a manager answers Docker's control API.

Consequences you must plan around:

| Intent | Correct command | Why not `-r <worker>` |
|--------|-----------------|------------------------|
| Per-node disk / CPU | `servel node ls -r <swarm>` (shows DISK + AVAIL per node) | `servel df -r <worker>` refuses — it would report the **manager's** filesystem |
| Node disk detail | `servel df -r <swarm> --nodes` | same |
| Reclaim images on a worker | `servel node prune <node>` | `servel prune -r <worker>` refuses — it would prune the **manager** |
| Shell / raw command on the node | `servel ssh <worker-remote> -c '<cmd>'` | works: hops `ssh -J <manager>` to the node and prints the host it landed on |

`servel ssh` is the only host-local verb that is safe against a worker remote — it resolves the node's real address from the swarm (repairing a stale `host` in the config if the connect-path self-heal had overwritten it with a manager's IP) and jumps through the manager. Everything else that reports host-local state refuses with a message naming the right command.

Historic trap (fixed 2026-08-08): `servel ssh norastech -c hostname` printed `Running on norastech` and returned `KN-MANAGER`; `servel df -r norastech` printed `Server: norastech` above the manager's disk. If you see a servel build old enough to do this, verify with `hostname` before trusting any per-node reading.

## Non-Interactive / Agent Use (never hang on a prompt)

When driving servel from an agent or script, a confirmation prompt with no TTY would block. Two global switches prevent that:

- **`--yes`** — global flag on ALL commands; assumes yes to every confirmation prompt. One switch instead of remembering per-command `--force` vs `--yes`. (Long-only — no `-y` shorthand, to avoid collisions.) Example: `servel rm myapp --yes`, `servel prune --yes`.
- **`SERVEL_NONINTERACTIVE=1`** (env) — asserts "I cannot answer prompts." Any command that would prompt **fails fast with a clear `use --flag` error instead of blocking** (mirrors `CI=true`). Set it once in the agent's environment.

Without `--yes`/consent in a non-interactive context, destructive commands (`rm`, `prune`, `upgrade`) abort rather than proceed — a closed stdin is never read as "yes". Pair `SERVEL_NONINTERACTIVE=1` with explicit `--yes` (or `--force`) on the commands you intend to run unattended.

**Zero-prompt onboarding:** `servel remote add <name> root@<ip> --yes` pairs AND provisions a fresh server with no prompts at all — the Let's Encrypt email and primary domain are optional (`--email`/`--domain` flags on `remote add`, or set later via `servel server provision --email/--domain`; SSL works without an email). The install one-liner forwards it: `curl -fsSL https://servel.dev/install.sh | sh -s -- --remote root@<ip> --yes` (the script itself also accepts `-y`).

## Machine-Readable Output (`--json`)

`--json` is a **global flag** (bound on root, inherited by all commands). When set, the command emits a single structured JSON object/array on **stdout** and routes all human output (progress, spinners, banners) to **stderr** — so an agent can parse stdout deterministically. Color is auto-disabled. Errors still emit a JSON object with a populated `error` field AND a non-zero exit code (read both).

Commands with a typed `--json` envelope (prefer these when parsing):
- `ps --json` → `{deployments:[{...,replicas:"1/2"}],total_count,protected_count,dev_sessions?,unknown_count?}`; `replicas` is a string and uses `?/2` when task observation is unavailable. Also: `infra --json`, `inspect --json`, `find --json`
- `deploy --json` → `{deployment_id, name, url, status, elapsed_seconds, error}`
- `rm --json` → `{name, removed, error}`
- `add --json` → `{name, type, category, status, connection_env, ...}`
- `doctor --json` → `{checks:[{name,status,message}], healthy, ...}`
- `remote status --json` → full `ServerMetrics` (server_info, resources.cpu/memory/disk, services, network, …)
- `logs --json` → `{lines:[...]}` bounded, or NDJSON (`{app,line}` per line) with `--follow`

`add` over SSH self-corrects a lost exit-status: a long remote deploy (e.g. supabase, 13 services) can finish successfully yet end the SSH session without an exit-status. `servel add` detects this specific case, re-queries `infra status <name>`, and exits **0** with the verified envelope (`status: running` or `degraded`) instead of a spurious non-zero — so a deployed-but-degraded stack is distinguishable from a hard failure by `status`, not just exit code. A genuinely failed create indexes nothing, so the re-query finds nothing and the non-zero exit is preserved.

The MCP tools already return JSON internally and carry annotations (read-only / destructive hints) so MCP hosts can gate destructive calls in their own UI.

## Quick Reference

### Deploy

**IMPORTANT: Always use `--verbose` flag when deploying.** It shows full build output, making it much easier to diagnose issues and understand what's happening. Without it, build output is summarized and critical context is lost.

```bash
servel deploy --verbose               # Auto-detect & deploy (ALWAYS use --verbose)
servel deploy --verbose --preview --ttl 24h     # Preview with cleanup
servel deploy --verbose --link-infra db,redis   # Link infrastructure (internal DNS by default when same-swarm)
servel deploy --verbose --link-infra db --public   # Force public-domain hostnames (e.g. for cross-swarm)
servel deploy --verbose --link-infra db --internal # Hard-require same-swarm internal DNS
servel deploy --verbose --dry-run               # Show plan only
servel deploy --verbose --no-registry           # Skip registry (single-node)
servel deploy --verbose --env staging           # Multi-environment
servel deploy --verbose --rebuild               # Force rebuild, skip cache
servel deploy --verbose --new                   # Force new deployment with unique subdomain
servel deploy --verbose --supersede             # Cancel prior in-flight build for this project, then deploy (skip queue wait)
servel deploy --verbose --dashboard             # Real-time TUI dashboard during deploy
servel deploy --verbose --save                  # Persist flags to servel.yaml
servel deploy --exclude target --exclude web/.next  # Extra excludes (merged with built-ins + .servelignore)
servel deploy --memory 1g --cpu 0.5   # Resource limits
servel deploy --quiet                 # Minimal output (only final result)
# No domain configured at all? The deploy still gets a live HTTPS URL: servel
# auto-claims a magic subdomain (https://<app>-<hash6>.<magic-domain>) and wires
# the route + SSL before service creation. Redeploys REUSE the same grant
# (persisted in spec as magic_subdomain). Claim failure never fails the deploy —
# it prints "subdomain pending — retry: servel domains claim <name>"; the daemon
# retries in the background. Custom domain later via `servel domains add`.
servel ps                             # List deployments; task-derived replicas show ?/N when unavailable, never fabricated 0/N
servel ps --all-servers               # List across all servers. Reads deployments/*/meta.json directly, so records whose
                                      # Docker service is gone show STATUS=orphaned (JSON: "orphaned":true) instead of a stale
                                      # "running". Only running/degraded records are flagged; stopped/failed/superseded are history.
servel ps --tree                      # Tree view with dependencies
servel logs <name> -f                 # Follow logs
servel watch <name>                   # Watch deploy progress — high-level phases/steps TUI (reads progress.json)
                                      # Exits 0 on converged-running, NON-ZERO on failed deploy (safe for && chains)
servel watch <name> --follow          # Wait for deploy to start, then watch
servel attach [name|id-prefix]        # Stream raw build.log of in-progress build (BuildKit, nixpacks, bun output with full ANSI color via PTY)
                                      # No-arg: current dir's project. Ctrl+C detaches; build keeps running. Aliases: at
                                      # Use watch for "where am I in the pipeline?", attach for "why is this failing?"
servel rm <name>                      # Remove. If no Docker service exists for <name>, rm falls back to the stored records
                                      # and purges the orphaned generation dirs (+ their dangling Traefik routes) instead of
                                      # failing "not found". Refuses while any service still backs the name, incl. compose
                                      # sub-services (servel-<app>-<svc>-<env>). Matches app name OR deployment-ID prefix, and
                                      # purges EVERY matching generation. --dry-run lists them without touching anything.
servel rollback <name>                # Rollback version
servel promote <src> <tgt>            # Promote deployment (env + domains). Target KEEPS its own domains (merge,
                                      # not replace); plan lists kept/lost. Secrets hydrated from canonical .env.age
                                      # (never stale spec.json). Emits critical audit event.
servel promote src tgt --swap         # Bidirectional domain swap
servel promote src tgt --dry-run      # Preview promotion plan
servel promote src tgt --merge-env    # Merge env vars (vs replace). REPLACE (default) removes target-only
                                      # keys INCLUDING target-only secrets (--env-rm on the live service).
                                      # --dry-run lists every removal; secret keys shown by name + `(secret)`,
                                      # values never printed. Use --merge-env to keep target-only keys.
servel promote src tgt --rebuild      # Rebuild after (NEXT_PUBLIC_*)
servel promote src tgt --cleanup-source  # Remove source after
servel scale <name> 3                 # Scale replicas
servel scale <name> 0                 # Scale to 0 (same as stop)
servel restart <name>                 # Restart deployment
servel stop <name>                    # Stop deployment (scales to 0)
servel start <name>                   # Start stopped deployment
servel stop @mydb                     # Stop infra (warns if deployments are linked to it)
servel stop @mydb --with-linked       # Stop infra AND its linked deployments (apps first, then infra; records pre-stop replica counts)
servel start @mydb --with-linked      # Start infra first, then linked deployments at their recorded pre-stop replica
                                      # counts (falls back to 1 when no record; apps retry until infra ready)
servel rename <old> <new>             # Rename deployment
servel exec <name> sh                 # Shell into container
servel exec <name> -- cmd args        # Run command in container
servel inspect <name>                 # Detailed deployment info (secrets redacted as *** by default; --show-secrets to reveal)
servel history <name>                 # Deployment history
servel versions <name>                # Available versions
servel find myapp                    # Find across all servers
servel find @mydb                    # Find infrastructure only
servel find --type postgres          # Filter by infra type
```

**Build args injected automatically:**
- `SERVEL_GIT_COMMIT` -- Git commit SHA (available during build)
- `SERVEL_GIT_BRANCH` -- Git branch name (available during build)
- `SERVEL_DEPLOY_TIME` -- Deploy timestamp in RFC3339 UTC (always set)

Use `ARG SERVEL_GIT_COMMIT` + `ENV SERVEL_GIT_COMMIT=$SERVEL_GIT_COMMIT` in Dockerfile to persist at runtime.

**Detection priority:** `servel.yaml` -> `docker-compose.yml` -> `Dockerfile` -> preset -> Nixpacks

**Smart mode (default):** Detects what changed -> config-only (~8s), static-only (~10s), or full build.

**Key flags:**
- `--name, -n` -- Deployment name. **Multi-service-from-one-repo: pass `--name <svc>` on EVERY deploy.** Deploy state is keyed per project dir, not per service; a repo that swaps `servel.yaml` to deploy several differently-named services (e.g. `cp deploy/<svc>.yaml servel.yaml; servel deploy`) shares one cached identity, and `--name` pins each service so they don't collide. Without `--name`, a `servel.yaml` `name:` that differs from the dir's last deployment is treated as a **separate deployment** (deploys under the yaml name, leaves the previous service running — it is NOT renamed/removed; that was the 2026-06-14 clobber, now non-destructive). Real renames: `servel rename <old> <new>`.
- `--domain, -d` -- Domain for routing
- `--preview` -- Preview environment
- `--ttl` -- Preview lifetime (1h, 6h, 1d, 7d, 2w)
- `--link-infra` -- Link infrastructure (comma-separated). Defaults to internal Docker DNS (overlay alias) when same-swarm; the app auto-attaches to `servel-infra-{name}-network` so injected hosts resolve via Docker DNS instead of going through Traefik. Internal-only ports (e.g. postgres 5432) only work this way. Attachment is **reconciled on every deploy** (additive — operator-added networks survive), so adding a new link to an existing app and re-running `servel deploy` is enough; pre-fix, late-added links were silently dropped on the update path and showed up as `ENOTFOUND kong` / `ENOTFOUND <service>` at runtime. Connection env (DATABASE_URL etc.) is injected server-side via a trusted out-of-band channel (not process-env, so it bypasses the public-prefix env allowlist and never appears in `ps`), and link resolution is now **deterministic regardless of which manager smart-selection picks** — it falls back to a swarm-global lookup when the infra's node-local meta isn't on the SSH'd manager (previously failed `access:internal but not on deploy target swarm`).
  - **Connection env is auto-injected — no `servel env set`/`secrets set` needed.** `--link-infra <name>` (or the `infra:` block in servel.yaml) ships the resolved connection vars (`DATABASE_URL`, `<PREFIX>_HOST/PORT/USER/PASSWORD/DB`, `REDIS_URL`, `SUPABASE_*`, etc.) straight into the running container. They are delivered over a private in-band channel (a 0600 temp file, like migrated secrets), so DB passwords never appear in `ps`/argv, and they reach the container regardless of variable name (trusted server-resolved values bypass the public-prefix build-arg allowlist that gates raw shell env). Legacy bare-named infra (`servel-infra-{name}` with no `-{type}` suffix, deployed before the canonical convention) are discovered automatically and resolve to the correct host. Fixed 2026-05-29 (prior bug: network attached but `DATABASE_URL` was dropped by the allowlist → app reached 1/1 then crashed on stale env).
- **Secret-store reconciliation** — `servel deploy` is additive: it writes declared secrets but never removes orphans. Stale values silently shadow infra-injected values via the `if !exists` rule. Two ways to fix drift:
  - `servel secrets reconcile <app>` — interactive cleanup (`--dry-run` to preview, `--yes` to skip prompts).
  - `prune_secrets: true` in servel.yaml — auto-reconcile on every `servel deploy`. Orphan = in encrypted store, not in `secrets:` block, not injected by an `infra:` link.
- `--skip-migrations` -- Skip the `migrations:` schema gate for this deploy (see **Deploy-time migrations** below).
- `--public` -- Force linked infra to use public-domain hostnames (overrides per-link `access:internal` in servel.yaml). Mutually exclusive with `--internal`.
- `--internal` -- Force internal Docker DNS for every link; **errors** if any linked infra isn't on the deploy target swarm.
- `--no-registry` -- Skip registry push
- `--rebuild` -- Force rebuild
- `--no-smart` -- Disable smart detection
- `--verbose` -- Show full build output **(always recommended)**
- `--quiet, -q` -- Minimal output (only final result)
- `--dashboard` -- Real-time TUI dashboard
- `--env` -- Target environment
- `--build-on <node>` -- Build on specific node
- `--local-build` -- Build locally, push to registry
- `--force-low-ram` -- Proceed with a remote build even when no build node has the framework's recommended RAM. By default servel now FAST-FAILS before a doomed under-RAM build (e.g. Next.js 16 + Turbopack needs ~4GB) and tells you to use `--local-build`; pass this to override and try anyway.
- `--new` -- Force new deployment with unique subdomain
- `--converge-timeout <duration>` -- Convergence wait time (default: 5m)
- `--force-server` -- Suppress server mismatch warnings
- `--author <name>` -- Override deployment author
- `--save` -- Persist deploy flags to servel.yaml
- `--skip-scan` -- Skip vulnerability scanning
- `--scan-block <severity>` -- Block on severity (critical, high, medium, low)
- `--include <pattern>` -- Override exclusions (e.g. `--include .next` for pre-built)
- `--exclude <pattern>` -- Extra exclusion patterns (repeatable; merged with built-ins + `.servelignore`)

### Deploy-time migrations (schema gate)

A `migrations:` block in `servel.yaml` makes `servel deploy` run tracked SQL migrations against the linked database BEFORE the new image builds:

```yaml
migrations:
  path: ./migrations   # required — relative dir of .sql files
  infra: mydb           # optional — defaults to the sole linked DB-capable infra
  service: db            # optional passthrough (e.g. Supabase `db` service)
  database: myapp        # optional passthrough
  preview: false          # default false — preview deploys skip migrations
```

- Uses the same tracking engine as `servel infra sql <dir>` — already-applied files skip, only new files run.
- **Migration failure ABORTS the deploy** — no image build, no rollout, previous version keeps running.
- **Preview deploys skip migrations by default** (branch/PR deploys must not mutate shared/prod DB); set `preview: true` to opt in.
- Target resolution: explicit `infra:` wins; otherwise the sole linked DB-capable infra. Zero or 2+ candidates is a hard error naming them.
- v1 is same-server only — a linked infra on another node/swarm errors with a `--skip-migrations` escape hatch (run manually via `servel infra sql` instead).
- No `migrations:` block but a linked DB-capable infra + a conventional `migrations/`/`db/migrations/`/`supabase/migrations/` dir exists locally → prints ONE hint line suggesting the block. Never auto-runs without it.
- Adopting on a project with an existing schema: `servel infra sql @mydb ./migrations --baseline` first, so deploy doesn't try to re-run migrations the DB already has.
- **Destructive-DDL warning (detection-only, never blocks)**: migrations apply BEFORE the image build, so a
  build failure leaves the OLD image against the NEW schema. Deploy warns when the SQL matches `DROP COLUMN`,
  `DROP TABLE`, `DROP CONSTRAINT`, `RENAME COLUMN`, `RENAME TO`, `SET NOT NULL`, `ALTER COLUMN … TYPE`, or
  `ADD CONSTRAINT … NOT NULL`. Advice: expand-contract — ship additive changes with the code, drop old columns
  a deploy later. Lightweight regex, not a SQL parser: false positives possible. **Known nag**: it scans ALL
  `.sql` files, not just pending ones, so one historical destructive migration re-warns on every deploy.

### Package Selection (what gets shipped)

Build context exclusion layers (later overrides earlier):
1. Built-in `DeployExclusions` (`.git`, `node_modules`, `.next`, `dist`, `build`, `vendor`, `.env*`, `*.log`, `coverage`...)
2. `.gitignore` (honored automatically in git repos)
3. **`.servelignore`** -- gitignore-style file at project root; always loaded. Use for git-tracked-but-don't-ship paths (e.g. `target/` build artifacts):
   ```
   target
   web/.next
   *.tsbuildinfo
   ```
4. `servel.yaml: deploy.exclude_patterns: [...]` -- explicit, in-config
5. `--exclude <pattern>` -- one-shot CLI override
6. `--include <pattern>` / `deploy.include` -- un-exclude (for `--skip-build` shipping `.next/`, `dist/`, etc.)

### Deploy Aliases

Define deployment presets in `servel.yaml`:
```yaml
deploy:
  aliases:
    preview:
      ttl: "0"
      domain: "{branch}.preview.myapp.com"
      no_index: true
    quick:
      fast: true
      local: true
    staging:
      env: staging
      domain: "staging.myapp.com"
```
Usage: `servel deploy preview`, `servel deploy quick`
- If a directory exists with that name -> deploys directory
- Otherwise -> applies alias settings

### Infrastructure (45+ types)

| Category | Types |
|----------|-------|
| Database | postgres, mysql, mongodb, clickhouse, redis, libsql + HA variants (postgres-ha, mysql-ha, mongodb-ha, redis-ha) |
| Queue | rabbitmq |
| Search | meilisearch, typesense |
| Platform | supabase, supabase-ha, chatwoot, typebot, convex, affine, forgejo, clawdbot, maily, surfsense |
| Analytics | plausible, umami, openreplay, highlight |
| Monitoring | prometheus, grafana, loki, promtail, uptimekuma, gatus, peekaping |
| Realtime | livekit, livekit-egress, hocuspocus, y-sweet |
| Storage | minio |
| Email | posteio |
| CI | woodpecker, woodpecker-agent |
| Blockchain | bitcoin, ipfs, lnd |

**Naming rules** (`<name>` must be a DNS label):
- Lowercase `a-z`, digits `0-9`, hyphens — must start/end alphanumeric, max 63 chars. NO dots, underscores, uppercase.
- For domains use `--domain example.com`, NEVER `--name example.com`.
- Generate DNS-safe names automatically: `treachery-ai-production`, not `treachery.ai`.

```bash
servel add postgres --name db         # Create
servel add redis,postgres --prefix app # Bundle multiple
servel add postgres --name db --ha    # High-availability
servel add supabase --name supa       # Full platform stack
servel add chatwoot --var Domain=chat.example.com  # Auto-init on first deploy
# --var uses TEMPLATE variable names (Go-identifier rule), NOT POSIX env vars.
# Most templates use TitleCase (Password, JwtSecret, AdminEmail); some use UPPER_SNAKE.
# Find canonical names: `servel infra vars <type>` or `servel add <type> --advanced`.
servel infra status                   # Health check all (distinguishes "warming" from "failed" — see below)
servel infra vars db                  # View env vars
servel infra update db --memory 2g    # Update config (memory, cpu, domain, node, env)
                                      # — multi-service stacks AND single-replica stateful infra get a blast-radius
                                      #   preview + confirmation prompt before any restart. Skip with --yes.
                                      # — stateful single-replica services use the FULL safety set automatically:
                                      #   stop-first + 120s grace + dnsrr + pause-on-failure + 60s monitor.
                                      #   Closes the agentkarma-db checkpoint-PANIC class (2026-05-09).
                                      # — multi-service rolls preflight free RAM. Operator sees concrete
                                      #   "KN-MANAGER: needs 2048MB, 1820MB free — INSUFFICIENT" if the wave
                                      #   wouldn't fit. Diff-aware skip means per-service env changes don't
                                      #   roll the whole stack anymore.
servel infra update db --version v2       # Repair a record whose stored template version is "unknown".
                                      #   Symptom: EVERY template-dependent op (infra rotate above all) fails with
                                      #   "template <type> has an unresolved version" while `servel infra` still
                                      #   shows status: running — the listing gives no hint. The old error blamed
                                      #   "client/server version skew" and said to run `servel upgrade`; that was
                                      #   WRONG (reproduces with identical client+server builds) and is now fixed.
                                      #   servel does NOT auto-resolve to latest: supabase v2 has 10 rotatable
                                      #   credentials vs v1's 5, so guessing would rotate SECRET_KEY_BASE /
                                      #   VAULT_ENC_KEY / PG_META_CRYPTO_KEY / LOGFLARE_* on a stack lacking them.
                                      #   Value is validated against the type's declared versions AND must load.
servel infra customize db --service db --memory 4GB
servel infra customize mysupabase --service meta --health-cmd "bash -c 'exec 3<>/dev/tcp/127.0.0.1/8080'"  # override broken template healthcheck probe (live-applies + survives recreate); 'none' disables; --clear-health reverts
                                      # — per-service live apply: rolls ONLY db, not the other 12 services.
                                      #   Override persisted in spec; survives recreate.
servel infra upgrade db --image postgres:16  # Safely upgrade: auto-backup → persist pre-upgrade image set → apply → health-gate (convergence + template readiness) → AUTO-ROLLBACK to prior image set + alert on gate failure (single-service AND stack paths). --skip-health bypasses gate AND rollback (operator intent). Stateful stays stop-first.
servel infra upgrade supa --service auth --image supabase/gotrue:v2.186.0  # Upgrade specific service
servel infra domains add db --domain db.example.com  # Add domain alias
servel infra domains remove db --domain db.example.com
servel infra labels db --add key=val  # View/modify Docker labels
servel infra run db                  # List available actions
servel infra run db psql             # Run action (e.g. interactive shell)
servel infra run mysupabase deploy-functions ./supabase/functions  # Upload files + run
servel infra run mysupabase add-auth-provider --var Provider=google --var ClientID=123  # Enable OAuth; omit ClientSecret from --var to get a hidden prompt. Any var/env name containing password|secret|token|key|credential is auto-detected sensitive across ALL infra types — hidden prompt + encrypted at rest, no per-template annotation needed.
servel infra run db schema --dry-run # Preview action
servel infra run-hooks db             # Execute lifecycle hooks
servel infra run-hooks db --init      # Run post-init hooks
servel infra archives                 # Manage archived credentials
servel logs @db -f                    # Follow infra logs (@ prefix)
servel logs @chatwoot --service rails -f  # Multi-service infra logs
servel infra backup db                # One-shot backup. Volume archives STREAM from the node (tar to stdout over
                                      #   SSH) — the source node needs ZERO free space regardless of volume size.
                                      #   Space is preflighted on the DESTINATION; failure names free vs. required.
                                      #   Send elsewhere with --output when the default path is tight.
                                      #   Leftover `_servel_backup.tar.gz` inside a volume = pre-fix servel staged
                                      #   there; servel warns but NEVER deletes it (may still be streaming).
servel infra backup db --schedule "0 3 * * *"  # Schedule daily backup (alias for restic schedule add)
                                      # — backup-able types (postgres/mysql/mongodb/redis/supabase + HA) get a daily
                                      #   schedule INSTALLED AUTOMATICALLY at `servel add` time. Override with
                                      #   --backup-schedule "..." or skip with --no-backup-schedule.
servel infra restore db backup.sql.gz # Restore
servel infra rotate db                # Rotate credentials
servel infra restart db --force       # Force restart
servel infra start db                 # Start
servel infra start db --with-linked   # Start infra first, then linked deployments at recorded pre-stop replica counts (else 1)
servel infra stop db                  # Stop (scales to 0; warns if deployments are linked)
servel infra stop db --with-linked    # Stop linked deployments first, then the infra (records pre-stop replica counts)
servel infra stop db --service vector # Stop specific sub-service (multi-container infra)
servel infra rename old new           # Rename
servel infra rm db                    # Remove
servel infra check                    # Orphaned constraints, port conflicts, stuck services, state mismatches, overlay reachability
servel infra check mydb               # SAME checks + the FULL resilience scan, scoped to this infra
servel infra check --all-nodes        # Cluster-wide resilience scan across every configured remote (skips swarm workers)
# The resilience layer runs ONLY with a [name] or --all-nodes — a bare `infra check` never runs it.
# When it does run, EVERY class runs; the name narrows SCOPE, it does not select classes:
#   A.1 bind-source missing · C.1 unmanaged mounts · F.1 state-file drift
#   B.1 Traefik ACME mount · B.2 CF origin CA root · B.3 stale node.id pin
#   B.4 undersized secret · B.5 missing servel.managed label · B.6 stateful safety floors
#   G.1 Postgres near max_connections · G.2 ClickHouse system.* log bloat
# All detect-and-alert; none mutate. Full descriptions: `servel infra check --help`.
servel infra repair @mydb             # Auto-rsync missing bind sources; coalesce duplicate node pins (`multiple_node_pins`); alert on spec/state drift (resilience layer)
servel infra repair @mydb --dry-run   # Preview what the resilience layer would auto-repair
servel infra sql @mydb schema.sql     # Run SQL file against database (raw; single files aren't tracked by default)
servel infra sql @mydb ./migrations   # Run all .sql files in directory, alphabetical order, TRACKED BY DEFAULT (skips already-applied)
servel infra sql @mydb "SELECT 1"     # Run inline SQL
servel infra sql @mydb schema.sql --dry-run  # Preview without executing
servel infra sql @supabase migration.sql --service db  # Supabase (targets db service)
servel infra sql @mydb ./migrations --no-track   # Directory raw re-run-all (opt out of default tracking)
servel infra sql @mydb schema.sql --track        # Track a single file explicitly
servel infra sql @mydb ./migrations --status     # Show migration status
servel infra sql @mydb ./migrations --force      # Re-apply changed migrations
servel infra sql @mydb ./migrations --baseline   # Adopt tracking on a DB whose schema already exists (records files without executing them)
servel link db                        # Link (from app's project dir) -> saved to servel.yaml, injects DATABASE_URL on next deploy
servel unlink db                      # Unlink (persisted to servel.yaml)
servel deps myapp                     # Show dependencies
servel connect db                     # Quick connect to infra
servel connect mysupabase --open      # Open dashboard in browser, auto-logged-in (basic-auth dashboards)
```

**Database Migrations:** Use `servel infra sql` — this is the canonical way to run SQL migrations against any database infrastructure. Supports PostgreSQL, MySQL, MariaDB, CockroachDB, and Supabase. When given a directory, runs all `.sql` files in alphabetical order (prefix with `001_`, `002_`, etc.) and **tracking is ON by default** — already-applied files are skipped, a changed already-applied file errors (use `--force` to re-apply intentionally). Pass `--no-track` for the old raw re-run-all behavior. Single files/inline SQL stay raw unless you pass `--track`. Adopting tracking on a database whose schema already exists: `servel infra sql @mydb ./migrations --baseline` records every file as applied WITHOUT executing it. For Supabase, uses `supabase_admin` superuser on the `-db` service. For ORM migrations (Prisma, Drizzle), use `servel infra run <name> migrate` if a custom action is defined, or run via `servel exec`.

**Migrations on deploy (schema gate):** a `migrations:` block in `servel.yaml` makes `servel deploy` apply tracked migrations to the linked DB before the new image builds — failure aborts the deploy. See "Deploy-time migrations" below.

**Linking injects:** DATABASE_URL, REDIS_URL, MONGODB_URI, etc. based on infrastructure type.

**Lifecycle hooks:** Some templates auto-run setup commands on first deploy (e.g., Chatwoot runs `db:chatwoot_prepare`).

**Node pinning:**
- `--node hostname` -- By hostname
- `--alias db-node` -- By alias
- `--label storage=ssd` -- By node label

### Server Management (remote)

`servel server` is aliased to `servel remote`. Both work interchangeably.

**SSH host keys are TOFU (trust-on-first-use, OpenSSH `accept-new`)**: the first connection pins the server's key in `known_hosts` and prints its fingerprint once; any later mismatch is a **hard failure** with MITM guidance — never a prompt, never silently re-pinned. Legit rebuild/re-key: `ssh-keygen -R <host>` (or `"[<host>]:<port>"` for non-22) then retry. `--accept-new-host-keys` on `remote provision` is now a deprecated no-op (TOFU is the default).

```bash
servel ssh <server>                   # SSH into server
servel remote status                  # Cluster health (CPU, memory, disk)
servel remote add <name> user@host    # Add server (worker IP / cluster DNS auto-redirects to a swarm manager)
servel remote add prod root@1.2.3.4 --yes   # Zero-prompt add + provision (CI/agents) — no email/domain gate
servel remote add prod root@1.2.3.4 --email you@x.com --domain x.com  # Optional at add time: LE contact + primary domain
servel cloud connect hetzner          # BYOC: hidden prompt (preferred; age-encrypted, live-verified before save)
echo "$TOKEN" | servel cloud connect hetzner   # stdin pipe — CI-friendly, token never on argv
servel cloud connect hetzner --token "$TOKEN"  # scripting only — leaks to shell history
servel node add worker --provider hetzner  # BYOC: create VM on the provider + provision + swarm-join, one command
servel remote list                    # List servers
servel remote use <name>              # Switch default server
servel remote remove <name>           # Remove server
servel remote provision               # Automated setup — never prompts; email/domain are OPTIONAL flags
servel remote provision --email you@x.com  # Set LE contact any time (cert-expiry notices; SSL works WITHOUT it —
                                      # empty email = contact-less ACME registration, valid per RFC 8555)
servel remote provision --repair      # Repair corrupted keys/services
servel remote domain set example.com  # Set primary domain
servel remote keys add <name> --key-file pubkey.pub # Add deploy key
servel capacity                       # Capacity forecast + per-node health verdicts + recommendations + Reservation Health + Stateful Concentration + Underused Services. Cluster headline immediately under title (healthy/busy/strained/critical). Per-node reason lines for non-healthy nodes. LOAD column dim — never alarm-colored.
servel capacity --json                # JSON output (.cluster_status, .cluster_status_reason, per-node .verdict{level,reason} + .steal_pct + .system_pct, .rightsize, .stateful_moves, .underuse); unit_summary carries reserved_units/reserved_pct (canonical) + deprecated used_units/used_pct aliases + alloc_units/alloc_pct (0 if the units fetch soft-failed). .rightsize top entries carry an additive `display` field (servel identity: name / "@infra (svc)" / ~system) alongside the unchanged raw `Service` key
# Unit vocabulary (cap table + units --json): RESERVED/reserved_units = scheduler view (Docker reservations);
# ALLOC/alloc_units = declared limits + live estimates; ACTUAL/actual_units = observed usage. Gap between RESERVED
# and ACTUAL = phantom load. Deprecated used_units/used_pct are aliases — prefer reserved_units/reserved_pct on cap,
# alloc_units/alloc_pct on units. `servel infra`/`servel units` footers contextualize allocation inline with
# "· live ~Xu (Y%)" from the daemon snapshot (<=10min fresh; multi-node needs a daemon that stores per-node
# HasLive/Live*Pct — older daemons fall back to the pointer-only footer). Allocation is NEVER red anywhere
# (amber past 100%): >100% ALLOC = by-design Swarm
# oversubscription, not live exhaustion — red is reserved for live usage signals. The post-deploy teaser shows LIVE
# state only (capacity line + this deploy's dim "~+Nu" footprint); it has NO alloc line — allocation lives in cap/units.
servel df                             # Disk usage
servel df --volumes                   # Volume usage by category
servel df --nodes                     # Per-node usage
servel doctor                         # Diagnose issues
servel doctor --remote KN             # Remote server diagnostics — checks include B-class detectors (ACME mount path drift, CF Origin CA root install, stale swarm node.id pins, undersized sensitive secrets, missing servel.managed label, stateful safety-floor drift) emitted by the daemon's resilience layer
servel doctor --remote KN --fix       # DEPRECATED — per CLAUDE.md SELF-HEALING DISCIPLINE the daemon is the canonical self-healer. Today --fix still applies fixes for backwards compat (swap, middlewares, Traefik timeouts, Traefik config drift, ACME mount path, CF Origin CA root, managed-label coverage, missing tools restic/smartctl). Each class graduates to daemon-resident auto-repair with budget+circuit-breaker invariants in follow-up PRs; the flag will be removed once all classes are folded.
servel doctor migration --target X    # End-to-end migration self-test (ephemeral postgres probe)
servel bench migration --target X --size 1GB  # Compare fullcopy vs snapshot at real data size
servel move history                   # Audit log of past migrations (newest first)
servel move history @postgres --limit 50  # Filter to one infra

# Cross-remote migration (different operator-managed remote, NOT same swarm)
servel move @postgres --to-remote navola --to NAVOLA-Manager --plan  # ALWAYS safe — preflight only
SERVEL_EXPERIMENTAL_CROSS_REMOTE=1 \
  servel move @postgres --to-remote navola --to NAVOLA-Manager       # Execute (env gate required)
servel move @postgres --to-remote navola --from-remote KN --plan    # Explicit source override
servel move @supabase --to-remote navola --to manager --unsafe-leave-links  # Multi-service; acknowledge linked-app breakage
servel move @postgres --to-remote navola --abort                   # Idempotent partial-state cleanup on both remotes
servel move @postgres --to-remote navola --commit-source-cleanup    # Drop source after a successful migration (refuses if target not serving)
# Trust model: operator orchestrates from local CLI holding SSH for both remotes; data
# routes source → operator → target via two chained SSH sessions (operator-relayed restic).
# Routes (Traefik dynamic configs) ship automatically; DNS update is the operator's job.
servel dashboard                      # One-screen overview of every optional subsystem
servel dashboard --remote KN          # Same, targeting a specific server
servel dashboard --watch 5            # Refresh in place every 5 seconds (incident monitoring)
servel df --growers                   # Top disk consumers (curated scan + "Reclaim space:" action block; shows progress while scanning)
servel df --growers --top 20          # Show top 20 instead of default 10
servel cleanup                        # Remove expired environments
servel cleanup --force                # No confirmation
servel prune                          # Remove dangling images/containers
servel prune --all                    # Remove unused images/networks/cache
servel prune --all --volumes          # DATA LOSS: removes unused volumes
```

### Node Management

```bash
servel node ls                        # List swarm nodes (+ MEM/CPU/DISK/AVAIL per node when stats collect)
# DISK % is df semantics: used/(used+avail). NOT used/total — total counts ext4's ~5% root reserve,
# which under-reported by 6 points at the danger end (391G disk, 15G left = 97%, previously shown 91.2%).
# AVAIL is the free bytes an ordinary writer has; coloured by absolute headroom (<20G amber, <5G red).
# Same DISK/AVAIL pair appears in `servel df --nodes` and in the `servel df` filesystem summary.
servel node ps                        # Per-node service view (grouped by host)
servel node ps --node KN-MANAGER      # Filter to specific node
servel node ps --json                 # JSON output
servel node add worker user@host      # Add node to cluster
servel node remove <name>             # Remove node
servel node promote <name>            # Promote to manager
servel node health <name>             # Check node health
servel node specs <name>              # Node specifications
servel node drain <name>              # Drain for maintenance
servel node drain <name> --remove     # Drain then remove from cluster
servel node activate <name>           # Reactivate drained node
servel node balance --dry-run         # Preview rebalance plan (CLI default strategy: memory; daemon default: auto with memory→tasks fallthrough). Planner is CPU-aware: simulates live CPU burn per move and prefers groups that relieve the source node's bottleneck (memory OR CPU).
servel node balance --strategy auto   # Memory planner first, fallthrough to tasks if 0 migrations + task spread > delta
servel node balance --strategy tasks --task-delta 5  # Equalize stateless task count per node
servel node balance                   # Execute cluster rebalance
# Auto-rebalance (daemon) is intent-aware + flap-protected: data-bound deployments (persisted placement_state.data_node anchor) are NEVER candidates — moving them can't stick (home-node persistence refuses to follow), so they'd ping-pong every cycle. Anti-flap breaker: a group that migrated >=rebalance_flap_threshold (default 3) times within rebalance_flap_window (default 24h) is frozen from candidacy + alerted once ("group flapping — frozen from rebalance; investigate"). Tune: servel daemon config set rebalance_flap_threshold=4 (negative disables).
servel node schedule <name> --at "2026-02-01 03:00"  # One-time drain
servel node schedule <name> --in 2h   # Relative time drain
servel node schedule <name> --cron "0 3 * * *"       # Recurring drain
servel node schedule <name> --cron "0 3 * * *" --reactivate "0 7 * * *"  # Drain + reactivate
servel node schedule ls               # List scheduled actions
servel node schedule cancel <name>    # Cancel schedule
servel node prune <node>              # Reclaim Docker images on a worker (SSH hop manager -> node)
servel node prune --all               # All worker nodes
servel node prune <node> --dry-run    # Preview
# The disk reading printed around each prune is advisory: a failed probe warns and the prune still runs.
servel node install --all             # Install servel CLI on all workers
servel node upgrade --all             # Upgrade servel CLI on all nodes
servel node alias <hostname> <alias>  # Set friendly alias
servel node label <hostname> key=val  # Add/remove node labels
```

### Auto-Update

Signed self-upgrade for the CLI, daemon, and configured remotes. Notify-only by default; opt in to auto-apply per machine. See [AUTO_UPDATE.md](references/AUTO_UPDATE.md) for the full pipeline (sha256 + ed25519 verification, key rotation, daemon notifier, rolling fleet upgrades).

```bash
servel upgrade                        # self-upgrade CLI (verify → stage .next → smoke → swap → probation)
servel upgrade --check                # check only, no install
servel upgrade --no-twin              # skip twin-serving probation; direct verified swap (escape hatch)
servel upgrade --rollback             # restore previously-installed binary
servel upgrade --pin v0.4.2           # auto-apply ceiling
servel upgrade --set-mode apply       # opt in to auto-apply (default: notify)
servel upgrade --servers              # also roll the configured remotes

servel upgrade-servers                # fleet upgrade — rolling + health-gated + auto-revert BY DEFAULT
servel upgrade-servers --no-rolling   # opt out: upgrade all at once, no health gate
servel upgrade-servers --server KN    # upgrade one specific remote

servel selfcheck                      # offline binary smoke (config + state-dir + docker ping); no network
```

**Twin-serving upgrades (default).** The new binary is never trusted until it proves health: download → stage as `<binary>.next` → verify checksum+signature → pre-swap smoke (`<next> version --json` + `<next> selfcheck`) → atomic swap → **post-swap probation**. A bad binary bricks both SSH JSON-RPC and the daemon, so after the swap a **sentinel** (launched from the *previous* binary, never the unproven one) watches the daemon heartbeat at `/var/servel/daemon/health.json` for 3 fresh ticks carrying the new build id within 120s. Fail → auto-revert to `.prev`, restart the daemon, critical audit + alert. Auto-revert is budgeted (3/24h/surface; exhausted → alert-only). Servers without the daemon degrade to pre-swap smoke only. `--no-twin` opts out.

**Cross-server (second line of defense).** `servel upgrade-servers` is rolling-by-default and applies the SAME heartbeat verdict across SSH: after each server's `sudo servel upgrade --force` (which runs the server-local sentinel = first line of defense), the client gates on `version --json` + a fresh `/var/servel/daemon/health.json` carrying the new build id. On failure it auto-reverts remotely (`sudo servel upgrade --rollback` + daemon restart), budgeted per `server:<name>` — UNLESS the server-local sentinel already reverted (old build fresh → no double rollback). First failure aborts the rest.

Daemon-side: every `servel remote status` reads `/var/servel/cache/update.json` (written by the daemon's once-per-24h check) and surfaces "new servel release available" inline. Auto-apply on the daemon is **never** done — operators run `servel remote upgrade` (or `servel upgrade-servers`) to roll the fleet.

### Swap Management

Two-tier autonomous swap (zram + elastic disk swapfile). Daemon configures, monitors, and resizes within per-node policy bounds. See [SWAP.md](references/SWAP.md) for full details.

```bash
servel node swap status                      # cluster-wide table: zram, disk, recommendations
servel node swap status <hostname>           # detailed view (devices, advisor reasons)
servel node swap status --json               # JSON for scripting

servel node swap enable --all --apply-now    # roll cluster default to every node, install now
servel node swap enable kn-deployments --max 16G   # raise this node's cap (memory-bound workload)
servel node swap enable worker-1 --no-zram   # disable zram (kernel module missing)
servel node swap enable kn-deployments --zram-algo zstd   # higher compression at CPU cost

servel node swap disable worker-1            # stop daemon's elastic loop, leave runtime alone
servel node swap disable --all --purge --yes # tear down zram + truncate /swapfile fleet-wide

servel node swap resize kn-deployments --to 6G   # one-shot manual disk-tier resize (gated)
servel node swap resize worker-1 --to 2G --force # bypass safety gates (deploy/mem/disk pressure)
```

Defaults: zram **on** (lz4, 2G), disk swapfile **2G→8G elastic**, swappiness 10. The daemon never auto-applies advisor recommendations — operators run `servel node swap enable` to apply them.


### Secrets

```bash
servel secrets set API_KEY            # Set (interactive prompt, never pass values inline)
servel secrets list                   # List keys
servel secrets get API_KEY            # Get value (output is masked by default)
servel secrets rm API_KEY             # Remove
servel secrets rotate API_KEY         # Rotate
servel secrets backup                 # Backup all secrets
servel secrets scan                   # Scan for exposed secrets
servel secrets copy <src> <dst>       # Copy secrets: deployment ↔ deployment ↔ .env
servel redeploy <name>                # Apply secret changes to running container (no rebuild)
servel deploy --migrate-secrets       # Auto-detect *_KEY, *_SECRET, *_PASSWORD
```

**Reading redacted output (`inspect`, `env vars`, `infra inspect`, `infra vars`):** two rules run before serialization, so `--json` is redacted too.

1. **By name** — key contains `PASS`/`PWD`/`SECRET`/`KEY`/`TOKEN`/`CREDENTIAL`/`AUTH`/`PRIVATE`/`JWT`/`SALT` anywhere (case-insensitive) → value becomes `***`. Covers `GOTRUE_SMTP_PASS`, `PGPASSWORD`, and PascalCase spec vars like `JwtSecret`.
2. **By value shape** — any `scheme://user:password@host` has its password replaced regardless of key name: `postgres://supabase_admin:***@db:5432/postgres`. Catches `SUPABASE_DB_URL`, `GOTRUE_DB_DATABASE_URL`.

The connection block of `infra vars` gets rule 2 as well — most hub templates build `Connection.URL` from a `connection_template` with the password inline.

`***` is a mask, never a real value — do not copy it into configs or connection strings. `--show-secrets` bypasses both rules on `inspect`, `env vars` and `infra vars`; `infra inspect` has no reveal flag.

**Applying secret changes to a running service:** `secrets copy` writes to encrypted `.env.age` on disk. To push values into the live container, run `servel redeploy <name>` — it diffs `spec.Env ∪ .env.age` against the live service env and emits `--env-add` / `--env-rm` (preserves user-added env vars; degraded-safe if inspect fails). `servel deploy <name>` does **not** accept a bare deployment name (requires path or yaml alias) — use `redeploy` for config-only application from any directory.

**Applying `servel.yaml` env edits without rebuild (2026-05-15):** `servel redeploy` now refreshes `spec.Env` from local `./servel.yaml` when its name matches the target deployment, BEFORE computing the env diff. Closes the trap where editing `servel.yaml` (e.g. swapping `localhost` for swarm hostnames) had no effect on `servel redeploy` because it only reapplied stored spec env. Secrets stay untouched. Use `--no-refresh` to opt out. Name mismatch (running redeploy from a different project's directory) skips refresh with a warning.

**`secrets copy` / `env copy` endpoint syntax** (same for both):

| Form | Meaning |
|---|---|
| `myapp` | Encrypted secrets / env of a deployment |
| `myapp@staging` | Deployment pinned to an environment |
| `./.env`, `path.env` | Local file (any path with `/` or `.env`) |
| `file:./path.env` | Explicit file form |
| `-` | Stdin (source) or stdout (destination) |

Default is **merge**; `--replace` for full replacement. Values never touch disk unencrypted in transit. Local writes are atomic with `0600` perms. Git-tracked files are refused unless `--force`. Production targets (env=production or deployment name contains `prod`) require typed `copy` confirmation. Every copy emits a `secret.copy`/`env.copy` audit entry with src/dst and `[+added ~updated -removed]` counts — never values. Common flags: `--keys`, `--exclude`, `--prefix OLD=NEW`, `--dry-run`, `--replace`, `--force`, `--prod-env`, `--show-keys`. Infrastructure endpoints (`@mydb`) and cross-server copies are v2.

### Domains & Routing

```bash
servel domains add myapp app.com      # Add domain (auto-SSL)
servel domains claim [myapp]          # Claim/retry the magic subdomain (domainless deploys auto-claim;
                                      # this is the manual retry after "subdomain pending"). Idempotent.
servel domains ls                     # List all domains
servel domains rm myapp app.com       # Remove domain
servel domains redirect old.com new.com # Create redirect
servel domains remove-redirect old.com # Remove redirect
servel domains list-redirects         # List redirects
servel routes <name>                  # Show deployment routes
```

### Traefik (Routing Layer)

```bash
servel traefik status                 # Router status
servel traefik status --history       # With historical events
servel traefik logs                   # Traefik logs
servel traefik logs -f --level error  # Follow with level filter
servel traefik logs --since 1h -n 100 # Recent logs with tail count
servel traefik routes <name>          # Detailed route info
servel traefik certs                  # SSL certificate info
servel traefik test <domain>          # Test domain routing
servel traefik debug <deployment>     # Debug routing config for a deployment
servel traefik restart                # Restart Traefik
```

**Slow uploads 502 at ~60s?** Traefik v3.x ships a 60s `entryPoints.<name>.transport.respondingTimeouts.readTimeout` default. Servers provisioned before that knob was set in `traefik/config.go` inherit the broken default. Run `servel doctor --remote <name> --fix` — the `Traefik Timeouts` check resolves the live `traefik.yml` from the docker mount (handles legacy `traefik.yaml`), fills in `300s` only when missing (never downgrades higher operator-chosen values), and force-restarts Traefik. Idempotent.

### Visitor IP / Forwarded Headers

**Tell apps to read `X-Real-Ip`. One header, one line, every language.** Servel preconfigures Traefik's entrypoints with `forwardedHeaders.trustedIPs = <all Cloudflare CIDRs>`, so Traefik computes `X-Real-Ip` as the rightmost-not-trusted entry of `X-Forwarded-For` (falling back to socket source). Result: with or without Cloudflare in front, `X-Real-Ip` is the visitor IP. Spoofing via injected XFF entries from outside the trusted set is structurally impossible.

```js
const ip = req.headers['x-real-ip'];     // Node / Express / Next.js
```
```go
ip := r.Header.Get("X-Real-Ip")          // Go
```
```py
ip = request.headers.get("x-real-ip")    # FastAPI / Starlette
ip = request.META.get("HTTP_X_REAL_IP")  # Django
```
```ruby
ip = request.remote_ip                   # Rails (walks XFF, equivalent)
```

**Do not** recommend the multi-fallback CF-Connecting-IP → XFF → socket pattern unless the app already uses it — that's defensive overkill on a Servel deploy. `X-Real-Ip` is sufficient.

**When to mention other headers:** `CF-Connecting-IP` lets the app distinguish "came via CF" from "direct hit" (same IP value, different presence). `X-Forwarded-For` is for audit / multi-hop debugging. Socket `RemoteAddr` is the Traefik overlay IP — useless as visitor identity.

**CF SSL mode matters:** "Flexible" silently terminates TLS at CF and re-opens plaintext to the origin. Run `servel verify cf-ssl <domain>` and require **Full (strict)** — Servel issues a real LE cert at the origin.

**CF IP range drift:** static slices `CloudflareIPv4Ranges` / `CloudflareIPv6Ranges` in `src/internal/traefik/config.go` (current as of Jan 2026). If CF adds edges, refresh the slices and redeploy Traefik (`servel remote provision` or restart `~traefik`). Yearly refresh is fine.

**OpenReplay infra (sessions → GeoIP):** the bundled nginx-openreplay needs the real client IP for country resolution. Swarm ingress source-NATs incoming connections, so X-Forwarded-For collapses to `10.0.0.x` on multi-node and every session resolves to "Unknown Country". The template's `BehindCloudflare` var defaults to `auto` — it probes the domain at deploy time and rewrites nginx to use `CF-Connecting-IP` when Cloudflare is detected. Override with `true` (force) or `false` (legacy XFF) only when auto-detection is wrong.

Full reference: `website/content/docs/reference/visitor-ip.mdx`.

**OpenReplay ClickHouse system logs (disk bomb):** ClickHouse retains its own telemetry tables forever by default. On a Servel-managed OpenReplay this outgrows the business data by four orders of magnitude — `system.trace_log` hit 139.79 GiB / 5.7B rows and `system.text_log` 22.27 GiB while `experimental.*` + `product_analytics.*` held a few MB (2026-08-08, and ~110 GB in a first occurrence in June 2026; ≈2 GB/day). Instances created from openreplay/v1.yaml **from 2026-08-08 onward are born bounded**: `file_templates` write two files, mounted into the `clickhouse` service via `overrides.volumes`.

- `data/servel-clickhouse-system-logs.xml` → `config.d/` — TTL 3d on `trace_log`/`text_log`, 7d on `metric_log`, `asynchronous_metric_log`, `query_log`, `query_thread_log`, `query_views_log`, `part_log`, `error_log`, `processors_profile_log`, 30d on `crash_log`; plus `global_profiler_real_time_period_ns` / `global_profiler_cpu_time_period_ns` / `total_memory_profiler_step` = `0`.
- `data/servel-clickhouse-profiler.xml` → `users.d/` — `query_profiler_real_time_period_ns` / `query_profiler_cpu_time_period_ns` / `memory_profiler_step` = `0` on the `default` profile.

**Both halves are required.** The *global* profiler is a server setting (default 10 s period, samples ClickHouse's own background threads) and completely ignores the per-profile `query_profiler_*` values — a users.d file alone leaves an idle instance writing thousands of trace rows. Measured on 26.2-alpine, identical 80M-row scan + 120 s idle: guarded 0 rows, stock 11 562 rows / 1.16 MiB.

Check any ClickHouse-backed infra:

```bash
servel exec @<infra> --service clickhouse -- clickhouse-client -q \
  "SELECT table, formatReadableSize(sum(bytes_on_disk)) FROM system.parts WHERE database='system' GROUP BY table ORDER BY sum(bytes_on_disk) DESC"
servel exec @<infra> --service clickhouse -- clickhouse-client -q "SHOW CREATE TABLE system.trace_log"   # want: TTL event_date + toIntervalDay(3)
```

Templates never retro-apply. **Pre-existing instances** must be fixed in place with `ALTER TABLE system.<t> MODIFY TTL event_date + INTERVAL 3 DAY` (persists in table metadata inside the data volume, survives restart and recreate). Dropping >50 GB needs `--max_table_size_to_drop=0 --max_partition_size_to_drop=0`. Never bolt the config file onto an existing instance: ClickHouse renames the old table to `<name>_0` and the orphan keeps unlimited retention. The standalone `clickhouse` hub template (`databases/clickhouse`) carries the same two files (verified on its pinned 24.8-alpine: all 11 TTLs present, 0 vs 113 trace rows against a stock instance). Reaching it required teaching the single-image deploy path to write `file_templates` and to redirect relative bind sources — that path previously did neither, so a template-declared config file silently became an empty directory.

### Verification

```bash
servel verify <name>                  # Full verification
servel verify config                  # Verify configuration
servel verify health <name>           # Current Swarm tasks + HTTP; 5+ current-generation restarts fail; Docker health healthy/starting/unhealthy/none; unavailable evidence stays unknown
servel verify ssl <domain>            # Check SSL certificates
servel verify cf-ssl [project]        # Classify CF→origin SSL mode (Full strict / Flexible / Off)
servel verify dns <domain>            # Check DNS configuration
servel verify routing <name>          # Check Traefik routing
servel verify dependencies <name>     # Check dependencies
servel verify resources               # Check resource availability; no local/empty stats = unknown, SSH or stats JSON failure = error; --quiet keeps both
```

### Dev Mode

```bash
servel dev                            # Start dev session
servel dev --team                     # Bidirectional sync (collaboration)
servel dev --port 3001                # Custom port
servel dev --domain staging.app.com   # Custom domain
servel dev --no-sync                  # One-time upload only
servel dev --conflict-policy newer-wins # Sync conflict resolution
servel dev --link-env myapp-prod      # Pull plaintext env from a deployment
servel dev --link-env myapp-prod --secrets   # +decrypted secrets (ACL: dev:env:pull or owner)
servel dev --link-infra @postgres,@redis     # Inject @infra connection vars (auto-tunneled)
servel dev --link-env myapp-prod --only DATABASE_URL,REDIS_URL  # Whitelist filter
servel dev --link-env myapp-prod --exclude SENTRY_DSN           # Blacklist filter
servel dev --link-infra @postgres --no-tunnel  # Disable auto-tunnel (warns per internal host)
servel dev list                       # Active sessions
servel dev logs <id> -f               # Follow session logs
servel dev stop <id>                  # Stop session
servel tunnel                         # Expose localhost publicly
servel tunnel start <port>            # Start tunnel on port
servel tunnel list                    # List active tunnels
servel tunnel stop <id>               # Stop tunnel
servel port-forward @db:5432          # Forward remote port locally
servel port-forward @db:5432 -- drizzle-kit push  # Ephemeral tunnel + run command
servel pf @db:5432 -- drizzle-kit push            # Short alias
servel pf @db:5432 --env-file .env.local -- drizzle-kit push  # With env file
servel port-forward @db:5432 --detach  # Background tunnel
servel port-forward list               # Show active tunnels
servel port-forward stop <id>          # Stop tunnel
```

**Conflict policies:** remote-wins, local-wins, newer-wins, backup

#### Linking Production Env into Dev (`--link-env` / `--link-infra`)

When the user wants the dev container to run with prod-shaped env (real DB URL, real connection bundles, optionally real secrets), reach for `--link-env` / `--link-infra`. **Never** ask the user to copy/paste from `servel env vars` or `.env`; the link flags exist so they don't have to.

| Goal | Flag |
|------|------|
| Pull plaintext env from a real deployment | `--link-env <deployment>` |
| Inject `DATABASE_URL`/`REDIS_URL`/etc. from one or more `@infra` | `--link-infra @name[,@name...]` |
| Also pull decrypted secrets (opt-in, ACL-gated) | `--secrets` |
| Keep only specific keys | `--only KEY[,KEY...]` |
| Drop specific keys | `--exclude KEY[,KEY...]` |
| Pass internal-host values through unchanged (advanced) | `--no-tunnel` |

Auth invariants (server-side, in `dev_env.go:authorizeDevEnvPull`):

1. Empty `SERVEL_GATE_USER` → allow (direct SSH = root-equivalent).
2. Gate user equals `deployment.DeployedBy` → allow (owner bypass).
3. Gate user has `dev:env:pull` permission in any scope → allow.
4. Otherwise → deny with explicit `deny_reason` in the audit log.

Role-based grants (admin, super_admin) deliberately do NOT include `dev:env:pull` — pulling prod secrets to a laptop is opt-in per user. Grant explicitly: `servel access scope add <user> --server <name> --permissions dev:env:pull`.

Auto-tunnel: when a linked value contains `servel-infra-*` or `servel-system-*` with a port (e.g. `postgres://...@servel-infra-pg:5432/db`), `servel dev` spawns a `portfwd.Manager` per host:port pair and rewrites the value to `localhost:N` before injecting. Bare hostnames with no port surface a warning instead — there's no port to forward. `--no-tunnel` skips the rewrite and warns per detected host. Now that `servel deploy --link-infra` defaults to internal Docker DNS, prod env vars pulled via `--link-env` will more often contain `servel-infra-*` hosts — the auto-tunnel handles them transparently.

Cache: pulled env is written to `~/.servel/dev/sessions/<sid>/env.cache` (mode 0600, dir 0700, 1h TTL, JSON with provenance). Removed on dev exit; stale caches from crashed sessions are evicted on next `servel dev` start. Cache files with non-0600 perms are refused on read.

Cross-server limitation: the linked deployment / `@infra` must live on the same Docker swarm as the dev session's SSH endpoint. `--link-server` is not implemented (future work).

Audit: every pull (allow OR deny) emits action `dev.env.pull`. Metadata records key names only — never values. Inspect with `servel audit list --action dev.env.pull --limit 20`.

See [DEV_LINK_ENV.md](references/DEV_LINK_ENV.md) for the full reference.

### Environment Variables

```bash
servel env set <name> KEY=VALUE       # Set env var (no rebuild, restarts service)
servel env vars <name>                # Show env vars (secrets redacted as ***; --show-secrets to reveal)
servel env list                       # List environments
servel env copy <src> <dst>           # Copy env vars: deployment ↔ deployment ↔ .env (plain vars)
servel config show <name>             # Show deployment config
servel config show --server           # Show server config (/var/servel/config.yaml) over SSH
servel config sync <name>             # Sync config to servel.yaml
servel config sync --dry-run          # Preview sync
servel set-env-file .env              # Set env_file in servel.yaml (.local rejected)
```

**Configuration knobs** — operate on either `~/.servel/config.yaml` (`--client`, default) or `/var/servel/config.yaml` (`--server`, over SSH; pair with `--remote <name>`):

```bash
# Auto-routes to --server because build_queue.* is server-only
servel config set build_queue.max_concurrent 2
servel config set log_retention.max_age_days 14 --remote KN
servel config set auto_update.mode apply --client

servel config get build_queue.max_concurrent --remote KN
servel config get auto_update.mode --client --json

servel config list --server --section build_queue --defaults  # tree + defaults
servel config list --client                                   # all client keys

servel config wizard --server --remote KN                     # walk every key interactively (promptui select for bool/enum, edit-inline for others)
servel config wizard --server --section build_queue           # scope wizard to one section
servel config wizard --server --editor                        # open $EDITOR on YAML w/ type:X default:Y annotations
servel config wizard --server --editor --editor-bin nvim      # override editor binary

servel config validate                                        # validate live client config
servel config validate --server --remote KN                   # pull + validate server config
servel config validate ./draft.yaml --server                  # dry-run a yaml file vs schema
```

**Type coercion** for `set` and `wizard`: bool accepts `true/false/yes/no/y/n/on/off/1/0`; duration uses Go syntax (`10m`, `24h`); `[]string` is comma-separated; `map[string]string` is `key=value,key=value`. Secret-like fields (`*token*`, `*password*`, `*auth_key*`) print as `***` unless `--raw`/`--include-secrets`.

**Allocating an unset section is safe**: setting one leaf inside a previously-nil section (build_queue, log_retention, build_cache, registry_retention, access, auto_update, telemetry) populates siblings from canonical defaults — never zero-initializes them. So `servel config set build_queue.max_concurrent 2` on a fresh server writes `{enabled: true, max_concurrent: 2, queue_timeout: 10m}`, not `{max_concurrent: 2}` with `enabled: false`.

**Operator identity (`user.name` / `user.email`) — the `git config user.*` parallel (added 2026-05-31).** Servel stamps a self-declared operator onto every audit entry so the log answers "which human ran this" on single-tenant boxes where everyone SSHes in as `root`:

```bash
servel config set user.name  "Kerem Noras"      # client config (~/.servel/config.yaml `user:` block)
servel config set user.email "kerem@noras.tech"
servel config get user.email
```

It is **attribution, not authentication**: the access gate's authenticated principal stays authoritative in the audit `user` field; operator rides alongside in `operator`/`operator_email`. **Destructive commands are GATED on it** — `remove`, `rollback`, `secrets {set,delete,rotate,rotate-all,import,migrate}`, `node {remove,forget,drain}`, `infra {remove,restore,rotate}` (deletes/overwrites DB/queue data), and `access` mutations (`user {create,delete,disable,enable,modify,rename}`, `scope {set,add,remove}`, `role create`, `request {approve,deny,revoke,extend,modify}`) refuse to run until an operator **email** is set (read-only commands and `--dry-run` are never gated; server-side re-execs are exempt — the gate is client-side). For automation/CI, satisfy the gate WITHOUT storing config by exporting the identity — agents running destructive commands non-interactively MUST set these or the command errors:

```bash
SERVEL_OPERATOR_NAME="ci-bot" SERVEL_OPERATOR_EMAIL="ci@example.com" servel rollback myapp --yes
```

The identity auto-propagates to ALL server-side audit writes (`deploy`, `rollback`, every `access`/`infra`/etc. command routed over SSH, and MCP tools) via `SERVEL_OPERATOR_NAME`/`SERVEL_OPERATOR_EMAIL` injected into the remote command string. `servel audit list` shows operator in place of the bare login; `--details` prints both; `--json` / `servel --json audit list` and CSV export include `operator`/`operator_email`.

**Build queue is cluster-aware on multi-node remotes**: `max_concurrent` becomes per-host (each host gets its own slot pool), and queued deploys fan out across hosts instead of serializing through the manager. Override individual hosts with `build_queue.per_host_concurrency` (map of `hostname → slot count`; hostname is used — not swarm node ID — so config survives node rejoins). The queue tracks BuildKit cache affinity (last-built host per project) and per-host RAM reservations so parallel acquires don't pile onto a stale free-RAM snapshot. FIFO + priority ordering is preserved across hosts via a fairness gate (entries past total cluster capacity wait). Single-host remotes keep the legacy global pool — no behavior change.

**Common server keys**: `build_queue.{enabled,max_concurrent,queue_timeout,priority_deployments,per_host_concurrency}`, `log_retention.{max_age_days,max_size_mb,compress,schedule}`, `build_cache.{max_size_gb,max_age_days,prune_on_deploy,registry_export,registry_export_mode}`, `registry_retention.{keep_per_repo,older_than,always_keep}`, `deployment_retention`, `port_range_start`, `port_range_end`, `access.{enabled,audit_retention_days}`. Run `servel config list --server --defaults` for the full inventory on any remote.

**Registry-backed build cache (default-on, multi-node)**: when a registry is configured, builds use a registry-stored BuildKit layer cache (`--cache-to`/`--cache-from type=registry,mode=max` on the `docker-container` builder) under a single `<image>:buildcache` tag, overwritten each build. It survives builder resets and is shared across nodes (a fresh/other node pulls warm cache instead of building cold), and the deps `RUN --mount=type=cache` now survives transient BuildKit session-error recovery (restart-before-recreate). Disk-bounded: tag overwritten per build (no per-version accumulation), stale layers reclaimed by daily registry garbage-collect, `:buildcache` preserved by retention. Disable/tune: `build_cache.registry_export` (bool, default true), `build_cache.registry_export_mode` (`max` default / `min`). The image cache (skip-build on unchanged source hash) is separate and unchanged.

**Note:** `env copy` targets plain Docker service env vars; `secrets copy` targets the encrypted store. Same endpoint syntax and flags for both (see `secrets copy` section above). Use `env copy` for non-sensitive config, `secrets copy` for credentials.

**Note:** `servel deploy` auto-reads `.env` + `.env.local` from project dir and injects as Docker env vars. This is NOT persistent — vars are re-sent each deploy. For persistent encrypted storage, use `servel secrets`. See [Environment Variables & Secrets](#environment-variables--secrets) workflow for full details.

### Alerts

```bash
servel alerts setup                   # Interactive wizard
servel alerts add telegram            # Add Telegram channel
servel alerts add slack               # Add Slack channel
servel alerts add discord             # Add Discord channel
servel alerts add webhook             # Add webhook
servel alerts test                    # Test notifications
# Outside-in probes (daemon, default-on): every routed public domain HEAD-probed via real DNS/CDN every 5min; any HTTP status = reachable, transport failures + expired certs = down; 3 strikes → domain_unreachable alert (domain→service→node). Skip: `docker service update --label-add servel.probe.skip=true <svc>`; disable: `servel daemon config set domain_probes_enabled=false`. Catches the inside-checks-green-but-world-sees-down class.
# Backups default-on (daemon): stateful infra auto-enrolled into daily restic schedules (opt-out: infra backup.auto_enroll:false / backup_auto_enroll_enabled=false); restic auto-installed when missing; weekly restic check + monthly restore-to-scratch drill — backup_verify_failed alert = your safety net is broken, act.
# Containment (daemon, default-on): crash-loop quarantine — service storming >=2 consecutive windows is scaled to 0 (budget 3/24h cluster-wide, 1/stack; stack root-folding quarantines only the -db root; >5 unrelated storms = node-level breaker, alert only; release: servel scale <name> 1 / servel infra repair <infra>). Infra dup-generation reconciler force-converges services running MORE tasks than desired (sustained 2 checks, budget 5/24h).
# Noise control (daemon-side, /var/servel/alerts.yaml): EDGE-TRIGGERED — each drift class (condition|service|server) is a FIRING/RESOLVED state machine; an alert fires ONCE on entry, is suppressed while unchanged (NO re-fire per cooldown), re-delivers only on severity/fingerprint change, recovers once on the resolving edge; state persisted (restart does not replay). Flap damping: N consecutive fails to alert + N consecutive clean cycles to recover, so oscillating probes emit no unreachable/repaired pairs. Critical heartbeat: still-FIRING criticals re-paged every reminder_interval (1h); warnings/operator-intent (e.g. parked scale-0) fire once + audit only, never nag. digest_threshold 5 + digest_window 60s collapse bursts into ONE summary; telegram bell rings ONLY for critical. suppress_cooldown now = RESOLVED-entry grace/reap window (not a re-fire interval).
servel alerts status                  # Show alert status
servel alerts history                 # View alert history
servel alerts pause 2h                # Maintenance mode (pause alerts)
```

### Status Page

```bash
# Stand up a public status page at status.<primary-domain>
servel status enable

# Custom domain + auth (credentials printed once — record them)
servel status enable --domain health.example.com --auth

# Pin to a specific node + custom title
servel status enable --node KN-MANAGER --title "My Cluster Status"

# Use a non-default infra name
servel status enable --name health

# Show page URL, auth state, monitored count, live up/down
servel status list
servel status list --name health

# Tear it down (prompts unless --yes)
servel status disable
servel status disable --yes

# Bare command = list
servel status
```

**How it works:**
- `enable` enumerates every public Traefik route (file-provider + label-routed deployments) via the same inventory the daemon's outside-in prober watches, then renders a deterministic gatus config.yaml (endpoints sorted by group+name, names de-duplicated for gatus), dogfoods `servel add gatus <name>`, seeds the config.yaml into the gatus `/config` named volume, and writes the Traefik route if absent.
- `disable` removes the gatus infra + all matching Traefik route files (`infra-<name>-*.yml`).
- `list` reads the seeded config.yaml for monitored count + auth state; queries the gatus API (internal Swarm DNS) for live up/down (best-effort, silent on failure).
- The status page is itself cluster-local — a full ingress outage takes it down too. Complement with an external monitor for the page's own URL.

**Flags:**
- `--domain <d>` — status-page domain (default: `status.<primary-domain>`)
- `--auth` — generate bcrypt basic auth; credentials printed once, not stored in plaintext
- `--node <hostname>` — pin gatus service to a node
- `--name <n>` — infra name for the status page (default: `status`)
- `--title <t>` — dashboard title (default: `Servel Status`)

### CI/CD

```bash
servel ci setup                       # Interactive wizard (init + token creation)
servel ci setup github-actions        # Setup specific provider
servel ci init github-actions         # Generate workflow only (no token)
servel ci init gitlab-ci --legacy-ssh # Legacy SSH key-based template
servel ci list                        # List pipelines
servel ci run <config>                # Run built-in CI
servel ci run <config> --domain x.com # Auto-route CI service
servel ci status <run-id>             # Check run status
servel ci logs <run-id>               # View CI logs
servel ci recent                      # Recent runs
servel ci cancel <run-id>             # Cancel run
servel ci retry <run-id>              # Retry run
```

### Access Control

```bash
servel auth login                     # User authentication
servel auth logout                    # Logout
servel auth whoami                    # Current user info
servel auth enable <name>             # Enable basic auth
servel auth disable <name>            # Disable basic auth
servel access user                    # User management
servel access user create --name bob --ssh-key key.pub  # Add user with key
servel access user create --name bob --generate-key     # Generate keypair for user
servel access scope add bob --server KN --permissions deploy  # Grant extra perms on one server
servel access scope add bob --server KN --permissions deploy,logs --infra mydb --deployment app
servel access scope show bob --json   # Verify effective scopes
# Scope perms are ADDITIVE (extend role; never subtract). Fixes "your role X lacks Y permission"
# without changing the user's global role. --permissions + --infra + --deployment + --env compose.
servel access role                    # Role management
servel access setup                   # Initialize access control on server
servel access setup --rotate-join-key # Rotate join key
servel access invite --role deployer   # Generate invite token
servel access invite --embed          # Write invite token to .servel/access.yaml
servel access invite ls               # List pending invites
servel access invite revoke <id>      # Revoke invite
servel access invite rotate <id>      # Rotate token (new token, old revoked)
servel access invite clean            # Remove expired/used invites
servel access join <token>            # Join server (idempotent -- safe for CI reruns)
servel access join <token> -i ~/.ssh/key  # Join with specific SSH key (multi-identity local testing)
# Identity = one remote per key. Same host + different key → new remote (dedupe is by host+key, not host alone).
# Switch identity = switch --remote. Default identity = the remote named by default_remote in ~/.servel/config.yaml.
servel access leave                   # Leave a server you joined
servel access request                 # In project dir: public request or check approval status (state.json w/ join_key_seed → no prior access needed; access.yaml → join)
servel access request create [srv] --reason "..." --duration 2h  # Request JIT access
servel access request create --deployments app --infra mydb      # Custom scope
servel access request list --status pending  # List access requests (shows SCOPE column)
servel access request approve <id>        # Interactive TUI for scope selection
servel access request approve <id> -y     # Skip TUI, approve full scope
servel access request approve <id> --only-infra mydb             # Narrow scope on approve
servel access request approve <id> --no-infra                    # Strip infra from grant
servel access request modify <id>         # Adjust scope of approved request (TUI)
servel access request extend <id> --by 4h # Extend expiry
servel access request revoke <id>         # Immediately revoke approved request
servel access request deny <id>       # Deny request
servel access request cancel <id>     # Cancel own pending request
servel access request expire-check    # Expire overdue grants (daemon runs this)
servel access request-hint "msg"      # Set hint shown when access denied
servel access request-hint --project myapp "msg"  # Project-specific hint
# After approval, ANY command auto-configures the remote
servel ps    # (in project dir) → auto-joins if approved, then runs
```

### IP Bans

```bash
# Server-wide bans (ipset+iptables, propagated to all swarm nodes)
servel ban 1.2.3.4                          # Block IP/CIDR everywhere
servel ban 10.0.0.0/8 --reason "scanner"    # With reason
servel unban 1.2.3.4                        # Remove server-wide ban
servel ban ls                               # List server-wide bans
servel ban clear --yes                      # Clear all server-wide bans
servel ban sync                             # Replay bans to all nodes (after node rejoin)

# Per-deployment bans (Traefik denyip plugin — needs one-time setup)
servel remote setup-granularban             # One-time: install denyip Traefik plugin
servel ban myapp 1.2.3.4                    # Block IP from a specific deployment
servel ban @chatwoot 5.6.7.0/24             # Block CIDR from infrastructure
servel unban myapp 1.2.3.4                  # Remove per-deployment ban
servel ban ls myapp                         # List bans for a target
servel ban clear myapp --yes                # Clear all bans for a target

# Aliases: `block` / `unblock`
```

**When to use which:**
- Use **server-wide bans** for known-bad IPs (scanners, brute force, abusers). Drops at the kernel before traffic touches any service.
- Use **per-deployment bans** when you need to block IPs from one specific app but allow them on others. Operates at the HTTP middleware layer.
- Use **`--allow-ip`** at deploy time when you want a strict allowlist (e.g., admin dashboard restricted to office IPs).
- Bans survive `servel deploy` and `servel rollback` automatically — no manual re-application.
- `servel rm <name>` cleans up the deployment's ban state automatically.

### Security Checkpoint (Vercel-style bot/DDoS gate)

Per-deployment ForwardAuth challenge (PoW or Cloudflare Turnstile) that runs as a system service alongside Traefik. Stateless HMAC cookies; no Redis.

```bash
# Master switch — heuristic-gated by default
servel security checkpoint enable myapp
servel security checkpoint enable myapp --mode always
servel security checkpoint enable myapp --verifier turnstile

# Runtime mode flips (no redeploy needed)
servel security checkpoint attack myapp                 # → under-attack
servel security checkpoint normal myapp                 # → revert

# Disable / status
servel security checkpoint disable myapp
servel security checkpoint status [myapp]

# Rotate HMAC key (with grace period — old cookies still valid)
servel security checkpoint rotate-key

# Aliases: `servel sec checkpoint *`
```

**Modes:** `off` | `suspicious` (default) | `always` | `under-attack` (auto-set on RPS spike).

**Per-route + per-domain config in `servel.yaml`:**
```yaml
security:
  checkpoint:
    enabled: true
    mode: suspicious
    cookie_ttl: 24h
    fail_mode: open                    # 'closed' to block when daemon down
    routes:
      - match: "/api/webhooks/*"
        skip: true                     # Stripe, GitHub
      - match: "/admin/*"
        mode: always
    bypass:
      cidrs: ["10.0.0.0/8"]
      user_agents: ["Stripe/*"]
```

**System service:** `servel-system-checkpoint` deployed automatically by `servel server provision`. ForwardAuth target via `servel-checkpoint@file` middleware. State at `/var/servel/checkpoint/{hmac.key,policies.json}`. Inert until a deployment opts in.

**When to use:** Public-facing app under bot/scraper pressure, or anticipating burst traffic. Don't enable on internal-only services or APIs called by known partners (use bypass CIDRs/UAs instead).

### Registry

**Default behavior:** `servel deploy` autodetects the registry from `git remote get-url origin`. GitHub → `ghcr.io/<owner>/<repo>`, GitLab → `registry.gitlab.com/<group>/<project>`. No git remote → self-hosted. Override per-project with `servel.yaml: registry: <url|self-hosted|named>`.

Resolver priority: `--registry` flag > `servel.yaml: registry:` > git autodetect > server default > self-hosted.

```bash
servel registry                       # Cross-registry table (every configured registry)
servel registry myregistry            # Per-repo listing for one registry
servel registry tags <repo>           # Bare, host/ns/name, or @<project>
servel registry rm <repo>:<tag>       # Same resolution as tags
servel registry info                  # Registry info + capabilities
servel registry du                    # Per-repo disk usage (self-hosted; hint at >5GB unique)
servel registry retain --keep 10      # Trim old versions, keep 10 newest per repo (dry-run + --older-than 30d)
servel registry migrate               # Auto-detect project + remote from .servel/state.json (run from project dir)
servel registry migrate <project>     # Move project to its auto-detected registry (ghcr/gitlab)
servel registry migrate --all --continue-on-error  # Bulk migrate
servel registry decommission          # Tear down self-hosted (after migrate, --keep-volume for safety)
```

**Always-preserved tags** for retain: `latest`, `stable`, `main`, `master`. Daily systemd timer runs retention + GC at `/var/servel/scripts/registry-retain.sh`.

### Auth Setup (External Registries)

```bash
docker login ghcr.io
servel auth registry add ghcr.io --import-docker-config  # Easiest: read from ~/.docker/config.json (falls back to credsStore/credHelpers — Docker Desktop, pass, secretservice)
servel auth registry add ghcr.io --username k-nrs --password $GHCR_TOKEN  # Explicit
servel auth registry test ghcr.io                        # Verify before bulk migrations
servel auth registry ls                                  # List configured
servel auth registry rm ghcr.io                          # Remove
```

Token scopes: GHCR needs `read:packages`+`write:packages` (+`delete:packages` for `registry rm`). GitLab needs deploy token with `read_registry`+`write_registry`, or PAT with `api`. Stored Age-encrypted at `/var/servel/secrets/registry-auth/<sha256>.age` + merged into `~/.docker/config.json`.

**Multi-node TODO:** `auth registry add` lands on one host today. Workaround: run `--remote <node>` per worker until cross-node distribution lands.

### Volumes

```bash
servel volumes                        # List all volumes
servel volumes --dangling             # Unused volumes (Links=0)
servel volumes --orphaned             # Volumes whose owner was deleted
servel volumes --json                 # JSON output
servel volumes inspect <name>         # Detailed volume information
```

### Audit

```bash
servel audit list                     # View audit logs (default)
servel audit list --user bob --since 7d  # Filter by user and time
servel audit list --app myapp --severity high --details  # With details
servel audit stats                    # Action counts, failure rates
servel audit export --format csv -o audit.csv  # Export to CSV/JSON
servel audit export --format json -o audit.json --since 30d
servel audit rotate --keep-days 90    # Retention policy (default: 90 days)
```

### Bastion (SSH Gateway)

```bash
servel bastion start                  # Start bastion server
servel bastion start --listen :2222   # Custom listen address
servel bastion restart                # Restart bastion
servel bastion install --start        # Install as systemd service
servel bastion uninstall              # Remove systemd service
servel bastion session list           # List recorded sessions
servel bastion session play <id>      # Playback recorded session
servel bastion session play <id> --speed 2.0  # Fast playback
servel bastion session info <id>      # Session metadata
servel bastion session commands <id>  # Extract commands from session
```

### Advanced

```bash
servel detect                         # Detect project build type
servel detect --verbose               # Detailed detection info
servel init                           # Initialize servel.yaml
servel validate                       # Validate servel.yaml
servel upgrade                        # Self-upgrade CLI (signed sha256+ed25519, atomic swap)
servel upgrade --check                # Check for update; print result, exit
servel upgrade --rollback             # Restore previously-installed binary (.prev)
servel upgrade --pin v0.4.2           # Pin auto-apply ceiling
servel upgrade --pin off              # Clear pin
servel upgrade --set-mode notify      # notify | apply | off — default notify
servel upgrade --servers              # Self-upgrade then run upgrade-servers
servel upgrade-servers                # Upgrade servel on all servers (sequential)
servel upgrade-servers --rolling      # One-at-a-time with health gate (recommended for prod)
servel upgrade-servers --server KN    # Upgrade specific server
servel upgrade-servers --dry-run      # Preview plan
servel tag <name> <tag>               # Add tags to deployment
servel untag <name> <tag>             # Remove tags
servel reconcile                      # Discover/fix unlabeled services and missing state
servel reconcile --dry-run            # Preview reconciliation
servel reconcile --deployments        # Only deployments
servel reconcile --infra              # Only infrastructure
servel queue                          # Show build queue: active builds + waiting (aliases: bq, build-queue)
servel queue clean                    # Force cleanup stale build queue entries
servel telemetry                      # Show telemetry status
servel telemetry enable               # Enable anonymous telemetry
servel telemetry disable              # Disable anonymous telemetry
```

## Common Workflows

### Deploy with Database

```bash
servel add postgres --name mydb
servel deploy --verbose --link-infra mydb
# App receives DATABASE_URL, DB_HOST, DB_PORT, DB_PASSWORD
```

### Preview Deployments

```bash
servel deploy --verbose --preview --ttl 24h
# Returns: https://myapp-pr42.example.com
```

### Multi-Environment

```bash
servel deploy --verbose --env production
servel deploy --verbose --env staging
servel deploy --verbose --preview
```

### Environment Variables & Secrets

**How `servel deploy` handles .env files:**
- Reads `.env` (base) then `.env.local` (override) from the project directory
- Injects them as **Docker service environment variables** (not servel secrets)
- Vars are sent on every deploy — if you delete `.env.local`, those vars won't be in the next deploy
- `.env.local` is auto-detected and takes priority when present
- `.env` files are **excluded from the deployment package** (never uploaded to the server as files)

**This is NOT automatic migration to secrets.** The vars live as plain Docker service env vars unless you explicitly use secrets:

```bash
# Option 1: servel.yaml env block (fine for non-secret config)
# env:
#   NODE_ENV: production
#   API_URL: https://api.example.com

# Option 2: servel secrets (encrypted, persistent, recommended for API keys)
servel secrets set API_KEY             # Interactive prompt (recommended)
servel secrets set DB_PASSWORD         # Never pass secret values inline

# Option 3: Auto-detect sensitive vars during deploy
servel deploy --verbose --migrate-secrets
# Detects *_KEY, *_SECRET, *_PASSWORD patterns → prompts to encrypt

# Option 4: servel.yaml secrets block (keys loaded from .env at deploy time)
# secrets:
#   - API_KEY        # Value pulled from .env/.env.local during deploy
#   - DB_PASSWORD    # If not in .env, assumed already on server (encrypted)
```

**Priority (highest wins):** **infra-link injection** > servel secrets > servel.yaml `env:` > `env_file:` > `.env` > `.env.local`

**Important distinctions:**
- **Infra-link wins** (since 2026-05-10): when `infra: - name: mydb` is in servel.yaml, those env vars come from the live infra spec at deploy time and override anything in `.env*` / `env_file` / `yaml env:`. Conflicts show as a single dim line: `(infra-link overrode N key(s) from .env / env_file / yaml env: …)`. To opt out: `--public` or `access: public` per link.
- **`.env.local` is dev-only** by Next.js convention. Today it's still loaded for `servel deploy` with a soft notice; a future release will skip it so it stays clean for `next dev` / `bun dev` localhost work. For production-only overrides, use `env_file:` (e.g. `env_file: .env.production`).
- `env_file:` in servel.yaml rejects `.local` files (dev-only convention)
- `servel set-env-file .env` — convenience command to set `env_file` in servel.yaml
- `servel env set myapp KEY=VALUE` — update env var on running deployment without rebuild
- NEXT_PUBLIC_* vars are kept in build-time env AND runtime (Next.js needs them at build time)
- **Stale-key drift defense**: `servel secrets reconcile <app>` (interactive) or `prune_secrets: true` in servel.yaml (auto on every deploy). Removes encrypted secrets no longer declared and not injected by infra.

### Log Observability

**All application logs are accessible via `servel logs` — no SSH, no Docker commands, no log aggregator needed.**

```bash
servel logs myapp -f                  # Follow logs (live tail)
servel logs myapp --since 1h          # Last hour
servel logs myapp -n 200              # Last 200 lines
servel logs @mydb -f                  # Infrastructure logs (@ prefix)
servel logs @chatwoot --service rails -f  # Multi-service infra logs
servel logs ~traefik                  # System service logs (~ prefix)
```

When debugging issues in a servel-deployed project, **always start with `servel logs`** — it streams container stdout/stderr directly to your terminal.

### Debug Container

```bash
servel logs myapp -f                  # View logs
servel exec myapp sh                  # Shell into container
servel exec myapp -- cat /app/.env    # Run command
servel logs @mydb -f                  # View infra logs (@ prefix)
servel exec @chatwoot --service rails sh  # Multi-service infra
```

### Backup & Restore

```bash
servel infra backup mydb
servel infra restore mydb backup-2024-01-15.sql.gz
```

### CI/CD Deployment (No SSH Keys in Repo)

**Recommended: Use `servel ci setup` -- one command to generate workflow + token.**

```bash
# One-command setup (interactive wizard)
servel ci setup

# Or step by step:
# 1. Create a deployer token (on your machine, one-time)
servel access invite --role deployer --expiry 8760h --uses 10000

# 2. Store token as CI secret (e.g., SERVEL_TOKEN in GitHub Actions)

# 3. In CI pipeline:
servel access join $SERVEL_TOKEN    # Idempotent -- safe for ephemeral CI runners
servel deploy --verbose             # Deploy
```

**Why tokens > SSH keys:**
- No private key material in CI secrets
- Built-in MITM protection (host fingerprint in token)
- Built-in expiry + use limits
- Scoped to deployer role (no shell access)
- Each join generates ephemeral SSH keypair
- Idempotent join -- reruns don't fail or waste invite uses

**GitHub Actions example:**
```yaml
steps:
  - uses: actions/checkout@v4
  - run: curl -fsSL https://servel.dev/install.sh | bash  # Official installer (pinned to latest stable)
  - run: servel access join ${{ secrets.SERVEL_TOKEN }}
  - run: servel deploy --verbose
```

**Token rotation:**
```bash
servel access invite rotate <id-prefix>    # New token, old revoked
servel access setup --rotate-join-key      # Invalidate ALL tokens (emergency)
```

### Troubleshoot Routing

```bash
servel verify dns app.example.com     # Check DNS
servel verify ssl app.example.com     # Check SSL
servel traefik test app.example.com   # Test routing
servel traefik debug myapp            # Debug routing config
servel traefik logs                   # View Traefik logs
```

## Configuration (servel.yaml)

### Complete Reference

```yaml
name: myapp
domain: app.example.com
domains:                              # Multiple domains
  - app.example.com
  - api.example.com
port: 3000

# Registry override (default: autodetect from git origin)
# registry: ghcr.io/k-nrs/myapp       # explicit external
# registry: self-hosted                # force self-hosted (e.g. local-only experiments)
# registry: corporate-internal         # named registry from server config
# (omit entirely)                      # autodetect — GitHub→ghcr, GitLab→registry.gitlab.com

# Cloudflare proxy support
# PREFER Full (strict) on the CF dashboard + leave this UNSET so Servel keeps the
# origin HTTPS redirect on as defense-in-depth. Flexible = CF↔origin plaintext.
# Verify posture with: servel verify cf-ssl <project>
cloudflare: true                      # ONLY for CF Flexible SSL; skips HTTPS redirect to avoid loops
www: redirect                         # WWW handling: "redirect" (www->apex), "redirect-to-www" (apex->www)

# Environment
env:
  NODE_ENV: production
env_file: ".env"                      # Load from file (excludes .env.local patterns)
secrets:
  - API_KEY
  - DB_PASSWORD

# Build configuration
build:
  preset: bun                         # bun, node, python, go
  dockerfile: Dockerfile
  context: "./app"                    # Build context directory (default: .)
  compose: docker-compose.yml         # Compose file path
  service: web                        # Service name for compose (when multiple)
  buildCommand: bun run build
  startCommand: bun run start
  installCommand: pnpm install --frozen-lockfile  # Override install (Nixpacks)
  outputDirectory: dist               # Static file output dir
  workspace: apps/web                 # Monorepo workspace target
  args:                               # Docker build args
    NODE_ENV: production
    BUILD_DATE: "2024-01-15"
  cache_invalidate:                   # Additional cache invalidation patterns
    - "content/**"
    - "public/**/*.md"

# Resources & scaling
resources:
  memory: 512M
  cpus: 0.5
  auto_reserve: true                  # default on: measured-baseline Swarm reservations at deploy (scheduling hint, never throttles); false to opt out
replicas: 2
node: manager-1                       # Pin to specific node hostname

# Placement constraints
placement:
  strategy: manager_only              # any, manager_only, spread
  constraints:
    - "node.role==manager"
    - "node.labels.storage==ssd"

# Health checks
healthcheck:
  type: http                          # http, tcp, cmd, none
  path: /health
  interval: 30s
  timeout: 10s
  retries: 3
# type: http auto-degrades to a TCP listen-check on images without curl/wget
#   (Rust/scratch/distroless) — won't false-kill a listening-but-curl-less app.
# type: none now AUTHORITATIVELY disables the healthcheck (Docker NONE sentinel)
#   and CLEARS any prior one on redeploy — use it to truly turn health checking
#   off (e.g. a curl-less image where you don't want even the TCP probe).
# Omitting the block → default TCP probe on the service port (listening = healthy).

# Update strategy
update:
  order: start-first                  # start-first (default, zero-downtime), stop-first
                                      # WARNING: stop-first + replicas:1 + no volumes = guaranteed 404
                                      # window every deploy (deploy prints a warning). Keep start-first
                                      # for stateless apps; stop-first is for single-volume stateful only.
  failure_action: rollback            # rollback, pause, continue
  parallelism: 2                      # Tasks updated at once (default: 1)
  delay: "10s"                        # Delay between updates (default: 5s)
  convergence_timeout: "10m"          # Max wait for convergence (default: 5m)
  retry:                              # Automatic retry on failure
    enabled: true
    max_attempts: 5                   # Default: 3
    initial_interval: "30s"
    max_interval: "5m"
    retryable_errors:
      - "connection refused"
      - "timeout"

# Infrastructure links
infra:
  - name: mydb
    prefix: DB                        # -> DB_HOST, DB_PORT, DB_PASSWORD

# Infrastructure auto-creation
requires:
  critical:                           # Must exist before deploy
    - postgres
    - redis: shared-redis             # With custom name
    - postgres:                       # With full config
        name: mydb
        timeout: 10m
        resources:
          memory: 2GB
          storage: 20GB
  optional:                           # Created if missing, deploy continues without
    - meilisearch

# Routes (advanced multi-domain/path routing)
routes:
  - type: http                        # http, tcp, udp, redirect
    domain: app.example.com
    port: 3000
    cloudflare: true                  # Per-route CF support (Flexible only — prefer Full (strict) + leave unset)
    auth:                             # Per-route authentication
      type: basic
      username: admin
      password: "${ADMIN_PASSWORD}"   # Resolved from servel secrets at deploy time (never hardcode)
      rules:                          # Path-based auth rules
        - paths: ["/admin/*"]
          username: admin
          password: "${ADMIN_PASSWORD}"
        - paths: ["/api/*"]
          username: api_user
          password: "${API_PASSWORD}"
    middlewares:                       # Per-route middleware override
      rate_limit:
        average: 100
  - type: http
    domain: api.example.com
    path: /v1/*                       # Path-based routing
    port: 3001
  - type: redirect                    # Domain redirect
    domain: old.example.com
    redirect_to: new.example.com
    permanent: true                   # 301 (true) or 302 (false)
  - type: tcp                         # TCP passthrough
    expose: 5432                      # External port
    port: 5432

# Middlewares (Traefik middleware configuration)
middlewares:
  rate_limit:
    average: 100
    burst: 50
  ip_allowlist:
    - "10.0.0.0/8"
    - "192.168.1.0/24"
  cors:
    origins: ["https://app.example.com"]
    methods: ["GET", "POST", "PUT", "DELETE"]
    headers: ["Content-Type", "Authorization"]
    exposed_headers: ["X-Total-Count"]
    max_age: 3600
    credentials: true
  compress:
    enabled: true
    excluded_content_types: ["image/png", "image/jpeg"]
    min_response_body_bytes: 1024
  security_headers:
    preset: strict                    # "strict" or "relaxed" presets
    # OR custom:
    sts_seconds: 31536000
    sts_include_subdomains: true
    sts_preload: false
    frame_options: "DENY"
    content_type_nosniff: true
    csp: "default-src 'self'; script-src 'self' 'unsafe-inline'"
    referrer_policy: "strict-origin-when-cross-origin"
    permissions_policy: "camera=(), microphone=()"
  headers:
    request:
      X-Custom-Header: "value"
    response:
      X-Powered-By: "Servel"
      Cache-Control: "public, max-age=3600"
  request_limit:
    max_body_size: "10MB"
  timeouts:
    read: "60s"
    write: "60s"
    idle: "90s"
    response_forwarding_flush_interval: "100ms"
  retry:
    attempts: 3
    initial_interval: "100ms"
  circuit_breaker:
    expression: "NetworkErrorRatio() > 0.5"
    check_period: "10s"
    fallback_duration: "30s"
    recovery_duration: "10s"
  sticky_sessions:
    cookie_name: "srv_session"
    secure: true
    http_only: true
    same_site: "Lax"
  redirects:                          # Path-based redirects
    - from: "/old-path"
      to: "/new-path"
      permanent: true
    - from: "/blog/(.*)"
      to: "/articles/$1"

# Authentication
auth:
  type: basic                         # basic, none
  username: admin
  password: "${AUTH_PASSWORD}"        # Resolved from servel secrets (never hardcode values)
  rules:                              # Path-based rules
    - paths: ["/admin/*"]
      username: admin
      password: "${ADMIN_PASSWORD}"   # Use: servel secrets set ADMIN_PASSWORD

# Persistent storage
persist:
  - /app/data
  - /app/uploads

# Volumes (advanced mount configuration)
volumes:
  - source: ./data
    target: /app/data
    type: bind                        # bind (default), volume
    readonly: false
    consistency: local                # "local" (default) or "strong" (DRBD replication)
    replicas: 2                       # For consistency=strong

# Tags & networking
tags: ["production", "critical"]
network: per-project                  # per-project (default), by-tag, global, or custom name

# Custom actions (run commands in containers)
actions:
  migrate:
    command: npx prisma migrate deploy
    output: migration.log
    outputs: [types.ts, schema.ts]
    env: ["MIGRATION_ENV=prod"]
    workdir: /app
    service: api                      # For multi-service compose
    user: app
  seed:
    command: npm run seed
  deploy-functions:                   # Upload files to container before running
    service: functions
    inputs:
      - source: "{{.FunctionsDir}}"
        target: /home/deno/functions
    vars:
      - name: FunctionsDir
        default: "./supabase/functions"
    command: ls /home/deno/functions/
    confirm: "Upload functions?"
  gen-types:                           # Supabase: download OpenAPI schema
    service: rest
    command: curl -s http://localhost:3000/
    output: openapi.json
  create-bucket:                       # Supabase: create storage bucket
    service: storage
    command: create-bucket
    vars:
      - name: BucketName
  list-users:                          # Supabase: list auth users
    service: auth
    command: list-users
  db-size:                             # Supabase: database size report
    service: db
    command: psql -U postgres -c "SELECT pg_size_pretty(pg_database_size(current_database()))"

# Deploy configuration
deploy:
  local: true                         # Skip registry, use local images
  no_cache: true                      # Disable build cache
  pull: true                          # Always pull latest base images
  no_cleanup: true                    # Skip cleanup of old images
  skip_build: true                    # Skip build, use pre-built output
  fast: true                          # Fast mode: skip convergence, minimal delay
  build_memory: "4g"                  # Build memory limit
  build_cpus: 2.0                     # Build CPU limit
  build_timeout: "1h"                 # Build timeout (default: 30m)
  registry: "docker.io"              # Named registry
  registry_path: "myorg/myproject"   # Override image path
  include:                            # Override exclusions
    - ".next"
    - "dist"
  exclude_patterns:                   # Extra excludes merged with built-ins + .servelignore
    - "target"
    - "web/node_modules"
  aliases:
    preview:
      ttl: "7d"
      domain: "{branch}.preview.myapp.com"
      no_index: true
      build_memory: "2g"
      memory: "512M"
      verbose: true
    quick:
      fast: true
      local: true

# Dev mode settings
dev:
  command: bun run dev
  port: 3000
  domain: myapp-dev.local             # Custom dev domain
  dockerfile: Dockerfile.dev          # Dev-specific Dockerfile
  compose: docker-compose.dev.yml     # Dev-specific compose
  service: web                        # Compose service name
  build_method: nixpacks              # Force: nixpacks, dockerfile, compose, auto
  env:
    DEBUG: "1"
    LOG_LEVEL: debug
  nixpacks:                           # Nixpacks overrides
    provider: node
    node_version: "20"
    install_cmd: "pnpm install"
    build_cmd: "pnpm build"
    start_cmd: "pnpm dev"
  sync:
    ignore:
      - "*.log"
      - "node_modules"
      - ".cache"

# Multi-environment overrides
environments:
  production:
    domain: myapp.com
    replicas: 5
    branches: ["main"]                # Auto-deploy from these branches
    auth:                             # Environment-specific auth
      type: basic
      username: admin
  staging:
    domain: staging.myapp.com
    branches: ["develop"]
    noIndex: true                     # Prevent search indexing
  preview:
    ttl: 7d                           # Auto-cleanup for branch deploys
    domain: "{branch}.preview.myapp.com"
```

## Aliases

| Command | Aliases |
|---------|---------|
| deploy | d, push |
| remove | rm, delete |
| logs | log |
| exec | x, run |
| rollback | rb |
| inspect | i, info |
| ps | ls, list |
| verify | v, check |
| doctor | dr |
| port-forward | pf |
| watch | w |
| remote | srv, server |
| infra | infrastructure |
| domains | dom |
| alerts | alert, alrt |
| connect | conn |
| tunnel | tun |
| rename | (no aliases — `mv`/`move` belong to `servel move`) |
| dashboard | dash |
| move | mv |
| add | create, new |
| stats | stat |
| access | acl |
| find | search, where |
| capacity | cap, forecast |
| registry | reg |
| redeploy | rd |
| reconcile | sync |

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Build fails | `servel logs <name>` (deploy should already use `--verbose`) |
| Port conflict | Use `--port` flag |
| Domain not working | `servel verify dns <domain>` |
| SSL issues | `servel verify ssl <domain>` |
| Container exits | Check start command and port |
| Smart mode wrong | Use `--no-smart` for full rebuild |
| Routing broken | `servel traefik test <domain>` then `servel traefik debug <name>` |
| Health check fails | `servel verify health <name>` |
| Cloudflare redirect loop | Set `cloudflare: true` in servel.yaml OR (preferred) switch CF SSL/TLS to Full (strict) and leave `cloudflare` unset |
| CF↔origin leg unencrypted | Run `servel verify cf-ssl <project>` — Flexible mode = plaintext between CF and origin. Switch CF SSL/TLS to Full (strict). |
| Stale/orphaned state | `servel reconcile --dry-run` to preview, then `servel reconcile` |
| Deploy ended with status `degraded` | Image is running but the public URL probe failed twice (once before and once after auto-respawn). Multi-domain deploys probe **every** HTTP route — a single failing route triggers respawn. Check `servel logs traefik` first, then `servel verify routing <name>`. To force another respawn cycle: `servel restart <name>`. To skip the probe on the next deploy: `--skip-probe`. To set a non-default probe path: `servel.yaml` `post_deploy.probe.path: /healthz` (or `--probe-path /healthz`). **The probe path now defaults to the declared `healthcheck.path` (http type) when set, else `/`** — so API-only services that 404 on `/` just need an HTTP `healthcheck:` block (no separate probe config). Services with NO healthcheck block (TCP fallback) that 404 on `/` still need an explicit `post_deploy.probe.path`. |
| `superseded` status in `servel ps` / multiple generations of one app | **Normal, not an error.** Each deploy is an immutable generation; Servel keeps the last `deployment_retention` (default **3**) and marks superseded (old) generations `superseded`. Exactly one generation per app is `running`. A leader-side daemon reconcile pass (`generation_reconcile_budget`, default 50) collapses historical duplicates. The live generation's dir is never GC'd, even after `rollback`. |
| `metadata not found` / app unmanageable by name though service is healthy | A running service's `servel.deployment.id` label pointed at a pruned generation dir (dangling label). Name resolution **self-heals** to the current live generation on disk automatically — `logs/exec/redeploy <name>` work without manual repair. A redeploy (or the daemon reconcile) re-points the label persistently. |
| Service returned 404 from `servel-errors` middleware on one of its domains after a successful deploy | Stale-Traefik-VIP class. **Layered healing as of 2026-05-19**: post-deploy probe and 5-min runtime watchdog now (a) **skip stopped services** — `0/0` replicas means intentionally paused, not broken (audit `routing.skip_stopped`); (b) check for **orphan** — if the swarm service vanished, mark `routing.orphan` and stop retrying (closes the 600+ noisy-loop class observed on navola); (c) force-update the **backend** first; (d) if backend respawn doesn't restore reachability, **escalate** by force-updating Traefik itself (`servel-system-traefik`). Watchdog escalation threshold: 6 consecutive failed probes; Traefik-repair cooldown: 1h per service. **Cluster-wide budget:** max 4 Traefik force-updates per rolling 24h window — once exhausted, escalation is refused (audit `routing.traefik_repair_budget_exceeded`) and only backend repairs run. Founding incident 2026-05-19: a single 0/0 deployment caused hourly Traefik rolls that wiped acme.json each time and tripped LE 5/week per-domain rate limits cluster-wide. Skip-stopped + budget cap close that class together. To force immediately: `servel restart <name>`. To disable watchdog: `servel config set routing_health_enabled=false --server`. To disable auto-repair (keep alerts): `servel config set routing_auto_repair=false --server`. Audit log: `routing.probe`, `routing.repair`, `routing.traefik_repair`, `routing.orphan`, `routing.skip_stopped`, `routing.traefik_repair_budget_exceeded`. Alert conditions: `deployment_routing_unreachable`, `deployment_routing_repaired`. **Provider hardening:** Traefik static config now sets `exposedByDefault: false` + provider constraint `Label(\`servel.managed\`,\`true\`)`. Existing servers need `servel doctor --remote <name> --fix` to migrate retroactively (`Traefik Config` + `Managed Labels` checks). |
| Service works but is **invisible** to `servel ps` / `stats` / alerts / rightsize, and/or gets no Traefik routing despite correct `traefik.*` labels | **B.5 — missing `servel.managed=true`.** Traefik's swarm provider is constrained to ``Label(`servel.managed`,`true`)``, and every listing surface filters `--filter label=servel.managed=true`, so an unlabeled `servel-*` service is simultaneously unrouted and unlistable. Confirm with `docker service inspect <svc> --format '{{index .Spec.Labels "servel.managed"}}'` (empty = drift). **Two causes, check which:** (a) **Swarm reverted it** — `docker service update --label-add` is a spec-mutating rolling update; if the task can't converge inside `UpdateConfig.Monitor` then `FailureAction=rollback` restores the old spec and drops the label. Proof: `docker service inspect <svc> --format '{{json .UpdateStatus}}'` shows `rollback_completed`, and `.UpdatedAt` matches its `CompletedAt`. Stateful services (Postgres) are the usual victims. (b) **Created outside the chokepoint** — spot it by `.Spec.UpdateConfig` carrying Docker defaults (`pause`/`5s`) instead of servel's `rollback`/`3m5s`/`start-first`. **Do NOT blind-fix by re-running the label-add**: on cause (a) it restarts the DB and silently reverts again. Make the service healthy enough to converge first, then `docker service update --label-add servel.managed=true <svc>` and re-verify. Detected on every resilience tick and in `servel infra check` / `servel doctor`. Founding case: navola 2026-08-05, 4 Supabase `-db` services + 1 deployment; the 2026-05-20 one-shot backfill had reported `failed=0` because it trusted the update's exit code. |
| Node disk fills with no deployment to blame, or an unrelated infra on the same node reports "low free space" (poste.io "3% free" on a 1.6 GB volume) | **G.2 — ClickHouse system-log bloat.** ClickHouse keeps its OWN telemetry tables forever by default. On noras-openreplay (2026-08-08, second occurrence after ~110 GB in June 2026) `system.trace_log` was 139.79 GiB / 5.7B rows and `system.text_log` 22.27 GiB — 239 GB of a 391 GB disk — while `experimental.*` + `product_analytics.*` held a few MB. ≈2 GB/day. Confirm: `servel exec @<infra> --service clickhouse -- clickhouse-client -q "SELECT table, formatReadableSize(sum(bytes_on_disk)) FROM system.parts WHERE database='system' AND active GROUP BY table ORDER BY sum(bytes_on_disk) DESC"`. **Fix an EXISTING instance with `ALTER TABLE system.<t> MODIFY TTL event_date + INTERVAL 3 DAY`** (trace_log/text_log; 7 DAY for the metric/query/part logs) — TTL lives in table metadata inside the data volume, so it survives restart and recreate. Dropping >50 GB needs `--max_table_size_to_drop=0 --max_partition_size_to_drop=0`. **NEVER bolt the hub template's config.d file onto a running instance**: ClickHouse treats a changed system-table definition as a new table, renames the old one to `<name>_0`, and the orphan keeps unlimited retention forever (that is what the dead `metric_log_0` / `query_log_1` tables were). Instances created from openreplay/v1.yaml or databases/clickhouse/v1.yaml on or after 2026-08-08 are born bounded. Servel detects but NEVER truncates — alert-only by design. |
| Postgres PANIC `could not locate a valid checkpoint record`, truncated WAL, or a DB that corrupts after any `docker service update` | **B.6 — stateful safety-floor drift.** Single-replica stateful services must run `--update-order stop-first`, `--update-failure-action pause`, `--update-monitor >=1m0s`, `--stop-grace-period >=2m0s`. `start-first` races two postmasters for one data directory; `grace=10s` (the compose default) SIGKILLs mid-checkpoint and truncates the WAL; `rollback` rolls again while the first roll may still be flushing. **These floors are applied at CREATE/DEPLOY time only** (`infra.applyStatefulUpdateOrder`, landed 2026-05-09) — anything older, or never redeployed since, silently keeps Swarm defaults. A 2026-08-06 sweep found 27 such services across navola + KN. Check with `docker service inspect <svc> --format 'order={{.Spec.UpdateConfig.Order}} failure={{.Spec.UpdateConfig.FailureAction}} monitor={{.Spec.UpdateConfig.Monitor}} grace={{.Spec.TaskTemplate.ContainerSpec.StopGracePeriod}}'`. **Repairing RESTARTS the database, under the current unsafe grace period** — back up first (`servel infra backup <name>`), do it per service, never fleet-wide in one sweep. Reported by `servel infra check <name>` or `servel infra check --all-nodes` — a BARE `servel infra check` skips the resilience scan entirely (gated on `name != "" \|\| allNodes`), so B-class findings silently will not appear unless you scope it. Same gate applies to B.1-B.5. |
| Volume orphaned | `servel volumes --orphaned` to find, `servel volumes inspect <name>` for details |
| `name must be valid as a DNS name component` / `not a valid DNS label` | Name has dots/uppercase/underscores. Use DNS-safe name (`my-app-prod`) and pass domain via `--domain`, not `--name`. |
| `service db not found: no services found` after deploy | Hook ran before ServiceIDs persisted. Re-run: `servel infra run-hooks <name> --init --force`. |
| `<svc>: no running tasks` on Supabase analytics/realtime/supavisor | Memory limit too low. Bump per-service: `servel infra update <name> --memory <svc>:1G`. NEVER drop Erlang services below 512MB. |
| Supabase `/realtime/v1/*` → 503 `{"message":"name resolution failed"}` (REST/Auth fine) | Kong's bundled kong.yml hardcodes the realtime upstream `realtime-dev.supabase-realtime:4000` (a Compose `container_name` trick). **Docker Swarm overlay DNS does NOT resolve aliases containing dots** (proven: NXDOMAIN even when the alias is registered on the service spec), so `network_alias_overrides` is a silent no-op — the only fix is `source_substitutions` rewriting kong.yml → bare `realtime`. Template (v1+v2) carries it; if a running stack predates distribution, live-fix in the kong container: `sed -i 's/realtime-dev.supabase-realtime/realtime/g' /home/kong/kong.yml && kong reload` (graceful) and patch the bind source `/var/servel/infrastructure/<infra>/volumes/api/kong.yml` so it survives a kong restart. NB: kong key-auth returns 401 before proxying, so reproduce the 503 with `-H "apikey: <ANON>"`. |
| `access denied: command requires X permission` | Either change role, or extend scope: `servel access scope add <user> --server <s> --permissions X`. |
| `--var <Name>` rejected / value not picked up | `--var` uses Go-identifier names (`^[A-Za-z_][A-Za-z0-9_]*$`). List canonical names with `servel infra vars <type>`. |

**Diagnostic commands:**
```bash
servel doctor                         # System check
servel verify health <name>           # Health check
servel verify dns <domain>            # DNS check
servel verify ssl <domain>            # SSL check
servel traefik status                 # Routing status
servel traefik debug <name>           # Debug specific routing
servel logs <name>                    # View logs
servel inspect <name>                 # Deployment details
servel reconcile --dry-run            # Find state mismatches
servel audit list --severity high     # Recent high-severity events
```

## Project Context Detection

For advanced cases, check these local files to understand the deployment context before running servel commands.

### .servel/ Directory (Project State)

Located at `<project-root>/.servel/`. Created automatically after first deploy. Contains deployment state that tells you **which server** this project deploys to and its current configuration.

**Files:**
- `.servel/state.json` -- Production environment state
- `.servel/state.<env>.json` -- Other environment states (e.g., `state.staging.json`)

**State file structure:**
```json
{
  "version": 1,
  "server": "KN",
  "server_fingerprint": "uuid-...",
  "deployment_id": "myapp",
  "environment": "production",
  "build_type": "dockerfile",
  "project_name": "myapp",
  "domain": "myapp.example.com",
  "install_command": "",
  "build_command": "",
  "start_command": "",
  "port": "",
  "runtime": "",
  "tags": [],
  "network_mode": "",
  "network_name": ""
}
```

**How to use:**
1. Read `.servel/state.json` to determine the target server and deployment name
2. Check for multiple environments with `.servel/state.*.json`
3. The `server` field maps to a remote in `~/.servel/config.yaml`
4. If `.servel/` doesn't exist, the project hasn't been deployed yet

**List environments:**
```bash
ls .servel/state*.json  # See all deployed environments
```

### servel.yaml (Project Configuration)

Located at `<project-root>/servel.yaml`. Defines how the project should be built and deployed. This is the **declarative config** -- checked into version control. See the [Complete Reference](#complete-reference) section above for all available fields.

### Putting It Together

When working with a servel-managed project:
1. **Check `.servel/state.json`** -> Know which server, deployment name, and environment
2. **Check `servel.yaml`** -> Know build config, domains, infra links, resources
3. **No `.servel/` dir** -> Project not yet deployed (use `servel deploy` first)
4. **No `servel.yaml`** -> Auto-detected project (Dockerfile/compose/preset)

Example workflow:
```
# Understand current project deployment
cat .servel/state.json          # -> server: "KN", project_name: "myapp"
cat servel.yaml                 # -> domain, infra links, build config

# Now you know: myapp is deployed on KN server
servel logs myapp               # View logs
servel inspect myapp            # Full details
```

## Reference Files

- [Template Building Guide](references/TEMPLATES.md) - Create custom infrastructure types
- [Access Control](references/ACCESS.md) - Roles, permissions, scope composition, SSH gate path tiers, refusals
- [Analytics Reference](references/ANALYTICS.md) - Visitor analytics, privacy model, agent heuristics
- [Autonomous Remediation](references/AUTONOMOUS_REMEDIATION.md) - 2026-05-07 daemon remediations: rebalance auto-strategy, right-size advisor, load-aware placement, Reservations-only capacity, constraint drift auto-repair, stateful auto-rebalance (Tier 1 advisory always-on / Tier 2 opt-in)
- [Cross-Node Migration](references/MIGRATION.md) - `servel move`, strategies, pre-copy loop, evacuation, validator, autonomous Tier 2 invocation
- [Distributed Storage](references/STORAGE.md) - LINSTOR/DRBD substrate, `--replicated` volumes, replicated migration fast path (Phase 4)
- [Dashboard](references/DASHBOARD.md) - `servel dashboard` — one-screen view of every optional subsystem
- [Post-Deploy Probe](references/POST_DEPLOY_PROBE.md) - Routability probe + auto-respawn that runs after deploy and rollback (closes the stale-Traefik-VIP class of outages)
- [Dev Link Env](references/DEV_LINK_ENV.md) - `servel dev --link-env` / `--link-infra` — pull prod env (and optionally secrets) into dev, with auto-tunneling, ACL, audit, and cache lifecycle
- Full docs: https://servel.dev/docs
- Infrastructure Hub: https://hub.servel.dev
