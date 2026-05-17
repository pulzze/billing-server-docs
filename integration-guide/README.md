# Billing Integration Guide

This guide explains how external developers can integrate with the Interactor platform, from creating an account to making API calls.

## Overview

The Interactor platform consists of three core services:

| Service | Domain | Purpose |
|---------|--------|---------|
| Account Server | auth.interactor.com | Authentication, organizations, applications |
| Billing Server | billing.interactor.com | Plans, subscriptions, usage tracking |
| Interactor API | core.interactor.com | AI agents, workflows, service integrations |

## Integration Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER INTEGRATION FLOW                           │
└─────────────────────────────────────────────────────────────────────────────┘

Step 1: Register (auth.interactor.com)
    │
    ├── Create admin account + organization + application in one call
    └── Receive client_id and client_secret immediately
            │
            ▼
Step 2: Use API (core.interactor.com)   ← Start using immediately!
    │
    ├── Authenticate with client_id/client_secret
    ├── Auto-trial provisioned on first API call
    └── Access AI agents, workflows, and services
            │
            ▼
Step 3 (Optional): Upgrade Plan (billing.interactor.com)
    │
    └── Browse plans and upgrade before trial ends
```

### Streamlined Onboarding

The registration endpoint creates everything you need in a single call:

1. **Admin account** - Your login credentials
2. **Organization** - Represents your company (subscriptions are managed here)
3. **Default application** - Ready-to-use credentials (client_id + client_secret)

You receive your API credentials immediately and can start building right away.

### Auto-Provisioning

When your organization makes its first API call to Interactor, a **subscription is automatically created**. This means:

- **No manual subscription step required** - Start using the API immediately
- **Trial period** - If the plan includes a trial, you'll start in trial status
- **Full API access** - All features available based on your plan
- **Upgrade anytime** - Visit the billing portal to change plans or add payment

## Step 1: Register

Register an admin account with your organization and get API credentials in a single call.

```bash
curl -X POST https://auth.interactor.com/api/v1/admin/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@yourcompany.com",
    "password": "SecurePassword123!",
    "org_name": "yourcompany"
  }'
```

**Response:**
```json
{
  "message": "Registration successful. Please check your email to verify your account.",
  "admin_id": "adm_abc123",
  "organization": {
    "id": "org_xyz789",
    "name": "yourcompany"
  },
  "application": {
    "id": "app_def456",
    "name": "default-app",
    "client_id": "app_7d8f9a2b3c4e5f6g",
    "client_secret": "sec_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0"
  }
}
```

**Important:** Save the `client_secret` immediately - it is only shown once!

You can optionally specify a custom app name:

```bash
curl -X POST https://auth.interactor.com/api/v1/admin/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@yourcompany.com",
    "password": "SecurePassword123!",
    "org_name": "yourcompany",
    "app_name": "my-production-app"
  }'
```

### Login (if already registered)

```bash
curl -X POST https://auth.interactor.com/api/v1/admin/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@yourcompany.com",
    "password": "SecurePassword123!"
  }'
```

### Managing Organizations and Applications

You can update your organization and application names anytime:

**Update Organization:**
```bash
curl -X PATCH https://auth.interactor.com/api/v1/admin/orgs/yourcompany \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "display_name": "Your Company Inc"
  }'
```

**Update Application:**
```bash
curl -X PATCH https://auth.interactor.com/api/v1/admin/orgs/yourcompany/applications/$APP_ID \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "production-app",
    "display_name": "Production Application"
  }'
```

**Create Additional Applications:**
```bash
curl -X POST https://auth.interactor.com/api/v1/admin/orgs/yourcompany/applications \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "staging-app",
    "display_name": "Staging Application"
  }'
```

**Note:** Organization names must be lowercase, alphanumeric with hyphens only. Subscriptions are managed at the organization level - all applications share the same subscription.

## Step 2: Use the Interactor API

With your application credentials from registration, you can start using the API immediately. A trial subscription is automatically provisioned on your first API call.

### Get Application Token

```bash
curl -X POST https://auth.interactor.com/oauth/token \
  -H "Content-Type: application/json" \
  -d '{
    "grant_type": "client_credentials",
    "client_id": "app_7d8f9a2b3c4e5f6g",
    "client_secret": "sec_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0"
  }'
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

### Create an AI Assistant

```bash
curl -X POST https://core.interactor.com/api/v1/agents/assistants \
  -H "Authorization: Bearer $APP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Support Agent",
    "system_prompt": "You are a helpful customer support agent.",
    "model": "gpt-4"
  }'
```

**Response:**
```json
{
  "data": {
    "id": "asst_jkl012",
    "name": "Support Agent",
    "created_at": "2024-02-15T12:00:00Z"
  }
}
```

On your first API call, a trial subscription is automatically created for your organization. You'll see this in logs:
```
[info] Auto-provisioned trial subscription: sub_xyz789
```

### Start a Conversation

```bash
curl -X POST https://core.interactor.com/api/v1/agents/asst_jkl012/rooms \
  -H "Authorization: Bearer $APP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "context": {"user_id": "customer_123"}
  }'
```

### Send a Message

```bash
curl -X POST https://core.interactor.com/api/v1/agents/rooms/$ROOM_ID/messages \
  -H "Authorization: Bearer $APP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "How do I reset my password?"
  }'
```

## Step 3: Browse and Upgrade Plans (Optional)

Before your trial expires, browse available plans and upgrade. This endpoint does not require authentication.

```bash
curl https://billing.interactor.com/api/public/plans
```

**Response:**
```json
{
  "data": [
    {
      "id": "plan_starter",
      "name": "Interactor Starter",
      "description": "Perfect for small projects and experimentation",
      "base_price": "29.00",
      "currency": "USD",
      "billing_period": "monthly",
      "trial_period_days": 14,
      "metrics": [
        {
          "name": "capability_calls",
          "description": "API capability invocations",
          "limit": 10000,
          "limit_type": "hard",
          "reset_period": "monthly"
        },
        {
          "name": "agent_sessions",
          "description": "AI agent conversation sessions",
          "limit": 100,
          "limit_type": "soft",
          "reset_period": "monthly"
        }
      ]
    },
    {
      "id": "plan_pro",
      "name": "Interactor Pro",
      "description": "For growing applications with higher volume",
      "base_price": "99.00",
      "currency": "USD",
      "billing_period": "monthly",
      "trial_period_days": 14,
      "metrics": [
        {
          "name": "capability_calls",
          "limit": 100000,
          "limit_type": "hard"
        },
        {
          "name": "agent_sessions",
          "limit": 1000,
          "limit_type": "soft"
        }
      ]
    }
  ]
}
```

### Get Plan Details

```bash
curl https://billing.interactor.com/api/public/plans/plan_starter
```

### Upgrade to a Paid Plan

When ready to upgrade from your trial, subscribe to a paid plan:

```bash
curl -X POST https://billing.interactor.com/api/subscriptions \
  -H "Authorization: Bearer $APP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "plan_id": "plan_pro"
  }'
```

**Response:**
```json
{
  "data": {
    "id": "sub_ghi789",
    "plan_id": "plan_pro",
    "subscriber_type": "organization",
    "subscriber_id": "org_xyz789",
    "status": "active",
    "on_trial": false,
    "current_period_start": "2024-02-15T00:00:00Z",
    "current_period_end": "2024-03-15T00:00:00Z"
  }
}
```

**Note:** Subscriptions are at the **organization level** - all applications under your organization share the same subscription.

## Error Handling

### No Subscription (402 Payment Required)

This error occurs when:
- Auto-trial provisioning is disabled on the server
- Your trial has expired and no paid subscription is active
- The billing server is unavailable and fail-open is disabled

```json
{
  "error": "subscription_required",
  "message": "An active subscription is required to access this API",
  "subscribe_url": "https://billing.interactor.com/org/marketplace"
}
```

**Solution:** Visit the billing portal to upgrade your subscription or contact support if you believe this is an error.

### Invalid Token (401 Unauthorized)

```json
{
  "error": "invalid_token",
  "message": "The access token is invalid or expired"
}
```

**Solution:** Refresh your token using the client credentials flow.

### Rate Limited (429 Too Many Requests)

```json
{
  "error": "rate_limited",
  "message": "Too many requests",
  "retry_after": 60
}
```

**Solution:** Wait for the specified time before retrying.

### Usage Limit Exceeded (429)

```json
{
  "error": "limit_exceeded",
  "message": "Monthly capability_calls limit exceeded",
  "limit": 10000,
  "current_usage": 10001,
  "reset_at": "2024-03-01T00:00:00Z"
}
```

**Solution:** Upgrade your plan or wait for the limit to reset.

## Code Examples

### Python

```python
import requests

# Configuration
AUTH_URL = "https://auth.interactor.com"
BILLING_URL = "https://billing.interactor.com"
API_URL = "https://core.interactor.com"
CLIENT_ID = "app_your_client_id"
CLIENT_SECRET = "sec_your_client_secret"

# Get access token
def get_token():
    response = requests.post(f"{AUTH_URL}/oauth/token", json={
        "grant_type": "client_credentials",
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET
    })
    return response.json()["access_token"]

# Create AI assistant
def create_assistant(token, name, system_prompt):
    response = requests.post(
        f"{API_URL}/api/v1/agents/assistants",
        headers={"Authorization": f"Bearer {token}"},
        json={
            "name": name,
            "system_prompt": system_prompt
        }
    )
    return response.json()["data"]

# Usage
token = get_token()
assistant = create_assistant(token, "My Agent", "You are helpful.")
print(f"Created assistant: {assistant['id']}")
```

### JavaScript/Node.js

```javascript
const axios = require('axios');

const AUTH_URL = 'https://auth.interactor.com';
const API_URL = 'https://core.interactor.com';
const CLIENT_ID = 'app_your_client_id';
const CLIENT_SECRET = 'sec_your_client_secret';

// Get access token
async function getToken() {
  const response = await axios.post(`${AUTH_URL}/oauth/token`, {
    grant_type: 'client_credentials',
    client_id: CLIENT_ID,
    client_secret: CLIENT_SECRET
  });
  return response.data.access_token;
}

// Create AI assistant
async function createAssistant(token, name, systemPrompt) {
  const response = await axios.post(
    `${API_URL}/api/v1/agents/assistants`,
    { name, system_prompt: systemPrompt },
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.data.data;
}

// Usage
(async () => {
  const token = await getToken();
  const assistant = await createAssistant(token, 'My Agent', 'You are helpful.');
  console.log(`Created assistant: ${assistant.id}`);
})();
```

### Elixir

```elixir
defmodule InteractorClient do
  @auth_url "https://auth.interactor.com"
  @api_url "https://core.interactor.com"
  @client_id "app_your_client_id"
  @client_secret "sec_your_client_secret"

  def get_token do
    body = Jason.encode!(%{
      grant_type: "client_credentials",
      client_id: @client_id,
      client_secret: @client_secret
    })

    {:ok, response} = HTTPoison.post(
      "#{@auth_url}/oauth/token",
      body,
      [{"Content-Type", "application/json"}]
    )

    Jason.decode!(response.body)["access_token"]
  end

  def create_assistant(token, name, system_prompt) do
    body = Jason.encode!(%{name: name, system_prompt: system_prompt})

    {:ok, response} = HTTPoison.post(
      "#{@api_url}/api/v1/agents/assistants",
      body,
      [
        {"Authorization", "Bearer #{token}"},
        {"Content-Type", "application/json"}
      ]
    )

    Jason.decode!(response.body)["data"]
  end
end

# Usage
token = InteractorClient.get_token()
assistant = InteractorClient.create_assistant(token, "My Agent", "You are helpful.")
IO.puts("Created assistant: #{assistant["id"]}")
```

## Managing Your Subscription

### View Subscription Status

```bash
curl https://billing.interactor.com/api/subscriptions \
  -H "Authorization: Bearer $APP_TOKEN"
```

### Check Usage Limits

```bash
curl https://billing.interactor.com/api/limits \
  -H "Authorization: Bearer $APP_TOKEN"
```

**Response:**
```json
{
  "data": {
    "metrics": [
      {
        "name": "capability_calls",
        "limit": 10000,
        "current_usage": 2500,
        "remaining": 7500,
        "reset_at": "2024-03-01T00:00:00Z",
        "limit_type": "hard"
      }
    ]
  }
}
```

### Change Plan

```bash
curl -X POST https://billing.interactor.com/api/subscriptions/$SUB_ID/change-plan \
  -H "Authorization: Bearer $APP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"plan_id": "plan_pro"}'
```

### Cancel Subscription

```bash
curl -X POST https://billing.interactor.com/api/subscriptions/$SUB_ID/cancel \
  -H "Authorization: Bearer $APP_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"cancellation_type": "end_of_cycle"}'
```

## Web Portal Access

For a visual interface, access the Billing Portal:

1. Go to https://billing.interactor.com/org/login
2. Sign in with your admin credentials (SSO via Account Server)
3. Navigate to:
   - **My Subscriptions** - View and manage your subscriptions
   - **Marketplace** - Browse and subscribe to available plans
   - **Payment Methods** - Manage payment information
   - **Invoices** - View billing history

## Security Best Practices

1. **Never expose client_secret in client-side code** - Use server-side authentication only
2. **Rotate secrets periodically** - Use the secret rotation endpoint
3. **Use environment variables** - Never hardcode credentials
4. **Implement token refresh** - Tokens expire after 1 hour
5. **Monitor usage** - Check limits proactively to avoid service interruption

## Support

- **Documentation:** https://docs.interactor.com
- **API Reference:** https://api.interactor.com/docs
- **Support:** support@interactor.com
- **Status:** https://status.interactor.com
