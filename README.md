# envshed

Agent skill for [envshed](https://app.envshed.com) — a secrets management CLI for syncing environment variables across teams, projects, and environments.

## Install

```bash
npx skills add irbano-tech/envshed-skill
```

## What This Skill Does

When installed, this skill gives your AI coding agent deep knowledge of the envshed CLI, including:

- All commands, subcommands, flags, and options
- Configuration files (`.envshed.json`, `~/.envshed/config.json`)
- Monorepo workspace setup and management
- Secret operations (pull, push, run, get, set, delete, override)
- Environment switching, comparison, and access control
- Version history and rollback
- Snapshots (create, list, restore)
- Cross-project secret linking
- Service tokens for CI/CD pipelines
- Import from .env, JSON, YAML, and CSV
- REST API reference (all v1 endpoints)
- Common workflows and best practices

## When It Activates

The skill activates automatically when you ask about:

- Environment variables or secrets management
- `.env` files and configuration
- `envshed` commands or setup
- CI/CD secret injection
- Monorepo workspace configuration

## Structure

```
skills/envshed/
├── SKILL.md                              # Core skill with all commands and workflows
└── references/
    ├── workspaces-monorepo.md            # Monorepo setup and workspace management
    ├── secret-management.md              # Versions, snapshots, linking, definitions
    ├── environments.md                   # Environment switching, comparison, access
    ├── ci-cd-service-tokens.md           # CI/CD setup and service token management
    └── api-reference.md                  # Full REST API v1 endpoint reference
```

## License

MIT
