# Reference

_Last verified: 2026-06-18_

Error handling, best practices, and SDK examples.

## Error Handling

### HTTP Status Codes

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `201` | Created |
| `204` | No Content (successful deletion) |
| `400` | Bad request (invalid or missing parameters) |
| `401` | Unauthorized (invalid or missing token) |
| `403` | Forbidden (no permission for this resource) |
| `404` | Not found |
| `409` | Conflict (e.g., metric in use, already subscribed, blocked sync) |
| `422` | Unprocessable entity (validation failure) |
| `429` | Rate limited |
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
      "used": 100
    }
  }
}
```

### Common Error Codes

| Code | Description |
|------|-------------|
| `invalid_token` | JWT token is invalid or expired |
| `missing_parameter` | Required parameter not provided |
| `invalid_parameters` | Unrecognized or conflicting parameters |
| `subscription_not_found` | No active subscription for this subscriber |
| `metric_not_found` | Metric name not recognized |
| `plan_not_found` | Plan ID does not exist |
| `limit_exceeded` | Hard limit has been reached |
| `payment_required` | Subscription is suspended due to payment failure |
| `subscriber_mismatch` | Provided subscriber_id doesn't match token identity |
| `no_active_subscriptions` | Subscriber has no active subscriptions |
| `no_subscription_for_metric` | No active subscription includes this metric |
| `already_subscribed` | Already has a subscription to this plan |
| `metric_in_use` | Cannot delete metric referenced by active plans |
| `stripe_not_connected` | Application must connect Stripe before creating plans |
| `not_trusted_service` | App is not in the trusted services list |

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

        response = self.billing_client.get_limits(subscriber_id)
        self.cache[subscriber_id] = (response.limits, response.recheck_after)
        return response.limits
```

### 3. Report Usage Asynchronously

For non-critical usage reporting, use async/background processing to avoid blocking requests:

```python
# Non-blocking
usage_queue.enqueue(UsageEvent(
    subscriber_id=subscriber_id,
    metric=metric,
    count=count,
    idempotency_key=request.id
))
```

### 4. Handle Billing Server Unavailability

Continue with cached limits if the billing server is unreachable:

```python
def get_limits_with_fallback(subscriber_id):
    try:
        return billing_client.get_limits(subscriber_id)
    except BillingServerUnavailable:
        if subscriber_id in limit_cache:
            return limit_cache[subscriber_id]
        # Fail open or closed based on your policy
        raise
```

### 5. Test with Stripe Test Cards

| Card Number | Behavior |
|-------------|----------|
| `4242424242424242` | Successful payment |
| `4000000000000002` | Declined |
| `4000000000009995` | Insufficient funds |

---

## SDK Examples

### Python

```python
import requests

class BillingClient:
    def __init__(self, base_url, get_token):
        self.base_url = base_url
        self.get_token = get_token

    def _headers(self):
        return {
            "Authorization": f"Bearer {self.get_token()}",
            "Content-Type": "application/json"
        }

    def get_limits(self, subscriber_id):
        response = requests.get(
            f"{self.base_url}/api/limits",
            headers=self._headers(),
            params={"subscriber_id": subscriber_id}
        )
        response.raise_for_status()
        return response.json()["data"]

    def report_usage(self, subscriber_id, metric, count, idempotency_key=None, external_user_id=None):
        payload = {
            "subscriber_id": subscriber_id,
            "metric_name": metric,
            "count": count
        }
        if idempotency_key:
            payload["idempotency_key"] = idempotency_key
        if external_user_id:
            payload["external_user_id"] = external_user_id

        response = requests.post(
            f"{self.base_url}/api/usage",
            headers=self._headers(),
            json=payload
        )
        response.raise_for_status()
        return response.json()["data"]
```

### Elixir

```elixir
defmodule BillingClient do
  def get_limits(subscriber_id) do
    url = "#{billing_url()}/api/limits?subscriber_id=#{subscriber_id}"

    case Req.get(url, headers: auth_headers()) do
      {:ok, %{status: 200, body: %{"data" => data}}} -> {:ok, data}
      {:ok, %{status: status, body: body}} -> {:error, {status, body}}
      {:error, reason} -> {:error, reason}
    end
  end

  def report_usage(params) do
    url = "#{billing_url()}/api/usage"

    case Req.post(url, json: params, headers: auth_headers()) do
      {:ok, %{status: 201, body: %{"data" => data}}} -> {:ok, data}
      {:ok, %{status: status, body: body}} -> {:error, {status, body}}
      {:error, reason} -> {:error, reason}
    end
  end
end
```

### TypeScript

```typescript
class BillingClient {
  constructor(
    private baseUrl: string,
    private getToken: () => Promise<string>
  ) {}

  private async headers() {
    return {
      'Authorization': `Bearer ${await this.getToken()}`,
      'Content-Type': 'application/json'
    };
  }

  async getLimits(subscriberId: string) {
    const response = await fetch(
      `${this.baseUrl}/api/limits?subscriber_id=${subscriberId}`,
      { headers: await this.headers() }
    );
    const json = await response.json();
    return json.data;
  }

  async reportUsage(params: {
    subscriberId: string;
    metric: string;
    count: number;
    idempotencyKey?: string;
    externalUserId?: string;
  }) {
    const body: Record<string, unknown> = {
      subscriber_id: params.subscriberId,
      metric_name: params.metric,
      count: params.count
    };
    if (params.idempotencyKey) body.idempotency_key = params.idempotencyKey;
    if (params.externalUserId) body.external_user_id = params.externalUserId;

    const response = await fetch(`${this.baseUrl}/api/usage`, {
      method: 'POST',
      headers: await this.headers(),
      body: JSON.stringify(body)
    });
    const json = await response.json();
    return json.data;
  }
}
```

---

## Health Check

```
GET /health
```

Response:
```json
{
  "status": "ok",
  "service": "billing-server"
}
```

No authentication required. Use this for monitoring and load balancer health checks.
