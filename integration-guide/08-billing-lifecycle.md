# Billing Lifecycle

Understanding subscription states, billing cycles, and payment handling.

## Subscription States

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
| `trial` | In trial period | Yes |
| `active` | Normal operation, payments current | Yes |
| `past_due` | Payment failed, within grace period | Yes (typically) |
| `suspended` | Grace period expired | No |
| `cancelled` | Terminated | No |

### Handling States in Your Application

```python
def check_subscription_access(subscriber_id):
    subscription = billing_client.get_subscription(subscriber_id)

    if subscription["status"] in ["active", "trial"]:
        return True

    if subscription["status"] == "past_due":
        return True  # or False depending on your grace period policy

    if subscription["status"] in ["pending", "suspended", "cancelled"]:
        return False

    return False
```

## Trial Subscriptions

Start a trial using the dedicated endpoint:

```
POST /api/subscriptions/start-trial
{
  "plan_id": "plan_pro_monthly"
}
```

Requirements:
- The plan must have `trial_period_days > 0`
- The subscriber must not have an existing non-cancelled subscription to the same plan

During trial:
- `status` is `"trial"` (or `"active"` with `on_trial: true`)
- `trial_end_date` indicates when the trial ends
- Full access to plan features and limits
- No payment required until trial ends

When the trial ends, behavior depends on the plan's `trial_end_behavior` configuration.

## Failed Payment Handling

**During Grace Period (Past Due):**
- The subscriber still has access (configurable per plan)
- Automatic payment retries occur (e.g., days 1, 3, 7)
- Subscriber receives payment failure notifications

**After Grace Period (Suspended):**
- Access is blocked
- Subscriber can reinstate by updating payment method and paying the balance

```
POST /api/subscriptions/:id/reinstate
```

## Cancellation Behavior

**End-of-cycle cancellation:**
- Subscription remains `active` until the billing period ends
- Usage is tracked and billed normally
- At period end, status changes to `cancelled`

**Immediate cancellation:**
- Status changes to `cancelled` immediately
- Prorated credit is issued for unused time
- Usage stops being tracked

## Billing Cycles

### Billing Period Options

| Period | Description |
|--------|-------------|
| Monthly | Every month |
| Quarterly | Every 3 months |
| Annual | Every 12 months |
| Custom | Every N days (configured via `billing_period_days`) |

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

Metric reset periods are **independent** of billing cycles:

| Reset Period | Behavior |
|--------------|----------|
| `per_minute` | Resets every minute (UTC) |
| `per_hour` | Resets every hour (UTC) |
| `per_day` | Resets at midnight UTC |
| `per_month` | Resets on 1st of month |
| `per_cycle` | Resets with billing cycle |

## Proration

When subscribers upgrade or downgrade mid-cycle, charges are prorated using daily calculation:

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

## Pricing Details

### Included Pricing

Units are tracked and may have limits, but no additional charge beyond the base price.

### Per-Unit Pricing

Each unit consumed is charged at a fixed rate. Optionally, some units are included free.

**Calculation:** If a subscriber uses 2,500 API calls with 1,000 included at $0.01/call:
- First 1,000 are included (free)
- Remaining 1,500 × $0.01 = **$15.00**

### Tiered Pricing (Graduated)

Different rates apply to different usage tiers. Each tier's rate only applies to usage within that tier.

**Calculation:** For 15,000 API calls with tiers at $0.02/$0.01/$0.005:
- First 1,000 × $0.02 = $20.00
- Next 9,000 × $0.01 = $90.00
- Final 5,000 × $0.005 = $25.00
- **Total: $135.00**

### Passthrough Pricing

For resources with variable underlying costs (like LLM APIs), your application reports the actual cost. The plan applies a markup and may include a free allowance.

**Calculation:** If actual LLM costs are $12.00 with $5.00 included and 20% markup:
- Subtract included amount: $12.00 - $5.00 = $7.00
- Apply markup: $7.00 × 1.20 = **$8.40**

Report passthrough costs using the `cost` field:
```json
POST /api/usage
{
  "subscriber_id": "user_123",
  "metric_name": "llm_costs",
  "cost": 0.0847,
  "metadata": {"model": "gpt-4", "input_tokens": 1500, "output_tokens": 340}
}
```

## Balance-Type Metrics

Balance-type metrics are prepaid credits that decrement with usage.

| Type | Behavior | Example |
|------|----------|---------|
| **Limit** | Usage accumulates and resets each period | "100 API calls/month" |
| **Balance** | Credits decrement with each use | "500 tokens to spend" |

Balance is automatically deducted when you report usage for a balance-type metric. The usage response includes `balance_remaining`.

Checking balance appears in the limits response:
```json
{
  "metric_type": "balance",
  "balance": 750,
  "low_balance_threshold": 100,
  "is_low": false,
  "allowed": true
}
```

See [Per-User Billing](07-per-user-billing.md) for managing credits at the subscription and per-user levels.
