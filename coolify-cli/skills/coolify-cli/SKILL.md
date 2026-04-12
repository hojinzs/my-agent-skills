---
name: coolify-cli
description: |
  A skill for deploying applications, checking deployment status, and managing settings
  via the Coolify CLI. Use this skill whenever the user mentions Coolify, deployment,
  server management, app status checks, environment variables, database management,
  or service deployment. Also trigger on requests like "deploy my app", "check server status",
  "add env vars", "push to coolify", or any infrastructure management task involving Coolify.
  Coolify is a self-hosted PaaS, and coolify-cli is a Go CLI tool for controlling it from the terminal.
---

# Coolify CLI Skill

Deploy applications, check status, and manage settings using the Coolify CLI.

## First Step: Verify Installation and Connection

Before starting any task, always perform the checks below. Do not skip this step.

### Step 1: Verify CLI Installation

```bash
coolify version
```

Note: the CLI uses `coolify version` as a subcommand, not `--version`.

If this command fails (command not found), the CLI is not installed. Guide the user through installation:

**Linux / macOS:**
```bash
curl -fsSL https://raw.githubusercontent.com/coollabsio/coolify-cli/main/scripts/install.sh | bash
```

**Homebrew:**
```bash
brew install coollabsio/coolify-cli/coolify-cli
```

**Windows PowerShell:**
```powershell
irm https://raw.githubusercontent.com/coollabsio/coolify-cli/main/scripts/install.ps1 | iex
```

**Go:**
```bash
go install github.com/coollabsio/coolify-cli/coolify@latest
```

After installation, verify again with `coolify version`.

### Step 2: Verify Context (Server Connection)

```bash
coolify context list
```

If no contexts exist, guide the user through setup:

1. Generate an API token from the Coolify dashboard at `/security/api-tokens`
2. Register a context:
```bash
# Coolify Cloud users
coolify context set-token cloud <token>

# Self-hosted users
coolify context add -d <name> <URL> <token>
```

### Step 3: Test Connection

```bash
coolify context verify
```

This command must succeed before proceeding. If it fails, the token may be expired or the URL may be incorrect — ask the user to verify.

## Core Workflows

### 1. Deploy

Deployment is the most common operation. There are three approaches:

```bash
# Deploy by UUID
coolify deploy uuid <uuid>

# Deploy by resource name
coolify deploy name <app-name>

# Deploy multiple apps at once (comma-separated)
coolify deploy batch <name1,name2,name3>
```

Useful deployment flags:
- `--force` / `-f`: Skip confirmation prompts
- `--pull-request-id <id>`: Deploy a PR preview
- `--docker-tag <tag>`: Deploy a specific Docker tag

### 2. Check Status

```bash
# List all apps
coolify app list

# Get specific app details
coolify app get <uuid>

# Stream app logs in real-time
coolify app logs <uuid> --follow

# View deployment history
coolify app deployments list <uuid>

# View build logs for a specific deployment (summary)
coolify app deployments logs <app-uuid> <deployment-uuid>

# View build logs with full Docker build output (recommended for debugging)
coolify app deployments logs <app-uuid> <deployment-uuid> --debuglogs

# List servers and their status
coolify server list
coolify server get <uuid> --resources

# List all resources across projects
coolify resources list
```

**Identifying apps by name vs UUID:** When multiple apps share the same name
(e.g., staging and production instances), always use `coolify app list --format=json`
to find the correct UUID and use UUID-based commands to avoid ambiguity.

### 3. Change Settings

#### Update App Settings
```bash
coolify app update <uuid> --name "new-name"
coolify app update <uuid> --domains "app.example.com"
coolify app update <uuid> --git-branch develop
coolify app update <uuid> --build-command "npm run build"
coolify app update <uuid> --ports-exposes 3000
```

#### Manage Environment Variables
```bash
# List env vars
coolify app env list <uuid>

# Add a new env var
coolify app env create <uuid> --key DB_HOST --value localhost

# Update an env var
coolify app env update <uuid> <env-key> --value new-value

# Delete an env var
coolify app env delete <uuid> <env-uuid>

# Sync from a .env file (updates existing + adds missing, does NOT delete)
coolify app env sync <uuid> --file .env.production
```

#### App Lifecycle Control
```bash
coolify app start <uuid>
coolify app stop <uuid>
coolify app restart <uuid>
```

### 4. Create an App

App creation commands vary by type. Required: `--project-uuid`, `--server-uuid`.

```bash
# From a public Git repository
coolify app create public \
  --project-uuid <p-uuid> --server-uuid <s-uuid> \
  --git-repository https://github.com/user/repo \
  --git-branch main --build-pack nixpacks --ports-exposes 3000

# From a Docker image
coolify app create dockerimage \
  --project-uuid <p-uuid> --server-uuid <s-uuid> \
  --docker-registry-image-name nginx --ports-exposes 80

# From a Dockerfile
coolify app create dockerfile \
  --project-uuid <p-uuid> --server-uuid <s-uuid> \
  --dockerfile "FROM node:18\nCOPY . .\nCMD [\"node\", \"index.js\"]"
```

If the required UUIDs are unknown, look them up first:
```bash
coolify server list          # Find server-uuid
coolify projects list        # Find project-uuid
```

## Multi-Environment (Context) Usage

When managing multiple Coolify instances (e.g., staging, production):

```bash
# Run a command against a specific context
coolify --context=staging deploy name my-app
coolify --context=production app logs <uuid> --follow

# Switch the default context
coolify context use production
```

## Output Formats

The default output is a table. Use JSON when scripting or parsing is needed:

```bash
coolify app list --format=json
coolify deploy name my-app --format=json
```

Use `--show-sensitive` / `-s` to reveal masked sensitive data (tokens, IPs, etc.).

## Resource Aliases

The CLI supports multiple aliases for convenience:
- `app` = `apps` = `application`
- `database` = `db`
- `service` = `svc`
- `server` = `servers`
- `private-key` = `key`
- `github` = `gh`

## Detailed Reference

For commands related to databases, services, servers, private keys, and GitHub Apps, read `references/commands.md`. Consult that file when you need to:

- Create/manage/backup databases
- Deploy and manage one-click services
- Add/remove/validate servers
- Manage SSH private keys
- Configure GitHub App integrations

## Monitoring a Deployment

After triggering a deploy, the response includes a `deployment_uuid`. Use it to track progress:

```bash
# Check deployment status (look for "status" field: in_progress, finished, failed)
coolify app deployments list <app-uuid> --format=json

# View build logs (summary only)
coolify app deployments logs <app-uuid> <deployment-uuid>

# View full Docker build output (use this to see actual build errors)
coolify app deployments logs <app-uuid> <deployment-uuid> --debuglogs
```

The `--debuglogs` flag is important because without it, you only see high-level status messages.
The actual build errors (npm failures, Dockerfile issues, compilation errors) only appear in debug logs.

**Polling pattern:** Deployments can take several minutes (especially with native module compilation).
Check the status periodically via `coolify app deployments list --format=json` and look at the
`status` field. Once it changes from `in_progress` to `finished` or `failed`, check the final logs.

## Debugging a Failed Deployment

When a deployment fails, follow this sequence:

1. **Get the failed deployment's logs with debug output:**
   ```bash
   coolify app deployments logs <app-uuid> <deployment-uuid> --debuglogs
   ```

2. **Common failure patterns and fixes:**

   | Error | Cause | Fix |
   |-------|-------|-----|
   | `npm ci` — "package.json and package-lock.json are in sync" | Lock file out of date or generated with a different Node version | Regenerate lock file with the same Node version as the Dockerfile |
   | `Cannot find module '@tailwindcss/...'` | Build-time deps in devDependencies + `NODE_ENV=production` skipping them | Use `npm ci --include=dev` in Dockerfile, or move deps to dependencies |
   | `failed to solve: process ... did not complete successfully` | Dockerfile build step failed | Read the lines above this error for the actual failure |
   | Healthcheck failed | App built but isn't responding on the expected port | Check app logs with `coolify app logs <uuid>`, verify port and health check path |

3. **Fix the root cause** in the source code or Dockerfile, push the changes, then redeploy:
   ```bash
   coolify deploy uuid <app-uuid> --force
   ```

4. **Verify the new deployment succeeds** by monitoring logs again.

## Workflow Guide

Follow the steps below based on the user's request. Always perform **"First Step: Verify Installation and Connection"** before proceeding.

**Deploy requests** (e.g., "deploy my app", "push to production"):
1. Identify the target app name or UUID (if unknown, run `coolify app list`)
2. If multiple apps share the same name, use `coolify app list --format=json` to find the correct UUID
3. Run `coolify deploy name <name>` or `coolify deploy uuid <uuid>`
4. Monitor the deployment and verify the result (see "Monitoring a Deployment" below)

**Status check requests** (e.g., "check app status", "show server info"):
1. Identify the target (app, server, or all resources)
2. Run the appropriate list/get command
3. Check logs if needed (`coolify app logs`)

**Settings change requests** (e.g., "update the domain", "add env var"):
1. Check current settings first (`coolify app get <uuid>`)
2. Confirm the intended changes with the user
3. Apply changes via `coolify app update` or `coolify app env`
4. Verify the updated state

**App creation requests** (e.g., "create a new app", "set up a new service"):
1. Look up server/project UUIDs (`coolify server list`, `coolify projects list`)
2. Determine the app type (public, dockerfile, dockerimage, github, deploy-key)
3. Gather required info (git repo, branch, port, etc.)
4. Run `coolify app create`
5. Set up env vars if needed via `coolify app env create` or `coolify app env sync`
