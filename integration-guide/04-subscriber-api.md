# Subscriber API

_Last verified: 2026-06-18_

Endpoints for subscribers to browse plans, manage subscriptions, check limits, report usage, and handle payments.

All endpoints require `Authorization: Bearer <jwt>` unless noted otherwise.

---

## Public Plans (No Auth Required)

Browse available plans without authentication. Useful for plan discovery and pricing pages.

### List Public Plans

```
GET /api/public/plans
```

Returns all plans with `visibility: "public"` and `is_active: true`, sorted by base price ascending. Includes nested metrics, pricing tiers, and limit overrides.

Response:
```json
{
  "data": [
    {
      "id": "plan_uuid",
      "application_id": "app_xyz",
      "application_name": "Acme API",
      "name": "Pro Monthly",
      "description": "Professional plan with expanded limits",
      "visibility": "public",
      "base_price": "29.00",
      "currency": "USD",
      "billing_period": "monthly",
      "trial_period_days": 14,
      "is_active": true,
      "metrics": [
        {
          "metric_id": "metric_messages",
          "name": "messages_per_day",
          "display_name": "Daily Messages",
          "limit": 100,
          "limit_type": "soft",
          "reset_period": "per_day",
          "pricing_type": "included",
          "pricing_tiers": [],
          "limit_overrides": []
        }
      ]
    }
  ]
}
```

### Get Public Plan Details

```
GET /api/public/plans/:id
```

Returns a single public plan with full details. Returns 404 if the plan doesn't exist, is not public, or is inactive.

---

## Plans (Authenticated)

### List Available Plans

```
GET /api/plans
```

Returns public plans plus your organization's private plans.

### Get Plan Details

```
GET /api/plans/:id
```

Returns full plan details including nested metrics, pricing tiers, and limit overrides.

---

## Subscriptions

### Create a Subscription

```
POST /api/subscriptions
```

Request:
```json
{
  "plan_id": "plan_pro_monthly",
  "stripe_customer_id": "cus_abc123"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `plan_id` | Yes | UUID of the plan to subscribe to |
| `stripe_customer_id` | No | Stripe customer ID for payment |

Response (201):
```json
{
  "data": {
    "id": "sub_abc123",
    "subscriber_type": "user",
    "subscriber_id": "user_123",
    "plan_id": "plan_pro_monthly",
    "plan_snapshot_id": "snap_123",
    "status": "pending",
    "current_period_start": "2024-01-15T00:00:00Z",
    "current_period_end": "2024-02-15T00:00:00Z",
    "on_trial": false,
    "plan": {
      "id": "plan_pro_monthly",
      "name": "Pro Monthly",
      "base_price": "29.00",
      "currency": "USD",
      "billing_period": "monthly"
    }
  }
}
```

Starting status depends on the plan's payment timing:

| Payment Timing | Starting Status | When It Becomes Active |
|----------------|-----------------|------------------------|
| Prepaid | `pending` | After first payment succeeds |
| Postpaid | `active` | Immediately (billed at cycle end) |
| Hybrid | `pending` | After base fee payment succeeds |

### Start a Trial

```
POST /api/subscriptions/start-trial
```

Request:
```json
{
  "plan_id": "plan_pro_monthly",
  "stripe_customer_id": "cus_abc123"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `plan_id` | Yes | UUID of the plan (must have `trial_period_days > 0`) |
| `stripe_customer_id` | No | Stripe customer ID |

Response (201): Same shape as Create Subscription, with `on_trial: true` and `trial_end_date` set.

Error cases:
- `400` — Plan has no trial period (`trial_period_days == 0`)
- `409` — Subscriber already has a non-cancelled subscription to this plan

### List Subscriptions

```
GET /api/subscriptions
```

Returns all subscriptions for the authenticated subscriber, with plan details.

### Get Subscription

```
GET /api/subscriptions/:id
```

### Change Plan (Upgrade/Downgrade)

```
POST /api/subscriptions/:id/change-plan
```

Request:
```json
{
  "plan_id": "plan_enterprise_monthly"
}
```

Changes take effect according to the plan's upgrade/downgrade behavior. Prorated charges are calculated automatically.

### Cancel Subscription

```
POST /api/subscriptions/:id/cancel
```

Request:
```json
{
  "cancellation_type": "end_of_cycle"
}
```

| Type | Behavior |
|------|----------|
| `"immediate"` | Cancel now, issue prorated credit |
| `"end_of_cycle"` | Cancel at end of current billing period (default) |

### Reinstate Subscription

```
POST /api/subscriptions/:id/reinstate
```

Reinstates a **suspended** subscription. No request body needed. The subscription must be in `suspended` status.

---

## Limits

### Get Effective Limits

```
GET /api/limits
```

Query parameters (choose one):

| Parameter | Mode | Description |
|-----------|------|-------------|
| `subscription_id` | Direct | Get limits for a specific subscription |
| `subscription_id` + `external_user_id` | Per-user | Get per-user limits within a subscription |
| `subscriber_id` | Subscriber | Get aggregated limits across all active subscriptions |
| *(none)* | Subscriber | Uses the authenticated token's identity |

**Direct Mode:**
```
GET /api/limits?subscription_id=sub_abc123
```

**Subscriber Mode:**
```
GET /api/limits?subscriber_id=user_123
```

**Per-User Mode:**
```
GET /api/limits?subscription_id=sub_abc123&external_user_id=user_456
```

Response (Subscriber Mode):
```json
{
  "data": {
    "subscriber_id": "user_123",
    "limits": {
      "messages_per_day": {
        "subscription_id": "sub_base",
        "metric_id": "metric_messages",
        "metric_name": "messages_per_day",
        "limit": 100,
        "used": 45,
        "remaining": 55,
        "limit_type": "soft",
        "recheck_after": 300
      }
    }
  }
}
```

Response fields:

| Field | Description |
|-------|-------------|
| `limit` | The current limit value (`null` if unlimited) |
| `used` | Usage count in the current reset period |
| `remaining` | Units remaining before limit is reached |
| `limit_type` | `"soft"`, `"hard"`, or `"unlimited"` |
| `recheck_after` | Suggested number of requests before rechecking with server |

---

## Usage

### Report Usage

```
POST /api/usage
```

**Direct Mode** (subscription_id + metric_id):
```json
{
  "subscription_id": "sub_abc123",
  "metric_id": "metric_messages",
  "count": 1,
  "idempotency_key": "req_abc123"
}
```

**Subscriber Mode** (subscriber_id + metric_name):
```json
{
  "subscriber_id": "user_123",
  "metric_name": "messages_per_day",
  "count": 1,
  "idempotency_key": "req_abc123"
}
```

**Passthrough cost** (for variable-cost metrics like LLM APIs):
```json
{
  "subscriber_id": "user_123",
  "metric_name": "llm_costs",
  "cost": 0.0847,
  "metadata": {
    "model": "gpt-4",
    "input_tokens": 1500,
    "output_tokens": 340
  },
  "idempotency_key": "req_xyz789"
}
```

All parameters:

| Field | Required | Description |
|-------|----------|-------------|
| `subscription_id` | Direct mode | Subscription UUID |
| `metric_id` | Direct mode | Metric UUID |
| `subscriber_id` | Subscriber mode (or omit for token identity) | Subscriber UUID |
| `metric_name` | Subscriber mode | Metric name string |
| `count` | No | Units consumed (for count-based metrics) |
| `cost` | No | Actual cost (for passthrough metrics) |
| `external_user_id` | No | Attribute usage to a specific end user |
| `idempotency_key` | No | Prevents duplicate reporting on retry |
| `metadata` | No | Arbitrary JSON context |

Response (201):
```json
{
  "data": {
    "id": "usage_evt_123",
    "subscription_id": "sub_abc123",
    "metric_id": "metric_messages",
    "count": 1,
    "reported_at": "2024-01-15T10:30:45Z",
    "metadata": {}
  },
  "routed_to": {
    "subscription_id": "sub_abc123",
    "metric_id": "metric_messages"
  }
}
```

The `routed_to` field shows which subscription and metric the usage was recorded against (useful in Subscriber Mode).

**Idempotency:** If you send the same `idempotency_key` within 24 hours, the duplicate is ignored and the original response is returned.

### Batch Usage Reporting

For high-volume applications, report multiple events in one request:

```
POST /api/usage/batch
```

Request:
```json
{
  "events": [
    {
      "subscriber_id": "user_123",
      "metric_name": "messages_per_day",
      "count": 5,
      "idempotency_key": "batch_1_evt_1"
    },
    {
      "subscriber_id": "user_456",
      "metric_name": "api_calls",
      "count": 3,
      "idempotency_key": "batch_1_evt_2"
    }
  ]
}
```

Response (201):
```json
{
  "data": [
    {"id": "usage_1", "subscription_id": "sub_abc", "metric_id": "metric_msg", "count": 5},
    {"id": "usage_2", "subscription_id": "sub_def", "metric_id": "metric_api", "count": 3}
  ]
}
```

**Note:** Batch is all-or-nothing for event resolution. If any event fails to resolve (e.g., unknown metric), the entire batch fails.

---

## Invoices

### List Invoices

```
GET /api/invoices
```

Query parameters:
- `subscription_id` — Filter to a specific subscription (optional)

Returns invoices sorted by billing period end date (newest first), with line items included.

Response:
```json
{
  "data": [
    {
      "id": "inv_abc123",
      "invoice_number": "INV-2024-00042",
      "subscription_id": "sub_xyz",
      "status": "paid",
      "currency": "USD",
      "subtotal": "29.00",
      "total": "34.50",
      "amount_paid": "34.50",
      "amount_due": "0.00",
      "billing_period_start": "2024-01-01T00:00:00Z",
      "billing_period_end": "2024-01-31T23:59:59Z",
      "issued_at": "2024-02-01T00:00:00Z",
      "due_at": "2024-02-08T00:00:00Z",
      "paid_at": "2024-02-01T10:30:00Z",
      "line_items": [
        {
          "id": "li_1",
          "type": "base_price",
          "description": "Pro Monthly - Base subscription",
          "quantity": "1",
          "unit_price": "29.00",
          "amount": "29.00"
        }
      ]
    }
  ]
}
```

### Get Invoice Details

```
GET /api/invoices/:id
```

Returns a single invoice with full line item details.

---

## Payment Methods

Payment methods are managed through Stripe. The Billing Server never handles card numbers directly.

### Create Setup Intent

```
POST /api/payment-methods/setup
```

Request:
```json
{
  "subscription_id": "sub_abc123"
}
```

The `subscription_id` is required to look up the Stripe customer and determine which connected Stripe account to use.

Response:
```json
{
  "data": {
    "client_secret": "seti_1234_secret_5678"
  }
}
```

Use the `client_secret` with Stripe.js to securely collect card details:

```javascript
const stripe = Stripe('pk_test_...', {
  stripeAccount: 'acct_connected_account'
});
const { setupIntent, error } = await stripe.confirmCardSetup(clientSecret, {
  payment_method: { card: cardElement }
});
```

### List Payment Methods

```
GET /api/payment-methods?subscription_id=sub_abc123
```

The `subscription_id` query parameter is required.

Response:
```json
{
  "data": [
    {
      "id": "pm_abc123",
      "type": "card",
      "card": {
        "brand": "visa",
        "last4": "4242",
        "exp_month": 12,
        "exp_year": 2025
      },
      "created": 1705312200
    }
  ]
}
```

### Set Default Payment Method

```
POST /api/payment-methods/:id/set-default
```

Request:
```json
{
  "subscription_id": "sub_abc123"
}
```

### Delete Payment Method

```
DELETE /api/payment-methods/:id
```

Requires `subscription_id` in the request body. Cannot delete the default payment method if there are active subscriptions.

---

## Portal Session

Create a magic link for end users to access the billing portal.

### Create Portal Session

```
POST /api/portal/session
```

Request:
```json
{
  "subscriber_id": "user_123",
  "subscriber_name": "Jane Doe",
  "subscriber_type": "user"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `subscriber_id` | Yes | The subscriber's UUID |
| `subscriber_name` | No | Display name for the portal |
| `subscriber_type` | No | `"user"` (default) or `"organization"` |

Response:
```json
{
  "data": {
    "portal_url": "https://billing.example.com/portal/auth?session_token=<token>",
    "expires_in": 300
  }
}
```

The `portal_url` is a one-time-use link that expires in 5 minutes. When the user visits this URL, they are authenticated into the billing portal where they can:
- View their subscriptions and usage
- Manage payment methods
- View and download invoices
