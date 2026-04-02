# Servel Template Building Guide

Build infrastructure templates for `servel add <type>`. Templates can be:
- **Local**: Project-specific in `.servel/templates/`
- **Server**: Shared across projects in `/var/servel/templates/`

## Template Location & Structure

**Local (project-specific):**
```
.servel/templates/{category}/{type}/
├── meta.yaml    # Version metadata
└── v1.yaml      # Full template config
```

**Server-wide:**
```
/var/servel/templates/{category}/{type}/
├── meta.yaml
└── v1.yaml
```

**Categories:** `databases`, `queues`, `searches`, `platforms`, `caches`, `blockchains`, `monitoring`, `analytics`, `storage`, `email`, `ci`, `internal`, `realtime`

**Priority:** Local → Server → Hub (local templates override built-in)

## Quick Start

### 1. Simple Template (Single Image)

```yaml
# meta.yaml
type: myservice
category: database
latest: "v1"
default: "v1"
supported_versions:
  - version: "v1"
    status: stable
    released: "2026-01-26"
```

```yaml
# v1.yaml
type: myservice
version: "v1"
category: database
name: My Service
description: Brief one-line description

image: vendor/image:tag

environment:
  PASSWORD: "{{generate:password:32}}"
  USER: "{{.Username}}"
  DATABASE: "{{.Database}}"

ports:
  - container: 5432
    protocol: tcp
    internal: true

volumes:
  - target: /var/lib/data
    size: "{{.Storage}}"

health_check:
  test: ["CMD", "healthcheck-cmd"]
  interval: "30s"
  timeout: "5s"
  retries: 3
  start_period: "10s"

default_resources:
  memory: "512MB"
  cpu: 0.5
  storage: "10GB"
  replicas: 1

connection_template: "protocol://{{.Username}}:{{.Env.PASSWORD}}@{{.Host}}:{{.Port}}/{{.Database}}"
connection_password_key: PASSWORD

connection_vars:
  HOST: "{{ .Host }}"
  PORT: "{{ .Port | default 5432 }}"
  USER: "{{ .Username }}"
  PASSWORD: "{{ .Password }}"
  DATABASE_URL: "protocol://{{ urlEncode .Username }}:{{ urlEncode .Password }}@{{ .Host }}:{{ .Port }}/{{ .Database }}"

supports_backup: true
backup_command: ["backup", "-o", "/backup/{{.Name}}-{{.Timestamp}}.sql"]
restore_command: ["restore", "-i", "{{.BackupFile}}"]

required_flags:
  - name

docs_url: "https://docs.example.com"

rotatable_credentials:
  - name: PASSWORD
    generator: "{{generate:password:32}}"
    restart_required: true
    description: Main password
```

### 2. Compose-Based Template (Multi-Service)

```yaml
type: myplatform
version: "v1"
category: platform
name: My Platform
description: Multi-service platform with database and API

source:
  type: git
  repo: https://github.com/vendor/project.git
  paths:
    - docker/docker-compose.yml
    - docker/.env.example
    - docker/volumes/
  ref: v1.2.0  # Pin to stable release

placement:
  strategy: colocate  # All services on same node
  sticky: true        # Keep on same node after deploy

deployment_stages:
  - name: core
    services: [db, cache]
    wait_for_healthy: true
    timeout: "3m"
  - name: api
    services: [api, worker]
    wait_for_healthy: true
    timeout: "2m"
  - name: frontend
    services: [web]
    wait_for_healthy: true
    timeout: "2m"

health_check_overrides:
  db:
    test: ["CMD", "pg_isready", "-U", "postgres"]
    interval: "10s"
    timeout: "5s"
    retries: 10
    start_period: "60s"
  api:
    type: "none"  # Disable if no healthcheck binary available

template_vars:
  - name: Domain
    description: Main domain for the service
    required: true
  - name: AdminEmail
    description: Admin email address
    required: false
    default: "admin@example.com"

env_mappings:
  Domain: BASE_URL
  AdminEmail: ADMIN_EMAIL
env_file_path: ".env"

rotatable_credentials:
  - name: DB_PASSWORD
    generator: "{{generate:password:32}}"
    restart_required: true
    linked_services: [db, api, worker]
    description: Database password

  - name: JWT_SECRET
    generator: "{{generate:secret:64}}"
    restart_required: true
    linked_services: [api, worker]
    service_env_mapping:
      api: API_JWT_SECRET
      worker: WORKER_JWT_SECRET

  - name: API_KEY
    generator: "{{generate:jwt:service_role}}"
    derives_from: JWT_SECRET
    restart_required: true
    linked_services: [api]

overrides:
  environment:
    api:
      PUBLIC_URL: "https://{{ .Domain }}"
      ORG_NAME: "{{ .Name }}"
    web:
      API_URL: "https://{{ .Domain }}/api"

  routing:
    web:
      port: 3000
      domain: "{{ .Domain }}"
      https: true
      protocol: http
    db:
      port: 5432
      domain: "{{ .Name }}-db.{{ .PrimaryDomain }}"
      protocol: postgres

  volumes:
    db:
      - target: /var/lib/postgresql/data
        size: "{{.Storage}}"

default_resources:
  memory: "4GB"
  cpu: 2.0
  storage: "50GB"
  replicas: 1

connection_template: "https://{{ .Domain }}"

connection_vars:
  URL: "https://{{ .Var.Domain }}"
  API_KEY: "{{ .Env.API_KEY }}"

post_restore:
  - name: sync_passwords
    service: db
    command: ["psql", "-U", "postgres", "-c", "ALTER USER app WITH PASSWORD '{{.Env.DB_PASSWORD}}'"]
    description: Reset passwords after restore

post_init:
  - name: prepare_database
    service: rails
    command: ["bundle", "exec", "rails", "db:chatwoot_prepare"]
    description: Initialize database schema
    run_once: true
    timeout: "10m"

actions:
  shell:
    service: db
    command: psql -U postgres
    description: Database shell
    interactive: true

  logs:
    service: api
    command: tail -f /var/log/app.log
    description: View API logs

docs_url: "https://docs.myplatform.com"

setup_notes: |
  Post-Deployment:
  1. Access: https://{{.Domain}}
  2. Admin: {{.Env.ADMIN_EMAIL}}
```

## Field Reference

### Core Metadata (Required)

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | **Must match folder name exactly** |
| `version` | string | Template version (e.g., "v1") |
| `category` | string | Singular form: database, queue, search, platform, cache, blockchain, monitoring, analytics, storage, email, ci, realtime |
| `name` | string | Display name |
| `description` | string | One-line description |

### Source (Choose One)

**Image-based:**
```yaml
image: vendor/image:tag
command: ["optional", "override"]  # Optional
```

**Compose-based:**
```yaml
source:
  type: git
  repo: https://github.com/owner/repo.git
  paths:
    - docker-compose.yml
    - .env.example
    - volumes/
  ref: v1.0.0  # Pin version
```

### Port Configuration

```yaml
ports:
  - container: 8080            # Required: 1-65535
    protocol: tcp              # tcp, udp, http, postgres, mysql, redis, mongodb
    mode: auto                 # auto, traefik, traefik-tcp, traefik-tcp-passthrough
    domain: "{{ .Domain }}"    # For HTTP/TCP routing
    internal: true             # Hide from external access (default: false)
    sni_capable: true          # TLS SNI support (default: false)
    optional: true             # Can be disabled (default: false)
    name: http                 # Short name for 'servel connect'
    description: "HTTP API"    # User-facing description
```

**Mode Reference:**
- `auto`: Traefik auto-detection (HTTP → traefik, TCP/UDP → traefik-tcp)
- `traefik`: HTTP only via Traefik (ports 80/443)
- `traefik-tcp`: TCP/UDP port forwarding via Traefik
- `traefik-tcp-passthrough`: Raw TCP passthrough (container handles TLS)

### Volumes

```yaml
volumes:
  - target: /var/lib/data     # Must be absolute path
    size: "{{.Storage}}"      # Size spec or variable
    readonly: false           # Optional: read-only mount
```

### Health Checks

**For image-based templates:**
```yaml
health_check:
  test: ["CMD", "pg_isready", "-U", "postgres"]
  interval: "30s"
  timeout: "5s"
  retries: 3
  start_period: "10s"
  type: "none"               # Use to disable health check
```

**For compose-based templates (override per-service):**
```yaml
health_check_overrides:
  service-name:
    test: ["CMD", "healthcheck"]
    interval: "10s"
    timeout: "5s"
    retries: 10
    start_period: "60s"
    type: "none"             # Disable for this service
```

### Resources

```yaml
default_resources:
  memory: "512MB"    # 512MB, 1GB, 4G, etc.
  cpu: 0.5           # CPU cores (float)
  storage: "10GB"    # Storage allocation
  replicas: 1        # Instances
```

### Template Variables

User-provided values via `--var Key=value`:

```yaml
template_vars:
  - name: Domain
    description: Main domain
    required: true
  - name: AdminEmail
    description: Admin email
    required: false
    default: "admin@example.com"
```

**Access in templates:** `{{ .Domain }}`, `{{ .AdminEmail }}`

### Environment Mappings

Map template vars to .env file:

```yaml
env_mappings:
  Domain: BASE_URL           # {{ .Domain }} → BASE_URL in .env
  AdminEmail: ADMIN_EMAIL
env_file_path: ".env"        # Where to write
```

### Rotatable Credentials

```yaml
rotatable_credentials:
  - name: DB_PASSWORD
    generator: "{{generate:password:32}}"
    restart_required: true
    linked_services: [db, api]
    description: Database password

  - name: JWT_SECRET
    generator: "{{generate:secret:64}}"
    restart_required: true
    linked_services: [api, auth]
    service_env_mapping:      # Map to different env vars per service
      api: API_JWT_SECRET
      auth: AUTH_JWT_SECRET

  - name: API_KEY
    generator: "{{generate:jwt:service_role}}"
    derives_from: JWT_SECRET  # Regenerated when parent rotates
    linked_services: [api]
```

**Generators:**
- `{{generate:password:N}}` - N alphanumeric chars
- `{{generate:secret:N}}` - N hex chars (cryptographic)
- `{{generate:jwt:anon}}` - JWT with anon claims
- `{{generate:jwt:service_role}}` - JWT with service_role claims
- `{{generate:basicauth}}` - Full Basic Auth header
- `{{generate:basicauth:username}}` - Username part
- `{{generate:basicauth:password}}` - Password part

### Connection Configuration

```yaml
# Single URL template
connection_template: "postgresql://{{.Username}}:{{.Env.POSTGRES_PASSWORD}}@{{.Host}}:{{.Port}}/{{.Database}}"
connection_password_key: POSTGRES_PASSWORD

# Default credentials (for services without env var support)
connection_username_template: "admin@{{.Domain}}"
connection_default_password: "admin"

# Auto-inject vars when using --link-infra
connection_vars:
  HOST: "{{ .Host }}"
  PORT: "{{ .Port | default 5432 }}"
  USER: "{{ .Username }}"
  PASSWORD: "{{ .Password }}"
  DATABASE_URL: "postgresql://{{ urlEncode .Username }}:{{ urlEncode .Password }}@{{ .Host }}:{{ .Port }}/{{ .Database }}"
```

**Template Variables:**
- `.Host` - Service hostname
- `.Port` - Exposed port
- `.Username`, `.Password`, `.Database` - Credentials
- `.Env.VAR` - Environment variable value
- `.Var.FieldName` - User template variable
- `.Domain` - Primary domain
- `.PrimaryDomain` - Root domain
- `.Name` - Service name

**Filters:**
- `urlEncode` - URL-encode value
- `default VALUE` - Fallback if empty

### Compose Overrides

```yaml
overrides:
  environment:
    service-name:
      VAR: "value"
      DYNAMIC: "{{ .Domain }}"
      FROM_CREDS: "${DB_PASSWORD}"

  routing:
    service-name:
      port: 8000
      domain: "{{ .Domain }}"
      https: true
      protocol: http          # http, tcp, postgres, mysql, redis, mongodb
      websocket: true         # Enable WebSocket support
      headers:
        - "X-Custom: value"

  volumes:
    service-name:
      - target: /data
        size: "{{.Storage}}"
```

### Deployment Stages (Compose Only)

```yaml
deployment_stages:
  - name: core
    services: [db, cache]
    wait_for_healthy: true
    timeout: "3m"

  - name: api
    services: [api, worker]
    wait_for_healthy: true
    timeout: "2m"
```

### Lifecycle Hooks (Post-Init)

Run commands after successful deployment (all services healthy):

```yaml
post_init:
  - name: prepare_database
    service: rails
    command: ["bundle", "exec", "rails", "db:migrate"]
    description: Run database migrations
    run_once: true       # Only on first deployment (tracked in spec.json)
    timeout: "10m"       # Default: 5m
```

**Features:**
- `run_once: true` - Hook only executes on first deployment, state tracked in `HooksExecuted`
- Hooks run sequentially after all deployment stages complete
- Failures are logged but don't fail deployment (can re-run manually)

### Service Overrides (Image/Command)

Override images or commands per service (for compose-based templates):

```yaml
overrides:
  services:
    postgres:
      image: pgvector/pgvector:pg16   # Override default image
      command: ["postgres", "-c", "shared_preload_libraries=vectors"]
    rails:
      entrypoint: ["bundle", "exec"]
```

**Use case:** Upstream compose uses base image but you need a variant (e.g., pgvector for AI features).

### Placement

```yaml
placement:
  strategy: colocate         # any, colocate, manager_only, spread
  sticky: true               # Pin to same node
  constraints:
    - "node.role==manager"
    - "node.hostname!=worker-1"
```

### Backup & Restore

```yaml
supports_backup: true

backup_command:
  - "pg_dump"
  - "-U"
  - "{{.Username}}"
  - "-f"
  - "/backup/{{.Name}}-{{.Timestamp}}.sql"

restore_command:
  - "psql"
  - "-U"
  - "{{.Username}}"
  - "-f"
  - "{{.BackupFile}}"

post_restore:
  - name: sync_passwords
    service: db
    command: ["psql", "-c", "ALTER USER app WITH PASSWORD '{{.Env.DB_PASSWORD}}'"]
    description: Reset passwords
```

### Actions

```yaml
actions:
  psql:
    service: db
    command: psql -U postgres
    description: Database shell
    interactive: true

  schema:
    service: db
    command: pg_dump -s postgres
    description: Dump schema

  deploy-functions:
    service: functions
    description: Upload edge functions
    vars:
      - name: FunctionsDir
        default: "./supabase/functions"
    inputs:
      - source: "{{.FunctionsDir}}"
        target: /home/deno/functions
    command: ls /home/deno/functions/
    confirm: "Upload functions?"
```

Action fields: `service`, `command`, `description`, `interactive`, `user`, `workdir`, `output`, `outputs`, `inputs`, `vars`, `env`, `confirm`

**inputs** upload local files/dirs to the container before command execution:
- `source`: local path (supports `{{.VarName}}` templates)
- `target`: absolute container path

**vars** define user-provided variables via `--var Name=value` or positional args.

Usage: `servel infra run name action [args...]`

### Config Files

Generate config files from templates:

```yaml
config_files:
  - path: /etc/app.yaml
    content: |
      log_level: info
      api_key: {{.Env.API_KEY}}
      api_secret: {{.Env.API_SECRET}}
      ws_url: {{.Env.WS_URL}}

      {{if .Env.S3_BUCKET}}
      s3:
        bucket: {{.Env.S3_BUCKET}}
        access_key: {{.Env.S3_ACCESS_KEY}}
      {{end}}

command:
  - "--config"
  - "/etc/app.yaml"
```

### Docker Capabilities

Add Linux capabilities:

```yaml
cap_add:
  - SYS_ADMIN    # For Chrome sandboxing
  - NET_ADMIN    # For network operations
```

### Linkable Infrastructure

Auto-inject vars when linking to specific infra types:

```yaml
linkable_infra:
  - type: livekit
    env_vars:
      WS_URL: "{{ .LinkedInfra.connection_template }}"
      API_KEY: "{{ .LinkedInfra.Env.LIVEKIT_API_KEY }}"
      API_SECRET: "{{ .LinkedInfra.Env.LIVEKIT_API_SECRET }}"
  - type: minio
    env_vars:
      S3_ENDPOINT: "{{ .LinkedInfra.connection_template }}"
      S3_ACCESS_KEY: "{{ .LinkedInfra.Env.MINIO_ACCESS_KEY }}"
```

### Upgrade Configuration

Safe version upgrade handling:

```yaml
upgrade:
  pre_upgrade_backup: true
  backup_services: [db]
  health_verification:
    - service: db
      check: "pg_isready -U postgres"
    - service: api
      check: "curl -f http://localhost:8080/health"
```

### Documentation

```yaml
docs_url: "https://docs.example.com"

setup_notes: |
  Post-Deployment:
  1. Access at https://{{.Domain}}
  2. Login with {{.Env.ADMIN_USER}}:{{.Env.ADMIN_PASS}}

required_flags:
  - name
  - domain
  - var:Domain
```

## Validation Rules

| Rule | Example |
|------|---------|
| `type` must match folder name | Folder `uptimekuma` → type `uptimekuma` |
| Category must be singular | `database` not `databases` |
| Port range 1-65535 | `container: 8080` |
| Volume target must be absolute | `/var/lib/data` not `./data` |
| Memory format | `512MB`, `1GB`, `4G` |
| Duration format | `30s`, `2m`, `1h` |
| No mixed source types | Either `image` OR `source`, never both |

## Common Patterns

### Satellite Service (Links to Parent)
```yaml
type: myservice-worker
category: realtime

image: vendor/worker:latest

cap_add:
  - SYS_ADMIN  # If needed

config_files:
  - path: /etc/config.yaml
    content: |
      api_key: {{.Env.PARENT_API_KEY}}
      ws_url: {{.Env.PARENT_WS_URL}}

linkable_infra:
  - type: myservice
    env_vars:
      PARENT_WS_URL: "{{ .LinkedInfra.connection_template }}"
      PARENT_API_KEY: "{{ .LinkedInfra.Env.API_KEY }}"
      PARENT_API_SECRET: "{{ .LinkedInfra.Env.API_SECRET }}"

required_flags:
  - name
  - var:ParentUrl  # Fallback if not linked
```

### Database Template
```yaml
type: mydb
category: database
image: vendor/mydb:latest
environment:
  PASSWORD: "{{generate:password:32}}"
  USER: "{{.Username}}"
  DATABASE: "{{.Database}}"
ports:
  - container: 5432
    protocol: tcp
    internal: true
volumes:
  - target: /var/lib/data
    size: "{{.Storage}}"
health_check:
  test: ["CMD", "mydb-check"]
connection_template: "mydb://{{.Username}}:{{.Env.PASSWORD}}@{{.Host}}:{{.Port}}/{{.Database}}"
connection_password_key: PASSWORD
supports_backup: true
```

### Platform with Multi-Domain
```yaml
overrides:
  routing:
    api:
      port: 8000
      domain: "{{ .Domain }}"
      protocol: http
    db:
      port: 5432
      domain: "{{ .Name }}-db.{{ .PrimaryDomain }}"
      protocol: postgres
```

### Email Server (Complex Ports)
```yaml
ports:
  - container: 25
    protocol: tcp
    mode: auto
    sni_capable: false
    name: smtp
  - container: 465
    protocol: tcp
    mode: auto
    sni_capable: true
    name: smtps
  - container: 443
    protocol: tcp
    mode: traefik-tcp-passthrough
    domain: "{{.Host}}"
    name: https
```

### Service with Derived Credentials
```yaml
rotatable_credentials:
  - name: JWT_SECRET
    generator: "{{generate:secret:64}}"
    linked_services: [api, auth]

  - name: ANON_KEY
    generator: "{{generate:jwt:anon}}"
    derives_from: JWT_SECRET
    linked_services: [api]

  - name: SERVICE_KEY
    generator: "{{generate:jwt:service_role}}"
    derives_from: JWT_SECRET
    linked_services: [api, admin]
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Folder `uptime-kuma`, type `uptimekuma` | **Must match exactly** |
| `category: databases` | Use singular: `category: database` |
| `target: ./data` | Use absolute: `target: /var/lib/data` |
| `image:` AND `source:` both present | Choose one only |
| Health check uses unavailable binary | Verify binary exists in container |
| `start_period: "5s"` for complex service | Increase to `30s`+ |
| Service name with uppercase | Docker requires lowercase |

## Health Check Guidelines

| Container Type | Recommended Check |
|---------------|-------------------|
| PostgreSQL | `pg_isready -U user` |
| MySQL | `mysqladmin ping` |
| Redis | `redis-cli ping` |
| HTTP service | `wget --spider http://localhost:PORT/health` |
| Alpine without tools | `nc -z localhost PORT` |
| Scratch/minimal image | `type: "none"` |

## Testing Template

**Local template (recommended for development):**
```bash
# Create template in project
mkdir -p .servel/templates/databases/mydb
# Add meta.yaml and v1.yaml

# Test with dry-run
servel add mydb --name test --dry-run

# Deploy for real
servel add mydb --name test
```

**Server-wide template:**
```bash
# Copy to server templates dir
servel ssh myserver
mkdir -p /var/servel/templates/databases/mydb
# Add meta.yaml and v1.yaml

# Test from any project
servel add mydb --name test --dry-run
```

**Verify:** Check generated compose, env vars, routing labels.

## Server-Side Template Caching

On deployment, critical template fields are cached in `spec.json`:
- `rotatable_credentials` - For offline credential rotation
- `actions` - For `servel infra run name action`
- `post_restore` - For backup restore hooks
- `post_init` - For lifecycle hooks
- `deployment_stages` - For staged deployment tracking

This enables these features to work without network access to the hub.

