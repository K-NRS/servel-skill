---
name: servel
description: Self-hosted deployment platform specialist. Use when deploying applications, managing infrastructure, or working with Docker Swarm deployments. Enforces dogfooding policy - ALWAYS use servel commands instead of direct Docker operations. Triggers for deploy, infrastructure, postgres, redis, supabase, backup, restore, logs, exec, shell, container, docker, secrets, SSL, domains, dev mode, CI/CD, alerts, traefik, routing, server management, capacity, rebalance, registry, node management, access control, volumes, audit, bastion, tunnel, or port-forward.
---

# Servel

Deploy applications and infrastructure to Docker Swarm with Vercel-like simplicity. Auto-detects project type, provisions SSL, zero-downtime rolling updates.

## Always Use Servel Commands

**Never use raw Docker or SSH commands for operations servel handles.** Servel wraps these with proper state tracking, routing, and safety.

| Instead of... | Use servel |
|---|---|
| `ssh user@server` | `servel ssh <server>` |
| `docker exec` | `servel exec <name> sh` or `servel exec <name> -- cmd` |
| `docker logs` | `servel logs <name> -f` |
| `docker service create/update/rm` | `servel deploy`, `servel rm` |
| `docker stack/compose` | `servel deploy` (compose auto-detected) |
| `docker ps` / `docker service ls` | `servel ps`, `servel node ps` |
| Manual DB setup | `servel add postgres --name mydb` |
| `rsync` / `scp` to server | `servel dev` |
| Manual backups / `pg_dump` | `servel infra backup <name>` |
| `docker exec psql < file.sql` | `servel infra sql @mydb file.sql` |
| Raw `curl` health checks | `servel verify health <name>` |

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
|   +- Link to app? -> servel link myapp --infra db
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
+- Volumes -> servel volumes | servel volumes inspect <name>
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

## Quick Reference

### Deploy

**IMPORTANT: Always use `--verbose` flag when deploying.** It shows full build output, making it much easier to diagnose issues and understand what's happening. Without it, build output is summarized and critical context is lost.

```bash
servel deploy --verbose               # Auto-detect & deploy (ALWAYS use --verbose)
servel deploy --verbose --preview --ttl 24h     # Preview with cleanup
servel deploy --verbose --link-infra db,redis   # Link infrastructure
servel deploy --verbose --dry-run               # Show plan only
servel deploy --verbose --no-registry           # Skip registry (single-node)
servel deploy --verbose --env staging           # Multi-environment
servel deploy --verbose --rebuild               # Force rebuild, skip cache
servel deploy --verbose --new                   # Force new deployment with unique subdomain
servel deploy --verbose --dashboard             # Real-time TUI dashboard during deploy
servel deploy --verbose --save                  # Persist flags to servel.yaml
servel deploy --memory 1g --cpu 0.5   # Resource limits
servel deploy --quiet                 # Minimal output (only final result)
servel ps                             # List deployments
servel ps --all-servers               # List across all servers
servel ps --tree                      # Tree view with dependencies
servel logs <name> -f                 # Follow logs
servel watch <name>                   # Watch deploy progress (TUI)
servel rm <name>                      # Remove
servel rollback <name>                # Rollback version
servel promote <src> <tgt>            # Promote deployment (env + domains)
servel promote src tgt --swap         # Bidirectional domain swap
servel promote src tgt --dry-run      # Preview promotion plan
servel promote src tgt --merge-env    # Merge env vars (vs replace)
servel promote src tgt --rebuild      # Rebuild after (NEXT_PUBLIC_*)
servel promote src tgt --cleanup-source  # Remove source after
servel scale <name> 3                 # Scale replicas
servel scale <name> 0                 # Scale to 0 (same as stop)
servel restart <name>                 # Restart deployment
servel stop <name>                    # Stop deployment (scales to 0)
servel start <name>                   # Start stopped deployment
servel rename <old> <new>             # Rename deployment
servel exec <name> sh                 # Shell into container
servel exec <name> -- cmd args        # Run command in container
servel inspect <name>                 # Detailed deployment info
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
- `--name, -n` -- Deployment name
- `--domain, -d` -- Domain for routing
- `--preview` -- Preview environment
- `--ttl` -- Preview lifetime (1h, 6h, 1d, 7d, 2w)
- `--link-infra` -- Link infrastructure (comma-separated)
- `--no-registry` -- Skip registry push
- `--rebuild` -- Force rebuild
- `--no-smart` -- Disable smart detection
- `--verbose` -- Show full build output **(always recommended)**
- `--quiet, -q` -- Minimal output (only final result)
- `--dashboard` -- Real-time TUI dashboard
- `--env` -- Target environment
- `--build-on <node>` -- Build on specific node
- `--local-build` -- Build locally, push to registry
- `--new` -- Force new deployment with unique subdomain
- `--converge-timeout <duration>` -- Convergence wait time (default: 5m)
- `--force-server` -- Suppress server mismatch warnings
- `--author <name>` -- Override deployment author
- `--save` -- Persist deploy flags to servel.yaml
- `--skip-scan` -- Skip vulnerability scanning
- `--scan-block <severity>` -- Block on severity (critical, high, medium, low)

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
| Analytics | plausible, umami, openreplay |
| Monitoring | prometheus, grafana, loki, promtail, uptimekuma, gatus, peekaping |
| Realtime | livekit, livekit-egress, hocuspocus, y-sweet |
| Storage | minio |
| Email | posteio |
| CI | woodpecker, woodpecker-agent |
| Blockchain | bitcoin, ipfs, lnd |

```bash
servel add postgres --name db         # Create
servel add redis,postgres --prefix app # Bundle multiple
servel add postgres --name db --ha    # High-availability
servel add supabase --name supa       # Full platform stack
servel add chatwoot --var Domain=chat.example.com  # Auto-init on first deploy
servel infra status                   # Health check all
servel infra vars db                  # View env vars
servel infra update db --memory 2g    # Update config (memory, cpu, domain, node, env)
servel infra upgrade db --image postgres:16  # Safely upgrade image (auto-backup + health check)
servel infra upgrade supa --service auth --image supabase/gotrue:v2.186.0  # Upgrade specific service
servel infra domains add db --domain db.example.com  # Add domain alias
servel infra domains remove db --domain db.example.com
servel infra labels db --add key=val  # View/modify Docker labels
servel infra run db                  # List available actions
servel infra run db psql             # Run action (e.g. interactive shell)
servel infra run mysupabase deploy-functions ./supabase/functions  # Upload files + run
servel infra run db schema --dry-run # Preview action
servel infra run-hooks db             # Execute lifecycle hooks
servel infra run-hooks db --init      # Run post-init hooks
servel infra archives                 # Manage archived credentials
servel logs @db -f                    # Follow infra logs (@ prefix)
servel logs @chatwoot --service rails -f  # Multi-service infra logs
servel infra backup db                # Backup
servel infra restore db backup.sql.gz # Restore
servel infra rotate db                # Rotate credentials
servel infra restart db --force       # Force restart
servel infra start db                 # Start
servel infra stop db                  # Stop (scales to 0)
servel infra stop db --service vector # Stop specific sub-service (multi-container infra)
servel infra rename old new           # Rename
servel infra rm db                    # Remove
servel infra check                    # Diagnose all (orphaned constraints, port conflicts, stuck services)
servel infra check mydb               # Check specific infra
servel infra sql @mydb schema.sql     # Run SQL file against database
servel infra sql @mydb ./migrations   # Run all .sql files in directory (alphabetical order)
servel infra sql @mydb "SELECT 1"     # Run inline SQL
servel infra sql @mydb schema.sql --dry-run  # Preview without executing
servel infra sql @supabase migration.sql --service db  # Supabase (targets db service)
servel infra sql @mydb ./migrations --track      # Tracked migrations (skip applied)
servel infra sql @mydb ./migrations --track --status  # Show migration status
servel infra sql @mydb ./migrations --track --force   # Re-apply changed migrations
servel link myapp --infra db          # Link -> injects DATABASE_URL
servel unlink myapp --infra db        # Unlink
servel deps myapp                     # Show dependencies
servel connect db                     # Quick connect to infra
```

**Database Migrations:** Use `servel infra sql` — this is the canonical way to run SQL migrations against any database infrastructure. Supports PostgreSQL, MySQL, MariaDB, CockroachDB, and Supabase. When given a directory, runs all `.sql` files in alphabetical order (prefix with `001_`, `002_`, etc.). For Supabase, uses `supabase_admin` superuser on the `-db` service. For ORM migrations (Prisma, Drizzle), use `servel infra run <name> migrate` if a custom action is defined, or run via `servel exec`.

**Linking injects:** DATABASE_URL, REDIS_URL, MONGODB_URI, etc. based on infrastructure type.

**Lifecycle hooks:** Some templates auto-run setup commands on first deploy (e.g., Chatwoot runs `db:chatwoot_prepare`).

**Node pinning:**
- `--node hostname` -- By hostname
- `--alias db-node` -- By alias
- `--label storage=ssd` -- By node label

### Server Management (remote)

`servel server` is aliased to `servel remote`. Both work interchangeably.

```bash
servel ssh <server>                   # SSH into server
servel remote status                  # Cluster health (CPU, memory, disk)
servel remote add <name> user@host    # Add server
servel remote list                    # List servers
servel remote use <name>              # Switch default server
servel remote remove <name>           # Remove server
servel remote provision               # Automated setup
servel remote provision --repair      # Repair corrupted keys/services
servel remote domain set example.com  # Set primary domain
servel remote keys add <name> --key-file pubkey.pub # Add deploy key
servel capacity                       # Capacity forecast + recommendations
servel capacity --json                # JSON output
servel df                             # Disk usage
servel df --volumes                   # Volume usage by category
servel df --nodes                     # Per-node usage
servel doctor                         # Diagnose issues
servel doctor --remote KN             # Remote server diagnostics
servel cleanup                        # Remove expired environments
servel cleanup --force                # No confirmation
servel prune                          # Remove dangling images/containers
servel prune --all                    # Remove unused images/networks/cache
servel prune --all --volumes          # DATA LOSS: removes unused volumes
```

### Node Management

```bash
servel node ls                        # List swarm nodes
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
servel node balance --dry-run         # Preview rebalance plan
servel node balance                   # Execute cluster rebalance
servel node schedule <name> --at "2026-02-01 03:00"  # One-time drain
servel node schedule <name> --in 2h   # Relative time drain
servel node schedule <name> --cron "0 3 * * *"       # Recurring drain
servel node schedule <name> --cron "0 3 * * *" --reactivate "0 7 * * *"  # Drain + reactivate
servel node schedule ls               # List scheduled actions
servel node schedule cancel <name>    # Cancel schedule
servel node install --all             # Install servel CLI on all workers
servel node upgrade --all             # Upgrade servel CLI on all nodes
servel node alias <hostname> <alias>  # Set friendly alias
servel node label <hostname> key=val  # Add/remove node labels
```

### Secrets

```bash
servel secrets set API_KEY            # Set (interactive prompt, never pass values inline)
servel secrets list                   # List keys
servel secrets get API_KEY            # Get value (output is masked by default)
servel secrets rm API_KEY             # Remove
servel secrets rotate API_KEY         # Rotate
servel secrets backup                 # Backup all secrets
servel secrets scan                   # Scan for exposed secrets
servel deploy --migrate-secrets       # Auto-detect *_KEY, *_SECRET, *_PASSWORD
```

### Domains & Routing

```bash
servel domains add myapp app.com      # Add domain (auto-SSL)
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

### Verification

```bash
servel verify <name>                  # Full verification
servel verify config                  # Verify configuration
servel verify health <name>           # Check service health
servel verify ssl <domain>            # Check SSL certificates
servel verify dns <domain>            # Check DNS configuration
servel verify routing <name>          # Check Traefik routing
servel verify dependencies <name>     # Check dependencies
servel verify resources               # Check resource availability
```

### Dev Mode

```bash
servel dev                            # Start dev session
servel dev --team                     # Bidirectional sync (collaboration)
servel dev --port 3001                # Custom port
servel dev --domain staging.app.com   # Custom domain
servel dev --no-sync                  # One-time upload only
servel dev --conflict-policy newer-wins # Sync conflict resolution
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

### Environment Variables

```bash
servel env set <name> KEY=VALUE       # Set env var (no rebuild, restarts service)
servel env vars <name>                # Show env vars (secrets masked)
servel env list                       # List environments
servel config show <name>             # Show deployment config
servel config sync <name>             # Sync config to servel.yaml
servel config sync --dry-run          # Preview sync
servel set-env-file .env              # Set env_file in servel.yaml (.local rejected)
```

**Note:** `servel deploy` auto-reads `.env` + `.env.local` from project dir and injects as Docker env vars. This is NOT persistent — vars are re-sent each deploy. For persistent encrypted storage, use `servel secrets`. See [Environment Variables & Secrets](#environment-variables--secrets) workflow for full details.

### Alerts

```bash
servel alerts setup                   # Interactive wizard
servel alerts add telegram            # Add Telegram channel
servel alerts add slack               # Add Slack channel
servel alerts add discord             # Add Discord channel
servel alerts add webhook             # Add webhook
servel alerts test                    # Test notifications
servel alerts status                  # Show alert status
servel alerts history                 # View alert history
servel alerts pause 2h                # Maintenance mode (pause alerts)
```

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
servel access role                    # Role management
servel access setup                   # Initialize access control on server
servel access setup --rotate-join-key # Rotate join key
servel access invite --role deployer   # Generate invite token
servel access invite ls               # List pending invites
servel access invite revoke <id>      # Revoke invite
servel access invite rotate <id>      # Rotate token (new token, old revoked)
servel access invite clean            # Remove expired/used invites
servel access join <token>            # Join server (idempotent -- safe for CI reruns)
```

### Registry

```bash
servel registry                       # Browse registry repos
servel registry tags myapp            # List tags with sizes
servel registry rm myapp:v1.0         # Delete tag (confirmation required)
servel registry info                  # Registry info + capabilities
servel registry du                    # Per-repo disk usage breakdown
```

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
servel upgrade                        # Self-upgrade CLI
servel upgrade-servers                # Upgrade servel on all servers
servel upgrade-servers --remote KN    # Upgrade specific server
servel tag <name> <tag>               # Add tags to deployment
servel untag <name> <tag>             # Remove tags
servel reconcile                      # Discover/fix unlabeled services and missing state
servel reconcile --dry-run            # Preview reconciliation
servel reconcile --deployments        # Only deployments
servel reconcile --infra              # Only infrastructure
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

**Priority (highest wins):** servel secrets > servel.yaml `env:` > `env_file:` > `.env`/`.env.local`

**Important distinctions:**
- `env_file:` in servel.yaml rejects `.local` files (dev-only convention)
- `servel set-env-file .env` — convenience command to set `env_file` in servel.yaml
- `servel env set myapp KEY=VALUE` — update env var on running deployment without rebuild
- NEXT_PUBLIC_* vars are kept in build-time env AND runtime (Next.js needs them at build time)

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

# Cloudflare proxy support
cloudflare: true                      # Skip HTTPS redirect (prevents redirect loops with Flexible SSL)
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

# Update strategy
update:
  order: start-first                  # start-first, stop-first
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
    cloudflare: true                  # Per-route Cloudflare proxy support
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
| rename | mv, move |
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
| Cloudflare redirect loop | Set `cloudflare: true` in servel.yaml OR set Cloudflare SSL to Full (strict) |
| Stale/orphaned state | `servel reconcile --dry-run` to preview, then `servel reconcile` |
| Volume orphaned | `servel volumes --orphaned` to find, `servel volumes inspect <name>` for details |

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
- Full docs: https://servel.dev/docs
- Infrastructure Hub: https://hub.servel.dev
