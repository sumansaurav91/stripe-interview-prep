🎯 Stripe Mock Interview Set #2
1) Coding (45 min)
Question:

You’re building a dispute resolution system for Stripe. Given an array of payment events, return the longest streak of consecutive successful payments for a merchant.

Each event looks like:

{
  id: string;
  merchantId: string;
  status: "succeeded" | "failed" | "pending";
  timestamp: number; // Unix epoch seconds
}


A streak means consecutive chronological events for that merchant with status = "succeeded".

Return a map { merchantId: longestStreak }.

Handle millions of events efficiently.

Answer / Approach

Step 1: Group events by merchantId.

Step 2: Sort each group by timestamp.

Step 3: Scan sequentially to count streaks.

Complexity: O(n log n) due to sorting, O(n) scan.

Optimization: If upstream data is already time-ordered, drop sort → O(n).

TypeScript Solution
type Event = {
  id: string;
  merchantId: string;
  status: "succeeded" | "failed" | "pending";
  timestamp: number;
};

function longestStreak(events: Event[]): Record<string, number> {
  const grouped: Record<string, Event[]> = {};
  for (const e of events) {
    (grouped[e.merchantId] ||= []).push(e);
  }

  const result: Record<string, number> = {};
  for (const [merchantId, list] of Object.entries(grouped)) {
    list.sort((a, b) => a.timestamp - b.timestamp);
    let longest = 0, cur = 0;
    for (const e of list) {
      if (e.status === "succeeded") {
        cur++;
        longest = Math.max(longest, cur);
      } else {
        cur = 0;
      }
    }
    result[merchantId] = longest;
  }
  return result;
}

Edge cases

Merchant with no successes → streak = 0.

Multiple merchants → each independent.

Millions of events → watch memory; could stream events sorted by merchant,timestamp (merge-sort style).

Production considerations

Store events partitioned by (merchant_id, timestamp) in DB → avoid per-merchant sort.

Precompute streak counters in warehouse for dashboards.

2) Full-Stack / System Design (60 min)
Question:

“Design a refund service for Stripe merchants.”

Merchants can request refunds for charges. Requirements:

Support partial & full refunds.

Refunds should be idempotent (same refund request can be retried).

Must handle asynchronous processing (refunds can take minutes).

Provide API + dashboard view.

Scale to tens of millions of refunds/day.

Answer (Stripe-level)
Core APIs

POST /v1/refunds
Body: { charge_id, amount?, reason?, idempotency_key }

GET /v1/refunds/:id

GET /v1/refunds?charge_id=...

POST /v1/webhooks/refunds (for status updates)

Refund lifecycle

pending → processing → succeeded | failed.

DB Schema
CREATE TABLE refunds (
  id UUID PRIMARY KEY,
  charge_id UUID NOT NULL,
  merchant_id UUID NOT NULL,
  amount_cents BIGINT NOT NULL,
  status TEXT CHECK (status IN ('pending','processing','succeeded','failed')),
  reason TEXT,
  idempotency_key TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(merchant_id, idempotency_key) -- ensures idempotency
);

Flow

Request: Merchant POSTs refund with idempotency key.

Validation:

Ensure charge exists & refundable.

Check idempotency (same merchant + key → return existing record).

Persist: Insert refund row with pending.

Async Worker: Picks refund, calls external payment rails (ACH, card network).

Update: Refund becomes processing, then webhook updates → succeeded or failed.

Notify: Merchant can GET or receive webhook.

Scale

Partition refunds by (merchant_id, created_at) for queries.

Kafka/Kinesis queue for async refund jobs.

DLQ for failed attempts.

Ensure at-least-once delivery but with idempotency guard.

Frontend

Dashboard React UI:

“Request refund” modal.

Refund list view with filters (status/date).

Status updates via polling or WebSocket → real-time.

3) API / Product Design (45 min)
Question:

“Design an API for webhook management at Stripe.”

Merchants want to register, test, and manage webhooks.

Answer
Endpoints

POST /v1/webhook_endpoints
Body: { url, events: ["payment_intent.succeeded", "charge.refunded"], secret? }

GET /v1/webhook_endpoints

GET /v1/webhook_endpoints/:id

POST /v1/webhook_endpoints/:id/test

DELETE /v1/webhook_endpoints/:id

Features

Support multiple endpoints per merchant.

Allow filtering by event types → less noise.

Rotate secrets easily.

Provide retry policies + DLQ if endpoint down.

Developer experience

Provide dashboard UI for quick setup.

Expose Test webhook with example payload.

Clear error messages for misconfigured URLs (timeouts, 500).

Versioning

Webhooks tied to Stripe API version of account.

Breaking change → require opt-in per endpoint.

4) Debugging / Technical Deep Dive (30 min)
Scenario:

A merchant complains: “Refund requests sometimes succeed, sometimes fail with duplicate refund error.”

Logs:

POST /v1/refunds
Idempotency-Key: refund-123
Response: 200 OK { refund_id: r1, status: "succeeded" }

POST /v1/refunds
Idempotency-Key: refund-123
Body differs: { charge_id: c1, amount: 2000 } vs. { charge_id: c1, amount: 1500 }
Response: 409 Conflict duplicate refund

Answer

Root cause:

Same idempotency key used with different payloads.

Stripe guarantees idempotency only if request body is identical.

Different amount = conflict.

Debugging steps:

Compare stored payload hash vs. new.

Look at merchant integration code → are they reusing a static key?

Check client retries: is the library setting keys properly?

Fix:

Educate merchant: idempotency key must be unique per refund attempt, but consistent across retries of that same attempt.

Add middleware: hash request body on first use; reject mismatched reuse.

Improve error docs: clarify "duplicate refund" vs. "conflict on idempotency".

5) Behavioral / Stripe Values (30 min)
Q1: Tell me about a time you simplified a complex system for end users.

A (STAR):

S: Our EKS-based pipeline produced opaque errors → users confused.

T: Improve developer experience.

A: Built error mapper → mapped low-level infra errors to clear categories (auth, quota, network).

R: Ticket volume dropped 30%, dev onboarding much smoother. Shows “Users First.”

Q2: Describe a time you identified a scaling bottleneck before it became a production issue.

A:

S: High API traffic (200/s) → Postgres read spikes.

T: Anticipate failure before peak season.

A: Ran load tests, spotted missing index on (account_id, created_at). Added composite index + read replicas.

R: Sustained 10x load in production without incident. Shows “High Standards & Rigor.”

Q3: Why Stripe?

A:
“I’m excited by Stripe’s mission of growing the GDP of the internet. My experience building scalable APIs, idempotent systems, and event-driven pipelines aligns directly with Stripe’s challenges — correctness and developer experience. Stripe’s culture of rigor and empathy is exactly what I want to contribute to.”
