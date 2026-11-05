# Hosted Agents: MCP, Machine Tokens and OIDC

How an AI agent, a hosted agent, a scheduled routine or a CI job gets access to Envshed without a person pasting values.

## Which one to use

A hosted agent or routine runs without a person. Choose the first that fits:

1. **MCP OAuth, where a person can approve once.** No token to store, and it can be disconnected at any time.
2. **OIDC workload identity, on GitHub Actions.** No stored credential: the job proves which repository it runs in and gets a read-only session of at most an hour.
3. **A machine token, everywhere else.** Read-only, one environment, optional key limits, stops after 90 days without use, rotates with no downtime.

A plain [service token](ci-cd-service-tokens.md) also works, but it has no key limits and no idle expiry. Avoid calendar-expiry tokens for routines: they cause outages when they run out and do not limit exposure if they leak early.

For an interactive coding agent on your own machine, use [`envshed agent session`](agent-safety.md#envshed-agent-session) instead.

## MCP server

The Envshed MCP server lets an agent work with secrets without the CLI. **No tool returns a secret value and no tool accepts one.** It deals only in key names and statuses. It is available on every plan.

- Endpoint: `https://app.envshed.com/api/mcp` (Streamable HTTP)
- Sign-in: OAuth in the browser (authorization code with PKCE). Access tokens last 1 hour, refresh tokens rotate and last 30 days from last use.

### Connect Claude Code

```bash
claude mcp add --transport http envshed https://app.envshed.com/api/mcp
```

Then run `/mcp` inside Claude Code and follow the browser steps, or run `claude mcp login envshed`. Approve only a connection you started. Other MCP clients use the same URL.

Disconnect a client in the dashboard under **Settings > API Tokens > Connected MCP clients > Revoke**.

### Tools

| Tool | What it does |
|---|---|
| `list_organizations` | Organizations this connection can reach |
| `list_projects` | Projects in an organization |
| `list_environments` | Environments in a project |
| `list_secret_keys` | Key names in an environment, each `set`, `placeholder`, `disabled` or `missing` |
| `request_secret` | Ask a person for a secret and get the link where they enter it |
| `get_secret_request_status` | Whether a request is pending, fulfilled, declined or expired |
| `list_secret_requests` | Requests still waiting in an environment |
| `decline_secret_request` | Decline a request (needs write access) |
| `list_agent_sessions` | Live agent sessions (signed-in people only) |
| `revoke_agent_session` | End an agent session (signed-in people only) |

Every tool applies the same permissions as the REST API. A `missing` key is defined for the project but absent from that environment. Each call goes to the audit log and counts toward the organization's API rate limit.

### MCP with a token (no browser)

Pass a machine token or scoped service token as a Bearer header. In Claude Code `.mcp.json`, keep the token in a platform secret:

```json
{
  "mcpServers": {
    "envshed": {
      "type": "http",
      "url": "https://app.envshed.com/api/mcp",
      "headers": {
        "Authorization": "Bearer ${ENVSHED_TOKEN}"
      }
    }
  }
}
```

The token keeps its limits. Tokens cannot list or revoke agent sessions, and machine tokens and sessions cannot decline requests. A personal CLI token from `envshed login` is refused here on purpose.

### Getting values at runtime

The MCP server never hands out values. To run a program with its secrets, store a machine token as `ENVSHED_TOKEN` and run it through the CLI:

```bash
envshed run -- npm start
```

An agent with no shell can call a tool you write. The tool uses the Node SDK and returns only the result:

```typescript
import { EnvshedSecretsClient } from "@envshed/node";

const envshed = new EnvshedSecretsClient({
  token: process.env.ENVSHED_TOKEN!,
  org: "acme",
  project: "billing",
  env: "production",
});

export async function createInvoice(amount: number) {
  const { secrets } = await envshed.secrets.list();
  const response = await fetch("https://api.example.com/invoices", {
    method: "POST",
    headers: { Authorization: `Bearer ${secrets.BILLING_API_KEY}` },
    body: JSON.stringify({ amount }),
  });
  return { ok: response.ok, status: response.status };
}
```

A third-party connector with no shell and no code of your own cannot get values from Envshed today.

## Machine tokens

A read-only credential for a hosted agent or recurring routine. It reads one environment, optionally only certain keys, and the server enforces those limits. The organization owns it, so it survives the creator leaving.

**Create (dashboard only, owners and admins):** open the organization, choose **Hosted agents**, then **Connect a hosted agent**. Pick project, environment and optional keys. Copy the token (shown once) and the config snippet for GitHub Actions, a platform secret or `.mcp.json`.

**Use:** store it as `ENVSHED_TOKEN` on the platform.

```bash
envshed run -o acme -p billing -e production -- npm start
```

**What it can do:** the same routes as an agent session: read the environment's secrets (allowed keys only), read one allowed secret, secret requests for allowed keys, environment version, `whoami`. Everything else answers `403`.

**Idle expiry:** stops after 90 days without a request. An owner or admin can also set a maximum lifetime on the **Hosted agents** screen.

**Rotate:** choose **Rotate**. The replacement has the same scope; the old token keeps working for 24 hours.

**Revoke:** on the **Hosted agents** screen. `envshed token list-service` does not list machine tokens, and `revoke-service` and `update-service` refuse them.

## GitHub Actions with OIDC

Run a job with its secrets and no stored `ENVSHED_TOKEN`. The job presents its GitHub ID token and gets a short-lived, read-only session.

```bash
envshed run --oidc <policy-id> [-o <org>] [-p <project>] [-e <env>] -- <command...>
```

Or set `ENVSHED_OIDC_POLICY` instead of the flag (the flag wins).

### Set it up

1. Find the repository and owner ids: `gh api repos/OWNER/REPO --jq '.id, .owner.id'`.
2. In the dashboard, open the organization, choose **Hosted agents**, then **New trust policy** (owners and admins). Set the project and environment, optional keys, the repository and its ids, and optionally the exact ref (such as `refs/heads/main`), the workflow file and the GitHub environment. Sessions last up to 60 minutes. Policies cannot be edited: create a new one and delete the old.
3. Use it in the workflow:

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v6
  - name: Run with secrets
    run: npx envshed@0 run --oidc <policy-id> -- npm test
```

The policy id is not a secret. Without `id-token: write` the command stops and says so. The policy decides org, project and environment, so no `.envshed.json` or flags are needed; a flag naming something else is an error.

Set a ref (a protected branch) or a GitHub environment for anything sensitive. Without one, any branch can sign in. A policy needs one of the two unless the environment's name marks it as non-production.

### Behavior

- Read-only, limited to the policy's environment and keys, same routes as an agent session.
- Output masking is on under `--oidc` (`--no-redact` turns it off).
- `--watch` is refused, since a session lasts at most an hour. Use a machine token for long-running processes.
- `ENVSHED_TOKEN` set at the same time is an error.
- Only `push`, `pull_request`, `merge_group`, `schedule`, `workflow_dispatch`, `repository_dispatch`, `release` and `deployment` events are accepted. `pull_request_target`, `issue_comment` and `workflow_run` are refused.
- Disabling or deleting the policy ends its live sessions on their next request.
- GitHub Enterprise Server and customized issuers are not supported.

## What none of this protects against

Whatever a token or session can read, an agent holding it can read too, for example with `envshed run -- printenv`. Keep keys limited to what the job needs and store tokens as platform secrets. A person can still paste a value into a chat, and a password with no recognizable shape can still end up in a tool argument.
