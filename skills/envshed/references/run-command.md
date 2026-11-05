# envshed run

Run a command with secrets injected as environment variables. No `.env` file is written.

```bash
envshed run [-o <org>] [-p <project>] [-e <env>] [options] -- <command...>
```

| Option | Description | Default |
|---|---|---|
| `-o, --org <slug>` | Organization | From config |
| `-p, --project <slug>` | Project | From config |
| `-e, --env <slug>` | Environment (overrides `ENVSHED_ENV`) | From config |
| `--watch` | Restart the command when remote secrets change | Off |
| `--interval <seconds>` | Watch poll interval, minimum 10 | `30` |
| `--redact` | Mask secret values in stdout and stderr | On in agent mode and with `--oidc` |
| `--no-redact` | Do not mask, even in agent mode | |
| `--oidc <policy>` | Sign in with a GitHub Actions ID token and this trust policy id | `ENVSHED_OIDC_POLICY` |

## How it works

1. Fetches the environment's secrets. References (`${KEY}`, `${env.KEY}`) are already resolved by the server. Disabled secrets and placeholders without your override are not injected (a dim `N disabled secrets skipped` line is printed).
2. Merges them into the current environment. Secrets win over existing variables.
3. Removes the CLI's own `ENVSHED_TOKEN` and `ENVSHED_API_URL` from the child's environment.
4. Starts the command and exits with its exit code.

## Always put `--` before the command

Without `--`, `run` still reads its own options anywhere in the line: `envshed run node app.js -e prod` runs against `prod`, and `envshed run node app.js --no-redact` turns masking off. After `--`, every argument goes to the command unchanged: `envshed run -- node app.js -e prod` passes `-e prod` to `node`.

## Several arguments vs one argument

**Several arguments** go to the program exactly as your shell left them, like `env`, `sudo` or `docker exec`. No shell sits in between, so quotes, `$`, globs and parentheses inside an argument stay literal:

```bash
envshed run -- node -e "console.log('ready')"
envshed run -- awk '{print $1}' access.log
envshed run -- FOO=1 node server.js     # FOO is set for the program, as in a shell
```

**A single argument** is a shell line, run by `/bin/sh` (`cmd.exe` on Windows). Use it to read an injected variable, chain commands or redirect:

```bash
envshed run -- 'npm test && npm run build'
envshed run -- 'psql "$DATABASE_URL" -c "select 1"'
envshed run -- sh -c 'psql "$DATABASE_URL"'   # the same, spelled out
```

Before CLI 0.20.0, several arguments were joined into one shell line. Now `envshed run -- echo '$API_KEY'` prints the literal text. Write `envshed run -- 'echo $API_KEY'` to have the shell expand it (and never do that as an agent).

## Exit codes and signals

- `run` exits with the child's exit code, including death by signal.
- If the program cannot be found, `run` prints an error and exits `127`, as a shell would.
- With `--oidc`, exit `1` means no ID token was available, the exchange was refused, or options conflict.
- With several arguments the program starts directly, so signals such as `SIGTERM` from `docker stop` reach it and its graceful shutdown runs, on every Linux image and in images with no shell. For a shell line, put `exec` in front: `envshed run -- "exec node server.js > app.log"`.
- On Windows, `npm` and other `.cmd` files are found and started the same way.

## Watch mode

```bash
envshed run --watch -- npm start
envshed run --watch --interval 60 -- node server.js
```

The CLI polls the environment version (ETag, cheap when nothing changed). On a change it sends `SIGTERM`, waits for exit, re-fetches and starts the command again. Ctrl+C stops the child and the watcher. On a `429` the interval doubles (up to 5 minutes) without restarting. `--watch` cannot be combined with `--oidc`.

## Monorepos

- Inside a workspace directory: that workspace's secrets, run in that directory.
- At the root: you must pick one with `envshed --workspace <path> run -- <cmd>`. `--all` is not supported for `run`.

## Sessions and tokens

- If you started an [agent session](agent-safety.md#envshed-agent-session) for this org, project and environment, `run` uses it and says so. An expired session is an error, not a fallback to your login.
- `ENVSHED_TOKEN` takes precedence over a stored session.
- `--oidc` keeps its session in memory and does not pass it, or the variables used to ask GitHub for another ID token, to the command. `--oidc` together with `ENVSHED_TOKEN` is an error. See [hosted-agents-mcp-oidc.md](hosted-agents-mcp-oidc.md).

## Output masking

When a coding agent runs a command, its output lands in the agent's context. Masking replaces known values with `[ENVSHED:KEY_NAME]`:

```
DATABASE_URL=[ENVSHED:DATABASE_URL]
PORT=3000
```

When it is on (first match wins):

| Priority | Source | Effect |
|---|---|---|
| 1 | `--redact` / `--no-redact` | On / off |
| 2 | `--oidc` | On (GitHub only masks `secrets.X` values) |
| 3 | `ENVSHED_AGENT_MODE` | `0`, `false`, `no`, `off` turn it off; any other non-empty value turns it on |
| 4 | `CLAUDE_CODE_CHILD_SESSION=1` | On. Claude Code sets it in commands it launches. |

No TTY does not turn masking on, so scripts and CI see normal output. Set `ENVSHED_AGENT_MODE=1` for other agents, for MCP servers launched by Claude Code, and for Claude Code versions that do not set the variable. A machine token in GitHub Actions needs `--redact`.

What is matched: each value as is, URL-encoded, base64 (several variants), JSON-escaped, as a string literal and shell-quoted, and each line of a multi-line value. Not masked, to keep output readable: values under 8 bytes, common words like `development` or `localhost`, bare URL schemes, AWS regions, digit-only values under 12 characters, and low-entropy values. The CLI prints on stderr how many values are masked and skipped.

Not covered: values split by other text or color codes, partial values, other encodings, and anything written to a file or sent over the network. Masking is a guard against accidents, not a barrier against an agent with a shell.

What changes for the command while masking is on: stdout and stderr become pipes, so tools may drop colors and progress bars (stdin stays connected); trailing bytes that could start a value are held briefly (50 ms, or up to 2 s from 8 bytes on); after exit the CLI waits up to 2 s for remaining output.

## Examples

```bash
envshed run -- npm start
envshed run -e production -- npm run seed
envshed run -- docker compose up
envshed --workspace apps/api run -- npm start
envshed run -- 'npm test && npm run build'
```
