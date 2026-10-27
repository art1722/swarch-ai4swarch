**Language:** [ภาษาไทย](05-reflection.md) | English

# Individual reflection (≤ 150 words)

**Name / Student ID:** : Chawakorn Samosorn / 6631308221

**My role on the review board:** : I did every roles, namely architect, AI analyst, product owner (card P), Site reliability engineer (SRE) (card S) and security & privacy officer (card X).

After Part A and Part B: which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

…
I would trust AI with tasks where the information it needs is within its context. For example, AI can recovered all 16 dependencies from the manifest files correctly in run B. In Run 2b, with the scenario as context, its ADR used every quality numbers.

I would keep the decisions that depend on information outside context, the decisions that have a high impact, and the final architectural decisions for humans. For instance, in my board meeting, the AI reviewer accepted the RabbitMQ design with changes. However, the role cards showed unseen factors, such that the sale was 3 weeks away, two engineers were on call, and the team had ever used RabbitMQ only on their laptops, etc. The board therefore deferred RabbitMQ and used the PostgreSQL outbox as the queue, an option the AI reviewer had suggested without knowing the constraints that made it the better choice.
