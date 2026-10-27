Agent: AI assistant using Copilot SDK in VS Code; model: not provided.

# Architecture review: ADR-0002

## Quality-attribute scenarios

1. **Order availability and performance:** A customer places an order while one or more post-order systems are unavailable, during the 11.11 peak of 200 orders/s. `checkoutservice` returns success without waiting for those systems; `PlaceOrder` p95 remains ≤500 ms. ( `docs/scenario.md`, “Quality goals and constraints”; `docs/adr/0002-use-asynchronous-post-order-processing.md`, “Decision”)
2. **Loyalty correctness and timeliness:** At 200 orders/s, a loyalty event is retried or redelivered after a consumer interruption. The consumer awards points exactly once per order and within 1 minute. (Scenario, “Quality goals and constraints”; ADR, “Decision”)
3. **Email throughput and recovery:** During the 200 orders/s peak, MailFast accepts at most 50 emails/s, has p95 1.5 s, p99 8 s, and fails about 0.5% of calls. The email consumer limits sends to 50/s, retains queued requests, and drains the backlog after the peak without affecting order success. The scenario gives no acceptable confirmation delay or peak duration, so a response-time measure for email cannot yet be set. (Scenario, “Quality goals and constraints”; ADR, “Decision” and “Open questions”)

## Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Non-risk** | Persisting the order and outbox event in one PostgreSQL transaction, then relaying to RabbitMQ, addresses the loss window between saving an order and publishing its event. | Supports order reliability while keeping the downstream calls off the checkout response path. This is well matched to the requirement that downstream outages never fail an order. (ADR, “Decision”; scenario, “Quality goals and constraints”) |
| **Trade-off point** | The email queue can grow by about 150 messages/s while 200 orders/s arrive and MailFast is capped at 50/s. | The design protects checkout performance and availability, but shifts pressure into email timeliness, queue storage, and recovery capacity. The ADR acknowledges growth but has no backlog-size or drain-time target. (Scenario, “Quality goals and constraints”; ADR, “Consequences”) |
| **Sensitivity point** | Peak duration, MailFast’s sustained throughput, and the maximum acceptable confirmation delay determine required queue capacity and post-peak drain time. | The one-minute loyalty target is specified, but no email delay target or peak duration is given. Without these values, capacity and alert thresholds cannot be validated. (Scenario, “Quality goals and constraints”; ADR, “Open questions”) |
| **Risk** | “Exactly once” points depend on atomically recording the event/order ID with the points award; the ADR does not state the transaction or recovery behavior if only one write succeeds. | An event-ID check alone may not prevent duplicate awards across a crash boundary. Define the atomicity boundary and test crashes around the award and deduplication writes. (ADR, “Decision”; scenario, “Quality goals and constraints”) |
| **Risk** | The one-minute loyalty deadline may conflict with the requirement to accept orders while loyalty is down. | Durable retry preserves the event but cannot ensure points are awarded within one minute during a sufficiently long outage or backlog. Specify the operating conditions for the deadline and an outage/recovery objective. (Scenario, “Quality goals and constraints”; ADR, “Decision” and “Open questions”) |

## Missing or weak

- The ADR correctly identifies the email throughput mismatch, but does not state a peak duration, target email delay, maximum backlog, or required drain rate. These are necessary to size RabbitMQ queues and PostgreSQL outbox retention.
- The loyalty design says to record the event/order ID “alongside” the points award but does not specify atomicity, uniqueness constraints, or how the consumer recovers from a crash between those operations.
- The claim that a saved order “never loses its event” is true for the transactional outbox record, not by itself for eventual delivery. Retention, retry limits, dead-letter handling, alert ownership, and replay procedures remain open.
- Options cover synchronous calls, direct asynchronous calls, a coordinator, and outbox plus RabbitMQ. They do not compare a durable database-backed queue without a separate broker; explain why RabbitMQ’s additional operations are preferable for this four-developer team.
- `OrderPlaced` includes user ID, items, and totals for both loyalty and analytics. The ADR says personal data reaches only the needed consumer, but does not justify which fields each consumer needs or whether the event can be minimized further.

## Verdict

**Accept with changes.** Before implementation approval:

1. Define the loyalty one-minute deadline’s operating conditions, outage objective, and atomic idempotency/recovery design.
2. Set email delay, peak-duration, backlog-capacity, and drain-time targets; validate queue and outbox sizing against them.
3. Clarify delivery guarantees and ownership for outbox retention, dead-lettering, monitoring, and replay, and justify the event fields sent to each consumer.
