Agent: GitHub Copilot (Copilot SDK in VS Code); model: not disclosed

# ADR-0002: Publish an order-placed event for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Not specified

## Context and problem

After an order is placed, the system must send a confirmation email through MailFast, replacing the mock `emailservice` logic; award loyalty points through a new `loyaltyservice`; and record an "order placed" analytics event for the marketing team. Marketing expects additional post-order actions next year: SMS, fraud scoring, and a recommendations update.

The design must support the required actions now and allow new post-order actions without making each addition require changes to the order-placement path. No throughput, latency, delivery, or infrastructure constraints were provided.

## Decision drivers

- **Modifiability:** Add future post-order actions with minimal changes to order placement.
- **Availability:** A failure in a post-order action should not prevent unrelated actions from being attempted.
- **Reliability:** Provide a way to handle failed or repeated event processing; required guarantees are not specified.
- **Performance:** Avoid unnecessarily delaying order placement; acceptable latency is not specified.

## Considered options

1. Publish an `order placed` event and have independent consumers handle email, loyalty, and marketing analytics.
2. Make synchronous calls to MailFast, `loyaltyservice`, and the marketing analytics destination from the order-placement flow.
3. Use a single post-order coordinator that invokes all actions, with a defined interface for adding actions.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Event with independent consumers | Supports future actions by adding consumers (modifiability); isolates consumer failures from one another (availability); order placement need not wait for each action (performance). | Requires event-delivery infrastructure and operational ownership (cost, modifiability); actions may complete later than order placement (consistency); retries can cause duplicate effects unless consumers are idempotent (reliability). |
| 2. Synchronous calls from order placement | Simple request flow with direct responses (simplicity); action completion can be reported immediately (consistency). | Adds dependencies and latency to order placement (performance); a downstream outage can affect order placement (availability); future actions require modifying the order-placement flow (modifiability). |
| 3. Post-order coordinator | Centralizes action sequencing and error handling (modifiability); can expose a single integration point to order placement (simplicity). | Coordinator can become a bottleneck or failure point (availability, performance); each new action still changes or configures the coordinator (modifiability); its retry and partial-failure behavior needs explicit design (reliability). |

## Decision

Choose option 1: after an order is successfully placed, publish an `order placed` event. Independent consumers handle the MailFast confirmation email, loyalty-point award through `loyaltyservice`, and marketing analytics event. Keep the event contract and delivery mechanism technology-neutral until the open infrastructure and reliability questions are answered.

This separates post-order actions from the order-placement path and allows SMS, fraud scoring, and recommendation updates to be added as consumers. The event should carry a stable order identifier and the minimum information consumers need; its exact schema and data-handling constraints remain open.

## Consequences

- **Positive:** New post-order actions can be added as independent consumers rather than changing the order-placement flow (quality attribute: modifiability).
- **Positive:** A failure in one consumer need not block the other consumers from processing the event (quality attribute: availability).
- **Positive:** Order placement can avoid waiting for all post-order actions to finish (quality attribute: performance).
- **Negative:** Post-order actions may complete after the order-placement response, so confirmation, points, and analytics may be temporarily delayed (quality attribute: consistency).
- **Negative:** Event delivery and consumer operations add infrastructure and monitoring responsibilities (quality attribute: cost, operability).
- **Risks and mitigations:** Retries or redelivery may repeat side effects. Define delivery guarantees, stable event identifiers, idempotent consumer behavior, retry limits, and a way to inspect or recover failed events before implementation (quality attribute: reliability).
- **Risks and mitigations:** The event could expose unnecessary customer or order data. Minimize its payload and define access and retention rules before implementation (quality attribute: security, privacy).

## Validation

- Verify that one successfully placed order produces an `order placed` event that can be consumed independently by the email, loyalty, and analytics actions.
- Verify that a failure in one consumer does not prevent the other consumers from processing the event.
- Verify the agreed retry and duplicate-handling behavior for each consumer, including that retries do not award loyalty points or send email more than intended.
- Measure order-placement latency and post-order completion latency against targets agreed by the deciders; no target values were provided.
- Review the design when the future SMS, fraud-scoring, and recommendations actions are planned.

## Open questions

- Which event-delivery technology and hosting model are available, and who will operate them?
- What delivery guarantees are required, and how should retries, dead-letter handling, replay, and event ordering work?
- What are the acceptable order-placement and post-order completion latency targets, expected event volume, and availability targets?
- What MailFast API, authentication, rate limits, failure behavior, and email retry policy apply?
- What are the rules for loyalty-point calculation, duplicate awards, and correction or reversal?
- Which marketing analytics destination and event schema should be used?
- What customer/order data may be included in the event, and what access, privacy, and retention requirements apply?
- Who owns each consumer and the event contract, and who are the decision-makers?
