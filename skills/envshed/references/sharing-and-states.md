# Secret States, References, Links and Sharing

## Disable, delete, placeholder

| | Disable | Delete | Placeholder |
|---|---|---|---|
| Value kept | Yes | No | No shared value |
| Shows in dashboard | Yes (greyed, toggleable) | No | Yes |
| Included in `pull` / `run` / API | No | No | No (key listed, waits for a personal override) |
| Reversible | Yes, `enable` | Restore from history or a snapshot | Set an override |
| Use for | Turning a variable off for a while | Removing it for good | A key each person fills in personally |

### Disabled secrets

```bash
envshed secret disable LEGACY_API_KEY            # prompts for confirmation
envshed secret disable LEGACY_API_KEY --yes -e production
envshed secret enable LEGACY_API_KEY
```

- Per environment. Disabling in `production` does not touch `staging`.
- `pull` and `run` omit the value and print `N disabled secrets skipped`. The collection API lists the key in a `disabled` array. Requesting a disabled secret directly returns `404`.
- A reference to a disabled secret resolves as missing.
- The dashboard can disable or enable several secrets at once.

### Placeholders

```bash
envshed secret set GITHUB_TOKEN --placeholder
```

A placeholder declares a key without a shared value. `pull` warns about placeholders without your override and prints links to set one. `push` to a placeholder saves your value as a **personal override**, so it does not change the shared value. `envshed request` exits `5` for a key that is a placeholder or disabled.

## Personal overrides

```bash
envshed secret override set LOCAL_DB_URL      # masked prompt, or stdin
envshed secret override remove LOCAL_DB_URL   # -y skips confirmation
```

Your override is visible only to you and replaces the team value in your `pull` and `run`. Overrides need a user token; service tokens cannot set or remove them. Share links share the team value, never overrides.

## Secret references

| Token | Meaning |
|---|---|
| `${KEY}` | Another secret in the same environment |
| `${env.KEY}` | A secret in another environment of the same project (`env` is its slug) |
| `$${KEY}` | Escape: the literal text `${KEY}` |

```ini
DB_USER=app
DB_PASS=<password>
DB_HOST=db.internal
DATABASE_URL=postgres://${DB_USER}:${DB_PASS}@${DB_HOST}:5432/app
API_BASE=${prod.API_URL}
```

- Stored verbatim and resolved on the server when read: `pull`, `run`, the single-secret API and dashboard export resolve them. Promote and copy keep the literal token, so it rebinds to the destination's values.
- Nesting is allowed up to a depth limit. Cycles are detected.
- Cross-environment references need read access to the target environment. A token scoped to one environment cannot read another through a reference.
- An unresolved reference stays literal in the value and is listed in `referenceErrors` (`key` or `env.key`). Agent sessions and machine tokens resolve only inside their allowed keys.
- Cross-project references are not supported. Use a secret link for that.

## Secret links (cross-project)

```bash
envshed link DATABASE_URL --to-project api --to-env development
envshed --workspace apps/web link SHARED_KEY --to-project api --to-env staging
envshed unlink DATABASE_URL
```

- Linked secrets form a link group (two or more members). A secret belongs to one group at a time.
- When you `push` a changed linked secret, the new value propagates to the other members.
- If the other side changed too, `push` asks: **Keep mine** (propagate yours), **Keep theirs** (accept the linked value), or **Unlink**.
- `unlink` keeps the current value. A group left with fewer than two members is deleted.
- In a monorepo use `-w` / `--workspace`; bulk mode (`--all`) is not supported. Needs permission to update secrets in the project.

## Share links

Give someone, with or without an Envshed account, a temporary encrypted view of some values. A link is a snapshot of the values at creation time.

```bash
envshed share [create] [KEY...] [options]
envshed share list [-o <org>]
envshed share revoke <id> [-o <org>] [-y]
envshed share open <url> [--file <path>] [-y]
```

`create` is the default subcommand, so `envshed share DATABASE_URL` works. For a key literally named `list`, `revoke`, `open` or `create`, write `envshed share create <KEY>`.

| `create` option | Description | Default |
|---|---|---|
| `-o`, `-p`, `-e` | Context | Config |
| `--expires <duration>` | Lifetime such as `15m`, `8h`, `7d`; maximum `30d` | `24h` |
| `--burn` | One-time link, destroyed after the first open | Off |
| `--max-views <n>` | Maximum opens (ignored with `--burn`) | Until expiry |
| `--password` | Password protection, prompted or read from stdin | Off |
| `--note <text>` | Note for the recipient and the list | None |
| `-y, --yes` | Skip the confirmation when sharing every secret | Off |
| `--output json` | JSON output | Table |

```bash
envshed share DATABASE_URL --expires 1h --burn
envshed share STRIPE_KEY STRIPE_WEBHOOK_SECRET --expires 3d --password --note "for the payments contractor"
envshed share --max-views 5 --yes        # every shareable secret in the environment
```

- With no keys, every shareable secret is included (confirmation unless `--yes`). Placeholders and disabled secrets are skipped and listed. At most 500 secrets per link.
- Needs read access to the environment. Service tokens cannot create share links.
- The URL looks like `https://app.envshed.com/s/<id>#<key>`. The key after `#` is shown once and never sent to the server; Envshed stores only a hash and cannot rebuild a lost link.
- Ten wrong passwords lock the link and wipe it. Revoked, used up, locked or expired links are wiped.
- `share list`: owners and admins see every link, others see their own. Creator or owner/admin can revoke, immediately and permanently.

**Opening** needs no login. Quote the URL, since `#` starts a shell comment:

```bash
envshed share open "https://app.envshed.com/s/<id>#<key>" --file .env
```

`--file` writes with mode `0600`. Without `--file` the values print to stdout, which is for a person's own terminal only; an agent must use `--file`. Opening counts as a view, and a one-time link asks for confirmation first.

## Secret definitions

Definitions document which keys a project expects, with a description and a required flag, without storing values. The MCP `list_secret_keys` tool reports a key defined for the project but absent from an environment as `missing`.

```bash
envshed definition list
envshed definition create DATABASE_URL -d "PostgreSQL connection string" --required
envshed definition update DATABASE_URL -d "Primary database" --no-required
envshed definition delete DATABASE_URL -y
```

The key is a positional argument. `update` needs at least one of `--description`, `--required` or `--no-required`. `delete` warns if active secrets use the key.

## Exposed

`envshed scan` marks a secret as **Exposed** when its value is found in an agent history. Saving a different value clears the mark. See [agent-safety.md](agent-safety.md#envshed-scan).
