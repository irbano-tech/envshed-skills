---
name: envshed
description: "Guide for the Envshed secrets manager and its envshed CLI: pull, push and inject environment variables across organizations, projects, environments and monorepo workspaces. Use when the user asks about env vars, secrets, .env files or envshed commands, and when a task needs an API key or credential, involves giving an AI agent access to secrets, the Envshed MCP server, secrets in CI (service tokens, machine tokens, GitHub Actions OIDC), sharing a secret with someone, or a secret that may have leaked into an agent history."
progressive_disclosure:
  entry_point:
    summary: "Secrets manager CLI for syncing and injecting environment variables, with agent safety, MCP and CI support"
    when_to_use: "When working with envshed, environment variables, secrets, .env files, agent access to secrets, MCP, or CI credentials."
    quick_start: "1. Read the agent rules below if you are an AI agent. 2. Use the command index. 3. Open a reference for depth."
  references:
    - agent-safety.md
    - run-command.md
    - hosted-agents-mcp-oidc.md
    - sharing-and-states.md
    - secret-management.md
    - environments.md
    - workspaces-monorepo.md
    - ci-cd-service-tokens.md
    - api-reference.md
---

# Envshed

Envshed is a secrets manager for teams. Secrets live encrypted on the server (AES-256-GCM) and reach your code through the `envshed` CLI, the web dashboard (https://app.envshed.com), the REST API, the Node SDK (`@envshed/node`) or the MCP server.

Install: `npm install -g envshed` (Node.js 18 or later). Check with `envshed --version`.

## If you are an AI agent, read this first

Whatever you see ends up in the chat history and at the model provider. Work so that you never see a secret value. These rules match the `envshed-agent-safety` skill that `envshed agent install` writes (see [agent-safety.md](references/agent-safety.md)):

1. **Never print a value.** Do not put one in a command line, a tool argument, a file, a commit or your reply.
2. **Use `envshed run -- <command>`** to give a program its secrets. The program gets the values, you do not. In agent mode the output is masked as `[ENVSHED:KEY_NAME]`.
3. **Do not run** `envshed secret get`, `envshed pull --stdout` (or `pull -f` aimed at the terminal), `envshed diff --show-values`, or `envshed share open` without `--file`. These are for a person in their own terminal.
4. **Do not run** `envshed run -- printenv`, `envshed run -- env`, or anything that prints the injected environment. Do not pass `--no-redact` and do not change `ENVSHED_AGENT_MODE`.
5. **Do not read** `.env`, `.env.local`, `.envrc` or files under `~/.envshed/`. `.env.example` is fine.
6. **Never ask for a secret in chat.** If one is missing, run `envshed request <KEY> --reason "<why>" --no-wait` and give the user the printed link. Run the same command again to check.
7. **If a value reached the chat,** tell the user at once. It needs rotating, which is their step. `envshed scan` reports leaked keys by name. Do not run `scan --clean` or `scan --undo` unless the user asks.
8. **A hook block is not an obstacle to work around.** Do what the message says. If it looks wrong, tell the user.

These protections reduce accidental leaks. They are not a security boundary: limits that must hold belong on the server (roles, scoped tokens, agent sessions).

## Quick start

```bash
envshed login                                        # browser device-code login
envshed init -o my-org -p my-app -e development      # writes .envshed.json
envshed pull                                         # writes .env
envshed run -- npm start                             # injects secrets, no file on disk
envshed push                                         # uploads local .env changes
envshed env staging                                  # switch the default environment and pull it
envshed branch                                       # show org / project / env context
```

To protect a Claude Code setup: `envshed agent install`, then restart Claude Code.

## Core concepts

| Concept | Description |
|---|---|
| Organization | Top-level team container. Org roles: `owner`, `admin`, `member`, `viewer`. |
| Project | A service or app in an organization. New projects get the organization's environments from **Settings > Environments** (by default `development`, `staging`, `production`). |
| Environment | A named set of secrets inside a project. A slug means the same environment in every project of the organization. |
| Secret | A key/value pair. Keys match `^[A-Za-z_][A-Za-z0-9_]*$`. Every change creates a version. |
| Placeholder | A key with no shared value. Each person fills it with a personal override. |
| Personal override | Your own value for a secret, visible only to you, used by your `pull` and `run`. |
| Disabled secret | Keeps its value but is left out of `pull`, `run` and the API until re-enabled. |
| Reference | `${KEY}` (same environment) or `${env.KEY}` (another environment of the same project) inside a value, resolved on the server at read time. |
| Secret link | The same key linked across projects. A push propagates the new value. |
| Snapshot | Point-in-time copy of a whole environment, restorable. |
| Share link | Expiring, encrypted link to some values for someone outside the team. |
| Secret request | Ask a person to fill a key in the dashboard, so the value never passes through a terminal or chat. |
| Agent session | Temporary, read-only, server-enforced access to one environment for an AI agent. |
| Service token | Org-owned token for CI and infrastructure. Scope `org`, `project` or `environment`; permission `read` or `read_write`. |
| Machine token | Read-only token for hosted agents and routines. One environment, optional key list, stops after 90 days idle. Created in the dashboard. |
| OIDC trust policy | Lets a GitHub Actions job get a short read-only session with no stored credential. |
| Workspace | A monorepo package mapped to its own project, environment and env file. |

## Configuration

**Project config `.envshed.json`** (commit it, it holds no secrets). The CLI looks for it from the current directory upward.

```json
{
  "org": "my-org",
  "project": "my-app",
  "defaultEnv": "development"
}
```

`apiUrl` in this file is honored only for loopback hosts (`localhost`, `127.0.0.1`, `::1`). For a remote instance, use `~/.envshed/config.json` or `ENVSHED_API_URL`. For monorepos, add a `workspaces` block (see [workspaces-monorepo.md](references/workspaces-monorepo.md)).

**Global config `~/.envshed/config.json`** holds `apiUrl` and `locale`. The login token goes to the OS keychain (macOS Keychain, Linux Secret Service) and falls back to this file (mode `0600`) where there is no keychain. On CI (`CI` set) or with `ENVSHED_KEYCHAIN=off`, the keychain is not used.

**Resolution order**

| Setting | Order, highest first |
|---|---|
| org, project, env | CLI flags (`-o`, `-p`, `-e`), then `ENVSHED_ENV` (env only), then `.envshed.json` |
| API URL | `--api-url` (on `login`, `init`), then loopback `apiUrl` in `.envshed.json`, then `~/.envshed/config.json`, then `ENVSHED_API_URL`, then `https://app.envshed.com` |
| Token | `ENVSHED_TOKEN`, then an OIDC session (`run --oidc`) or an agent session (`run` only), then the saved login (config file or keychain) |
| Output masking in `run` | `--redact` / `--no-redact`, then `--oidc` (on), then `ENVSHED_AGENT_MODE`, then `CLAUDE_CODE_CHILD_SESSION=1` (on), else off |

There is no per-command token flag. `ENVSHED_OIDC_POLICY` (or `--oidc`) together with `ENVSHED_TOKEN` is an error.

**Token lifetime.** A personal token, including the one `envshed login` saves, stops after 30 days without use; any request restarts the count. Run `envshed login` again when it stops. Machine tokens stop after 90 idle days. Service tokens have only the expiry you set.

## Command index

Most commands that touch secrets take `-o, --org`, `-p, --project` and `-e, --env`, and many take `--output json` and `-y, --yes`. Run `envshed <command> --help` for the full list.

**Global options:** `--locale <code>` (en, es, pt-BR), `-w, --workspace <path>`, `-a, --all` (all workspaces), `-V, --version`, `-h, --help`.

### Auth and context

| Command | What it does |
|---|---|
| `envshed login` | Browser device-code login (10 minute timeout) |
| `envshed login --token-stdin` | Save an existing token read from stdin or a masked prompt (`-t, --token` is deprecated) |
| `envshed login --api-url <url>` | Log in against another API URL |
| `envshed whoami` | Your email, or the token's org, scope and permission for a non-personal token |
| `envshed token show` / `token clear` | Show the masked token / remove it from keychain and config |
| `envshed token set --token-stdin` | Save a token (`token set <token>` is deprecated) |
| `envshed init [-o -p -e] [--api-url] [-w <path>]` | Create `.envshed.json`; interactive monorepo detection without flags |
| `envshed branch` | Show org / project / env (and workspaces) from local config, no login needed |

### Secrets

| Command | What it does |
|---|---|
| `envshed pull [-f <path>] [--stdout] [--format env\|json] [--force]` | Write secrets to `.env` (aborts on local changes unless `--force`) |
| `envshed push [-f <path>] [--force]` | Upload a local file. Never deletes remote keys |
| `envshed run [--watch] [--interval <s>] [--redact\|--no-redact] [--oidc <policy>] -- <cmd>` | Run a command with secrets injected |
| `envshed secret get <KEY>` | Print one value (human terminal only) |
| `envshed secret set <KEY> [--placeholder]` | Set a value from a masked prompt or stdin, never from an argument |
| `envshed secret delete <KEY>` | Delete a secret |
| `envshed secret disable <KEY>` / `secret enable <KEY>` | Hide from consumption / bring back, per environment |
| `envshed secret override set <KEY>` / `override remove <KEY>` | Your personal override (user token only) |
| `envshed diff <env1> <env2> [--show-values]` | Compare two environments, values masked by default |
| `envshed import [file] [--format env\|json\|yaml\|csv] [--force]` | Import from a file |
| `envshed import --from doppler --token <doppler-token> [--dry-run]` | Migrate from Doppler |
| `envshed versions <KEY> [--limit <n>]` | Version history (default 20) |
| `envshed rollback <KEY> --version <n>` | Restore a version as a new version |
| `envshed snapshot create [-n <name>] [-d <desc>]` / `list` / `restore <id>` | Environment snapshots |
| `envshed link <KEY> --to-project <slug> --to-env <slug>` / `unlink <KEY>` | Cross-project links |
| `envshed definition list` / `create <KEY>` / `update <KEY>` / `delete <KEY>` | Expected keys with `-d` description and `--required` |
| `envshed share [create] [KEY...]` / `list` / `revoke <id>` / `open <url>` | Expiring encrypted share links |

### Agents

| Command | What it does |
|---|---|
| `envshed request <KEY> --reason <text> [--timeout <d>] [--no-wait]` | Ask a person to fill a secret in the dashboard |
| `envshed agent install [--agent claude-code] [--scope user\|project]` | Install hooks and the `envshed-agent-safety` skill |
| `envshed agent uninstall [--agent] [--scope]` | Remove only what install added |
| `envshed agent session -e <env> [--ttl <d>] [--keys <list>] [--reason <text>]` | Temporary read-only session for an agent |
| `envshed agent session list` / `agent session revoke <id>` | List / end sessions |
| `envshed scan [--path <p>] [--formats-only] [--no-mark] [--clean [--yes]] [--undo [id]] [--forget-backups]` | Find vault values and token formats in agent histories |

### Organization, projects, environments

| Command | What it does |
|---|---|
| `envshed org list` / `org create -n <name> [-s <slug>] [-d <text>]` | Organizations (`orgs` is a legacy alias) |
| `envshed project list` / `create -n <name>` / `update <slug>` / `delete <slug>` | Projects (`projects` is a legacy alias) |
| `envshed env list` / `create -n <name>` / `update <slug>` / `delete <slug>` / `duplicate <src> -n <name>` | Environments (`envs` is a legacy alias) |
| `envshed env checkout <slug>` (or `env switch <slug>`, or `env <slug>`) | Change `defaultEnv` and pull; asks first if the env file has unpushed changes |
| `envshed rename-env <slug> [-n <name>] [-s <slug>]` | Rename an environment across the organization |
| `envshed env-version` | Environment version number and last update |
| `envshed member list` / `invite <email> -r <role>` / `role <email> -r <role>` / `remove <email>` | Org members (admin or owner, user token only) |
| `envshed webhook list` / `create --name --url --events` / `update <id>` / `delete <id>` / `test <id>` | Webhooks (admin or owner, user token only) |
| `envshed token create-service` / `list-service` / `update-service <id>` / `revoke-service --id <id>` | Service tokens |
| `envshed workspace add <path>` / `remove <path>` / `list` | Monorepo workspaces |
| `envshed completions bash\|zsh\|fish` | Shell completion script |

## Common workflows

### Daily development

```bash
envshed pull                    # sync .env (add --force to overwrite local edits)
envshed run -- pnpm dev         # or skip the file and inject at start
envshed run --watch -- pnpm dev # restart when remote secrets change (polls every 30s)
```

### A secret is missing (agent or human)

```bash
envshed request STRIPE_KEY --reason "payment tests" -e development --no-wait
# exit 2 = pending; share the printed link; run the same command again later
envshed run -e development -- npm test   # once it exits 0
```

A person can also fill it in a separate terminal with `envshed secret set STRIPE_KEY -e development`, which prompts for the value.

### Give an agent limited access

```bash
envshed agent session -e development --ttl 2h --keys API_KEY,DATABASE_URL
envshed run -e development -- npm test   # run uses the session, not your login
```

A production-like environment needs approval by a person in the dashboard. See [agent-safety.md](references/agent-safety.md).

### CI and hosted agents

Pick the first that fits (details in [hosted-agents-mcp-oidc.md](references/hosted-agents-mcp-oidc.md)):

1. A person can approve once: connect the MCP server with OAuth (`https://app.envshed.com/api/mcp`).
2. GitHub Actions: `envshed run --oidc <policy-id> -- <cmd>` with `permissions: id-token: write`. No stored token.
3. Anywhere else: a machine token (dashboard, **Hosted agents**) stored as `ENVSHED_TOKEN`, then `envshed run -- <cmd>`.

A scoped service token (`envshed token create-service`) also works for classic CI. See [ci-cd-service-tokens.md](references/ci-cd-service-tokens.md).

### Before a risky change

```bash
envshed snapshot create -n "before v2.1 deploy" -e production
envshed diff staging production          # values masked
# ...if it goes wrong:
envshed snapshot list -e production
envshed snapshot restore <snapshot-id> -e production
envshed rollback API_KEY --version 5 -e production   # or one key
```

### Hand a value to a contractor

```bash
envshed share DATABASE_URL --expires 1h --burn
envshed share STRIPE_KEY --expires 3d --password --note "for the payments contractor"
```

The URL is shown once. See [sharing-and-states.md](references/sharing-and-states.md).

### A value leaked into an agent history

```bash
envshed scan                 # report by key name, marks exposed keys on the server
envshed secret set API_KEY -e development    # rotate, in a separate terminal
envshed scan --clean         # optional, user's decision; undo with: envshed scan --undo
```

### Monorepo

```bash
envshed init                         # detects pnpm/npm workspaces
envshed pull                         # at the root: every workspace
envshed --workspace apps/api run -- npm start
```

### Migrate from Doppler

```bash
envshed import --from doppler --token <doppler-token> --org my-org --dry-run
envshed import --from doppler --token <doppler-token> --org my-org
```

## References

| File | Covers |
|---|---|
| [agent-safety.md](references/agent-safety.md) | Rules for agents, `request`, `agent install`, `agent session`, `scan`, `.envshed-allow` |
| [run-command.md](references/run-command.md) | `run` argv rules, exit codes, signals, watch mode, output masking |
| [hosted-agents-mcp-oidc.md](references/hosted-agents-mcp-oidc.md) | MCP server, machine tokens, GitHub Actions OIDC, which to use |
| [sharing-and-states.md](references/sharing-and-states.md) | Disabled, placeholders, overrides, references, links, share links, definitions |
| [secret-management.md](references/secret-management.md) | get/set/delete, pull/push, versions, rollback, snapshots, import, diff |
| [environments.md](references/environments.md) | Environment lifecycle, checkout, rename, env-version |
| [workspaces-monorepo.md](references/workspaces-monorepo.md) | Workspaces config and behavior |
| [ci-cd-service-tokens.md](references/ci-cd-service-tokens.md) | Tokens, CI examples, members, webhooks, completions |
| [api-reference.md](references/api-reference.md) | REST API v1 routes |

## Localization

CLI output is available in English (`en`), Spanish (`es`) and Brazilian Portuguese (`pt-BR`). Order: `--locale <code>`, then `locale` in `~/.envshed/config.json`, then the system locale (`LANG` / `LC_ALL`), then `en`. Agent hooks read only `LC_ALL` / `LANG`.
