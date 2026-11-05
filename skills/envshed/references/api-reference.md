# REST API Reference

Envshed has a REST API (v1). The CLI is built on it. Base URL:

```
https://app.envshed.com/api/v1
```

Every request needs a Bearer token:

```
Authorization: Bearer <token>
```

Personal tokens, service tokens, machine tokens and sessions all use this header, each with its own limits (see [ci-cd-service-tokens.md](ci-cd-service-tokens.md)). Machine tokens and sessions are refused on every route except reading their environment's secrets, reading one allowed secret, secret requests, the environment version and `/me`.

As an AI agent, do not print API responses that contain secret values. Use `envshed run` or a tool that returns only results.

## Secrets

| Method | Path | Description |
|---|---|---|
| `GET` | `/secrets/{org}/{project}/{env}` | All secrets in an environment (references resolved). `Accept: text/plain` returns `.env` format. |
| `PUT` | `/secrets/{org}/{project}/{env}` | Create or update secrets in bulk |
| `GET` | `/secrets/{org}/{project}/{env}/{key}` | One secret (references resolved). Disabled or missing keys return `404`. |
| `PUT` | `/secrets/{org}/{project}/{env}/{key}` | Update one secret |
| `DELETE` | `/secrets/{org}/{project}/{env}/{key}` | Delete one secret |
| `PUT` | `/secrets/{org}/{project}/{env}/{key}/override` | Set your personal override |
| `DELETE` | `/secrets/{org}/{project}/{env}/{key}/override` | Remove your personal override |

The collection response has `secrets`, `placeholders`, `version`, and when non-empty `disabled` (keys hidden on purpose) and `referenceErrors` (unresolved references):

```json
{
  "secrets": { "DATABASE_URL": "postgres://..." },
  "placeholders": [],
  "disabled": ["LEGACY_API_KEY"],
  "version": 42
}
```

## Versions and snapshots

| Method | Path | Description |
|---|---|---|
| `GET` | `/versions/{org}/{project}/{env}/{key}` | Version history of a secret |
| `POST` | `/versions/{org}/{project}/{env}/{key}/rollback` | Roll back, body `{"targetVersion": 2}` |
| `GET` | `/snapshots/{org}/{project}/{env}` | List snapshots |
| `POST` | `/snapshots/{org}/{project}/{env}` | Create a snapshot, body `{"name": "..."}` |
| `POST` | `/snapshots/{org}/{project}/{env}/restore` | Restore, body `{"snapshotId": "..."}` |

## Environment version

| Method | Path | Description |
|---|---|---|
| `GET` | `/version/{org}/{project}/{env}` | Current version (supports ETag / `304`, used by `run --watch`) |
| `GET` | `/env-version/{envId}` | Version by environment id |

## Organizations and members

| Method | Path | Description |
|---|---|---|
| `GET` | `/me` | Current user, or the token's type and scope |
| `GET` / `POST` | `/orgs` | List / create organizations |
| `GET` / `PUT` / `DELETE` | `/orgs/{org}` | Get / update / delete an organization |
| `GET` / `POST` | `/orgs/{org}/members` | List / invite members |
| `PATCH` / `DELETE` | `/orgs/{org}/members/{userId}` | Change role / remove |

## Projects and environments

| Method | Path | Description |
|---|---|---|
| `GET` / `POST` | `/projects/{org}` | List / create projects |
| `GET` / `PUT` / `DELETE` | `/projects/{org}/{project}` | Get / update / delete a project |
| `GET` / `POST` | `/environments/{org}/{project}` | List / create environments |
| `GET` / `PUT` / `DELETE` | `/environments/{org}/{project}/{env}` | Get / update / delete an environment |
| `POST` | `/environments/{org}/{project}/{env}/duplicate` | Duplicate with all secrets |

## Definitions and links

| Method | Path | Description |
|---|---|---|
| `GET` / `POST` | `/secret-definitions/{org}/{project}` | List / create definitions |
| `GET` / `PUT` / `DELETE` | `/secret-definitions/{org}/{project}/{key}` | Get / update / delete a definition |
| `GET` / `POST` / `DELETE` | `/secret-links/{org}/{project}/{env}` | Links for an environment |
| `POST` | `/secret-links/resolve` | Resolve link conflicts |

## Share links

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/share-links/{org}` | User token | Create a link for `keys` in `projectSlug` / `envSlug` (omit `keys` for all) |
| `GET` | `/share-links/{org}` | User token | List links visible to you |
| `DELETE` | `/share-links/{org}/{id}` | User token | Revoke |
| `GET` | `/shared/{id}` | None | Public status and metadata |
| `POST` | `/shared/{id}` | None | Open, body `{"key": "...", "password": "..."}` |

Service tokens cannot create share links.

## Tokens

| Method | Path | Description |
|---|---|---|
| `GET` / `POST` | `/tokens` | List / create personal tokens |
| `DELETE` | `/tokens?id={id}` | Delete a personal token |
| `GET` / `POST` | `/service-tokens/{org}` | List / create service tokens |
| `PATCH` | `/service-tokens/{org}/{tokenId}` | Update name, description or active status |
| `DELETE` | `/service-tokens/{org}?id={tokenId}` | Revoke |
| `POST` | `/device` | Start the CLI device-code flow |

Create a service token:

```bash
curl -X POST https://app.envshed.com/api/v1/service-tokens/my-org \
  -H "Authorization: Bearer $ENVSHED_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "CI Token", "scope": "project", "projectId": "<project-id>", "permission": "read"}'
```

The service-token routes do not list, update or revoke machine tokens. Manage those on the **Hosted agents** screen.

## Webhooks and notifications

| Method | Path | Description |
|---|---|---|
| `GET` / `POST` | `/webhooks/{org}` | List / create webhooks |
| `PUT` / `DELETE` | `/webhooks/{org}/{id}` | Update / delete |
| `POST` | `/webhooks/{org}/{id}/test` | Send a test event |
| `GET` | `/notifications` | Your notifications |

## Agents

Secret requests, agent sessions and the MCP server have their own routes. Use them through `envshed request`, `envshed agent session` and the MCP endpoint `https://app.envshed.com/api/mcp` (see [hosted-agents-mcp-oidc.md](hosted-agents-mcp-oidc.md)).

## Roles

| Level | Roles |
|---|---|
| Organization | `owner`, `admin`, `member`, `viewer`. Owners and admins can reach every project. |
| Project | `admin`, `editor`, `viewer` |
