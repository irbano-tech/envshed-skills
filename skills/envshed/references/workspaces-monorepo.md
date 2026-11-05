# Workspaces and Monorepos

Workspaces map packages in a monorepo to their own Envshed project, environment and env file.

## Setup

```bash
envshed init                         # at the root: detects pnpm-workspace.yaml or package.json workspaces
envshed init --workspace apps/web    # add one workspace (prompts for project, env, file)
envshed workspace add apps/web       # same, after initial setup
envshed workspace remove apps/web
envshed workspace list
```

## Configuration

```json
{
  "org": "my-org",
  "workspaces": {
    "apps/web": {
      "project": "web-frontend",
      "defaultEnv": "development",
      "file": ".env.local"
    },
    "apps/api": {
      "project": "backend-api",
      "defaultEnv": "development"
    },
    "packages/shared": {
      "org": "other-org",
      "project": "shared-lib"
    }
  }
}
```

| Field | Description | Required | Default |
|---|---|---|---|
| `project` | Envshed project slug | Yes | |
| `defaultEnv` | Current environment slug | No | |
| `file` | Env file name, relative to the workspace | No | `.env` |
| `org` | Override the top-level org | No | Top-level `org` |

A top-level `project` and `defaultEnv` can sit alongside `workspaces` when the root is a project too (for example shared infrastructure secrets). Without `workspaces`, everything works in single-project mode.

## How commands pick a workspace

| Where you are | Behavior |
|---|---|
| Inside a workspace directory | Only that workspace |
| At the root | `pull`, `push` and `env checkout` act on every workspace (and the root project, if set); `run` needs `--workspace` |
| `-w, --workspace <path>` | Only that workspace, from anywhere |
| `-a, --all` | Every workspace and the root project, even when a root project is configured |

`--workspace` and `--all` are global options. Put them before the command, as the docs do:

```bash
envshed --workspace apps/api pull
envshed --workspace apps/web push
envshed --workspace apps/api run -- npm start
envshed --workspace apps/web env checkout staging
envshed --all pull
```

The CLI finds the current directory from `INIT_CWD` when run through pnpm or npm scripts, so `pnpm --filter` does not confuse it.

- `run` always needs one workspace context; `--all` is not supported.
- `link` and `unlink` do not support `--all`; use `--workspace`.
- `-f` on `pull` or `push` overrides the workspace's `file` for that call.

## Output

`envshed pull` at the root:

```
  my-org / web-frontend / development
✔ Wrote 12 secrets to apps/web/.env.local
  my-org / backend-api / development
✔ Wrote 8 secrets to apps/api/.env
Pulled secrets for 2 workspaces
```

`envshed branch` in workspace mode marks the current workspace with `*`:

```
  org:     my-org
    Path          Project         Env          File
* apps/web      web-frontend    development  .env.local
  apps/api      backend-api     development  .env
```

## Linked secrets across workspaces

Workspaces often map to different projects that share a key such as `DATABASE_URL`. Link it once (`envshed --workspace apps/web link DATABASE_URL --to-project backend-api --to-env development`) and a push from either side propagates the value. See [sharing-and-states.md](sharing-and-states.md#secret-links-cross-project).
