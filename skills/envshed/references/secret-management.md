# Secret Management

Day-to-day secret operations: pull, push, single secrets, history, snapshots, import and diff. For disabled secrets, placeholders, overrides, references, links, share links and definitions see [sharing-and-states.md](sharing-and-states.md).

Commands marked **human terminal only** print values. An AI agent must not run them; use `envshed run` instead (see [agent-safety.md](agent-safety.md)).

## Pull

```bash
envshed pull [-o <org>] [-p <project>] [-e <env>] [-f <path>] [--stdout] [--format env|json] [--force]
```

| Option | Description | Default |
|---|---|---|
| `-f, --file <path>` | Output file | `.env` (or the workspace's `file`) |
| `--stdout` | Print instead of writing a file (**human terminal only**) | |
| `--format <format>` | `env` or `json` | `env` |
| `--force` | Overwrite without checking for changes | |

- **Safe pull:** if pulling would modify or delete values in the existing file, the command aborts and lists them (`~ KEY (modified)`, `- KEY (deleted)`). Run again with `--force` to overwrite.
- Warns about placeholders that have no personal override, with links to set one.
- Disabled secrets are never written (`N disabled secrets skipped`).
- References are already resolved by the server.
- Values with carriage returns or every dotenv quote character need `--format json`.
- The CLI caches the environment version in `~/.envshed/version-cache.json` to notice when the remote changed.

## Push

```bash
envshed push [-o <org>] [-p <project>] [-e <env>] [-f <path>] [--force]
```

- **Safe push:** if remote values would be modified, or remote keys are missing from your file, the command aborts and lists them. `--force` pushes anyway.
- Remote keys missing from the local file are **kept**, never deleted.
- Changes to a placeholder are saved as your personal override.
- Linked secrets propagate. On a conflict you choose keep mine, keep theirs, or unlink.

## Single secrets

```bash
envshed secret get <KEY>                 # human terminal only
envshed secret set <KEY>                 # masked prompt, or value from stdin
envshed secret set <KEY> --placeholder   # declare a key with no shared value
envshed secret delete <KEY> [-y]
envshed secret disable <KEY> [-y]
envshed secret enable <KEY> [-y]
```

All take `-o`, `-p`, `-e` and `--output json`. `secret set` never takes the value as an argument or flag. When stdin is piped it reads the value from stdin (30 second timeout), so a script can do:

```bash
my-secret-tool read db-url | envshed secret set DATABASE_URL -e production
```

`secret get` prints a hint on stderr at a terminal to avoid terminal history exposure. Requesting a disabled secret returns not found.

## Version history and rollback

Every change creates a version (`created`, `updated`, `rolled_back`) with author and time.

```bash
envshed versions DATABASE_URL              # last 20
envshed versions API_KEY --limit 50 -e production
envshed rollback DATABASE_URL --version 2 -e production
```

A rollback is non-destructive: rolling back from v3 to v2 creates v4 with the v2 value. It is in the audit log and bumps the environment version.

## Snapshots

A snapshot records every secret's version in an environment at a point in time.

```bash
envshed snapshot create [-n <name>] [-d <description>] [-e <env>]
envshed snapshot list [-e <env>]
envshed snapshot restore <snapshot-id> [-e <env>]
```

Restoring sets every secret back to its snapshot value, creating a `rolled_back` version for each. Secrets deleted after the snapshot are skipped. Use snapshots before deploys and bulk changes.

## Import

```bash
envshed import [file] [options]
```

| Option | Description |
|---|---|
| `[file]` | `.env`, `.json`, `.yaml`, `.yml` or `.csv`. Omit with `--from`. |
| `--format <format>` | `env`, `json`, `yaml`, `csv` (auto-detected from the extension) |
| `--force` | Overwrite existing secrets without confirmation |
| `-o`, `-p`, `-e` | Target context |
| `--from <source>` | Migrate from an external source. Currently `doppler`. |
| `--token <token>` | Token for the `--from` source (Doppler service account token; env `DOPPLER_TOKEN`) |
| `--doppler-project <name>` | One Doppler project (default: every accessible project) |
| `--doppler-config <name>` | One Doppler config (requires `--doppler-project`) |
| `--doppler-api-url <url>` | Override the Doppler API URL (default `https://api.doppler.com`) |
| `--config-map <mapping>` | Rename configs, such as `"prd=production,stg=staging"` |
| `--dry-run` | Print the migration plan without writing |

Formats:

- `.env`: `KEY=VALUE`, multi-line values in double quotes.
- JSON: a flat object, or Infisical's export format (array of `key` / `value` objects).
- YAML: a flat mapping. `yes`, `no`, `true`, `false`, `on`, `off` are kept as strings.
- CSV: header row with `key` and `value` columns.

Before writing, the CLI lists keys that would be overwritten with a different value and asks to confirm (skip with `--force`).

```bash
envshed import .env
envshed import secrets.json --format json
envshed import production.env --env production --force
envshed import --from doppler --token <doppler-token> --org my-org --dry-run
envshed import --from doppler --token <doppler-token> --org my-org \
  --doppler-project backend --config-map "prd=production,stg=staging"
```

The Doppler migration creates matching Envshed projects and environments and pushes their secrets. Prefer setting `DOPPLER_TOKEN` over passing `--token` on the command line.

## Diff

```bash
envshed diff <env1> <env2> [-o <org>] [-p <project>] [--show-values]
```

Compares two environments of the same project: added (`+`), removed (`-`) and changed (`~`) keys, with a summary line. Values are masked as `***`. `--show-values` prints them with a warning on stderr (**human terminal only**, never in CI logs). If either environment cannot be read, the diff aborts with no partial output.
