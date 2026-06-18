# Overview

_Last verified: 2026-06-18_

This guide explains how to integrate your application with the Billing Server for usage tracking, limit enforcement, and billing.

## What the Billing Server Provides

- **Usage Tracking**: Report consumption of metrics (API calls, messages, storage, LLM costs, etc.)
- **Limit Enforcement**: Check subscriber limits before processing requests
- **Subscription Management**: Create and manage subscriptions to payment plans
- **Billing**: Automated invoicing and payment processing via Stripe

## Architecture

```
Your Application ──────► Billing Server ──────► Stripe Platform
       │                       │                     │
       │                       ▼                     ▼
       │                 Account Server        Your Stripe Account
       │                   (Identity)          (via Stripe Connect)
       ▼
   Your Users
```

Each application connects its own Stripe account via Stripe Connect. Payments for subscriptions to your plans go directly to your connected Stripe account.

**Your application is responsible for:**
1. Authenticating users via Account Server
2. Checking limits before processing requests
3. Reporting usage after processing requests
4. Enforcing hard limits (blocking requests when limits are reached)

**The Billing Server handles:**
1. Tracking usage across all subscribers
2. Calculating effective limits (including overrides)
3. Generating invoices and processing payments
4. Sending billing event notifications

## Core Concepts

### Metrics

A **Metric** is a measurable unit that your application tracks. Examples:
- `messages_per_day` — Number of messages sent
- `api_calls` — API requests made
- `storage_gb` — Storage consumed
- `llm_costs` — LLM API costs (passthrough pricing)

Metrics are registered by your application and referenced in payment plans.

### Payment Plans

A **Payment Plan** defines pricing and limits for subscribers. Plans include:
- Base price (flat fee per billing cycle)
- Metrics with limits and pricing
- Billing cycle configuration (monthly, quarterly, annual, or custom)
- Upgrade/downgrade behavior

**Plans are application-scoped**: Each plan belongs to a specific application, not an organization. This enables:
- Multiple applications within one organization, each with its own plans
- Each application can have its own Stripe account (via Stripe Connect)
- Revenue goes directly to the application's connected Stripe account

Plans have visibility settings:
- **Private**: Only visible within the vendor's organization
- **Public**: Discoverable by any organization (including via the unauthenticated public plans API)
- **Unlisted**: Accessible via direct ID but not discoverable

### Subscriptions

A **Subscription** links a subscriber (user or application) to a payment plan. Subscriptions track:
- Current billing period
- Subscription status (pending, active, trial, past_due, suspended, cancelled)
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

This is automatic — you don't need to manage snapshots directly.

### Usage Events

A **Usage Event** records consumption of a metric. Your application reports usage after processing requests.

### Dual-Role Applications (Owner & Provider)

An application can participate in the billing server in **two distinct roles simultaneously**:

**As a Subscription Owner (consumer):**
Your application subscribes to another service's plan. For example, a chatbot app subscribes to an AI platform's plan to access LLM capabilities. In this role:
- Your application's organization is the **subscriber**
- You consume metrics defined by the upstream service (e.g., `capability_calls`, `llm_costs`)
- Your end users can be tracked as `external_user_id`s within your subscription
- **Allocation profiles** control how much of your quota each end user can consume

**As a Subscription Provider (vendor):**
Your application defines its own metrics, creates its own plans, and sells subscriptions to your end users. In this role:
- You register your own metrics (e.g., `messages`, `ai_requests`)
- You create plans with pricing (Free, Pro, Enterprise)
- End users subscribe to **your** plans and pay **you** (via your connected Stripe account)
- You collect and report usage for your own metrics
- You enforce limits based on your subscribers' plan terms

**These roles are independent.** Your subscription to an upstream service is a cost you manage internally. The subscriptions your users purchase from you are your revenue stream.

**Example — AI Chatbot App:**
```
Role 1: Owner (consuming AI Platform)
  Subscription: Chatbot App → AI Platform "Business" plan
  Metrics consumed: capability_calls, llm_costs
  Allocation profiles: limit each workspace to $50/month of LLM costs
  Purpose: cost protection

Role 2: Provider (selling to workspaces)
  Plans offered: Free ($0, 100 msgs/mo), Pro ($29, 5000 msgs/mo)
  Metrics defined: messages, ai_responses
  Subscribers: individual Slack/Discord workspaces
  Purpose: revenue
```

### Pricing Types

Plans can use different pricing models for each metric:

| Type | Description | Use Case |
|------|-------------|----------|
| **Included** | No charge, just tracking and limits | Features bundled with the plan |
| **Per-unit** | Fixed rate per unit consumed | Simple pay-per-use ($0.01/call) |
| **Tiered** | Rate decreases at volume thresholds (graduated) | Encouraging higher usage |
| **Passthrough** | Actual cost + markup | Variable-cost resources (LLM APIs) |

See [Billing Lifecycle](08-billing-lifecycle.md) for detailed pricing calculations and examples.
