# envshed

Agent skill for [Envshed](https://app.envshed.com), a secrets manager and CLI for syncing environment variables across teams, projects and environments, and for keeping secret values out of AI agent chats.

## Install

```bash
npx skills add irbano-tech/envshed-skills
```

## What This Skill Does

When installed, this skill gives your AI coding agent working knowledge of Envshed and the `envshed` CLI (checked against CLI 0.22.0):

- Every command, subcommand and flag, with the current defaults
- Safety rules for AI agents: never print values, use `envshed run`, ask for missing secrets with `envshed request`
- `envshed run` in depth: argument handling, exit codes, signals, watch mode and output masking
- Agent protection: `envshed agent install` (Claude Code hooks and the `envshed-agent-safety` skill), temporary `envshed agent session` access with production approval, and `envshed scan` for values leaked into agent histories
- Hosted agents: the Envshed MCP server (OAuth sign-in, key names only), machine tokens and GitHub Actions OIDC (`envshed run --oidc`)
- Secret states and features: disabled secrets, placeholders, personal overrides, `${KEY}` and `${env.KEY}` references, cross-project links, share links, definitions
- Versions and rollback, snapshots, diff, and import from files or Doppler
- Configuration (`.envshed.json`, `~/.envshed/config.json`, keychain storage) and resolution order
- Monorepo workspaces
- Personal, service and machine tokens, token lifetime, CI/CD examples, members and webhooks
- REST API v1 routes

## When It Activates

The skill activates automatically when you ask about:

- Environment variables, secrets, or `.env` files
- `envshed` commands or setup
- A task that needs an API key or other credential
- Giving an AI agent access to secrets, or the Envshed MCP server
- CI/CD secrets, service tokens, machine tokens or GitHub Actions OIDC
- Sharing a secret, or a secret that may have leaked
- Monorepo workspace configuration

## Structure

```
skills/envshed/
├── SKILL.md                              # Agent rules, concepts, command index, workflows
└── references/
    ├── agent-safety.md                   # request, agent install, agent session, scan, .envshed-allow
    ├── run-command.md                    # envshed run: argv, exit codes, signals, watch, masking
    ├── hosted-agents-mcp-oidc.md         # MCP server, machine tokens, GitHub Actions OIDC
    ├── sharing-and-states.md             # Disabled, placeholders, overrides, references, links, share links
    ├── secret-management.md              # pull/push, single secrets, versions, snapshots, import, diff
    ├── environments.md                   # Environment lifecycle, checkout, rename, env-version
    ├── workspaces-monorepo.md            # Monorepo setup and workspace behavior
    ├── ci-cd-service-tokens.md           # Tokens, CI/CD, members, webhooks, completions
    └── api-reference.md                  # REST API v1 routes
```

The separate `envshed-agent-safety` skill, installed by `envshed agent install`, holds the short rule set an agent follows inside a project. This skill is consistent with it and covers the rest of the product.

## License

MIT
