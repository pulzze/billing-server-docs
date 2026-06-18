# Authentication

_Last verified: 2026-06-18_

All API requests (except public endpoints) require a valid JWT token from Account Server.

## Token Types

The Billing Server accepts three types of JWT tokens, each resolving subscriber identity differently:

| Token Type | `type` claim | Subscriber Identity | Use Case |
|-----------|-------------|-------------------|----------|
| **User** | `"user"` | `{:user, sub}` | End users accessing their own subscriptions |
| **App** | `"app"` | `{:application, sub}` | Application-level access (service-to-service) |
| **Org** | `"org"` | `{:organization, org_id}` | Organization admin access |

The token type determines how endpoints like `/api/limits` and `/api/usage` resolve the subscriber when no explicit `subscriber_id` is provided.

## Making Authenticated Requests

Include the token in the `Authorization` header:

```bash
curl -X GET https://billing.example.com/api/limits \
  -H "Authorization: Bearer <account_server_jwt>" \
  -H "Content-Type: application/json"
```

The Billing Server validates the token with Account Server's JWKS and extracts:
- `sub` — the subscriber identity (user_id, application_id, or org_id)
- `org_id` — the subscriber's organization
- `client_id` — the application client ID (app tokens only)
- `app_name` — the application name (app tokens only)

## Public Endpoints (No Auth Required)

These endpoints do not require authentication:

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Service health check |
| `GET /api/public/plans` | Browse public plans |
| `GET /api/public/plans/:id` | Get public plan details |

## Service-to-Service Authentication

For backend services that need to manage subscriptions programmatically (e.g., auto-provisioning on user signup), use the Service API with an **app token**.

### How It Works

1. Your service authenticates with Account Server using OAuth client credentials
2. Your service receives an app-type JWT
3. Your service calls the Billing Server's `/api/service/*` endpoints
4. The Billing Server validates the JWT and checks the trusted services list

### Prerequisites

Your application must be registered as a **trusted service** with the Billing Server. Contact your administrator to add your application to the trusted services list.

### Obtaining a Service Token

```bash
# Get an app token from Account Server
curl -X POST https://account-server.example.com/oauth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "your_client_id",
    "client_secret": "your_client_secret"
  }'
```

The returned JWT will have `"type": "app"` and can be used with all Billing Server endpoints, including the `/api/service/*` scope.
