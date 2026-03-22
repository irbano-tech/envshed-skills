# Workspaces & Monorepo Support

Envshed has first-class monorepo support with automatic workspace detection and per-workspace configuration.

## How It Works

When `.envshed.json` contains a `workspaces` key, envshed operates in workspace mode. Each workspace maps a directory path to a project, environment, and output file.

## Setup

### Auto-Detection

`envshed init` automatically detects monorepo packages from:
- `pnpm-workspace.yaml` — reads `packages` globs
- `package.json` `workspaces` field — reads workspace paths

It prompts you to select which packages should be configured as workspaces.

### Manual Setup

```bash
# Add a workspace interactively
envshed workspace add apps/api

# Or use init for a single workspace
envshed init -w apps/api
```

## Configuration

### `.envshed.json` with Workspaces

```json
{
  "org": "my-org",
  "apiUrl": "https://app.envshed.com",
  "workspaces": {
    "apps/api": {
      "project": "api-backend",
      "defaultEnv": "development",
      "file": ".env"
    },
    "apps/web": {
      "project": "web-frontend",
      "defaultEnv": "development",
      "file": ".env.local"
    },
    "packages/shared": {
      "org": "other-org",
      "project": "shared-lib",
      "defaultEnv": "development",
      "file": ".env"
    }
  }
}
```

Each workspace can override:
- `org` — different organization (cross-org monorepos)
- `project` — the envshed project to map to
- `defaultEnv` — environment to use by default
- `file` — output filename for pulled secrets (default: `.env`)

## Workspace Commands

### Targeting a Specific Workspace

Use `-w` / `--workspace` to target a single workspace:

```bash
envshed pull -w apps/api           # Pull secrets for apps/api only
envshed push -w apps/web           # Push secrets for apps/web only
envshed env staging -w apps/api    # Switch env for apps/api only
envshed branch -w apps/api         # Show context for apps/api
```

### Bulk Mode (`--all`)

Use `-a` / `--all` to run a command across all workspaces:

```bash
envshed pull --all                 # Pull secrets for every workspace
envshed push --all                 # Push secrets for every workspace
```

**Note:** `envshed run` does not support `--all` — you must target a specific workspace.

### Managing Workspaces

```bash
envshed workspace list             # Show all configured workspaces
envshed workspace add <path>       # Add a workspace (interactive)
envshed workspace remove <path>    # Remove a workspace from config
```

## Branch Context in Workspace Mode

`envshed branch` displays a table when in workspace mode:

```
Workspace       Project         Environment
apps/api        api-backend     development
apps/web        web-frontend    staging
packages/shared shared-lib      development
```

## Bulk Push with Linked Secrets

When pushing all workspaces (`--all`), if any secrets are linked across projects, envshed:
1. Pushes each workspace in sequence
2. Detects linked secret conflicts
3. Prompts for resolution (keep mine / keep theirs / unlink)
4. Auto-pulls after all pushes to sync propagated changes

## Auto-Detection of Workspace Context

When running commands from within a workspace directory (e.g., `cd apps/api && envshed pull`), envshed auto-detects the workspace from `INIT_CWD` (pnpm/npm) or the current working directory relative to the `.envshed.json` location.

## Workspace File Placement

Pulled secrets are written to `<workspace-path>/<file>`. For example, with the config above:
- `apps/api/.env`
- `apps/web/.env.local`
- `packages/shared/.env`
