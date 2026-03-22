# Environment Management

Detailed guide for creating, switching, listing, comparing, and managing environments in envshed.

## What Are Environments?

Environments are named sets of secrets within a project (e.g., `development`, `staging`, `production`). Each environment has its own secret values, version history, and access controls.

## Switching Environments

### Basic Switch

```bash
envshed env checkout staging         # Switch to staging
envshed env checkout production      # Switch to production
```

The `checkout` subcommand is aliased as `switch`.

### Change Detection on Switch

When switching environments, if local changes exist against the current environment, envshed prompts:

1. **Push changes** — push your local `.env` to the current environment, then switch
2. **Discard** — discard local changes and switch
3. **Keep** — keep local files unchanged, only update `.envshed.json` config
4. **Cancel** — abort the switch

### Auto-Create

If the target environment doesn't exist on the server, envshed offers to create it automatically.

### Workspace-Specific Switch

```bash
envshed env checkout staging -w apps/api  # Switch only apps/api to staging
```

## Listing Environments

```bash
envshed envs                         # Uses .envshed.json context
envshed envs -o my-org -p my-project # Explicit org/project
```

## Environment Version

Each environment has a version number that increments on every secret change. Use this to detect staleness.

```bash
envshed env-version                  # Current project's default env
envshed env-version -e production    # Specific environment
```

Output: version number and last update timestamp.

## Renaming Environments

```bash
envshed rename-env my-env -n "New Name"          # Rename display name
envshed rename-env my-env -s new-slug             # Change slug
envshed rename-env my-env -n "New Name" -s new-slug  # Both
```

## Comparing Environments

```bash
envshed diff staging production                # Show key differences
envshed diff staging production --show-values  # Include actual values
```

The diff output shows:
- Keys only in the first environment
- Keys only in the second environment
- Keys present in both but with different values

## Environment Duplication

Clone an environment with all its secrets:

```bash
envshed env duplicate staging -n "staging-copy"
envshed env duplicate production -n "pre-release" -o my-org -p my-project
```

Also available via API: `POST /api/v1/environments/{orgSlug}/{projectSlug}/{envSlug}/duplicate`

## Access Control

Environments support role-based access control:

| Role | Permissions |
|------|-------------|
| **admin** | Full control: read, write, delete secrets, manage access |
| **editor** | Read and write secrets |
| **viewer** | Read-only access to secrets |

Access is configured per-environment in the web dashboard.

## Staleness Detection

When you `pull` or `run`, envshed compares the remote environment version against your local version cache (`~/.envshed/version-cache.json`). If the remote is newer, it warns you that secrets may be stale.

## Best Practices

- Use consistent environment names across projects (e.g., `development`, `staging`, `production`)
- Create snapshots before making bulk changes to production environments
- Use `envshed diff` to review differences before promoting secrets between environments
- Set up environment-level access control to restrict who can modify production secrets
