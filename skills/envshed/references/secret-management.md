# Secret Management

Detailed guide for managing individual secrets, versions, snapshots, linking, definitions, and import/export.

## Individual Secret Operations

### Get, Set, Delete

```bash
envshed secret get DATABASE_URL               # Print secret value
envshed secret set API_KEY                    # Prompts for value (hidden input)
envshed secret delete OLD_KEY                 # Delete a secret
```

### Personal Overrides

Personal overrides let individual users set their own values for specific secrets without affecting the team. Useful for local database URLs, API keys to personal sandboxes, etc.

```bash
envshed secret override set LOCAL_DB    # Prompts for value (hidden input or stdin)
envshed secret override remove LOCAL_DB # Remove override (-y to skip confirmation)
```

Overrides are encrypted per-user and take precedence when pulling secrets.

## Breaking Change Detection

Both `pull` and `push` detect breaking changes:

- **Pull:** Warns if your local `.env` has been modified since the last pull. Options: overwrite, keep local, or cancel.
- **Push:** Warns if the remote environment has been updated by someone else since your last pull. Options: force push, pull first, or cancel.

Use `--force` to skip these checks.

## Version History & Rollback

Every secret change is versioned automatically.

### View History

```bash
envshed versions DATABASE_URL              # Last 20 versions
envshed versions API_KEY --limit 50        # More history
envshed versions SECRET -e production      # Specific environment
```

Output columns: Version number, Type (created/modified/rolled_back), Date, Comment.

### Rollback

```bash
envshed rollback DATABASE_URL --version 3
envshed rollback API_KEY --version 7 -e production
```

Creates a new version entry with type `rolled_back`.

## Snapshots

Snapshots capture the full state of an environment at a point in time.

### Create

```bash
envshed snapshot create                                      # Auto-named
envshed snapshot create -n "pre-migration"                   # Named
envshed snapshot create -n "pre-deploy-v2" -d "Description"  # With description
envshed snapshot create -e production -n "prod-backup"       # Specific env
```

### List

```bash
envshed snapshot list
envshed snapshot list -e production
```

Displays: ID, Name, Date Created.

### Restore

```bash
envshed snapshot restore <snapshot-id>
envshed snapshot restore abc123 -e production
```

Restores all secrets to the exact state captured in the snapshot.

## Secret Linking (Cross-Project)

Link secrets across projects so changes propagate automatically. When the source secret changes, linked secrets update too.

### Create a Link

```bash
envshed link DATABASE_URL --to-project shared-infra --to-env production
envshed link REDIS_URL --to-project shared-infra --to-env production -o my-org
```

### Remove a Link

```bash
envshed unlink DATABASE_URL
envshed unlink REDIS_URL -e staging
```

### Conflict Resolution During Push

When pushing changes to secrets that are linked across projects, envshed prompts:
1. **Keep mine** — overwrite the linked value
2. **Keep theirs** — discard your change
3. **Unlink** — break the link and keep both values independent

## Secret Definitions (Schema)

Definitions document what secrets a project expects — key name, whether it's required, and a description. Useful for onboarding and validation.

```bash
envshed definition list                                     # List all definitions
envshed definition create -k DATABASE_URL -d "PostgreSQL connection string" --required
envshed definition update -k DATABASE_URL -d "Updated description"
envshed definition delete -k DATABASE_URL
```

### Placeholder Detection

When pulling, envshed warns about secrets that are still placeholders (matching definitions but not yet set to real values). These typically need personal overrides set in the dashboard.

## Import Secrets

Import from multiple formats:

```bash
envshed import .env.local --format env       # dotenv format
envshed import secrets.json --format json    # JSON object {"KEY": "value"}
envshed import secrets.yaml --format yaml    # YAML mapping
envshed import secrets.csv --format csv      # CSV with key,value columns
```

Format is auto-detected from file extension if `--format` is omitted.

## Environment Comparison

Compare secrets between two environments:

```bash
envshed diff staging production                # Show key differences
envshed diff staging production --show-values  # Include actual values
```

Output shows which keys are added, removed, or modified between environments.

## Encryption

- Secrets are encrypted at rest using **AES-256-GCM**
- Each secret stores: `encrypted_value`, `iv` (initialization vector), `tag` (authentication tag)
- Personal overrides use the same encryption, keyed per-user
