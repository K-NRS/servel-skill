# Access Control

How servel users, roles, scopes, and the SSH gate compose. Use when granting
access, debugging "permission denied" errors, or auditing who can do what.

## Roles

System roles, ordered by power. Each row inherits everything above plus
adds its own permissions.

| Role | Description | Permissions added |
|------|-------------|-------------------|
| `viewer` | Read-only | `logs`, `status`, `metrics` |
| `developer` | + project visibility | `config-read`, `port-forward` |
| `deployer` | + ship code | `deploy`, `update`, `restart`, `scale` |
| `operator` | + day-2 ops | `shell`, `backup`, `restore`, `infra-create`, `deploy-delete` |
| `admin` | + platform config | `secrets-read`, `secrets-write`, `config-write`, `infra-delete` |
| `super_admin` | + run the platform | `shell-root`, `user-manage` |

Custom roles can be defined; they validate against the same permission
catalog. System roles cannot be deleted.

## Permissions catalog

Granular permission tokens. Used as `--permissions <list>` on `servel access
scope add` and as the `Permissions` field on roles.

| Permission | Grants |
|------------|--------|
| `logs` | View service logs |
| `status` | View service status, listings |
| `metrics` | View metrics and monitoring |
| `config-read` | Read configuration (secrets redacted) |
| `port-forward` | Port-forward to services |
| `secrets-read` | View decrypted secret values |
| `shell` | Interactive shell into containers |
| `shell-root` | Root shell (host) — dangerous |
| `restart` | Restart services |
| `scale` | Scale replicas up/down |
| `update` | Generic updates (prune, abort, cache) |
| `deploy` | `servel deploy` / `redeploy` |
| `backup` | Create backups |
| `restore` | Restore from backup |
| `deploy-delete` | `servel rm <app>` |
| `infra-delete` | `servel rm @name` |
| `infra-create` | `servel add`, `servel link` |
| `config-write` | Modify env, domains, routes |
| `secrets-write` | Create/modify/delete secrets |
| `user-manage` | Manage access users / scopes / invites |

Legacy aliases (still validate, mapped server-side via `PermissionFallbacks`):
`delete` → `deploy-delete`/`infra-delete`, `secrets` → `secrets-read`/`secrets-write`,
`infra-manage` → `infra-create`/`infra-delete`. Prefer the granular names.

Authoritative source: `internal/types/access.go` (constants `PermXxx`). The
website auto-generates a permissions doc from the same file via
`tools/gen-permissions-doc`.

## Scope model

A **scope** is a per-server grant attached to a user. It composes with the
user's role.

```
effective_perms(user, server) = role.perms ∪ scope.permissions
effective_resources(user, server) = scope.{infra_names, deployment_names, environments}
```

Two key rules:

1. **Permissions are ADDITIVE.** A scope can add perms to a role on a single
   server. It cannot subtract.
2. **Resource lists are RESTRICTIVE.** When `infra_names`/`deployment_names`/
   `environments` are non-empty, the user is limited to those resources on
   that server. Empty list = no restriction (all visible to the role).
   `allow_all: true` overrides any restriction.

Typical use cases:

| Goal | Command |
|------|---------|
| Let a `developer` deploy on one server | `servel access scope add bob --server KN --permissions deploy` |
| Restrict to specific infra | `servel access scope add bob --server KN --infra mydb,redis` |
| Restrict to a single env | `servel access scope add bob --server KN --env staging` |
| Combine: deploy + restart, only on `app` deployment | `servel access scope add bob --server KN --permissions deploy,restart --deployment app` |
| Inspect effective scope | `servel access scope show bob --json` |

Resource restrictions are enforced on the command's target:

- `deploy` is scoped by app name (`--name`) and environment (`--env` / `--preview`). The server re-checks the name and environment it resolves from `servel.yaml`, so a scoped deployer cannot deploy another app.
- A name-restricted user is denied state-changing commands that name no target (an explicit target is required): `abort`, `connect`, `tunnel start`, `prune`, `node balance`, and `deploy` without `--name`.
- `servel pf` works only for the user's own deployments/infra. Port-forward to system services (`~traefik`, registry) is denied for name-restricted users.
- `~system` targets are read-only for name-restricted users: `logs ~traefik` and `inspect ~traefik` are OK; `exec`, `restart`, `scale` and `rm` on `~x` are denied.
- EVERY resource a command names is checked: both apps of `env copy`/`secrets copy`/`rename`, the app (2nd arg) of `domains remove <domain> <app>`, each infra of `link a,b` (bare names = infra), the name in `add <type> <name>`. Words a command doesn't take as a resource are refused: `servel infra link app db` is just `servel infra` (a listing; there is no `infra link`), so it is denied. To link, run `servel link <infra>` from the app dir.
- Raw commands the client sends are mapped to resources and denied when they touch another resource or can't be mapped:
  - `docker service|container …` operands, through the service's `servel.infra.name` label or deployment service name;
  - `/var/servel/{deployments/<id>,infrastructure/<name>,operation-logs/<name>,tmp/<app>-<pid>,deploys/<app>-<pid>}`;
  - `kill`/`pkill` of another app's deploy job;
  - `xargs` fed by anything but `docker service ps <own svc> -q`;
  - volume, image and system prunes;
  - `--network-add` and `traefik.*`/`servel.*` labels (except router `…middlewares` and `servel.managed`).
- Also checked:
  - `add --name` (it wins over the positional);
  - `exec --target`;
  - the linked apps of `--with-linked`;
  - `remote subdomain-claim --deployment`;
  - deploy `--linked-infra-network servel-infra-<x>-network` (that is `@x`).
- After `infra <sub>`, a bare name is an INFRA (`infra stop app` checks `@app`).
- Restricted users are refused cluster secrets at any tier (`secrets/master.key`, `letsencrypt/acme.json`, `deploy-keys/`, `backup-password`).
- State-changing commands tiered `config-read` (`remote cleanup`, `logs prune`, `audit rotate`, …) need a target.
- A router `…middlewares` label must name a router the user's own service already defines.
- Only dangling `docker image prune` is allowed.
- Environment-only scopes are checked on raw commands too, and refused what can't be mapped.
- Listing every deployment's `meta.json` (ps-level) stays OK. Walking every `spec.json` does not.
- Consequence: views that inspect ALL services (`domains list`, `verify`, `dev`, the per-node CPU line of `remote status`) fail for name-restricted users. Service specs carry every app's secrets as env.
- Unrestricted scopes (no names) see no change.
- If `scopes.yaml` can't be read, access is DENIED (it used to lift all restrictions).

Scopes are stored at `/var/servel/access/scopes.yaml` keyed by username.

## SSH gate path classification

Every command issued over the access gate is classified by its path target,
then matched against the user's effective permissions.

| Path | Required permission |
|------|---------------------|
| `/var/servel/secrets/**` | `secrets-read` |
| `/var/servel/access/audit.log` | `user-manage` |
| `/var/servel/access/policies/**` | `user-manage` |
| `/var/servel/access/join-keys/**` | `user-manage` |
| `/var/servel/access/scopes.yaml` | `user-manage` |
| `/var/servel/access/invites/**` | `user-manage` |
| `/var/servel/access/users.yaml` | `status` (legitimate read by `servel remote status`) |
| `/var/servel/access/roles.yaml` | `status` (same reason) |
| `/var/servel/access/**` (other) | `user-manage` (default-closed) |
| `/var/servel/config.yaml` | `config-read` |
| `/var/servel/traefik/**`, `routes/**`, `middlewares/**` | `config-read` |
| `/var/servel/**` (deployments, infra, metrics, archives, ci, cluster, tmp, ports) | `status` |
| system-wide `/proc` files (`meminfo`, `loadavg`, `drbd`, ...), `/proc/sys/`, `/proc/pressure/`, `/sys/` | `status` |
| `/etc/docker/daemon.json`, `/etc/os-release`, `/etc/ssh/ssh_host_{ecdsa,ed25519,rsa}_key.pub`, `*/.ssh/authorized_keys` | `status` |
| `/etc/ssh/sshd_config`, `/etc/fstab`, `/etc/exports`, `~/.ssh/id_{ed25519,rsa}.pub` | `config-read` |
| any other `/proc/...` (`/proc/<pid>/root`, `/proc/self/...`, `environ`) | `shell-root` |
| anything else (`/etc/shadow`, `ssh_host_*_key`, `*/.ssh/id_*`, `/var/lib/docker/**`, relative paths) | `secrets-read` |

Unknown paths fail closed (since 2026-10-11; before, they read at
`config-read`, so a `developer` key could `cat /etc/shadow` or
`/proc/self/root/var/servel/secrets/master.key`). Host files are allowlisted
by exact path only. `ls`, `stat` and a `find` walk see names and sizes, not
contents, so on an unknown path they need `config-read`; `find -exec cat {}`
still needs `secrets-read`. `du` is `status` on any path. A gated script
that reads a host file outside the table needs a role with `secrets-read`
(admin) — or add the exact path to the gate's allowlist in
`internal/access/gate/gate.go` `classifyPathSensitivity`.

## Refusals (gate-level)

The gate refuses the entire command before classification when it sees:

- **Command substitution**: `$(...)` or backticks — unless the substitution is
  one of a small known-safe allowlist. Error: `command substitution (\` or $()) is not permitted`.
- **Path traversal**: any `/..` segment. Error: `path traversal ('..') is not permitted`.
- **Unmatched quotes**: parser bails before classification.
- **Stdout redirect on a read path**: `cat /var/servel/x > /tmp/y` reclassifies
  as `config-write` (or `deploy` for scratch paths). Stderr-only (`2>/dev/null`)
  is fine.

Pipes (`|`) are allowed; each segment is classified independently and the
required permission is the union.

## Gated deploys

A `deploy`-only key cannot run scripts. The server owns the background deploy job: the client sends `servel deploy ... --background-job <id>` (hidden), and `--detach` and SSH-drop survival still work. Server feature `server-owned-deploy-job`; new CLI + old server falls back automatically. An old CLI on a gated key against an upgraded server fails with `this server no longer runs scripts uploaded by the client; upgrade your servel CLI` — run `servel upgrade`. Root keys are unaffected. Credential files from gated callers must be client-written under `/var/servel/tmp/.servel-cred-*/`. Reading one back through the gate needs `secrets-read` (all roles); writing stays `deploy`.

For every gated role (sessions run as root):
- `docker service create|update` options are vetted like `docker run`:
  - a bind mount outside `/var/servel` or from deploy scratch, `volume-driver`/`volume-opt`, `--network host`, `--cap-add`, `--env-file`, or any unknown option or mount field needs `shell-root`;
  - a `/var/servel` bind needs `config-write` plus that region's write tier;
  - `--publish` needs `config-write`;
  - `--secret`/`--config` need `secrets-read`.
  - `--mount` is parsed as docker does (CSV, case-insensitive keys).
- `pkill` accepts only `servel deploy.*<job>` and `servel ci server start`. `kill`/`pkill` may signal only servel deploy jobs (`kill -TERM -1` is denied).
- A group whose leader is gone cannot be signalled (a pid file is writable at deploy tier, so it proves nothing).
- Pids above 2^22 are refused, because `kill` truncates them to `pid_t`.

## Audit log

Every gate decision (allow + deny) is appended to:

```
/var/servel/access/audit.log
```

Best-effort — gate does not fail on log write errors. Read with:

```bash
servel audit list                        # recent entries
servel audit list --severity high        # only denials / dangerous ops
servel audit show <id>                   # detail one entry
```

The file itself requires `user-manage` to read directly via `cat`.

## Common errors → fix

| Error | Cause | Fix |
|-------|-------|-----|
| `access denied: command requires deploy permission` | Role lacks `deploy` | `servel access scope add <user> --server <s> --permissions deploy` |
| `access denied: requires secrets-read` | User read `/var/servel/secrets/*` without perm | Grant `secrets-read` via role change or scope add |
| `command substitution ... not permitted` | Shell invoked `$(...)` or backticks | Use literal arguments; the gate cannot reason about substituted strings |
| `path traversal not permitted` | Path contains `..` | Use absolute canonical paths |
| `your role X lacks Y permission` | Role-level gap | Either upgrade role or add scope with `--permissions` (additive, server-scoped) |

## Team authorized_keys lines (native forwarding)

- sshd grants `ssh -L`/`-R` before the ForceCommand runs, so the gate never sees them. `port-forward` is enforced IN the line: active holder (role or scope) → `restrict,port-forwarding,permitopen="localhost:*",command="servel remote gate --user=X" <type> <b64> # servel-access:X`; everyone else (incl. disabled/expired) → `restrict,command=...`. servel's own pf/connect/tunnel/dev-link-infra only dial `localhost:<port>`.
- Lines are DERIVED (`internal/access/authkeys.go` `rewrite`): every write re-renders every managed line under `authorized_keys.lock`; users/roles/scopes saves re-sync immediately; daemon re-checks every 5 min (`internal/daemon/authkeys_reconcile.go`, audit `access.authkeys_sync[_declined]`, alert `authkeys_unrepaired`).
- Keys are stored canonical (type + base64, comment dropped). `ParseSSHKey` rejects multi-key input and options prefixes.
- A line whose key ≠ the user's `ssh_fingerprint` in users.yaml (suspect, possible planted key) or that can't be re-rendered is QUARANTINED: commented out as `# servel-quarantined <reason>: <original>` (grants nothing, kept as evidence) + alert. Investigate before restoring (delete the prefix).
- Daemon budget (24/24h) gates only GRANTS; revocations/quarantines always apply.
- Manual `ssh -L host:port` to a non-localhost target is refused by design; use `servel port-forward`.

## Join key (`join_key_seed`)

- **What:** base64 ed25519 seed; `state.ProjectState.JoinKeySeed` (`internal/state/project.go`). Server copy at `/var/servel/access/join_key_seed`; deploy reads it into the project's `.servel/state.json` (`readServerJoinKeyAndFingerprint`, `internal/cli/clideploy/remote_state.go`) alongside `server_host` + `ssh_host_fingerprint`.
- **Server side:** `ConfigureJoinKey` (`internal/server/join_key.go`) writes the derived pubkey to `/var/servel/access/authorized_keys` (added to sshd's `AuthorizedKeysFile` by `internal/server/init_system.go`; `connectViaJoinKey` logs in as `root`) as `restrict,command="/usr/local/bin/servel server join-shell" <key>`.
- **What it can do:** join shell (`internal/server/join_shell.go`) accepts only `JOIN <token> <pubkey>` (valid invite token required), `REQUEST <pubkey> <email> <reason> <duration>` (creates a pending access request — admin approval required), `CHECK <pubkey>` (status). JOIN + REQUEST share a per-IP rate limit (5 / 15 min) and write to the audit log.
- **Why it's in the repo:** `servel access request` in a cloned project uses it via `connectViaJoinKey` (`internal/cli/access/access_request.go`) — no prior access needed.
- **Commit-safety conflict:** `docs/internal-docs/architecture/state-management-architecture.md` calls state.json "user-specific, gitignored" and committing it "controversial"; servel itself only adds `.servel/*.local.json` to `.gitignore`, leaving state.json tracked.
- **Leak impact:** server host + host fingerprint disclosed; anyone can file rate-limited access requests (social-engineering risk if approved blindly). No deploy / exec / shell.
- **Rotate:** `servel access setup --rotate-join-key` → `ConfigureJoinKey(force=true)`: old seed dead in every repo, outstanding invite tokens invalidated. Each repo's state.json needs refreshing (next deploy re-reads the seed).
- **Agent rule:** scanner hit → explain the above, don't treat it as a server credential, don't untrack state.json unilaterally; offer keep / untrack / rotate.

## Files at a glance

```
/var/servel/access/
├── users.yaml         # AccessUser (username, ssh fingerprint, role_id, status)
├── roles.yaml         # AccessRole (custom + system)
├── scopes.yaml        # per-user, per-server overrides (additive perms + resource restrictions)
├── policies/          # compiled gate policies
├── join-keys/         # invite & join key material
├── invites/           # pending invite tokens
└── audit.log          # gate decision log (append-only)
```

Source of truth for shapes: `internal/types/access.go`.
