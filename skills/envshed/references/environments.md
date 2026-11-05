# Environments

An environment is a named set of secrets in a project, such as `development`, `staging` or `production`. Each has its own values, version history and access rules. Environment slugs are shared across the organization: new projects get the environments listed under **Settings > Environments** (by default `development`, `staging` and `production`), and the dashboard lets you pick which ones to create.

## Lifecycle commands

```bash
envshed env list [-o <org>] [-p <project>] [--output json]
envshed env create -n <name> [-d <text>] [--output json]
envshed env update <slug> [-n <name>] [-d <text>] [--output json]
envshed env delete <slug> [-y] [--output json]
envshed env duplicate <source-slug> -n <name> [--output json]
```

- `env update` needs `--name` or `--description`. A new name changes the slug, and `.envshed.json` is updated if it was your `defaultEnv`.
- `env delete` removes the environment, all its secrets and its audit logs. It cannot be undone.
- `env duplicate` copies every secret into a new environment. The server checks read access to the source.
- `envshed envs` is a legacy alias of `env list`.

## Switching the active environment

```bash
envshed env checkout staging
envshed env switch staging      # alias
envshed env staging             # shorthand
```

Checkout verifies the environment exists (and offers to create it if not), updates `defaultEnv` in `.envshed.json`, and then pulls the new environment's secrets into your env file. If the file has unpushed changes against the current environment, it first asks what to do:

- **Push** the local changes to the current environment, then switch and pull.
- **Discard** them, then switch and pull.
- **Keep** the local file as it is and only change `defaultEnv`.
- **Cancel** the switch.

In a monorepo: inside a workspace directory only that workspace switches, at the root every workspace switches, and `envshed --workspace apps/web env checkout staging` switches one.

For a single command without switching, pass `-e <slug>` or set `ENVSHED_ENV`:

```bash
export ENVSHED_ENV=staging
envshed pull
envshed run -- npm start
```

`-e` always wins over `ENVSHED_ENV`.

## Renaming

```bash
envshed rename-env staging --name "Staging v2"           # slug follows the name
envshed rename-env staging --slug pre-prod               # slug only
envshed rename-env dev --name "Development" --slug development
```

- At least one of `--name` or `--slug`.
- Environments are organization-wide: the rename applies to every project that uses the slug, and the new slug must be unique in the organization. When other projects share the slug, an org owner or admin must do it.
- Secrets, service tokens and access rules keep working (they use the id). Update `defaultEnv` in any `.envshed.json` that used the old slug.
- Renaming can change whether agent sessions need approval, since that is read from the name.

## Environment version

```bash
envshed env-version [-o <org>] [-p <project>] [-e <env>]
# Version: 42 (updated 2/24/2026, 3:15:00 PM)
```

The version increases on every change. `envshed run --watch` polls it to know when to restart.

## Comparing

```bash
envshed diff staging production              # values masked as ***
```

See [secret-management.md](secret-management.md#diff).

## Production-like environments

There is no separate tier setting. Envshed reads words in the slug and name: `production`, `prod`, `prd`, `live`, `producao` and `produccion` mark production, while `dev`, `local`, `test`, `qa`, `staging`, `homolog`, `preview` and `sandbox` mark non-production. Unknown names count as production. This decides whether an agent session needs approval and whether an OIDC trust policy needs a ref or GitHub environment.

## Access

Org owners and admins can read every environment. Other members need explicit project or environment access, set in the dashboard. Project roles are `admin`, `editor` and `viewer`: admins and editors can fill secret requests, while viewers can read and ask.

## Tips

- Keep the same environment names across projects.
- Take a snapshot before bulk changes to production.
- Review `envshed diff` before promoting values between environments.
