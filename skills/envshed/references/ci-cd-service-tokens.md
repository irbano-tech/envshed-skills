# CI/CD & Service Tokens

Guide for using envshed in CI/CD pipelines with service tokens for secure, scoped, machine-to-machine access.

## Service Tokens

Service tokens provide scoped, non-interactive authentication for CI/CD pipelines, Docker builds, and automated systems.

### Creating a Service Token

```bash
envshed token create-service \
  -o my-org \
  -n "GitHub Actions" \
  -d "Used by CI pipeline" \
  -s project \                   # Scope: org | project | environment
  --project-id <id> \
  -p read \                      # Permission: read | read_write
  --expires-in 365               # Days until expiration (optional)
```

### Scopes

| Scope | Access |
|-------|--------|
| `org` | All projects and environments in the organization |
| `project` | All environments in a specific project |
| `environment` | A single environment only |

### Permissions

| Permission | Access |
|------------|--------|
| `read` | Pull secrets only |
| `read_write` | Pull and push secrets |

### Managing Service Tokens

```bash
envshed token list-service -o my-org                          # List all service tokens
envshed token update-service <id> -o my-org -n "new name"     # Update name/description
envshed token update-service <id> -o my-org --no-active       # Deactivate token
envshed token update-service <id> -o my-org --active          # Reactivate token
envshed token revoke-service -o my-org --id <token-id>        # Revoke permanently
```

## Using Tokens in CI/CD

### Environment Variable

Set `ENVSHED_TOKEN` in your CI environment. This takes precedence over `~/.envshed/config.json`.

```bash
export ENVSHED_TOKEN=envshed_svc_...
envshed pull -e production -f .env.production
```

### Direct Token Login

```bash
envshed login --token $ENVSHED_TOKEN
envshed pull -e production
```

## CI/CD Examples

### GitHub Actions

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install envshed
        run: npm install -g envshed

      - name: Pull production secrets
        env:
          ENVSHED_TOKEN: ${{ secrets.ENVSHED_TOKEN }}
        run: envshed pull -e production -f .env.production

      - name: Build and deploy
        run: |
          npm ci
          npm run build
```

### GitLab CI

```yaml
deploy:
  stage: deploy
  variables:
    ENVSHED_TOKEN: $ENVSHED_TOKEN
  script:
    - npm install -g envshed
    - envshed pull -e production -f .env.production
    - npm ci && npm run build
```

### Docker Build

```dockerfile
# Multi-stage build — secrets only in build stage
FROM node:20 AS builder
RUN npm install -g envshed
ARG ENVSHED_TOKEN
RUN envshed pull -e production -f .env.production
RUN npm ci && npm run build

FROM node:20-slim
COPY --from=builder /app/dist ./dist
# .env.production is NOT copied to final image
CMD ["node", "dist/server.js"]
```

### Vercel

Set `ENVSHED_TOKEN` in Vercel project environment variables, then use a build command:

```bash
npx envshed pull -e production -f .env.production && next build
```

## API Token (Personal)

For personal automation (scripts, local tools), use API tokens instead of service tokens:

```bash
envshed token set <your-api-token>    # Store token
envshed token show                     # Display masked token
envshed token clear                    # Remove stored token
```

Personal API tokens can also be created via:
- The web dashboard
- The API: `POST /api/v1/tokens`

## Security Best Practices

- Use the **narrowest scope possible** — prefer `environment` scope over `project` or `org`
- Use **read-only** permission unless the pipeline needs to push secrets
- Set **expiration dates** on service tokens
- Rotate tokens regularly
- Store tokens in your CI provider's secrets manager (e.g., GitHub Secrets, GitLab CI Variables)
- Never commit tokens to source control
- Use separate tokens for each pipeline/environment
