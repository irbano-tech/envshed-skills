# Migrating an existing project with an agent

Move a project's `.env` files into Envshed without reading them. The CLI reads the files and sends the values under the person's login; you work with key names, counts and exit codes.

Needs the CLI release after 0.24.0 (`envshed import` with `--on-conflict`, `--create-env` and `--output json`). On 0.24.0, `import --dry-run` on a file writes for real: do not use it there.

## Before you start

- The person signs in in their own terminal: `envshed login`. Do not ask for a token in the chat.
- Never read, `cat`, `grep` or print an env file. Never use `envshed secret set KEY value` to move values: the value would be in the command line.

## Steps

```bash
# 1. Organization and project
envshed orgs
envshed projects -o <org>
envshed project create --name <project> -o <org> --output json

# 2. Link the folder (writes .envshed.json, safe to commit)
envshed init -o <org> -p <project> -e development --yes --no-guide --no-import

# 3. Plan each file by key name; nothing is written
envshed import .env -e development --dry-run --output json
envshed import .env.production -e production --dry-run --output json

# 4. Import with an explicit conflict policy
envshed import .env -e development --on-conflict skip --output json

# 5. Check the app
envshed run -- <command>
```

`.env.<name>` files are read as env files. Map each file to its environment yourself; ask the user when the name does not say which one.

## Reading the result

| Field | Meaning |
| --- | --- |
| `keys.new` | Keys created |
| `keys.changed` | Keys that already exist with another value |
| `keys.skipped` | Changed keys left as they are (`--on-conflict skip`) |
| `keys.written` | Keys sent to Envshed; unchanged keys are never sent |
| `git.tracked` | The file is committed: its values are in the repository history and need rotating |
| `git.ignored` | The file is in `.gitignore` |

Exit codes: `0` done, `1` error, `2` keys already exist with another value and nothing was written. Error JSON: `{ "error": "conflict" | "env_not_found" | "file_not_found" | "format" | "parse" | "invalid_keys" | "usage" | "api", "message": "...", "keys": [...] }`.

## Decisions that belong to the user

- `keys.changed` not empty: ask whether to keep the current values (`--on-conflict skip`) or replace them (`--on-conflict overwrite`). Do not switch to `--force` on your own.
- `env_not_found`: ask before adding `--create-env`.
- `git.tracked: true`: tell the user the values are in git history and need rotating.
- Deleting the env file after `envshed run` works: ask first. Suggest `envshed scan` to find values in local agent histories.

## What does not exist

- The MCP server cannot create projects or import files; it never handles values.
- Agent sessions, machine tokens and OIDC sessions are read-only and cannot import.
