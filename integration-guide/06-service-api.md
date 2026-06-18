# Service API

_Last verified: 2026-06-18_

Endpoints for trusted backend services to manage subscriptions, check limits, and report usage on behalf of subscribers. All endpoints are under `/api/service/` and require an **app-type JWT**.

See [Authentication](02-authentication.md#service-to-service-authentication) for how to obtain a service token.

---

## Subscriptions

### Create Subscription

```
POST /api/service/subscriptions
```

Create a subscription on behalf of a subscriber (e.g., auto-provisioning on signup).

Request:
```json
{
  "subscriber_id": "org_xyz789",
  "subscriber_type": "organization",
  "plan_id": "plan_abc123"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `subscriber_id` | Yes | The user or organization ID to subscribe |
| `subscriber_type` | Yes | `"user"` or `"organization"` |
| `plan_id` | Yes | The plan to subscribe to |

Response (201):
```json
{
  "data": {
    "id": "sub_new123",
    "subscriber_id": "org_xyz789",
    "subscriber_type": "organization",
    "plan_id": "plan_abc123",
    "status": "active",
    "current_period_start": "2024-01-15T00:00:00Z",
    "current_period_end": "2024-02-15T00:00:00Z"
  }
}
```

**Security rules:**
- Your application must be in the trusted services list
- Can only create subscriptions to plans owned by your organization or with `visibility: "public"`
- The subscriber is specified in the request, not derived from the JWT

Error codes:
| Status | Code | Description |
|--------|------|-------------|
| `401` | `unauthorized` | Invalid or missing JWT |
| `403` | `not_trusted_service` | Your app is not in the trusted services list |
| `403` | `plan_not_accessible` | Cannot create subscription to this plan |
| `404` | `plan_not_found` | Plan ID does not exist |
| `409` | `already_subscribed` | Subscriber already has this subscription |

### Check Subscription Status

```
GET /api/service/subscriptions/check?subscriber_id=org_xyz789&subscriber_type=organization
```

Response (has subscription):
```json
{
  "data": {
    "has_subscription": true,
    "subscription": {
      "id": "sub_abc123",
      "plan_id": "plan_xyz",
      "status": "active"
    }
  }
}
```

Response (no subscription):
```json
{
  "data": {
    "has_subscription": false
  }
}
```

### Refresh Plan Snapshot

```
POST /api/service/subscriptions/:id/refresh-snapshot
```

Refreshes the plan snapshot for a subscription. Use this when a plan's terms have been updated and you want an existing subscription to pick up the new terms.

---

## Plans

### Look Up Plan by Role

```
GET /api/service/plans/by-role/:role
```

Find a plan by its role identifier. Useful for looking up well-known plans like the default free tier.

Example:
```
GET /api/service/plans/by-role/default_free
```

Response:
```json
{
  "data": {
    "id": "plan_free_123",
    "name": "Free",
    "role": "default_free",
    "base_price": "0.00",
    "is_active": true
  }
}
```

Returns 404 if no plan with the given role exists for your application.

---

## Limits

### Check Limits (Service)

```
GET /api/service/limits
```

Query parameters:
- `subscriber_id` — The subscriber to check
- `subscription_id` — Or a specific subscription
- `external_user_id` — Optional per-user check

Same behavior as the subscriber `/api/limits` endpoint, but authenticates with a service token instead of a user token. Useful for backend limit checks without a user context.

---

## Usage

### Report Usage (Service)

```
POST /api/service/usage
```

Same parameters as the subscriber `POST /api/usage` endpoint. Report usage on behalf of a subscriber using a service token.

```json
{
  "subscriber_id": "org_xyz789",
  "metric_name": "capability_calls",
  "count": 1,
  "external_user_id": "user_456",
  "idempotency_key": "req_abc123"
}
```

---

## Integration Example

Auto-provision a free subscription when a new organization signs up:

```python
class BillingService:
    def __init__(self, base_url, get_service_token):
        self.base_url = base_url
        self.get_service_token = get_service_token

    def ensure_subscription(self, org_id, plan_id):
        """Ensure organization has a subscription, create if needed."""
        token = self.get_service_token()
        headers = {"Authorization": f"Bearer {token}"}

        # Check if already subscribed
        response = requests.get(
            f"{self.base_url}/api/service/subscriptions/check",
            headers=headers,
            params={"subscriber_id": org_id, "subscriber_type": "organization"}
        )
        data = response.json()

        if data["data"]["has_subscription"]:
            return data["data"]["subscription"]

        # Create subscription
        response = requests.post(
            f"{self.base_url}/api/service/subscriptions",
            headers=headers,
            json={
                "subscriber_id": org_id,
                "subscriber_type": "organization",
                "plan_id": plan_id
            }
        )
        return response.json()["data"]

# Usage
billing = BillingService(
    base_url="https://billing.example.com",
    get_service_token=lambda: get_account_server_token()
)

def on_organization_created(org_id):
    subscription = billing.ensure_subscription(
        org_id=org_id,
        plan_id=os.environ["DEFAULT_FREE_PLAN_ID"]
    )
```
