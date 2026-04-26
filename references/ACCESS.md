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
| anything else | `config-read` (stricter than status) |

Unknown paths default to `config-read` rather than the loosest tier — fail
closed.

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
