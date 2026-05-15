# Vendor API

Endpoints for application owners to manage metrics, plans, pricing, Stripe Connect, and webhook subscriptions.

All endpoints are under `/api/vendor/` and require `Authorization: Bearer <jwt>`.

---

## Metrics

### Register a Metric

```
POST /api/vendor/metrics
```

Request:
```json
{
  "name": "messages_per_day",
  "display_name": "Daily Messages",
  "description": "Number of messages sent per day",
  "unit_label": "messages"
}
```

Response (201):
```json
{
  "data": {
    "id": "metric_abc123",
    "application_id": "app_xyz",
    "name": "messages_per_day",
    "display_name": "Daily Messages",
    "unit_label": "messages",
    "inserted_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T10:30:00Z"
  }
}
```

### List Metrics

```
GET /api/vendor/metrics
```

Returns all metrics registered by your application.

### Get Metric

```
GET /api/vendor/metrics/:id
```

### Update Metric

```
PUT /api/vendor/metrics/:id
```

### Delete Metric

```
DELETE /api/vendor/metrics/:id
```

Returns 204 No Content. Cannot delete metrics referenced by active plans (returns 409).

### Sync Metrics (Recommended)

```
POST /api/vendor/metrics/sync
```

Syncs your application's metric definitions with the billing server. This is the recommended way to manage metrics during deployment.

Request:
```json
{
  "metrics": [
    {
      "name": "capability_calls",
      "display_name": "API Capability Calls",
      "unit_label": "calls",
      "description": "Total capability executions",
      "metric_type": "limit"
    },
    {
      "name": "llm_costs",
      "display_name": "LLM API Costs",
      "unit_label": "USD",
      "description": "OpenAI/LLM token costs",
      "metric_type": "balance"
    }
  ]
}
```

Response:
```json
{
  "results": {
    "created": [
      {"name": "capability_calls", "id": "metric_abc123"}
    ],
    "updated": [
      {"name": "llm_costs", "id": "metric_def456", "changes": ["description"]}
    ],
    "unchanged": [],
    "deprecated": [],
    "blocked": []
  },
  "sync_successful": true,
  "warnings": []
}
```

Returns 409 if `sync_successful` is false (blocked changes prevent sync).

**Sync behavior:**

| Change Type | Active Plans Reference It? | Result |
|-------------|---------------------------|--------|
| Add new metric | N/A | Created |
| Update `display_name` | Any | Updated (cosmetic) |
| Update `description` | Any | Updated (cosmetic) |
| Update `unit_label` | No | Updated |
| Update `unit_label` | Yes | Blocked |
| Remove metric | No active subscriptions | Deprecated |
| Remove metric | Has active subscriptions | Blocked |

---

## Plans

### Create a Plan

```
POST /api/vendor/plans
```

Request:
```json
{
  "application_id": "app_abc123",
  "name": "Pro Monthly",
  "description": "Professional plan with expanded limits",
  "visibility": "public",
  "base_price": "29.00",
  "currency": "USD",
  "billing_period": "monthly",
  "payment_timing": "prepaid",
  "trial_period_days": 14
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `application_id` | Yes | — | Application UUID (must have connected Stripe account) |
| `name` | Yes | — | Display name for the plan |
| `description` | No | null | Plan description |
| `visibility` | No | `"private"` | `"private"`, `"public"`, or `"unlisted"` |
| `subscriber_scope` | No | `"any"` | `"user"`, `"organization"`, `"application"`, or `"any"`. Restricts which `subscriber_type` may subscribe. `"any"` (default) accepts all types; otherwise the subscription's `subscriber_type` must match this scope or creation/change is rejected with `incompatible_subscriber_type` (HTTP 422). |
| `base_price` | No | `"0.00"` | Base subscription fee |
| `currency` | No | `"USD"` | Currency code |
| `billing_period` | No | `"monthly"` | `"monthly"`, `"quarterly"`, `"annual"`, or `"custom"` |
| `billing_period_days` | No | — | Required when billing_period is `"custom"` |
| `payment_timing` | No | `"prepaid"` | `"prepaid"`, `"postpaid"`, or `"hybrid"` |
| `trial_period_days` | No | `0` | Free trial duration (0-365 days) |
| `trial_end_behavior` | No | — | What happens when trial ends |
| `role` | No | null | Role identifier (e.g., `"default_free"`) for lookup via Service API |
| `is_active` | No | `true` | Whether the plan accepts new subscriptions |
| `skip_stripe_check` | No | `false` | Bypass Stripe Connect requirement (dev only) |

Response (201):
```json
{
  "data": {
    "id": "plan_xyz789",
    "application_id": "app_abc123",
    "name": "Pro Monthly",
    "visibility": "public",
    "base_price": "29.00",
    "currency": "USD",
    "billing_period": "monthly",
    "is_active": true
  }
}
```

### List Your Plans

```
GET /api/vendor/plans?application_id=app_abc123
```

The `application_id` query parameter is required.

### Get Plan Details

```
GET /api/vendor/plans/:id
```

Returns full plan details including nested metrics, pricing tiers, and limit overrides.

### Update a Plan

```
PUT /api/vendor/plans/:id
```

Accepts the same fields as create. Changes to pricing or limits only affect new subscriptions — existing subscriptions keep their original terms (plan snapshot).

### Delete a Plan

```
DELETE /api/vendor/plans/:id
```

Plans with active subscriptions cannot be deleted. Deactivate them instead:

```json
PUT /api/vendor/plans/:id
{
  "application_id": "app_abc123",
  "is_active": false
}
```

---

## Plan Metrics

Configure which metrics a plan includes and how they're priced.

### List Plan Metrics

```
GET /api/vendor/plans/:plan_id/metrics
```

### Add Metric to Plan

```
POST /api/vendor/plans/:plan_id/metrics
```

Request:
```json
{
  "metric_id": "metric_messages",
  "limit": 100,
  "limit_type": "soft",
  "reset_period": "per_day",
  "pricing_type": "included"
}
```

### Update Plan Metric

```
PUT /api/vendor/plans/:plan_id/metrics/:id
```

### Remove Metric from Plan

```
DELETE /api/vendor/plans/:plan_id/metrics/:id
```

---

## Pricing Tiers

Configure graduated pricing for tiered metrics.

### List Tiers

```
GET /api/vendor/plans/:plan_id/metrics/:plan_metric_id/tiers
```

### Create Tier

```
POST /api/vendor/plans/:plan_id/metrics/:plan_metric_id/tiers
```

Request:
```json
{
  "from_units": 0,
  "to_units": 1000,
  "unit_price": "0.02"
}
```

### Update Tier

```
PUT /api/vendor/plans/:plan_id/metrics/:plan_metric_id/tiers/:id
```

### Delete Tier

```
DELETE /api/vendor/plans/:plan_id/metrics/:plan_metric_id/tiers/:id
```

---

## Limit Overrides

Configure dynamic limit overrides that trigger when usage thresholds are reached.

### List Overrides

```
GET /api/vendor/plans/:plan_id/metrics/:plan_metric_id/overrides
```

### Create Override

```
POST /api/vendor/plans/:plan_id/metrics/:plan_metric_id/overrides
```

Request:
```json
{
  "trigger_metric_id": "metric_messages",
  "trigger_threshold_percent": 100,
  "override_limit": 2,
  "override_limit_type": "hard"
}
```

### Update Override

```
PUT /api/vendor/plans/:plan_id/metrics/:plan_metric_id/overrides/:id
```

### Delete Override

```
DELETE /api/vendor/plans/:plan_id/metrics/:plan_metric_id/overrides/:id
```

---

## Vendor Subscriptions

View subscriptions to your plans.

```
GET /api/vendor/subscriptions?application_id=app_abc123
```

Query parameters:
- `application_id` — Required. Filter by application
- `plan_id` — Optional. Filter by specific plan

---

## Stripe Connect

Before creating plans or accepting payments, each application must connect a Stripe account.

### Check Connection Status

```
GET /api/vendor/applications/:application_id/stripe/status
```

Response (not connected):
```json
{
  "data": {
    "connected": false,
    "application_id": "app_abc123"
  }
}
```

Response (connected):
```json
{
  "data": {
    "connected": true,
    "application_id": "app_abc123",
    "stripe_account_id": "acct_1234567890",
    "status": "active",
    "business_name": "Acme Corp",
    "email": "billing@acme.com",
    "charges_enabled": true,
    "payouts_enabled": true,
    "details_submitted": true,
    "connected_at": "2024-01-15T10:30:00Z"
  }
}
```

### Initiate Connection

```
POST /api/vendor/applications/:application_id/stripe/authorize
```

Request (optional prefill data):
```json
{
  "email": "billing@yourcompany.com",
  "business_name": "Your Company Inc",
  "country": "US"
}
```

Response:
```json
{
  "data": {
    "url": "https://connect.stripe.com/oauth/authorize?..."
  }
}
```

Redirect the user to the returned URL. After they authorize on Stripe, they'll be redirected back.

### Sync Account Details

Refresh the account status from Stripe:

```
POST /api/vendor/applications/:application_id/stripe/sync
```

### Access Stripe Dashboard

Get a login link to the Stripe Express dashboard:

```
GET /api/vendor/applications/:application_id/stripe/dashboard
```

Response:
```json
{
  "data": {
    "url": "https://connect.stripe.com/express/..."
  }
}
```

### Disconnect Account

```
POST /api/vendor/applications/:application_id/stripe/disconnect
```

Returns 204 No Content.

**Warning**: Disconnecting prevents the application from accepting new payments. Existing subscriptions will continue until their next billing cycle fails.

### Generic Payment Connect

```
POST /api/vendor/applications/:application_id/payment/connect
```

Provider-agnostic payment connection endpoint. Currently routes to Stripe Connect.

---

## Webhooks

Register webhook endpoints to receive billing events.

### Create Webhook Endpoint

```
POST /api/vendor/webhooks
```

Request:
```json
{
  "url": "https://your-app.com/webhooks/billing",
  "secret": "whsec_your_secret_key",
  "events": [
    "subscription.created",
    "subscription.cancelled",
    "payment.succeeded",
    "payment.failed",
    "limit.exceeded"
  ]
}
```

### List Webhook Endpoints

```
GET /api/vendor/webhooks
```

### Get Webhook Endpoint

```
GET /api/vendor/webhooks/:id
```

### Update Webhook Endpoint

```
PUT /api/vendor/webhooks/:id
```

### Delete Webhook Endpoint

```
DELETE /api/vendor/webhooks/:id
```

### Webhook Payload Format

```json
{
  "id": "evt_abc123",
  "type": "limit.exceeded",
  "created_at": "2024-01-15T10:30:00Z",
  "data": {
    "subscription_id": "sub_xyz",
    "subscriber_id": "user_123",
    "metric": "messages_per_day",
    "limit": 100,
    "used": 101,
    "limit_type": "soft"
  }
}
```

### Verifying Webhook Signatures

Webhooks are signed using HMAC-SHA256:

```python
import hmac
import hashlib

def verify_webhook(payload, signature, secret):
    expected = hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)
```

### Event Types

| Event | Description |
|-------|-------------|
| `subscription.created` | New subscription created |
| `subscription.upgraded` | Subscriber upgraded to a higher plan |
| `subscription.downgraded` | Subscriber downgraded to a lower plan |
| `subscription.cancelled` | Subscription was cancelled |
| `subscription.suspended` | Subscription suspended due to payment failure |
| `subscription.reinstated` | Suspended subscription was reinstated |
| `payment.succeeded` | Payment was successful |
| `payment.failed` | Payment attempt failed |
| `invoice.created` | New invoice generated |
| `invoice.paid` | Invoice was paid |
| `limit.exceeded` | A soft or hard limit was exceeded |
