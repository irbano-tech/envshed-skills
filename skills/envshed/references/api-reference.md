# API Reference

Envshed provides a REST API (v1) for programmatic access to secrets, environments, projects, and organizations.

## Authentication

All API requests require a Bearer token in the `Authorization` header:

```
Authorization: Bearer envshed_...
```

Token types:
- **Personal API tokens** — user-scoped, created via dashboard or `POST /api/v1/tokens`
- **Service tokens** — scoped to org/project/environment, created via CLI or API
- **Device flow tokens** — issued after browser-based device code auth

## Base URL

```
https://app.envshed.com/api/v1
```

Override with `ENVSHED_API_URL` or the `apiUrl` field in `.envshed.json`.

## Response Format

- Default: JSON
- Set `Accept: text/plain` to receive secrets in `.env` format

## Endpoints

### User

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/me` | Get current user info |

### Organizations

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/orgs` | List user's organizations |
| `GET` | `/orgs/{orgSlug}` | Get org details |
| `POST` | `/orgs` | Create organization |
| `GET` | `/orgs/{orgSlug}/members` | List org members |
| `POST` | `/orgs/{orgSlug}/members` | Invite member (body: `{email, role}`) |
| `PATCH` | `/orgs/{orgSlug}/members/{userId}` | Update member role |
| `DELETE` | `/orgs/{orgSlug}/members/{userId}` | Remove member |

### Projects

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/projects/{orgSlug}` | List projects |
| `POST` | `/projects/{orgSlug}` | Create project |
| `GET` | `/projects/{orgSlug}/{projectSlug}` | Get project |
| `PUT` | `/projects/{orgSlug}/{projectSlug}` | Update project |
| `DELETE` | `/projects/{orgSlug}/{projectSlug}` | Delete project |

### Environments

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/environments/{orgSlug}/{projectSlug}` | List environments |
| `POST` | `/environments/{orgSlug}/{projectSlug}` | Create environment |
| `GET` | `/environments/{orgSlug}/{projectSlug}/{envSlug}` | Get environment |
| `PUT` | `/environments/{orgSlug}/{projectSlug}/{envSlug}` | Update environment |
| `DELETE` | `/environments/{orgSlug}/{projectSlug}/{envSlug}` | Delete environment |
| `POST` | `/environments/{orgSlug}/{projectSlug}/{envSlug}/duplicate` | Clone environment |

### Secrets

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/secrets/{orgSlug}/{projectSlug}/{envSlug}` | List all secrets |
| `POST` | `/secrets/{orgSlug}/{projectSlug}/{envSlug}` | Create/update secrets (bulk) |
| `GET` | `/secrets/{orgSlug}/{projectSlug}/{envSlug}/{key}` | Get single secret |
| `PUT` | `/secrets/{orgSlug}/{projectSlug}/{envSlug}/{key}` | Update secret |
| `DELETE` | `/secrets/{orgSlug}/{projectSlug}/{envSlug}/{key}` | Delete secret |
| `POST` | `/secrets/{orgSlug}/{projectSlug}/{envSlug}/{key}/override` | Set personal override |
| `DELETE` | `/secrets/{orgSlug}/{projectSlug}/{envSlug}/{key}/override` | Remove override |

### Versions

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/versions/{orgSlug}/{projectSlug}/{envSlug}/{key}` | List versions |
| `POST` | `/versions/{orgSlug}/{projectSlug}/{envSlug}/{key}/rollback` | Rollback to version |

### Secret Definitions

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/secret-definitions/{orgSlug}/{projectSlug}` | List definitions |
| `POST` | `/secret-definitions/{orgSlug}/{projectSlug}` | Create definition |
| `PUT` | `/secret-definitions/{orgSlug}/{projectSlug}/{key}` | Update definition |
| `DELETE` | `/secret-definitions/{orgSlug}/{projectSlug}/{key}` | Delete definition |

### Secret Links

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/secret-links/{orgSlug}/{projectSlug}/{envSlug}` | Create link |
| `DELETE` | `/secret-links/{orgSlug}/{projectSlug}/{envSlug}` | Delete link |
| `GET` | `/secret-links/resolve` | Resolve linked secrets |

### Snapshots

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/snapshots/{orgSlug}/{projectSlug}/{envSlug}` | Create snapshot |
| `GET` | `/snapshots/{orgSlug}/{projectSlug}/{envSlug}` | List snapshots |
| `POST` | `/snapshots/{orgSlug}/{projectSlug}/{envSlug}/restore` | Restore snapshot |

### Service Tokens

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/service-tokens/{orgSlug}` | Create service token |
| `GET` | `/service-tokens/{orgSlug}` | List service tokens |
| `DELETE` | `/service-tokens/{orgSlug}?id={tokenId}` | Revoke token |

### Personal API Tokens

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/tokens` | List user's tokens |
| `POST` | `/tokens` | Create token |
| `DELETE` | `/tokens/{id}` | Delete token |

### Device Auth Flow

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/device` | Initiate device code flow |
| `GET` | `/device?device_code={code}` | Poll for token |

### Environment Version

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/version/{orgSlug}/{projectSlug}/{envSlug}` | Get environment version |
| `GET` | `/env-version/{envId}` | Get version by environment ID |

### Webhooks

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/webhooks/{orgSlug}` | List webhooks |
| `POST` | `/webhooks/{orgSlug}` | Create webhook |
| `PUT` | `/webhooks/{orgSlug}/{id}` | Update webhook |
| `DELETE` | `/webhooks/{orgSlug}/{id}` | Delete webhook |
| `POST` | `/webhooks/{orgSlug}/{id}/test` | Test webhook |

### Notifications

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/notifications` | Get user notifications |

## Roles

### Organization Roles

| Role | Permissions |
|------|-------------|
| `owner` | Full control, billing, delete org |
| `admin` | Manage members, projects, and settings |
| `member` | Create and manage own projects |
| `viewer` | Read-only access |

### Project Roles

| Role | Permissions |
|------|-------------|
| `admin` | Full project control |
| `editor` | Read/write secrets |
| `viewer` | Read-only |

### Environment Roles

| Role | Permissions |
|------|-------------|
| `admin` | Full environment control |
| `editor` | Read/write secrets |
| `viewer` | Read-only |
