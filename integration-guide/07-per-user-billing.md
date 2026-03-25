# Per-User Billing

Manage individual limits and credit balances for end users within a subscription using **allocation profiles**.

## How It Works

Per-user billing uses a two-level model:

1. **Allocation Profiles** — Reusable templates that define per-user limits and balances for each metric
2. **Allocations** — Assign individual users to a profile, giving them the profile's limits

```
Subscription
  └── Allocation Profile: "Standard User"
  │     ├── messages_per_day: hard limit 100
  │     └── llm_costs: balance $10.00
  │
  └── Allocation Profile: "Power User"  (default)
  │     ├── messages_per_day: hard limit 500
  │     └── llm_costs: balance $50.00
  │
  └── Allocations
        ├── user_alice → "Power User" profile
        ├── user_bob → "Standard User" profile
        └── user_carol → "Power User" profile (default)
```

---

## Allocation Profiles

Profiles are reusable templates. Create profiles once, then assign many users to them.

### List Profiles

```
GET /api/subscriptions/:subscription_id/profiles
```

Response:
```json
{
  "data": [
    {
      "id": "prof_abc123",
      "subscription_id": "sub_xyz",
      "name": "Standard User",
      "description": "Default limits for regular users",
      "is_default": false,
      "metadata": {},
      "metrics": [
        {
          "metric_id": "metric_msg",
          "allocation_type": "limit"
        }
      ],
      "user_count": 15,
      "created_at": "2024-01-15T10:30:00Z",
      "updated_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

### Create Profile

```
POST /api/subscriptions/:subscription_id/profiles
```

Request:
```json
{
  "name": "Power User",
  "description": "Higher limits for premium users",
  "is_default": true,
  "metadata": {"tier": "premium"}
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `name` | Yes | — | Profile display name |
| `description` | No | null | Description |
| `is_default` | No | `false` | New allocations without an explicit profile use the default |
| `metadata` | No | `{}` | Arbitrary JSON metadata |

### Get Profile

```
GET /api/subscriptions/:subscription_id/profiles/:id
```

### Update Profile

```
PUT /api/subscriptions/:subscription_id/profiles/:id
```

### Delete Profile

```
DELETE /api/subscriptions/:subscription_id/profiles/:id
```

Returns 204 No Content. Cannot delete the only default profile.

### Set Default Profile

```
POST /api/subscriptions/:subscription_id/profiles/:id/set-default
```

Makes this profile the default for new allocations.

---

## Profile Metrics

Configure what limits or balances a profile provides for each metric.

### List Profile Metrics

```
GET /api/subscriptions/:subscription_id/profiles/:profile_id/metrics
```

Response:
```json
{
  "data": [
    {
      "id": "pm_abc123",
      "allocation_profile_id": "prof_abc",
      "metric_id": "metric_msg",
      "metric_name": "messages_per_day",
      "allocation_type": "limit",
      "limit": 100,
      "limit_type": "hard",
      "created_at": "2024-01-15T10:30:00Z"
    },
    {
      "id": "pm_def456",
      "allocation_profile_id": "prof_abc",
      "metric_id": "metric_credits",
      "metric_name": "ai_credits",
      "allocation_type": "balance",
      "initial_balance": "50.00",
      "low_balance_threshold": "5.00",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

### Add Metric to Profile

```
POST /api/subscriptions/:subscription_id/profiles/:profile_id/metrics
```

**Limit-type metric:**
```json
{
  "metric_id": "metric_msg",
  "allocation_type": "limit",
  "limit": 100,
  "limit_type": "hard"
}
```

**Balance-type metric:**
```json
{
  "metric_id": "metric_credits",
  "allocation_type": "balance",
  "initial_balance": "50.00",
  "low_balance_threshold": "5.00"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `metric_id` | Yes | — | Metric UUID |
| `allocation_type` | No | `"limit"` | `"limit"` or `"balance"` |
| `limit` | No | — | Limit value (for limit-type) |
| `limit_type` | No | `"hard"` | `"soft"`, `"hard"`, or `"unlimited"` |
| `initial_balance` | No | — | Starting balance (for balance-type) |
| `low_balance_threshold` | No | — | Low balance warning threshold |

### Update Profile Metric

```
PUT /api/subscriptions/:subscription_id/profiles/:profile_id/metrics/:metric_id
```

### Remove Metric from Profile

```
DELETE /api/subscriptions/:subscription_id/profiles/:profile_id/metrics/:metric_id
```

---

## Allocations

Assign individual users to allocation profiles.

### List Allocations

```
GET /api/subscriptions/:subscription_id/allocations
```

Query parameters:
- `profile_id` — Filter by profile (optional)
- `external_user_id` — Filter by user (optional)

Response:
```json
{
  "data": [
    {
      "id": "alloc_abc123",
      "subscription_id": "sub_xyz",
      "external_user_id": "user_alice",
      "profile_id": "prof_power",
      "profile": {
        "id": "prof_power",
        "name": "Power User",
        "is_default": true
      },
      "metadata": {},
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

### Create Allocation

```
POST /api/subscriptions/:subscription_id/allocations
```

Request:
```json
{
  "external_user_id": "user_alice",
  "profile_id": "prof_power",
  "metadata": {"team": "engineering"}
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `external_user_id` | Yes | — | Your identifier for the end user |
| `profile_id` | No | default profile | Which allocation profile to use |
| `metadata` | No | `{}` | Arbitrary JSON metadata |

If `profile_id` is omitted, the subscription's default profile is used.

### Get Allocation

```
GET /api/subscriptions/:subscription_id/allocations/:external_user_id
```

Returns the allocation with balance information:
```json
{
  "data": {
    "id": "alloc_abc123",
    "subscription_id": "sub_xyz",
    "external_user_id": "user_alice",
    "profile_id": "prof_power",
    "profile": {"id": "prof_power", "name": "Power User"},
    "metadata": {},
    "balances": [
      {
        "metric_id": "metric_credits",
        "metric_name": "ai_credits",
        "current_balance": "42.50"
      }
    ]
  }
}
```

### Update Allocation

```
PUT /api/subscriptions/:subscription_id/allocations/:external_user_id
```

If `profile_id` is included, reassigns the user to a different profile. Otherwise, updates metadata only.

### Delete Allocation

```
DELETE /api/subscriptions/:subscription_id/allocations/:external_user_id
```

### Assign Profile

```
POST /api/subscriptions/:subscription_id/allocations/:external_user_id/assign-profile
```

Request:
```json
{
  "profile_id": "prof_standard"
}
```

Reassigns the user to a different allocation profile.

---

## Per-User Credits

Add credits to a specific user's balance within an allocation.

### Add Credits

```
POST /api/subscriptions/:subscription_id/allocations/:external_user_id/credits
```

Request:
```json
{
  "metric_name": "ai_credits",
  "amount": 100,
  "reason": "grant",
  "idempotency_key": "grant_123"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `metric_name` or `metric_id` | Yes | Which metric to credit |
| `amount` | Yes | Positive number |
| `reason` | No | `"purchase"`, `"grant"` (default), or `"refund"` |
| `idempotency_key` | No | Prevents duplicate credits |
| `reference_type` | No | External reference type |
| `reference_id` | No | External reference ID |
| `metadata` | No | Arbitrary JSON |

Response (201):
```json
{
  "data": {
    "subscription_id": "sub_xyz",
    "external_user_id": "user_alice",
    "metric_id": "metric_credits",
    "amount": 100,
    "balance_after": "142.50"
  }
}
```

### List Transactions

```
GET /api/subscriptions/:subscription_id/allocations/:external_user_id/transactions?metric_name=ai_credits
```

The `metric_name` or `metric_id` parameter is required.

Query parameters:
- `metric_name` or `metric_id` — Required
- `limit` — Page size (default: 50)
- `offset` — Offset (default: 0)

---

## Subscription-Level Credits

Credits can also be managed at the subscription level (not per-user).

### Add Credits

```
POST /api/subscriptions/:subscription_id/credits
```

Request:
```json
{
  "metric_name": "ai_credits",
  "amount": 500,
  "reason": "purchase",
  "idempotency_key": "purchase_123"
}
```

Response (201):
```json
{
  "data": {
    "subscription_id": "sub_xyz",
    "metric_id": "metric_credits",
    "amount": 500,
    "balance_after": "1250.00",
    "transaction_id": "txn_abc123"
  }
}
```

### Get Balance

```
GET /api/subscriptions/:subscription_id/credits
```

Returns all balance-type metric balances. Optionally filter with `?metric_name=` or `?metric_id=`.

Response:
```json
{
  "data": [
    {
      "metric_id": "metric_credits",
      "metric_name": "ai_credits",
      "balance": "1250.00",
      "lifetime_credits": "2000.00",
      "lifetime_debits": "750.00"
    }
  ]
}
```

### List Transactions

```
GET /api/subscriptions/:subscription_id/transactions
```

Query parameters:
- `metric_name` or `metric_id` — Optional filter
- `limit` — Page size (default: 50)
- `offset` — Offset (default: 0)

Response:
```json
{
  "data": [
    {
      "id": "txn_abc123",
      "entity_type": "subscription",
      "entity_id": "sub_xyz",
      "metric_id": "metric_credits",
      "transaction_type": "credit",
      "amount": "500.00",
      "balance_before": "750.00",
      "balance_after": "1250.00",
      "reason": "purchase",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ],
  "pagination": {
    "limit": 50,
    "offset": 0,
    "has_more": false
  }
}
```

---

## Checking Per-User Limits

Include `external_user_id` when checking limits to get both subscription-level and per-user limits:

```
GET /api/limits?subscription_id=sub_xyz&external_user_id=user_alice
```

Both subscription-level limits and the user's allocation limits are evaluated. Both must pass for the request to be allowed.

## Reporting Per-User Usage

Include `external_user_id` in usage reports to attribute usage to a specific user:

```json
POST /api/usage
{
  "subscriber_id": "org_123",
  "metric_name": "api_calls",
  "count": 1,
  "external_user_id": "user_alice",
  "idempotency_key": "req_abc123"
}
```

Usage is tracked at both the subscription level and the individual user level.
