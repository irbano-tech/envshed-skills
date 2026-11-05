# Tokens, CI/CD and Team Admin

## Token types

| Token | Owner | Scope | Lifetime | Use for |
|---|---|---|---|---|
| Personal token (`envshed login`, dashboard **Settings > API Tokens**) | A user | Everything the user can reach | Stops after 30 days without use (any request restarts the count); an API-set `expiresAt` also applies | Your own terminal |
| Service token | Organization | `org`, `project` or `environment`; `read` or `read_write` | Optional `--expires-in` days | Classic CI, Kubernetes, infrastructure |
| Machine token | Organization | One environment, optional keys, read-only | Stops after 90 days idle; optional maximum lifetime | Hosted agents and routines |
| Agent session | Creator | One environment, optional keys, read-only | 1 minute to 24 hours | An AI agent on your machine |
| OIDC session | Trust policy | Policy's environment and keys, read-only | Up to 60 minutes | GitHub Actions |

See [hosted-agents-mcp-oidc.md](hosted-agents-mcp-oidc.md) for machine tokens and OIDC, and [agent-safety.md](agent-safety.md) for agent sessions.

## Personal token commands

```bash
envshed login                               # browser device-code flow
envshed login --token-stdin < token.txt     # save an existing token
my-secret-tool read envshed-token | envshed login --token-stdin
envshed token set --token-stdin             # masked prompt at a terminal
envshed token show                          # masked
envshed token clear                         # remove from keychain and config file
envshed whoami
```

`envshed login --token <token>` and `envshed token set <token>` still work but are deprecated: the token shows up in the process list and shell history. Without a browser, `envshed login` keeps the URL and code on screen so you can approve from another device.

## Service tokens

```bash
envshed token create-service -o my-org -n "GitHub Actions" \
  -s environment --project-id <project-id> --environment-id <environment-id> \
  -p read --expires-in 90
```

| Option | Description | Default |
|---|---|---|
| `-o, --org <slug>` | Organization (required) | |
| `-n, --name <name>` | Token name (required) | |
| `-d, --description <text>` | Description | |
| `-s, --scope <scope>` | `org`, `project` or `environment` | `org` |
| `--project-id <id>` | Required for `project` and `environment` scope | |
| `--environment-id <id>` | Required for `environment` scope | |
| `-p, --permission <perm>` | `read` or `read_write` | `read` |
| `--expires-in <days>` | Expiration in days | No expiration |

```bash
envshed token list-service -o my-org
envshed token update-service <id> -o my-org -n "New name"
envshed token update-service <id> -o my-org --no-active    # deactivate
envshed token update-service <id> -o my-org --active       # reactivate
envshed token revoke-service -o my-org --id <id>           # permanent
```

The token is shown once. Service tokens keep working when the creator leaves. They cannot set personal overrides, manage members or webhooks, or create share links. `list-service` does not show machine tokens or OIDC sessions, and `update-service` / `revoke-service` refuse machine tokens.

Best practices: narrowest scope (`environment` over `project` over `org`), `read` unless the job pushes, an expiry of 30 to 90 days, one token per pipeline, rotate by creating a new token, updating CI, then revoking the old one.

## Using a token in CI

Set `ENVSHED_TOKEN` as a CI secret. It takes precedence over any saved login, and the CLI does not touch the keychain when `CI` is set.

```bash
envshed run -o my-org -p backend -e production -- ./scripts/deploy.sh
envshed pull -o my-org -p backend -e production -f .env --force   # if a tool needs a file
```

Prefer `envshed run`: nothing is written to disk. Add `--redact` when the job's logs should mask values, since GitHub only masks `secrets.X`.

### GitHub Actions with OIDC (no stored token)

```yaml
permissions:
  id-token: write
  contents: read
steps:
  - uses: actions/checkout@v6
  - run: npx envshed@0 run --oidc <policy-id> -- npm test
```

See [hosted-agents-mcp-oidc.md](hosted-agents-mcp-oidc.md#github-actions-with-oidc).

### GitHub Actions with a token

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: npm install -g envshed
      - name: Deploy
        env:
          ENVSHED_TOKEN: ${{ secrets.ENVSHED_TOKEN }}
        run: envshed run --redact -o my-org -p backend -e production -- npm run deploy
```

There is also the `envshed/gh-setup-envshed@v1` action, which exports secrets as masked environment variables or writes a file:

```yaml
- uses: envshed/gh-setup-envshed@v1
  with:
    token: ${{ secrets.ENVSHED_TOKEN }}
    org: my-org
    project: backend
    environment: production     # default: production
    export-to: env              # env (default) or file
    file-path: .env             # with export-to: file
```

Use a service or machine token here, not a personal token, which stops after 30 days without use.

### GitLab CI

```yaml
deploy:
  image: node:22-slim
  script:
    - npm install -g envshed
    - envshed run -o my-org -p backend -e production -- ./deploy.sh
```

Store `ENVSHED_TOKEN` as a masked CI/CD variable.

### Docker

Wrap both the build and the start command so no `.env` is baked into the image:

```dockerfile
# build stage: token and environment passed as build args from CI
ARG ENVSHED_ENVIRONMENT
ARG ENVSHED_TOKEN
RUN pnpm add -g envshed && envshed run -e $ENVSHED_ENVIRONMENT -- pnpm build

# runtime stage: copy .envshed.json, install envshed, inject at start
RUN pnpm add -g envshed
CMD ["sh", "-c", "exec envshed run -e $ENVSHED_ENVIRONMENT -- node server.js"]
```

Pass `ENVSHED_TOKEN` and `ENVSHED_ENVIRONMENT` to the container at runtime. Build args can end up in image metadata, so keep the token out of the final stage. With several arguments after `--`, `envshed run` starts `node` directly, so `SIGTERM` from `docker stop` reaches it.

### Vercel, Netlify and other build hosts

Wrap the build command and store `ENVSHED_TOKEN` in the host's environment settings:

```bash
envshed run -e production -- pnpm build
```

## Members

Requires an org admin or owner and a user token.

```bash
envshed member list -o my-org
envshed member invite dev@example.com -r member -o my-org
envshed member role dev@example.com -r admin -o my-org
envshed member remove dev@example.com -o my-org -y
```

Roles: `owner`, `admin`, `member`, `viewer`. You must outrank the member's current role to change it. Removing the last owner fails.

## Webhooks

Requires an org admin or owner and a user token.

```bash
envshed webhook list -o my-org
envshed webhook create --name deploy-hook --url https://hooks.example.com/envshed --events push,delete -o my-org
envshed webhook update <id> --events push --no-active -o my-org
envshed webhook test <id> -o my-org          # shows status and latency, never the body
envshed webhook delete <id> -o my-org -y
```

`create` needs `--name`, an HTTPS `--url` and `--events`. `--type` defaults to `http`. The signing secret is shown once.

Verify deliveries: `X-Envshed-Signature` is `t=<unix-seconds>,v1=<hex-hmac>`. Compute HMAC-SHA256 with the signing secret over `<timestamp>.<raw-body>`, compare in constant time, and reject timestamps more than five minutes off.

## Shell completions

```bash
envshed completions bash >> ~/.bash_completion
mkdir -p ~/.zsh/completions && envshed completions zsh > ~/.zsh/completions/_envshed
envshed completions fish > ~/.config/fish/completions/envshed.fish
```

Completions are static (commands and flags, not slugs). Re-run after upgrading the CLI.
