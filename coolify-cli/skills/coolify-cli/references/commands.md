# Coolify CLI Detailed Command Reference

This document covers commands not included in the main SKILL.md.

## Table of Contents

1. [Databases](#databases)
2. [Database Backups](#database-backups)
3. [Services](#services)
4. [Servers](#servers)
5. [Private Keys](#private-keys)
6. [GitHub Apps](#github-apps)
7. [Teams](#teams)
8. [Storage](#storage)
9. [Context Management](#context-management)

---

## Databases

Aliases: `database`, `databases`, `db`, `dbs`

### Basic CRUD
```bash
coolify database list
coolify database get <uuid>
coolify database delete <uuid> [--delete-configurations] [--delete-volumes] [--docker-cleanup] [--delete-connected-networks]
```

### Create
Supported types: `postgresql`, `mysql`, `mariadb`, `mongodb`, `redis`, `keydb`, `clickhouse`, `dragonfly`

```bash
coolify database create <type> \
  --server-uuid <uuid> \
  --project-uuid <uuid> \
  [--environment-name <name>] \
  [--name <db-name>] \
  [--image <image>] \
  [--instant-deploy] \
  [--is-public] \
  [--public-port <port>] \
  [--limits-memory <mem>] \
  [--limits-cpus <cpus>]
```

### Lifecycle Control
```bash
coolify database start <uuid>
coolify database stop <uuid>
coolify database restart <uuid>
```

### Update
```bash
coolify database update <uuid> [--name <name>] [...]
```

---

## Database Backups

```bash
# List backups
coolify database backup list <db-uuid>

# Create a backup schedule
coolify database backup create <db-uuid> \
  [--frequency "0 2 * * *"] \
  [--enabled] \
  [--save-s3] \
  [--retention-policy-amount <n>] \
  [--retention-policy-days <n>]

# Manually trigger a backup
coolify database backup trigger <db-uuid> <backup-uuid>

# View backup execution history
coolify database backup executions <db-uuid> <backup-uuid>

# Update a backup schedule
coolify database backup update <db-uuid> <backup-uuid> [--frequency ...] [--enabled ...]

# Delete a backup schedule
coolify database backup delete <db-uuid> <backup-uuid>

# Delete a specific execution
coolify database backup delete-execution <db-uuid> <backup-uuid> <execution-uuid>
```

---

## Services

Aliases: `service`, `services`, `svc`

Deploy one-click services (WordPress, Plausible, Gitea, etc.).

```bash
# List available service types
coolify service create --list-types

# Service CRUD
coolify service list
coolify service get <uuid>
coolify service create <type> [--server-uuid ...] [--project-uuid ...]
coolify service delete <uuid>

# Lifecycle control
coolify service start <uuid>
coolify service stop <uuid>
coolify service restart <uuid>

# Environment variables (same pattern as app env commands)
coolify service env list <uuid>
coolify service env create <uuid> --key <KEY> --value <VALUE>
coolify service env update <uuid> <env-key> --value <new-value>
coolify service env delete <uuid> <env-uuid>
coolify service env sync <uuid> --file .env

# Storage (same pattern as app storage commands)
coolify service storage list <uuid>
coolify service storage create <uuid> --type persistent --mount-path /data --name my-vol
coolify service storage delete <uuid> <storage-uuid>
```

---

## Servers

Aliases: `server`, `servers`

```bash
# List / get servers
coolify server list
coolify server get <uuid>
coolify server get <uuid> --resources    # Include all resource statuses

# Add a server
coolify server add <name> <ip> <private-key-uuid> \
  [-p/--port <port>] \
  [-u/--user <user>] \
  [--validate]

# Remove a server
coolify server remove <uuid>

# Validate server connection
coolify server validate <uuid>

# List domains assigned to a server
coolify server domains <uuid>
```

---

## Private Keys

Aliases: `private-key`, `private-keys`, `key`, `keys`

```bash
# List keys
coolify private-key list

# Add a key (use @path to read from file)
coolify private-key add <name> <private-key>
coolify private-key add my-key @~/.ssh/id_rsa

# Remove a key
coolify private-key remove <uuid>
```

---

## GitHub Apps

Aliases: `github`, `gh`, `github-app`, `github-apps`

Access private repositories through GitHub Apps.

```bash
# List / get GitHub Apps
coolify github list
coolify github get <app-uuid>

# Create a GitHub App
coolify github create \
  --name <name> \
  --api-url <url> \
  --html-url <url> \
  --app-id <id> \
  --installation-id <id> \
  --client-id <id> \
  --client-secret <secret> \
  --private-key-uuid <uuid>

# Update / delete
coolify github update <app-uuid> [...]
coolify github delete <app-uuid>

# List accessible repositories
coolify github repos <app-uuid>

# List branches for a specific repository
coolify github branches <app-uuid> <owner/repo>
```

---

## Teams

Aliases: `team`, `teams`

Teams use numeric IDs, not UUIDs.

```bash
coolify team list
coolify team get <team-id>
coolify team current
coolify team members list [team-id]
```

---

## Storage

Mount persistent volumes or files to apps and services.

```bash
# List storage
coolify app storage list <app-uuid>

# Create a persistent volume
coolify app storage create <app-uuid> \
  --type persistent \
  --mount-path /data \
  --name my-volume \
  [--host-path /host/path] \
  [--is-directory]

# Create a file mount
coolify app storage create <app-uuid> \
  --type file \
  --mount-path /app/config.json \
  --name config \
  --content '{"key": "value"}'

# Update / delete
coolify app storage update <app-uuid> --uuid <storage-uuid> [...]
coolify app storage delete <app-uuid> <storage-uuid>
```

---

## Context Management

Detailed commands for managing multiple environments (staging, production, etc.).

```bash
# List contexts
coolify context list

# Add a context (-d: set as default, -f: force overwrite)
coolify context add [-d] [-f] <name> <url> <token>

# Get / update / delete a context
coolify context get <name>
coolify context update <name> [--name <new-name>] [--url <new-url>] [--token <new-token>]
coolify context delete <name>

# Update token only
coolify context set-token <name> <token>

# Switch default context
coolify context use <name>
coolify context set-default <name>

# Verify connection
coolify context verify

# Check remote Coolify API version
coolify context version
```

---

## Global Flags

Available on all commands:

| Flag | Description |
|------|-------------|
| `--context <name>` | Use a specific context |
| `--host <url>` | Override the Coolify host URL |
| `--token <token>` | Override the authentication token |
| `--format <format>` | Output format: `table` (default), `json`, `pretty` |
| `-s, --show-sensitive` | Show sensitive information |
| `-f, --force` | Skip confirmation prompts |
| `--debug` | Enable debug mode |
