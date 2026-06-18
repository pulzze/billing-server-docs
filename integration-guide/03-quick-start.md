# Quick Start

_Last verified: 2026-06-18_

Minimal steps to integrate billing into your application.

## 1. Register Your Metrics

Before creating plans, register the metrics your application will track:

```bash
POST /api/vendor/metrics
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "name": "messages_per_day",
  "display_name": "Daily Messages",
  "description": "Number of messages sent per day",
  "unit_label": "messages"
}
```

Or sync all metrics at once (recommended for deployments):

```bash
POST /api/vendor/metrics/sync
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "metrics": [
    {
      "name": "messages_per_day",
      "display_name": "Daily Messages",
      "unit_label": "messages"
    },
    {
      "name": "llm_costs",
      "display_name": "LLM API Costs",
      "unit_label": "USD",
      "metric_type": "balance"
    }
  ]
}
```

## 2. Check Limits Before Processing

Before processing a request, check the subscriber's limits:

```bash
GET /api/limits?subscriber_id=user_123
Authorization: Bearer <jwt>
```

Response:
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

## 3. Enforce Hard Limits

If a metric has `limit_type: "hard"` and `remaining <= 0`, block the request:

```python
limits = get_limits(subscriber_id)

metric = limits["messages_per_day"]
if metric["limit_type"] == "hard" and metric["remaining"] <= 0:
    return error_response(429, "Daily message limit reached")

# Process the request...
```

## 4. Report Usage After Processing

After successfully processing a request, report the usage:

```bash
POST /api/usage
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "subscriber_id": "user_123",
  "metric_name": "messages_per_day",
  "count": 1,
  "idempotency_key": "req_abc123"
}
```

## Integration Modes

The Billing Server supports two modes of operation for usage reporting and limit checking.

### Direct Mode (Subscription-Specific)

Use when you track subscription IDs in your application.

```json
POST /api/usage
{
  "subscription_id": "sub_abc123",
  "metric_id": "metric_messages",
  "count": 1
}
```

```
GET /api/limits?subscription_id=sub_abc123
```

### Subscriber Mode (Automatic Routing)

Use when you want the Billing Server to determine the correct subscription. The server finds the subscriber's active subscription that includes the specified metric.

```json
POST /api/usage
{
  "subscriber_id": "user_123",
  "metric_name": "messages_per_day",
  "count": 1
}
```

```
GET /api/limits?subscriber_id=user_123
```

If no `subscriber_id` is provided, the token identity is used automatically.

### Multiple Subscriptions

Subscribers can have multiple active subscriptions (e.g., base plan + add-ons). In Subscriber Mode:

1. **Usage reporting** routes to the subscription whose plan includes the specified metric
2. **Limit checking** aggregates limits across all active subscriptions

Each metric in the response shows which subscription provides that limit.
