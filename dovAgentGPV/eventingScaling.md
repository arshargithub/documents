Yep — your intuition is right. Scaling this cleanly is actually *much simpler* than it feels once you separate **ingest scaling** from **processing scaling**.

I’ll walk you through it in layers, from **mailbox → Kafka → LLM-heavy stages**, and I’ll call out the *non-obvious gotchas*.

---

## 1️⃣ Scaling the mailbox ingest layer (polling or webhook)

This layer should be **thin, dumb, and horizontally scalable**.

### Webhook-based ingest

Scaling model:

* Stateless HTTP service
* Fronted by:

  * ALB / NLB
  * API Gateway
* Scale by:

  * increasing pod replicas
  * increasing CPU for burst handling

Key properties:

* No affinity to mailboxes
* Any pod can handle any mailbox event
* Idempotency handled centrally (Redis / compacted Kafka topic)

So yes:

> **Just replicas + pod resources**

That’s exactly right.

---

### Polling-based ingest

Polling has one extra concern: **coordination**.

You don’t want:

* 10 pods polling the same mailbox

You *do* want:

* Mailboxes evenly distributed across pollers

#### Two clean patterns

### Pattern A: Static sharding

* Hash(mailbox_id) % num_pollers
* Each poller only polls mailboxes assigned to it

Works well if:

* Mailbox set is stable
* Scale changes are infrequent

---

### Pattern B: Work-queue driven polling (preferred)

* A scheduler emits:

  * `(mailbox_id, poll_window)` jobs to Kafka/Redis
* Poller pods:

  * pull jobs
  * poll mailbox
  * emit emails
  * ack job

This gives you:

* Dynamic scaling
* Backpressure
* Visibility

---

## 2️⃣ Kafka is your real scaling boundary

Once emails hit Kafka:

> **Mailbox count stops mattering**

Kafka now decouples:

* Ingest rate
* Processing rate
* Failure domains

This is the *core architectural win* you’re after.

---

## 3️⃣ Scaling classification + extraction (the expensive part)

This is where most people get it wrong.

### Key principle

> Scale by **partition count × consumer replicas**

Not by mailbox count.

---

### Example

* Topic partitions: 40
* Consumer group: `classifier-extractor`
* Max parallelism: 40

You can run:

* 10 pods → each handles ~4 partitions
* 40 pods → each handles 1 partition
* 80 pods → still max 40 active workers

Kafka enforces the ceiling.

---

### Autoscaling strategy

Use:

* Consumer lag (per partition)
* CPU / memory
* LLM request latency

HPA signals:

* lag > threshold → scale consumers
* lag shrinking → scale down

---

## 4️⃣ Dispatch stage scaling

Dispatch is usually:

* IO-bound
* Fast
* Retry-heavy

Best practices:

* Separate consumer group
* Separate topic (`extracted-events`)
* Independent scaling

This prevents slow downstream APIs from backing up extraction.

---

## 5️⃣ What about hundreds of thousands of emails?

This is the part execs worry about — but your design handles it naturally.

### What happens under massive burst

1. Ingest layer absorbs burst
2. Kafka log grows
3. Consumers fall behind
4. No data loss
5. Latency increases predictably

This is **controlled degradation**, not failure.

---

## 6️⃣ Where people accidentally break scaling

### ❌ Central DB in hot path

You already spotted this.

* Postgres as a state ledger in every stage
* High write amplification
* Lock contention
* Throttles scale

Kafka + object storage is the right backbone.

---

### ❌ Synchronous LLM calls in ingest

Never do extraction/classification in the webhook or poller.

Ingest should:

* persist
* enqueue
* return

---

### ❌ One mailbox = one queue

That explodes operational complexity and kills throughput.

You avoided this — good instinct.

---

## 7️⃣ Mental model to remember

Here’s the mantra that will keep you out of trouble:

> **Ingress scales by replicas. Processing scales by partitions. Correctness scales by keys.**

If you remember just that, you’ll design this right every time.

---

## 8️⃣ Exec-level one-liner (use this verbatim)

> “We can onboard thousands of mailboxes because ingest scales horizontally, Kafka absorbs bursts, and processing scales independently based on demand — mailbox count doesn’t affect system stability.”

If you want next:

* A concrete sizing example (insurance realistic volumes)
* HPA configs that actually work with Kafka
* Or failure scenarios and how the system recovers

You’re thinking like a platform owner now, not a feature builder — that’s exactly what Borealis leadership wants to see.
