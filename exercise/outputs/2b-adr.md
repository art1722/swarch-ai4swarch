Agent: AI assistant using Copilot SDK in VS Code; model: not provided.

# ADR-0002: Dispatch post-order actions asynchronously

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Not specified

## Context and problem

Requirement R-12 adds three actions after an order is placed: send its confirmation through MailFast, award loyalty points through a new `loyaltyservice`, and record an "order placed" analytics event for marketing. Marketing expects SMS, fraud scoring, and a recommendations update next year (`docs/scenario.md`, R-12).

Today, `checkoutservice.PlaceOrder` calls payment, shipping, and email synchronously over gRPC. Email failure is logged as a warning and does not fail the order (`docs/scenario.md`, "How it works today"; `system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`). The scenario sets a checkout p95 limit of 500 ms (currently approximately 300 ms), a peak of 200 orders/s, MailFast p95/p99 of 1.5/8 seconds, a 0.5% failure rate and a 50 emails/s rate limit. Loyalty must be awarded within one minute and exactly once; an order must not fail because email, loyalty, or analytics is unavailable (`docs/scenario.md`, "Quality goals and constraints").

Keeping these actions synchronous would couple order latency and availability to post-order systems. MailFast's 50 emails/s limit is below the 200 orders/s peak, so synchronous email also cannot keep pace with peak order throughput. The team has four developers, two with RabbitMQ experience, and prefers open-source, self-hosted components on Kubernetes (`docs/scenario.md`, "Quality goals and constraints").

## Decision drivers

1. **Availability:** orders must succeed when MailFast, loyalty, or analytics is unavailable.
2. **Performance:** keep `PlaceOrder` p95 at or below 500 ms, including at 200 orders/s.
3. **Correctness:** award loyalty points within one minute without awarding them twice.
4. **Modifiability:** make future post-order actions addable without serially extending checkout.
5. **Operational fit:** prefer self-hosted open-source components and use the team's RabbitMQ experience.

## Considered options

1. Call MailFast, `loyaltyservice`, and analytics synchronously from `checkoutservice`.
2. Have `checkoutservice` call each post-order system asynchronously without a broker.
3. Persist an order and an associated outbox record, then publish an order event through RabbitMQ to independent post-order consumers.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| Synchronous calls from checkout | Simple request flow; each action's result is immediately visible to checkout | MailFast's latency and rate limit conflict with checkout's latency and peak-load goals; outages or timeouts complicate order availability; each new action adds coupling |
| Direct asynchronous calls without a broker | Checkout need not wait for each remote response; fewer components than a broker-based design | Requires managing background work and retries in checkout; no durable buffering or independent consumer queues are specified; failures during process restart can lose work |
| Durable outbox and RabbitMQ event consumers | Durable buffering and retry support; separate queues isolate slow or unavailable actions; new actions can subscribe without adding another checkout call | Adds an outbox store, RabbitMQ, and consumer operations; duplicate delivery must be handled; MailFast's rate limit can create a backlog during peak traffic |

## Decision

Use a durable outbox and RabbitMQ to dispatch post-order actions asynchronously. Once the order is placed, persist its order record and an outbox event with a stable event ID and order ID. A publisher relays outbox events to a durable RabbitMQ topic exchange; use a separate durable queue and consumer for email, loyalty, and analytics so that one action's delay does not block the others.

`checkoutservice` should return the successful order response without waiting for these consumers. The event publisher and consumers retry transient failures. The loyalty consumer must record the event/order ID alongside the points award so a redelivery cannot award points twice. This provides effectively-once points processing through durable delivery and idempotent handling, rather than assuming exactly-once message delivery. The MailFast consumer must enforce the 50 emails/s limit.

The supplied checkout implementation shows no order or outbox persistence (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`); adding durable order/outbox storage is therefore part of this proposal, not an existing capability evidenced by the provided files. RabbitMQ is selected because it fits the stated self-hosting preference, Kubernetes environment, and team experience (`docs/scenario.md`, "Quality goals and constraints").

## Consequences

- **Positive:** checkout does not wait for MailFast, loyalty, or analytics responses (performance); these downstream outages do not prevent the order response (availability).
- **Positive:** durable buffering and independent queues allow post-order actions to be added as consumers rather than as sequential checkout calls (modifiability).
- **Positive:** a durable outbox and idempotent loyalty consumer support recovery and prevent duplicate point awards after redelivery (reliability, correctness).
- **Negative:** the outbox store, RabbitMQ, publisher, and consumers add operational and implementation work for a four-developer team (cost, modifiability).
- **Negative:** MailFast can process at most 50 emails/s while orders can arrive at 200/s; during sustained peak traffic, the email backlog can grow by approximately 150 messages/s before retries (performance, timeliness).
- **Risks and mitigations:** if the publisher or broker is unavailable, the outbox must retain events and be monitored until delivery recovers (availability). Define retention, retry limits, and dead-letter handling; alert on outbox age and queue depth.
- **Risks and mitigations:** retries may duplicate emails or analytics events. Use stable event IDs and deduplication where supported; confirm whether MailFast accepts an idempotency key before relying on one.

## Validation

- Load-test `PlaceOrder` at 200 orders/s and verify its p95 stays at or below 500 ms.
- Simulate MailFast, loyalty, and analytics outages and verify orders still succeed, events remain available for retry, and independent consumers continue processing.
- Replay and redeliver the same loyalty event; verify each order earns points once and within one minute under the agreed peak-load profile.
- Exercise MailFast failures and its rate limit; verify no more than 50 emails/s are sent and track queue age and drain time.
- Verify outbox recovery after publisher or RabbitMQ interruption, and check that event IDs, retries, and dead-letter alerts make undelivered work observable.

## Open questions

- Which durable store will hold the order and outbox, and what is the exact point at which the system considers an order placed?
- What peak duration and acceptable confirmation-email delay should size the MailFast backlog and its post-peak drain time?
- What are the loyalty and analytics throughput targets, and what peak-load profile must meet the one-minute loyalty deadline?
- Does MailFast support idempotency keys, and what duplicate-email behavior is acceptable if delivery succeeds but its acknowledgment is lost?
- What event retention, retry, and dead-letter policies should apply, and who owns monitoring and replay?
