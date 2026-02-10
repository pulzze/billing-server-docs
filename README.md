# Billing Server Integration Guide

This guide explains how to integrate your application with the Billing Server for usage tracking, limit enforcement, and billing.

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [Core Concepts](#core-concepts)
4. [Quick Start](#quick-start)
5. [API Reference](#api-reference)
6. [Understanding Pricing](#understanding-pricing)
7. [Usage Reporting](#usage-reporting)
8. [Limit Enforcement](#limit-enforcement)
9. [Subscription Lifecycle](#subscription-lifecycle)
10. [Billing Cycles](#billing-cycles)
11. [Payment Methods](#payment-methods)
12. [Webhooks](#webhooks)
13. [Best Practices](#best-practices)

---

## Overview

The Billing Server provides:

- **Usage Tracking**: Report consumption of metrics (API calls, messages, storage, etc.)
- **Limit Enforcement**: Check subscriber limits before processing requests
- **Subscription Management**: Create and manage subscriptions to payment plans
- **Billing**: Automated invoicing and payment processing via Stripe

### Architecture

```
Your Application ──────► Billing Server ──────► Stripe
       │                       │
       │                       ▼
       │                 Account Server
       │                   (Identity)
       ▼
   Your Users
```

Your application is responsible for:
1. Authenticating users via Account Server
2. Checking limits before processing requests
3. Reporting usage after processing requests
4. Enforcing hard limits (blocking requests when limits are reached)

The Billing Server handles:
1. Tracking usage across all subscribers
2. Calculating effective limits (including overrides)
3. Generating invoices and processing payments
4. Sending billing event notifications

---

## Authentication

All API requests require a valid JWT token from Account Server.

### Obtaining a Token

Your application should already have an Account Server JWT from authenticating the user or application making the request.

### Making Authenticated Requests

Include the token in the `Authorization` header:

```bash
curl -X GET https://billing.example.com/api/limits \
  -H "Authorization: Bearer <account_server_jwt>" \
  -H "Content-Type: application/json"
```

The Billing Server validates the token with Account Server and extracts:
- `user_id` or `application_id` (the subscriber)
- `organization_id` (the subscriber's organization)

---

## Core Concepts

### Metrics

A **Metric** is a measurable unit that your application tracks. Examples:
- `messages_per_day` - Number of messages sent
- `api_calls` - API requests made
- `storage_gb` - Storage consumed
- `llm_costs` - LLM API costs (passthrough pricing)

Metrics are registered by your application and referenced in payment plans.

### Payment Plans

A **Payment Plan** defines pricing and limits for subscribers. Plans include:
- Base price (flat fee per billing cycle)
- Metrics with limits and pricing
- Billing cycle configuration (monthly, quarterly, annual, or custom)
- Upgrade/downgrade behavior

Plans have visibility settings:
- **Private**: Only visible within the vendor's organization
- **Public**: Discoverable by any organization
- **Unlisted**: Accessible via direct ID but not discoverable

### Subscriptions

A **Subscription** links a subscriber (user or application) to a payment plan. Subscriptions track:
- Current billing period
- Subscription status (pending, active, past_due, suspended, cancelled)
- Plan terms at subscription time (frozen snapshot)

**Important notes:**
- A subscriber can have multiple subscriptions to *different* plans (enabling add-ons)
- A subscriber can only have one subscription per plan (upgrades replace existing subscriptions)
- Subscriptions can cross organization boundaries (Org A's users subscribing to Org B's plans)

### Plan Snapshots

When you create a subscription, the plan's terms are **frozen into a snapshot**. This means:
- If the vendor later changes the plan's pricing or limits, your subscription keeps the original terms
- Existing subscribers are protected from unexpected changes
- New subscribers get the updated terms

This is automatic—you don't need to manage snapshots directly.

### Usage Events

A **Usage Event** records consumption of a metric. Your application reports usage after processing requests.

---

## Quick Start

### 1. Register Your Metrics

Before creating plans, register the metrics your application will track:

```bash
POST /api/metrics
{
  "name": "messages_per_day",
  "display_name": "Daily Messages",
  "description": "Number of messages sent per day",
  "unit_label": "messages"
}
```

### 2. Check Limits Before Processing

Before processing a request, check the subscriber's limits:

```bash
GET /api/limits?subscriber_id=user_123
```

Response:
```json
{
  "limits": {
    "messages_per_day": {
      "limit": 100,
      "used": 45,
      "remaining": 55,
      "type": "soft",
      "exceeded": false
    },
    "messages_per_minute": {
      "limit": null,
      "type": "unlimited"
    }
  },
  "recheck_after": 50
}
```

### 3. Enforce Hard Limits

If a metric has `type: "hard"` and `exceeded: true`, block the request:

```python
limits = get_limits(subscriber_id)

if limits["messages_per_day"]["type"] == "hard" and limits["messages_per_day"]["exceeded"]:
    return error_response(429, "Daily message limit reached")

# Process the request...
```

### 4. Report Usage After Processing

After successfully processing a request, report the usage:

```bash
POST /api/usage
{
  "subscriber_id": "user_123",
  "metric": "messages_per_day",
  "count": 1
}
```

---

## API Reference

### Plans

#### List Available Plans

```
GET /api/plans
```

Returns plans visible to your organization (public plans and your organization's own plans).

Query parameters:
- `visibility` - Filter by visibility: `public`, `private`, `unlisted`
- `include_metrics` - Include plan metrics in response: `true`/`false`

Response:
```json
{
  "plans": [
    {
      "id": "plan_pro_monthly",
      "name": "Pro Monthly",
      "description": "Professional plan with expanded limits",
      "visibility": "public",
      "base_price": "29.00",
      "currency": "USD",
      "billing_period": "monthly",
      "vendor": {
        "organization_id": "org_xyz",
        "name": "Acme Corp"
      }
    }
  ]
}
```

#### Get Plan Details

```
GET /api/plans/{plan_id}
```

Response:
```json
{
  "id": "plan_pro_monthly",
  "name": "Pro Monthly",
  "description": "Professional plan with expanded limits",
  "visibility": "public",
  "base_price": "29.00",
  "currency": "USD",
  "billing_period": "monthly",
  "cycle_start": "anniversary",
  "payment_timing": "prepaid",
  "vendor": {
    "organization_id": "org_xyz",
    "name": "Acme Corp"
  },
  "metrics": [
    {
      "metric_id": "metric_messages",
      "name": "messages_per_day",
      "display_name": "Daily Messages",
      "limit": 100,
      "limit_type": "soft",
      "reset_period": "per_day",
      "pricing_type": "included"
    },
    {
      "metric_id": "metric_api",
      "name": "api_calls",
      "display_name": "API Calls",
      "limit": null,
      "limit_type": "unlimited",
      "pricing_type": "tiered",
      "tiers": [
        { "from_units": 0, "to_units": 1000, "unit_price": "0.02" },
        { "from_units": 1001, "to_units": 10000, "unit_price": "0.01" },
        { "from_units": 10001, "to_units": null, "unit_price": "0.005" }
      ]
    },
    {
      "metric_id": "metric_llm",
      "name": "llm_costs",
      "display_name": "LLM Usage",
      "pricing_type": "passthrough",
      "markup_percent": 20,
      "included_amount": "5.00"
    }
  ],
  "overrides": [
    {
      "trigger_metric": "messages_per_day",
      "trigger_threshold_percent": 100,
      "affected_metric": "messages_per_minute",
      "override_limit": 2,
      "override_limit_type": "hard"
    }
  ]
}
```

---

### Metrics

#### Register a Metric

```
POST /api/metrics
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

Response:
```json
{
  "id": "metric_abc123",
  "application_id": "app_xyz",
  "name": "messages_per_day",
  "display_name": "Daily Messages",
  "description": "Number of messages sent per day",
  "unit_label": "messages",
  "created_at": "2024-01-15T10:30:00Z"
}
```

#### List Metrics

```
GET /api/metrics
```

Returns all metrics registered by your application.

---

### Limits

#### Get Effective Limits

```
GET /api/limits?subscriber_id={subscriber_id}
```

Returns the current effective limits for a subscriber, including any active overrides.

Response:
```json
{
  "subscriber_id": "user_123",
  "subscription_id": "sub_abc",
  "limits": {
    "messages_per_day": {
      "limit": 100,
      "used": 45,
      "remaining": 55,
      "type": "soft",
      "exceeded": false,
      "resets_at": "2024-01-16T00:00:00Z"
    },
    "messages_per_minute": {
      "limit": 2,
      "used": 0,
      "remaining": 2,
      "type": "hard",
      "exceeded": false,
      "resets_at": "2024-01-15T10:31:00Z",
      "override_active": true,
      "override_reason": "messages_per_day exceeded 100%"
    }
  },
  "recheck_after": 50
}
```

**Response Fields:**

| Field | Description |
|-------|-------------|
| `limit` | The current limit value (`null` if unlimited) |
| `used` | Usage count in the current reset period |
| `remaining` | Units remaining before limit is reached |
| `type` | `"soft"`, `"hard"`, or `"unlimited"` |
| `exceeded` | Whether the limit has been exceeded |
| `resets_at` | When the usage counter resets |
| `override_active` | Whether an override is currently applied |
| `override_reason` | Why the override was triggered |
| `recheck_after` | Number of requests before rechecking with server |

---

### Usage

#### Report Usage

```
POST /api/usage
```

Request (count-based):
```json
{
  "subscriber_id": "user_123",
  "metric": "messages_per_day",
  "count": 1,
  "idempotency_key": "req_abc123"
}
```

Request (passthrough cost):
```json
{
  "subscriber_id": "user_123",
  "metric": "llm_costs",
  "cost": 0.0847,
  "metadata": {
    "model": "gpt-4",
    "input_tokens": 1500,
    "output_tokens": 340
  },
  "idempotency_key": "req_xyz789"
}
```

Response:
```json
{
  "id": "usage_evt_123",
  "subscriber_id": "user_123",
  "metric": "messages_per_day",
  "count": 1,
  "recorded_at": "2024-01-15T10:30:45Z",
  "limits": {
    "messages_per_day": {
      "limit": 100,
      "used": 46,
      "remaining": 54,
      "type": "soft",
      "exceeded": false
    }
  },
  "recheck_after": 50
}
```

**Idempotency:**

The `idempotency_key` field enables safe retries. If you send the same key within 24 hours, the duplicate is ignored and the original response is returned.

#### Batch Usage Reporting

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
      "metric": "messages_per_day",
      "count": 5,
      "timestamp": "2024-01-15T10:30:00Z",
      "idempotency_key": "batch_1_evt_1"
    },
    {
      "subscriber_id": "user_456",
      "metric": "api_calls",
      "count": 3,
      "timestamp": "2024-01-15T10:30:01Z",
      "idempotency_key": "batch_1_evt_2"
    }
  ]
}
```

Response:
```json
{
  "processed": 2,
  "results": [
    { "idempotency_key": "batch_1_evt_1", "status": "recorded" },
    { "idempotency_key": "batch_1_evt_2", "status": "recorded" }
  ]
}
```

---

### Subscriptions

#### Create a Subscription

```
POST /api/subscriptions
```

Request:
```json
{
  "subscriber_type": "user",
  "subscriber_id": "user_123",
  "plan_id": "plan_pro_monthly"
}
```

Response:
```json
{
  "id": "sub_abc123",
  "subscriber_type": "user",
  "subscriber_id": "user_123",
  "plan_id": "plan_pro_monthly",
  "plan_snapshot_id": "snap_123",
  "status": "pending",
  "current_period_start": "2024-01-15T00:00:00Z",
  "current_period_end": "2024-02-15T00:00:00Z",
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Starting status depends on the plan's payment timing:**

| Payment Timing | Starting Status | When it becomes Active |
|----------------|-----------------|------------------------|
| Prepaid | `pending` | After first payment succeeds |
| Postpaid | `active` | Immediately (billed at cycle end) |
| Hybrid | `pending` | After base fee payment succeeds |

The `plan_snapshot_id` references the frozen plan terms. Even if the plan is later modified, this subscription keeps its original terms.

#### Get Subscription

```
GET /api/subscriptions/{subscription_id}
```

#### List Subscriber's Subscriptions

```
GET /api/subscriptions?subscriber_id={subscriber_id}
```

#### Change Plan (Upgrade/Downgrade)

```
POST /api/subscriptions/{subscription_id}/change-plan
```

Request:
```json
{
  "new_plan_id": "plan_enterprise_monthly",
  "effective": "immediate"
}
```

The `effective` field can be:
- `"immediate"` - Change now, prorate charges
- `"next_cycle"` - Change at the start of the next billing cycle

#### Cancel Subscription

```
POST /api/subscriptions/{subscription_id}/cancel
```

Request:
```json
{
  "cancellation_type": "end_of_cycle"
}
```

The `cancellation_type` can be:
- `"immediate"` - Cancel now, issue prorated credit
- `"end_of_cycle"` - Cancel at end of current billing period

---

### Invoices

#### List Invoices

```
GET /api/invoices?subscriber_id={subscriber_id}
```

Query parameters:
- `subscriber_id` - Required. The subscriber's ID
- `status` - Filter by status: `draft`, `open`, `paid`, `failed`, `void`
- `from` - Start date (ISO 8601)
- `to` - End date (ISO 8601)

Response:
```json
{
  "invoices": [
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
      "paid_at": "2024-02-01T10:30:00Z"
    }
  ]
}
```

#### Get Invoice Details

```
GET /api/invoices/{invoice_id}
```

Response:
```json
{
  "id": "inv_abc123",
  "invoice_number": "INV-2024-00042",
  "subscription_id": "sub_xyz",
  "plan_name": "Pro Monthly",
  "status": "paid",
  "currency": "USD",
  "billing_period_start": "2024-01-01T00:00:00Z",
  "billing_period_end": "2024-01-31T23:59:59Z",
  "line_items": [
    {
      "description": "Pro Monthly - Base subscription",
      "type": "base_price",
      "quantity": 1,
      "unit_price": "29.00",
      "amount": "29.00"
    },
    {
      "description": "API Calls (1,500 calls)",
      "type": "usage",
      "metric": "api_calls",
      "quantity": 1500,
      "unit_price": "variable",
      "amount": "25.00",
      "breakdown": [
        { "tier": "0-1000", "quantity": 1000, "rate": "0.02", "amount": "20.00" },
        { "tier": "1001-10000", "quantity": 500, "rate": "0.01", "amount": "5.00" }
      ]
    }
  ],
  "subtotal": "54.00",
  "adjustments": [
    {
      "description": "Plan upgrade credit (15 days remaining)",
      "amount": "-20.00"
    }
  ],
  "total": "34.00",
  "amount_paid": "34.00",
  "amount_due": "0.00",
  "issued_at": "2024-02-01T00:00:00Z",
  "due_at": "2024-02-08T00:00:00Z",
  "paid_at": "2024-02-01T10:30:00Z",
  "payment_method": "Visa ending in 4242"
}
```

---

## Understanding Pricing

Payment plans can use different pricing models for each metric. Understanding these helps you explain costs to your users.

### Pricing Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Included** | No charge, just tracking and limits | Features bundled with the plan |
| **Per-unit** | Fixed rate per unit consumed | Simple pay-per-use ($0.01/call) |
| **Tiered** | Rate decreases at volume thresholds | Encouraging higher usage |
| **Passthrough** | Actual cost + markup | Variable-cost resources (LLM APIs) |

### Included Pricing

Units are tracked and may have limits, but there's no additional charge beyond the base plan price.

```json
{
  "metric": "storage_gb",
  "pricing_type": "included",
  "limit": 10,
  "limit_type": "hard"
}
```

### Per-Unit Pricing

Each unit consumed is charged at a fixed rate. Optionally, some units are included free.

```json
{
  "metric": "api_calls",
  "pricing_type": "per_unit",
  "unit_price": "0.01",
  "included_units": 1000
}
```

**Calculation:** If a subscriber uses 2,500 API calls:
- First 1,000 are included (free)
- Remaining 1,500 × $0.01 = **$15.00**

### Tiered Pricing (Graduated)

Different rates apply to different usage tiers. Each tier's rate only applies to usage within that tier—this is **graduated pricing**, not volume pricing.

```json
{
  "metric": "api_calls",
  "pricing_type": "tiered",
  "tiers": [
    { "from_units": 0, "to_units": 1000, "unit_price": "0.02" },
    { "from_units": 1001, "to_units": 10000, "unit_price": "0.01" },
    { "from_units": 10001, "to_units": null, "unit_price": "0.005" }
  ]
}
```

**Calculation:** For 15,000 API calls:
- First 1,000 × $0.02 = $20.00
- Next 9,000 × $0.01 = $90.00
- Final 5,000 × $0.005 = $25.00
- **Total: $135.00**

### Passthrough Pricing

For resources with variable underlying costs (like different LLM models), your application reports the actual cost incurred. The plan applies a markup and may include a free allowance.

```json
{
  "metric": "llm_costs",
  "pricing_type": "passthrough",
  "markup_percent": 20,
  "included_amount": "5.00"
}
```

**Calculation:** If actual LLM costs are $12.00:
- Subtract included amount: $12.00 - $5.00 = $7.00
- Apply markup: $7.00 × 1.20 = **$8.40**

See [Usage Reporting](#passthrough-pricing-1) for how to report passthrough costs.

---

## Usage Reporting

### When to Report Usage

Report usage **after** successfully processing a request, not before. This ensures you only bill for work actually completed.

```python
# 1. Check limits
limits = billing_client.get_limits(subscriber_id)

# 2. Enforce hard limits
if is_hard_limited(limits, "messages_per_day"):
    return error_response(429, "Limit reached")

# 3. Process the request
result = process_message(request)

# 4. Report usage (only if successful)
billing_client.report_usage(
    subscriber_id=subscriber_id,
    metric="messages_per_day",
    count=1,
    idempotency_key=request.id
)

return result
```

### Passthrough Pricing

For resources with variable costs (like LLM APIs), report the actual cost:

```python
# Call the LLM
response = openai.chat.completions.create(
    model="gpt-4",
    messages=messages
)

# Calculate cost based on token usage
cost = calculate_llm_cost(
    model="gpt-4",
    input_tokens=response.usage.prompt_tokens,
    output_tokens=response.usage.completion_tokens
)

# Report the cost
billing_client.report_usage(
    subscriber_id=subscriber_id,
    metric="llm_costs",
    cost=cost,
    metadata={
        "model": "gpt-4",
        "input_tokens": response.usage.prompt_tokens,
        "output_tokens": response.usage.completion_tokens
    },
    idempotency_key=request.id
)
```

### Idempotency

Always provide an `idempotency_key` to enable safe retries:

```python
def report_usage_with_retry(subscriber_id, metric, count, request_id):
    for attempt in range(3):
        try:
            return billing_client.report_usage(
                subscriber_id=subscriber_id,
                metric=metric,
                count=count,
                idempotency_key=request_id  # Same key for all retries
            )
        except NetworkError:
            if attempt == 2:
                raise
            time.sleep(1)
```

If the network fails after the server processes the request but before you receive the response, retrying with the same key ensures the usage isn't double-counted.

---

## Limit Enforcement

### Caching Strategy

The Billing Server returns a `recheck_after` value indicating how many requests you can process before checking limits again:

```python
class LimitCache:
    def __init__(self, billing_client):
        self.billing_client = billing_client
        self.cache = {}  # subscriber_id -> (limits, requests_remaining)

    def get_limits(self, subscriber_id):
        if subscriber_id in self.cache:
            limits, remaining = self.cache[subscriber_id]
            if remaining > 0:
                self.cache[subscriber_id] = (limits, remaining - 1)
                return limits

        # Fetch fresh limits
        response = self.billing_client.get_limits(subscriber_id)
        self.cache[subscriber_id] = (response.limits, response.recheck_after)
        return response.limits
```

### Hard Limit Enforcement

When a hard limit is reached, block the request:

```python
def check_and_enforce_limits(subscriber_id, metric):
    limits = limit_cache.get_limits(subscriber_id)

    metric_limit = limits.get(metric)
    if not metric_limit:
        return True  # No limit configured

    if metric_limit["type"] == "unlimited":
        return True

    if metric_limit["type"] == "hard" and metric_limit["exceeded"]:
        raise LimitExceededError(
            f"{metric} hard limit reached. Resets at {metric_limit['resets_at']}"
        )

    return True
```

### Soft Limit Handling

Soft limits don't block requests but should trigger notifications:

```python
if metric_limit["type"] == "soft" and metric_limit["exceeded"]:
    # Log for monitoring
    logger.warning(f"Soft limit exceeded for {subscriber_id}: {metric}")

    # Optionally notify the user
    notify_user_soft_limit_exceeded(subscriber_id, metric)
```

### Handling Billing Server Unavailability

If the Billing Server is unreachable, continue with cached limits:

```python
def get_limits_with_fallback(subscriber_id):
    try:
        return billing_client.get_limits(subscriber_id)
    except BillingServerUnavailable:
        logger.warning("Billing server unavailable, using cached limits")

        if subscriber_id in limit_cache:
            return limit_cache[subscriber_id]

        # No cache available - fail open or closed based on your policy
        logger.error("No cached limits available")
        raise
```

---

## Subscription Lifecycle

Understanding how subscriptions transition between states helps you handle edge cases correctly.

### Subscription States

```
┌─────────┐    payment     ┌────────┐
│ Pending │───succeeds────►│ Active │◄─────────────────┐
└─────────┘                └────┬───┘                  │
     │                          │                      │
     │ payment                  │ payment              │ payment
     │ fails                    │ fails                │ succeeds
     ▼                          ▼                      │
┌─────────┐                ┌──────────┐               │
│Cancelled│◄───────────────│ Past Due │───────────────┘
└─────────┘  (no payment   └────┬─────┘
              method)           │
                                │ grace period
                                │ expires
                                ▼
                           ┌───────────┐
                           │ Suspended │
                           └─────┬─────┘
                                 │
                                 │ policy threshold
                                 ▼
                           ┌───────────┐
                           │ Cancelled │
                           └───────────┘
```

| State | Description | User Can Access Service? |
|-------|-------------|--------------------------|
| `pending` | Awaiting first payment (prepaid/hybrid plans) | No |
| `active` | Normal operation, payments current | Yes |
| `past_due` | Payment failed, within grace period | Yes (typically) |
| `suspended` | Grace period expired | No |
| `cancelled` | Terminated, cannot be reinstated | No |

### Failed Payment Handling

When a payment fails, the subscription enters `past_due` status. What happens next depends on the plan's configuration:

**During Grace Period (Past Due):**
- The subscriber still has access (configurable per plan)
- Automatic payment retries occur (e.g., days 1, 3, 7)
- Subscriber receives payment failure notifications

**After Grace Period (Suspended):**
- Access is blocked
- Subscriber can reinstate by updating payment method and paying the balance
- After a configured suspension period, the subscription may auto-cancel

**Handling in your application:**

```python
def check_subscription_access(subscriber_id):
    subscription = billing_client.get_subscription(subscriber_id)

    if subscription["status"] == "active":
        return True

    if subscription["status"] == "past_due":
        # Plan may allow access during grace period
        # Check your plan's configuration
        return True  # or False depending on plan settings

    if subscription["status"] in ["pending", "suspended", "cancelled"]:
        return False

    return False
```

### Reinstatement

Suspended subscriptions can be reinstated when the subscriber:
1. Updates their payment method
2. Pays the outstanding balance

```
POST /api/subscriptions/{subscription_id}/reinstate
```

Response:
```json
{
  "id": "sub_abc123",
  "status": "active",
  "amount_paid": "29.00",
  "reinstated_at": "2024-01-20T15:30:00Z"
}
```

### Cancellation Behavior

**End-of-cycle cancellation:**
- Subscription remains `active` until the billing period ends
- Usage is tracked and billed normally
- At period end, status changes to `cancelled`

**Immediate cancellation:**
- Status changes to `cancelled` immediately
- Prorated credit is issued for unused time
- Usage stops being tracked

```python
# Check if subscription is cancelled but still active until period end
if subscription["status"] == "active" and subscription.get("cancelled_at"):
    # Subscriber cancelled but period hasn't ended yet
    days_remaining = calculate_days_until(subscription["current_period_end"])
    show_cancellation_notice(days_remaining)
```

---

## Billing Cycles

Understanding billing cycles helps you explain charges to your users and handle upgrades/downgrades correctly.

### Billing Period Options

Plans can use different billing periods:

| Period | Description |
|--------|-------------|
| Monthly | Every month |
| Quarterly | Every 3 months |
| Annual | Every 12 months |
| Custom | Every N days (configured per plan) |

### Cycle Start Types

| Type | Description | Example |
|------|-------------|---------|
| Anniversary | Relative to subscription date | Subscribed Jan 15 → bills on 15th each month |
| Calendar | Aligned to calendar boundaries | Always bills on 1st of month |

### Payment Timing

| Timing | When Charged | Use Case |
|--------|--------------|----------|
| Prepaid | At cycle start | SaaS subscriptions, flat-rate plans |
| Postpaid | At cycle end | Usage-based plans |
| Hybrid | Base at start, usage at end | Base subscription + usage fees |

### Metric Reset Periods

Metric reset periods are **independent** of billing cycles. This allows flexible configurations:

| Reset Period | Behavior |
|--------------|----------|
| `per_minute` | Resets every minute (UTC) |
| `per_hour` | Resets every hour (UTC) |
| `per_day` | Resets at midnight UTC |
| `per_month` | Resets on 1st of month |
| `per_cycle` | Resets with billing cycle |

**Example:** A monthly billing plan can have:
- `messages_per_day`: 100/day limit (resets daily)
- `api_calls`: 10,000/cycle (resets monthly with billing)

### Proration

When subscribers upgrade or downgrade mid-cycle, charges are prorated using **daily calculation**:

```
daily_rate = plan_price / days_in_cycle
prorated_amount = daily_rate × days_remaining
```

**Upgrade Example:**
- Current plan: $10/month, subscribed Jan 1
- Upgrade to: $30/month on Jan 16 (15 days remaining in 31-day month)
- Credit for old plan: (15/31) × $10 = $4.84
- Charge for new plan: (15/31) × $30 = $14.52
- **Net charge: $9.68**

Proration behavior is configured per plan:
- `immediate`: Charge prorated difference now
- `prorate`: Adjust on next invoice
- `next_cycle`: New plan starts at next billing cycle

---

## Payment Methods

Payment methods are managed through Stripe. The Billing Server never handles card numbers directly.

### Adding a Payment Method

Use Stripe Elements in your frontend to securely collect card details:

```javascript
// 1. Create a SetupIntent
const response = await fetch('/api/payment-methods/setup', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${token}` }
});
const { client_secret } = await response.json();

// 2. Use Stripe Elements to collect card
const stripe = Stripe('pk_test_...');
const elements = stripe.elements();
const cardElement = elements.create('card');
cardElement.mount('#card-element');

// 3. Confirm the setup
const { setupIntent, error } = await stripe.confirmCardSetup(client_secret, {
  payment_method: {
    card: cardElement,
  }
});

if (setupIntent.status === 'succeeded') {
  // Card is saved and ready for billing
}
```

### Setup Intent API

```
POST /api/payment-methods/setup
```

Response:
```json
{
  "client_secret": "seti_1234_secret_5678",
  "stripe_customer_id": "cus_abc123"
}
```

### List Payment Methods

```
GET /api/payment-methods?subscriber_id={subscriber_id}
```

Response:
```json
{
  "payment_methods": [
    {
      "id": "pm_abc123",
      "type": "card",
      "card": {
        "brand": "visa",
        "last4": "4242",
        "exp_month": 12,
        "exp_year": 2025
      },
      "is_default": true,
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

### Set Default Payment Method

```
POST /api/payment-methods/{payment_method_id}/set-default
```

### Delete Payment Method

```
DELETE /api/payment-methods/{payment_method_id}
```

Note: Cannot delete the default payment method if there are active subscriptions.

---

## Webhooks

### Configuring Webhooks

Register a webhook endpoint to receive billing events:

```
POST /api/webhooks
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

### Webhook Payload

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

Webhooks are signed using HMAC-SHA256. Verify the signature before processing:

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

# In your webhook handler
@app.post("/webhooks/billing")
def handle_billing_webhook(request):
    signature = request.headers.get("X-Billing-Signature")

    if not verify_webhook(request.body, signature, WEBHOOK_SECRET):
        return Response(status=401)

    event = json.loads(request.body)

    if event["type"] == "payment.failed":
        handle_payment_failed(event["data"])
    elif event["type"] == "subscription.cancelled":
        handle_subscription_cancelled(event["data"])

    return Response(status=200)
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

---

## Best Practices

### 1. Always Use Idempotency Keys

Provide unique, deterministic keys for all usage reports:

```python
# Good: Use the request ID
idempotency_key = f"req_{request.id}"

# Good: Use a hash of the operation
idempotency_key = f"msg_{message.id}_{subscriber_id}"

# Bad: Random UUIDs (can't retry safely)
idempotency_key = str(uuid.uuid4())
```

### 2. Cache Limits Aggressively

Respect the `recheck_after` value to minimize API calls:

```python
# The server tells you when to check again
if cached_requests_remaining > 0:
    use_cached_limits()
else:
    fetch_fresh_limits()
```

### 3. Report Usage Asynchronously When Possible

For non-critical usage reporting, use async/background processing:

```python
# Synchronous (blocks request)
billing_client.report_usage(...)

# Asynchronous (non-blocking)
usage_queue.enqueue(UsageEvent(
    subscriber_id=subscriber_id,
    metric=metric,
    count=count,
    idempotency_key=request.id
))
```

### 4. Handle Edge Cases

```python
# What if subscriber has no subscription?
if not subscription:
    # Option 1: Block access
    return error_response(403, "No active subscription")

    # Option 2: Use free tier limits
    limits = FREE_TIER_LIMITS

# What if limit check fails?
try:
    limits = get_limits(subscriber_id)
except BillingServerError:
    # Fail open (allow) or closed (deny) based on your policy
    limits = FALLBACK_LIMITS
```

### 5. Monitor Billing Events

Set up alerts for critical billing events:

- Payment failures (retry or suspend access)
- Approaching limits (notify users proactively)
- Subscription cancellations (trigger retention workflows)

### 6. Test with Stripe Test Mode

Use Stripe test cards for development:

| Card Number | Behavior |
|-------------|----------|
| `4242424242424242` | Successful payment |
| `4000000000000002` | Declined |
| `4000000000009995` | Insufficient funds |

---

## Error Handling

### HTTP Status Codes

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `400` | Bad request (invalid parameters) |
| `401` | Unauthorized (invalid or missing token) |
| `403` | Forbidden (no permission for this resource) |
| `404` | Not found (subscriber, subscription, or metric not found) |
| `429` | Rate limited (too many requests) |
| `500` | Server error |

### Error Response Format

```json
{
  "error": {
    "code": "limit_exceeded",
    "message": "Hard limit reached for messages_per_day",
    "details": {
      "metric": "messages_per_day",
      "limit": 100,
      "used": 100,
      "resets_at": "2024-01-16T00:00:00Z"
    }
  }
}
```

### Common Error Codes

| Code | Description |
|------|-------------|
| `invalid_token` | JWT token is invalid or expired |
| `subscription_not_found` | No active subscription for this subscriber |
| `metric_not_found` | Metric name not recognized |
| `limit_exceeded` | Hard limit has been reached |
| `payment_required` | Subscription is suspended due to payment failure |
| `idempotency_conflict` | Idempotency key was reused with different parameters |

---

## SDK Examples

### Python

```python
from billing_client import BillingClient

client = BillingClient(
    base_url="https://billing.example.com",
    get_token=lambda: get_account_server_token()
)

# Check limits
limits = client.get_limits(subscriber_id="user_123")

# Report usage
client.report_usage(
    subscriber_id="user_123",
    metric="messages_per_day",
    count=1,
    idempotency_key="req_abc123"
)
```

### Elixir

```elixir
# Check limits
{:ok, limits} = BillingClient.get_limits("user_123")

# Report usage
{:ok, _} = BillingClient.report_usage(%{
  subscriber_id: "user_123",
  metric: "messages_per_day",
  count: 1,
  idempotency_key: "req_abc123"
})
```

### JavaScript/TypeScript

```typescript
import { BillingClient } from '@your-org/billing-client';

const client = new BillingClient({
  baseUrl: 'https://billing.example.com',
  getToken: () => getAccountServerToken()
});

// Check limits
const limits = await client.getLimits('user_123');

// Report usage
await client.reportUsage({
  subscriberId: 'user_123',
  metric: 'messages_per_day',
  count: 1,
  idempotencyKey: 'req_abc123'
});
```

---

## Support

- **API Status**: https://status.billing.example.com
- **Documentation**: https://docs.billing.example.com
- **Support**: support@example.com
