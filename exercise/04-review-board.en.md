**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; _plausible ≠ appropriate_), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role                                | Who              | Brings                                                             |
| ----------------------------------- | ---------------- | ------------------------------------------------------------------ |
| **Architect**                       | the ADR's author | defends the ADR                                                    |
| **AI analyst**                      |                  | reads out the AI's review only, and may not add their own opinions |
| **Product owner**                   |                  | role card P (business side)                                        |
| **Site reliability engineer (SRE)** |                  | role card S (operations side)                                      |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step                                                                                                                                                                        |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 3   | **Architect** presents the decision, the options considered and the main consequences                                                                                       |
| 3   | **AI analyst** reads the AI review's findings and verdict, word for word                                                                                                    |
| 7   | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2   | **Decision.** The board votes: accept / accept with changes / reject                                                                                                        |

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-use-asynchronous-post-order-processing.md` by Chawakorn Samosorn

**Board members and roles:** Chawakorn Samosorn, myself, doing every roles, namely architect, AI analyst, product owner (card P), Site reliability engineer (SRE) (card S) and security & privacy officer (card X).

**Decision:** accept with changes

**Required changes:**

1. Points should be shown at checkout immediately, but awarded asynchronously.
2. Fewer new stateful components for 11.11, which is to use the PostgreSQL outbox as the queue and add RabbitMQ after the sale.
3. The email path should be safe and visible, e.g. a fallback provider, encrypted personal data, and alerts.

**Who raised what?** List every concern the board discussed.

| #   | Concern                                                                                                                                                                                            | Raised by (AI review / architect / PO / SRE / security) | Was it in the AI's review?                                                                    | Could the AI have known? Why or why not?                                                                                                                                 |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | The campaign promises points "the moment" the order is placed, but the ADR awards them asynchronously within 1 minute.                                                                             | PO                                                      | No                                                                                            | No. The promise is only on role card P; the scenario says "within 1 minute".                                                                                             |
| 2   | Points are worth money (100 points = 10 THB): double points are a loss, missing points a complaint; what happens on cancellation or refund?                                                        | PO                                                      | Partly: it raised atomic deduplication, but not reversal on cancellation or refund            | Partly. "Exactly once" is in the scenario, but the points' value and refunds are not.                                                                                    |
| 3   | The sale is in 3 weeks: can PostgreSQL, a relay, RabbitMQ and three consumers be built and load-tested in time?                                                                                    | PO                                                      | No                                                                                            | No. The deadline is only on role card P.                                                                                                                                 |
| 4   | Future fraud scoring must cancel an order before it is confirmed, so it cannot be an asynchronous post-order consumer.                                                                             | PO                                                      | No                                                                                            | No. The scenario lists fraud scoring as a post-order action; the "before confirmation" constraint is only on the card.                                                   |
| 5   | RabbitMQ was only used on laptops, on-call is 2 people, and any new stateful component needs dashboards, alerts, backups and a runbook (about 1 week); the ADR adds two (PostgreSQL and RabbitMQ). | SRE                                                     | Partly: it asked why RabbitMQ's extra operations suit a four-developer team                   | Partly. Team size is in the scenario; the laptop-only experience and the ops rule are not.                                                                               |
| 6   | A highly available broker needs 3 nodes, but the cluster is near its node limit and budget approval takes 2 weeks.                                                                                 | SRE                                                     | No                                                                                            | No. Cluster capacity and budget process are only on role card S.                                                                                                         |
| 7   | Last year's 11.11 sale had a 20-minute email-provider outage that nobody noticed until customers complained; who is paged, and how are stuck messages dead-lettered and replayed?                  | SRE                                                     | Partly: it listed dead-lettering, monitoring, alert ownership and replay as open              | Partly. The scenario gives MailFast's failure rate, but not the outage history.                                                                                          |
| 8   | How do we trace one order across the asynchronous boundary when a customer says "I never got my email"?                                                                                            | SRE                                                     | No                                                                                            | Yes. Tracing across async consumers is a general concern it could have raised from the ADR alone.                                                                        |
| 9   | MailFast stores data in US data centres; legal has not approved it under PDPA. What is the fallback if approval misses 11.11?                                                                      | Security                                                | No                                                                                            | No. The data location and the legal status are only on role card X.                                                                                                      |
| 10  | Queues, outbox rows and logs holding email or shipping addresses are personal data: encryption at rest and a 30-day retention limit are required.                                                  | Security                                                | Partly: it raised outbox retention, but not encryption or a retention limit for personal data | No. The policy is only on role card X.                                                                                                                                   |
| 11  | Analytics may not receive email addresses or full names.                                                                                                                                           | Security                                                | Yes, indirectly: it asked to justify the fields in `OrderPlaced`                              | Yes. The ADR already excludes them by design (`OrderPlaced` carries no email, name or address).                                                                          |
| 12  | Loyalty "exactly once" depends on writing the dedup ID and the award atomically; a crash between the two writes can double-award.                                                                  | AI review                                               | Yes                                                                                           | Yes. It follows from the ADR's own wording ("alongside") and the scenario's "exactly once".                                                                              |
| 13  | The 1-minute loyalty deadline cannot hold while `loyaltyservice` is down longer than a minute.                                                                                                     | AI review                                               | Yes                                                                                           | Yes. Both requirements are in the scenario.                                                                                                                              |
| 14  | Checkout now depends on PostgreSQL: if the database is down, orders fail, and the card is charged before the order is saved.                                                                       | Architect                                               | No                                                                                            | Partly. The new dependency is visible in the ADR; the charge-before-save order is only visible in `checkoutservice/main.go`, which the reviewer was not allowed to read. |

**The AI's best finding** (one the humans had missed): the atomic deduplication gap, that is if the loyalty consumer crashes between recording the event ID and awarding the points, a redelivery can double-award. No stakeholder raised it, yet the product owner's card shows that double points cost real money.

**The AI's weakest finding** (generic, wrong or irrelevant): asking to justify each field in `OrderPlaced`, as the ADR already keeps email, name and address out of it, and loyalty clearly needs the user ID.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.
