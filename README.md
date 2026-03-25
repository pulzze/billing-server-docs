# Billing Server

Usage tracking, limit enforcement, and subscription management for the Interactor platform.

## Integration Guide

The integration guide is organized into chapters by topic:

1. [Overview](integration-guide/01-overview.md) — Architecture, core concepts, and how billing works
2. [Authentication](integration-guide/02-authentication.md) — Token types, making authenticated requests, service-to-service auth
3. [Quick Start](integration-guide/03-quick-start.md) — Minimal integration: register metrics, check limits, report usage
4. [Subscriber API](integration-guide/04-subscriber-api.md) — Plans, subscriptions, limits, usage, invoices, payment methods
5. [Vendor API](integration-guide/05-vendor-api.md) — Metrics management, plan CRUD, Stripe Connect, webhooks
6. [Service API](integration-guide/06-service-api.md) — Service-to-service endpoints for backend integrations
7. [Per-User Billing](integration-guide/07-per-user-billing.md) — Allocation profiles, per-user limits, credits, and balances
8. [Billing Lifecycle](integration-guide/08-billing-lifecycle.md) — Subscription states, billing cycles, proration, payments
9. [Reference](integration-guide/09-reference.md) — Error handling, best practices, SDK examples
