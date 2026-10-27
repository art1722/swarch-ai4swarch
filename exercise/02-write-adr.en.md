**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but _less complete than humans_; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)

```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)

```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion                                                             | Run a                                                                                                                                                                                                                                                              | Run b                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Uses the scenario's numbers (latency, peak, MailFast limits)?         | No. It had no scenario, so it uses no numbers and says so explicitly ("No throughput, latency, delivery, or infrastructure constraints were provided"; validation targets "no target values were provided").                                                       | Yes. It uses p95 ≤ 500 ms (today approximately 300 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / 0.5 % failures / 50 emails/s, loyalty within 1 minute and exactly once, "never fail", 4 developers (2 with RabbitMQ) and the self-hosted Kubernetes preference. It also derives a backlog growth of approximately 150 emails/s at peak (200 − 50). It omits the team's Go/gRPC strength.                                                     |
| Options considered (list them)                                        | 1. Publish an `order placed` event to independent consumers (chosen, technology-neutral, no broker named). 2. Synchronous calls from order placement. 3. A single post-order coordinator service.                                                                  | 1. Synchronous calls from `checkoutservice`. 2. Direct asynchronous calls from checkout without a broker. 3. Durable outbox + RabbitMQ with one queue and consumer per action (chosen). No other broker (e.g. Kafka, NATS) and no coordinator service were considered, so option 2 is a weak alternative.                                                                                                                                         |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | Only in general terms. It mentions stable event IDs, idempotent consumers and retry limits, but it did not know about "exactly once" or the 50 emails/s limit, so both are listed as open questions (delivery guarantees, MailFast rate limits, duplicate awards). | Yes. Loyalty: the consumer records the event/order ID together with the points award, giving effectively-once processing on top of at-least-once delivery (it explicitly does not assume exactly-once delivery). MailFast: the email consumer throttles to 50 emails/s; it states the backlog grows at peak and leaves peak duration and drain time as open questions. A separate queue per action keeps the email backlog from delaying loyalty. |
| Anything invented (a number, tool or fact not in the scenario)?       | No invented facts found. Every unknown is stated as unknown or listed under Open questions.                                                                                                                                                                        | No invented numbers found; the ~150 emails/s figure is correctly derived. The outbox, order store and topic exchange are new design proposals, and the ADR says the order/outbox store does not exist today (correct: `PlaceOrder` in `checkoutservice/main.go` stores nothing). One simplification: it repeats the scenario's "payment, shipping, email" sequence, although the `main.go` it read also calls cart, product catalog and currency. |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| #   | What you changed from the AI's draft                                                                                                                                          | Why                                                                                                                                                                                                              |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Chose a self-hosted PostgreSQL database for the order and outbox, written in one transaction.                                                                                 | The outbox requires that the order and its event are submiited in one transaction. PostgreSQL is open-source and self-hosted, and Redis is not appropiate in this case because ADR-0001 accepts losing its data. |
| 2   | Split the event into `OrderPlaced` for the case of no email, name or address; loyalty and analytics queues and `OrderConfirmationRequested` for the case of email queue only. | Data minimisation principle should be considered here. Only the email consumer needs personal data, so analytics and loyalty should not receive it.                                                              |
| 3   | Added a post-order coordinator service as an extra considered option.                                                                                                         | The coordinator is more realistic, taken from the zero-shot run 2a.                                                                                                                                              |
