# Agent Safety

How to use Envshed with AI coding agents so secret values stay out of the chat, the transcript and the model provider. Everything here reduces accidental leaks. None of it is a security boundary against an agent with shell access: that agent can reach anything your user can. Limits that must hold (who can read production, which keys a token sees) belong on the server, through roles, scoped tokens and agent sessions.

## Rules for the agent

These match the `envshed-agent-safety` skill that `envshed agent install` writes. That skill is separate from this one and does not change it.

- Never ask anyone to paste a secret into the chat. If one is missing, use `envshed request`, so a person enters the value in the dashboard.
- Never print a secret value, and never put one in a command line, a tool argument, a file, a commit or your reply.
- To use secrets at runtime, inject them: `envshed run -- <command>` in a shell, or the `@envshed/node` SDK inside your own tool, so the model sees only the result.
- Do not run `envshed secret get`, `envshed pull --stdout`, `envshed diff --show-values`, or `envshed share open` without `--file`.
- Do not read `.env` files or anything under `~/.envshed/`. `.env.example` is fine.
- Do not run `envshed run -- printenv`, `envshed run -- env` or anything else that prints the injected environment.
- If a value reached the chat, tell the user at once. It needs rotating, which is their step.
- A block or a refusal is not an obstacle to work around. Do what the message says, and if it looks wrong, tell the user.
- Do not run `envshed scan --clean` or `--undo` unless the user asks.

These apply whenever you are an AI agent, whether or not an agent session is active.

## envshed request

Ask a person to provide a secret. The value goes straight into Envshed and never passes through this terminal or the chat. The command reports only the key and the status.

```bash
envshed request <KEY> --reason "<why it is needed>" [options]
```

| Option | Description |
|---|---|
| `--reason <text>` | Required. Shown to whoever fills it. Never put a secret here. |
| `-o, --org`, `-p, --project`, `-e, --env` | Context |
| `--timeout <duration>` | How long to wait, such as `90s`, `10m` or `1h`. Default `10m`. |
| `--no-wait` | Create the request and exit |
| `--output json` | One JSON document for scripts and agents |

| Exit code | Status | Meaning |
|---|---|---|
| `0` | `fulfilled`, `exists` | The value is available. Use it with `envshed run`. |
| `1` | | Error, such as no access or an invalid option |
| `2` | `pending` | Request is open (`--no-wait` only) |
| `3` | `declined` | Someone declined. Do not retry, talk to the user. |
| `4` | `timeout`, `expired` | Stopped waiting, or the request expired |
| `5` | `unusable` | The key exists as a placeholder or is disabled, so `run` cannot use it |

Recommended loop for an agent, since shell tools limit how long a command can run:

1. `envshed request GITHUB_PAT --reason "configure the runner" -e development --no-wait`
2. Tell the user who can fill it and give them the printed link (or the printed `envshed secret set` command, for a separate terminal).
3. Run the same command again to check. An open request is reused, not duplicated.
4. On exit `0`, use `envshed run -e development -- <command>`.

Behavior:

- Existing keys are never overwritten. If the key has a value, the command says so and exits `0`.
- Unanswered requests close after 7 days. A decline sticks for an hour. Each environment can have 20 open requests.
- Asking needs read access to the environment (a viewer or a read-only service token can ask). Org owners and admins, project admins and editors with access, and read-write service tokens can fill or decline.
- Every step is in the audit log. Values are never logged.

## envshed agent install / uninstall

```bash
envshed agent install [--agent claude-code] [--scope user|project] [--output json]
envshed agent uninstall [--agent claude-code] [--scope user|project] [--output json]
```

Defaults: `--agent claude-code` (the only agent supported today), `--scope user`.

| | `--scope user` | `--scope project` |
|---|---|---|
| Settings | `~/.claude/settings.json` | `.claude/settings.json` |
| Skill | `~/.claude/skills/envshed-agent-safety/SKILL.md` | `.claude/skills/envshed-agent-safety/SKILL.md` |

User scope follows `CLAUDE_CONFIG_DIR` when set. Project scope uses the Git repository root, or the current directory outside a repository.

What it installs:

- A `UserPromptSubmit` hook that stops a prompt that looks like it contains a secret (known token formats, JWTs, private keys, URLs with a password, sensitive assignments such as `DB_PASSWORD=...`, phrases like "the password is ..." in English, Portuguese or Spanish, long high-entropy strings).
- A `PreToolUse` hook that stops tool calls that would bring a value into the session.
- The `envshed-agent-safety` skill with the rules above.

Install merges with existing settings, is safe to run again, and stops without changes if the settings file is not valid JSON. Restart the agent afterwards. Uninstall removes only Envshed's own entries and a skill file Envshed wrote.

The hooks call the `envshed` binary directly, never `npx`, so install the CLI with `npm install -g envshed`. With project scope each teammate needs it on `PATH`. Hooks work offline, make no network calls and do not read your token. A hook that crashes exits `1` and Claude Code lets the action continue unchecked.

What the `PreToolUse` hook blocks, and what to use instead:

| Blocked | Use instead |
|---|---|
| `envshed secret get` | `envshed run -- <command>` |
| `envshed pull --stdout`, and `pull -f` aimed at `-`, `/dev/stdout`, `/dev/stderr`, `/dev/tty` or `/dev/fd/N` | `envshed run -- <command>` |
| `envshed diff --show-values` | `envshed diff` |
| `envshed share open` without `--file` or with `--file` aimed at the terminal | `envshed share open --file <path>` |
| `envshed run -- printenv`, `run -- env`, and one-liners that print the environment | `envshed run -- <command>` |
| `run --no-redact`, setting `ENVSHED_AGENT_MODE`, unsetting `CLAUDE_CODE_CHILD_SESSION`, `env -i` in front of `envshed` or a shell | `envshed run -- <command>` |
| Reading `.env`, `.env.local`, `.envrc`, `prod.env` with Read, Grep, `cat`, `head`, `grep`, `sed`, `source` (including globs) | `envshed run -- <command>`, or read `.env.example` |
| Reading `~/.envshed/` or the `envshed` keychain entries | Nothing: `envshed` uses them on its own |

Still allowed: `envshed run -- npm test`, `envshed pull -f .env.local`, `ENVSHED_AGENT_MODE=1`, `.env.example`, `.env.sample`, `.env.template`, `cp .env.example .env`, `ls .env`, and text that only mentions `.env`.

A prompt blocked by the hook does not reach the model, but Claude Code may still have written it to local files. If it held a real secret, rotate it.

### False positives

- In a prompt, add `envshed:allow` anywhere and send it again.
- For recurring patterns, add them to `.envshed-allow` (patterns, never values). This is the user's decision, not the agent's.

### `.envshed-allow`

The hooks and `envshed scan` use the nearest `.envshed-allow`, walking up from the working directory to the Git root or your home directory. One entry per line, `#` for comments.

| Entry | Applies to | Meaning |
|---|---|---|
| `<pattern>` | Prompts (and scan's token formats) | Let a detected string through. Must contain `*` or `?` and match the whole string. |
| `rule:<id>` | Prompts (and scan's token formats) | Turn off one rule, such as `github-token`, `phrase`, `assignment` or `high-entropy` |
| `path:<glob>` | Tool calls | Let the agent read this `.env` file, for example a fixture with fake values |

```text
rec_*
rule:high-entropy
path:tests/fixtures/.env.test
```

A line with no wildcard is ignored, and so is a line that looks like a secret. `path:` never applies to `~/.envshed/config.json`. An agent can edit this file, so review changes to it.

## envshed agent session

Short-lived, read-only access to one environment, optionally to specific keys, without handing the agent your login. The server enforces the limits and records every read under the agent's name.

```bash
envshed agent session -e <env> [--ttl <duration>] [--keys <list>] [options]
envshed agent session list [-o <org>] [--output json]
envshed agent session revoke <id> [-o <org>]
```

| Option | Description |
|---|---|
| `-o`, `-p`, `-e` | Context. `-e` defaults to `ENVSHED_ENV`, then the project config. |
| `--ttl <duration>` | Lifetime such as `30m` or `2h`. Default `1h`, minimum 1 minute, maximum 24 hours. |
| `--keys <list>` | Comma-separated keys the session may read. Without it, every key in the environment. |
| `--agent <name>` | Name in the audit log. Defaults to `ENVSHED_AGENT_NAME`, then `claude-code` inside Claude Code, then `cli`. A label, not proof of identity. |
| `--reason <text>` | Shown to whoever approves. Never put a secret here. |
| `--timeout <duration>` | How long to wait for a production approval. Default `10m`. |
| `--no-wait` | Do not wait for approval. Exits `2`. |
| `--output json` | Never contains the token |

```bash
envshed agent session -e development --ttl 2h --keys API_KEY,DATABASE_URL
envshed run -e development -- npm test
```

How it behaves:

- The token is never printed. It is stored in the OS keychain, or `~/.envshed/sessions.json` (mode `0600`) without one. One local session per environment; starting another replaces the local copy (revoke the old one if you no longer trust it).
- Only `envshed run` uses a stored session, and only for the org, project and environment it was started for. `pull`, `secret get` and the rest keep using your login. `ENVSHED_TOKEN` still wins over a session.
- When a session expires, `envshed run` stops with an error instead of falling back to your login. Revoke it to go back to your login.
- A session can read the environment's secrets (allowed keys only; `${REF}` stays inside allowed keys), read one allowed secret, use secret requests for allowed keys, read the environment version and `whoami`. Everything else answers `403`.

**Production approval.** Envshed reads the words in the environment slug and name. A session starts without approval only when a word marks it as non-production (`dev`, `local`, `test`, `qa`, `staging`, `homolog`, `preview`, `sandbox`, with an optional trailing number) and no word marks it as production (`production`, `prod`, `prd`, `live`, `producao`, `produccion`). Unknown names count as production, so `main` asks for approval. A person who can edit secrets there approves in the dashboard; there is no API or CLI to approve. If the command timed out or used `--no-wait`, run the same command again to pick up the same request.

| Exit code | Meaning |
|---|---|
| 0 | Session active and stored |
| 1 | Error |
| 2 | Waiting for approval (`--no-wait`) |
| 3 | Declined |
| 4 | Stopped waiting, or the approval request expired |

`session list` shows live sessions (members see their own, owners and admins see all). `session revoke <id>` ends one on the server and removes the local copy; it cannot be undone.

## envshed scan

Search local agent histories for values from your vault and for known token formats. It reports key, file and date, never a value. By default it changes no file. It starts with Claude Code.

```bash
envshed scan [options]
```

| Option | Description |
|---|---|
| `-o`, `-p` | Context |
| `-e, --env <slug>` | Check only this environment. Default: every environment you can read. |
| `--path <path>` | Scan this file or directory instead of the agent history |
| `--formats-only` | Only known token formats, without reading the vault |
| `--no-mark` | Report only, do not mark secrets as exposed on the server |
| `--clean` | Replace found values in the files, after confirmation |
| `--yes` | Skip the `--clean` confirmation |
| `--undo [id]` | Restore lines changed by a clean (newest, or the given backup id) |
| `--forget-backups` | Delete all clean backups and their keys. A clean then cannot be undone. |
| `--output json` | One JSON document |

| Exit code | Meaning |
|---|---|
| 0 | Nothing found, or an undo finished |
| 1 | Error, or `--clean` was refused |
| 2 | Something was found, even if `--clean` removed it. It is still exposed until rotated. |

What it scans by default: `projects/` and `history.jsonl` in the Claude Code directory (`CLAUDE_CONFIG_DIR` or `~/.claude`), files ending in `.jsonl`, `.json`, `.txt`, `.md`, `.log`. Values are matched raw, JSON-escaped, URL-encoded and base64. Short or common values, org and project slug parts, placeholders, disabled secrets and environments you cannot read are skipped.

**Exposed mark.** Each vault value found marks that secret as **Exposed** on the server (key names only; files and values stay local). Marking needs write access. Saving a different value clears the mark. Use `--no-mark` for a local report only.

**Rotating.** The report prints the commands. Run them in a separate terminal, not in an agent chat:

```bash
envshed secret set API_KEY -e development
envshed request API_KEY -e development --reason "rotate after exposure"
```

A token found only by its format is not in the vault: revoke it with its provider.

**Cleaning** replaces each value with `[ENVSHED:KEY]` (or the format name). It needs confirmation, or `--yes` without a terminal or with `--output json`. Backups are kept in `~/.envshed/scan-backups/`, encrypted with a key in the OS keychain; without a keychain (CI, many containers, Windows, `ENVSHED_KEYCHAIN=off`) `--clean` refuses. `envshed scan --undo` restores the original lines. Cleaning does not replace rotating: a value an agent saw was probably sent to a model provider.

Finding nothing does not prove nothing leaked.

## Output masking in `envshed run`

In agent mode, `envshed run` replaces known values in stdout and stderr with `[ENVSHED:KEY_NAME]`. Agent mode is on when Claude Code sets `CLAUDE_CODE_CHILD_SESSION=1` or when `ENVSHED_AGENT_MODE` is set to anything other than `0`, `false`, `no`, `off`. Set `ENVSHED_AGENT_MODE=1` yourself for other agents and for MCP servers launched by Claude Code. Details in [run-command.md](run-command.md).
