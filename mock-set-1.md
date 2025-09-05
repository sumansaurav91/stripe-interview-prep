🎯 Stripe Full-Stack Engineer Mock Interview
1. Coding (45 min)

👉 Question:
You are building a transaction reconciliation tool for Stripe. Given two lists of transactions (one from Stripe’s internal ledger, and one from a bank statement), return all unmatched transactions.

Each transaction has:

{
  id: string,
  amount: number,
  currency: string
}


Return transactions that appear in one list but not the other.

Consider performance for large datasets (millions of records).

Assume id is unique within each list.

Follow-up:

What if transactions can appear in both lists with the same id but different amount (discrepancy)?

How would you handle it in a production system?

2. Full Stack / System Design (60 min)

👉 Question:
“Design a system for generating and sending invoices to businesses via Stripe.”

Discuss:

Backend API endpoints (create invoice, list invoices, pay invoice, webhook for payment updates).

Database schema (tables for users, invoices, payments, line_items).

Frontend flow (React UI for creating an invoice, showing status).

Scalability: Handling millions of invoices across regions.

Stripe considerations: Idempotency keys, retries, fraud prevention.

3. API / Product Design (45 min)

👉 Question:
“Imagine you’re designing an API for developers to set up subscription billing (like Netflix using Stripe). What would the API look like?”

Define 2–3 endpoints (e.g., POST /subscriptions, GET /subscriptions/:id, DELETE /subscriptions/:id).

How would you handle trial periods, cancellations, and failed payments?

How do you ensure the API is developer-friendly?

How do you version it without breaking old integrations?

4. Debugging / Technical Deep Dive (30 min)

👉 Scenario:
A Stripe dashboard shows “Payment Failed” for 5% of transactions. Logs show:

Error: PaymentIntent confirmation failed - idempotency key reused


Walk through what might be happening.

How would you debug it?

What fixes would you propose (code, infra, or product changes)?

5. Behavioral / Stripe Values (30 min)

👉 Sample Questions:

Users First: Tell me about a time you built something with a strong focus on the end-user experience.

High Standards: What’s a piece of code or system you’re most proud of, and why?

Rigor: Describe a technical decision you made that required deep analysis of tradeoffs.

Collaboration: Tell me about a time you worked across different teams and faced misalignment. How did you handle it?

Why Stripe? What excites you about Stripe’s mission?

✅ If you practice these thoroughly, you’ll be in great shape for Stripe.

Would you like me to also write out detailed solutions & ideal answers for each section (coding implementation, system design diagrams, API specs, and sample behavioral answers), so you can compare your attempt vs. “Stripe-level expectations”?

You said:
Yes
ChatGPT said:
Thought for 14s

Awesome — here’s a complete Stripe-style answer key you can practice against. It includes ideal solutions, tradeoffs, and sample phrasing you can reuse in an interview.

1) Coding — Reconciliation (ideal solution)
Problem

Given two large lists of transactions (Stripe ledger vs. bank), return:

unmatched: present in one list but not the other (by id)

discrepant: same id exists in both, but amount or currency differ

Constraints & goals

Scale to millions of rows

O(n + m) time, O(n + m) space (or streaming alternatives)

TypeScript reference solution
type Tx = { id: string; amount: number; currency: string };

function reconcile(stripe: Tx[], bank: Tx[]) {
  const map = new Map<string, Tx>();
  for (const t of stripe) map.set(t.id, t);

  const unmatched: Tx[] = [];
  const discrepant: { id: string; stripe?: Tx; bank?: Tx; reason: string }[] = [];

  // Walk bank, check against Stripe map
  for (const b of bank) {
    const s = map.get(b.id);
    if (!s) {
      unmatched.push(b); // in bank not in stripe
    } else {
      // same id exists; check discrepancy
      if (s.amount !== b.amount || s.currency !== b.currency) {
        discrepant.push({
          id: b.id,
          stripe: s,
          bank: b,
          reason: [
            s.amount !== b.amount ? `amount ${s.amount} != ${b.amount}` : "",
            s.currency !== b.currency ? `currency ${s.currency} != ${b.currency}` : "",
          ].filter(Boolean).join("; ")
        });
      }
      map.delete(b.id); // consumed
    }
  }

  // what remains in map are in stripe but not in bank
  for (const s of map.values()) unmatched.push(s);

  return { unmatched, discrepant };
}


Complexity: O(n + m) time; O(n + m) extra space.

Edge cases

Duplicate IDs within a single feed (should not happen; if it does, dedupe or flag as data quality issue).

Floating amounts vs. integer cents — store as integer minor units to avoid FP errors.

Different currencies: treat as discrepancy, not convertible.

Production considerations

Streaming/External sort for memory limits:

If feeds are too large, sort both by id (external merge sort) and do a linear merge in O(1) extra memory (besides buffers).

Data quality:

Enforce uniqueness constraints at ingest.

Validate schemas.

Auditability:

Persist a reconciliation run record with counts and samples.

Idempotent runs:

Key each run by (source snapshot ids, date) to avoid double-processing.

2) Full Stack / System Design — Invoicing via Stripe (ideal answer)
Goal

Build an end-to-end invoicing system: create invoices, send to customers, accept payments, track status, and surface results in a dashboard. Optimize for correctness, idempotency, and developer ergonomics.

High-level architecture

Frontend (React): Invoice builder, list/detail, payment status.

Backend (API, e.g., NestJS/Express): AuthN/Z, invoice CRUD, idempotent mutations, integration with Stripe (PaymentIntents/Invoices), webhooks receiver.

DB (PostgreSQL): accounts, customers, invoices, line_items, payments, events.

Async: Queue for email sending, PDF rendering, and webhook processing.

Storage: Object store for PDF invoices.

Observability: Metrics (invoices created/paid/failed), logs with correlation IDs, traces.

API endpoints (v1)

POST /v1/invoices
Body: { customer_id, currency, due_date, line_items: [{description, unit_price, quantity, tax_rate_id?}], memo?, send_email?: boolean }
Headers: Idempotency-Key: <uuid>

GET /v1/invoices/:id

POST /v1/invoices/:id/send (enqueue email + generate PDF)

POST /v1/invoices/:id/pay (creates Stripe PaymentIntent if not present; client_secret returned)

GET /v1/invoices?customer_id=&status=&cursor=&limit=

POST /v1/webhooks/stripe (verifies signature, enqueues events)

POST /v1/invoices/:id/cancel

POST /v1/invoices/:id/void

DB schema (PostgreSQL)
CREATE TABLE accounts (
  id UUID PRIMARY KEY,
  name TEXT NOT NULL
);

CREATE TABLE customers (
  id UUID PRIMARY KEY,
  account_id UUID REFERENCES accounts(id),
  email TEXT NOT NULL,
  name TEXT,
  UNIQUE (account_id, email)
);

CREATE TYPE invoice_status AS ENUM ('draft','sent','viewed','paid','void','past_due','uncollectible');

CREATE TABLE invoices (
  id UUID PRIMARY KEY,
  account_id UUID REFERENCES accounts(id),
  customer_id UUID REFERENCES customers(id),
  currency TEXT NOT NULL,
  status invoice_status NOT NULL DEFAULT 'draft',
  subtotal_cents BIGINT NOT NULL,
  tax_cents BIGINT NOT NULL DEFAULT 0,
  total_cents BIGINT NOT NULL,
  due_date TIMESTAMPTZ,
  stripe_payment_intent_id TEXT,
  pdf_url TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE line_items (
  id UUID PRIMARY KEY,
  invoice_id UUID REFERENCES invoices(id),
  description TEXT NOT NULL,
  unit_price_cents BIGINT NOT NULL,
  quantity INT NOT NULL,
  tax_rate NUMERIC(5,2) DEFAULT 0
);

CREATE TYPE payment_status AS ENUM('requires_payment_method','processing','succeeded','failed','canceled','refunded','partial');

CREATE TABLE payments (
  id UUID PRIMARY KEY,
  invoice_id UUID REFERENCES invoices(id),
  stripe_payment_intent_id TEXT,
  amount_cents BIGINT NOT NULL,
  status payment_status NOT NULL,
  raw JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Outbox/events for reliability
CREATE TABLE events (
  id UUID PRIMARY KEY,
  aggregate_type TEXT,
  aggregate_id UUID,
  type TEXT,
  payload JSONB,
  created_at TIMESTAMPTZ DEFAULT now(),
  processed_at TIMESTAMPTZ
);

Core flows

Create & send invoice

Client → POST /invoices (with Idempotency-Key).

Server validates, computes subtotal/total in minor units, writes invoice + line items in a transaction.

Enqueue PDF render & email job; update status: 'sent'.

Store PDF in object storage; email contains a hosted “Pay Now” link.

Pay invoice

Client hits POST /invoices/:id/pay.

Backend ensures one PaymentIntent per invoice (reuse if exists).

Return client_secret to the client to confirm payment via Stripe Elements.

Stripe emits webhooks (e.g., payment_intent.succeeded).

Webhook handler (idempotent) updates payments + invoices.status='paid'.

Webhooks

Verify signature.

Use idempotency table keyed by event.id.

Process asynchronously (queue) to avoid timeouts.

Idempotency

All create/update endpoints require Idempotency-Key.

Persist request hash; if a key is reused with different body, return 409 (conflict).

Retry-safe: client can retry without double effects.

Scale & reliability

Hot partitions: paginate by (created_at, id) and use composite indexes.

Multi-region: region-anchored accounts; write-local/read-global; replicate using logical replication. Webhook processing per region with global de-dupe by event.id.

Queues: DLQs for failed jobs; exponential backoff.

Backfills: use outbox → worker → Stripe to keep side effects reliable.

Security

OAuth/JWT per account; row-level access checks.

PII minimization; store only necessary customer data.

Encrypt at rest; rotate keys; strict secrets management.

3) API / Product Design — Subscriptions (ideal answer)
States

inactive → active → past_due → canceled (+ trialing, paused optional). Billing cycle anchor and proration rules matter.

Minimal endpoints (v1)

POST /v1/subscriptions

GET /v1/subscriptions/:id

DELETE /v1/subscriptions/:id (cancel now or at period end)

POST /v1/subscriptions/:id/pause / /resume (optional)

POST /v1/webhooks/stripe

Request/response examples

Create subscription

POST /v1/subscriptions
Idempotency-Key: 9b9e...

{
  "customer_id": "cus_123",
  "items": [
    { "price_id": "price_basic", "quantity": 1 }
  ],
  "trial_days": 14,
  "payment_behavior": "default_incomplete", 
  "collection_method": "charge_automatically", 
  "coupon_code": "WELCOME10"
}


Response

{
  "id": "sub_abc",
  "status": "trialing",
  "current_period_end": "2025-09-12T00:00:00Z",
  "latest_invoice_id": "inv_987",
  "client_secret": "pi_client_secret_..."  // if payment needed
}


Get

GET /v1/subscriptions/sub_abc


Response includes items, status, next invoice estimate, past_due reasons.

Cancel

DELETE /v1/subscriptions/sub_abc?at_period_end=true

Key behaviors & tradeoffs

Trials: set trial_end = now + N days; convert automatically at period end; remind via email.

Failed payments:

Retry schedule (e.g., 1h, 24h, 3d, 7d).

Dunning: send emails, update portal banner, eventually mark past_due → canceled.

Proration:

When changing tier mid-cycle, compute delta for remaining period; optionally disable via proration_behavior=none.

Versioning:

URL versioning (/v1, /v2) + Stripe-Version style header for behavior changes.

Deprecate gracefully with sunset headers and migration docs.

DX:

Consistent error shapes:

{ "error": { "code": "invalid_request", "message": "price_id unknown", "param": "items[0].price_id" } }


Idempotency required for all writes.

Webhook types documented with examples.

SDKs in JS, Python, Java with typed models.

4) Debugging / Deep Dive — “idempotency key reused”
Symptom

5% of payments show: PaymentIntent confirmation failed - idempotency key reused.

Likely causes

Client retries with the same idempotency key but a different request body
Stripe treats this as a conflict (safety feature). E.g., first attempt: amount=1000; retry sends amount=1200 with same key → failure.

Server reuses a single idempotency key across multiple distinct operations
e.g., same key for different PaymentIntents.

Race conditions
Multiple clicks “Pay” generate concurrent confirmations with one key.

Webhook confusion
App treats webhook re-deliveries as “we should confirm again” and accidentally re-hits confirm with an old key.

Debug plan (what you’d say aloud)

Correlate by request_id / idempotency key in logs.

Compare first-seen request body hash vs. retries. If mismatch → root cause.

Inspect client code for key generation scope (should be 1:1 with a logical operation).

Check network timeouts: client may retry before server returns, causing double-confirm.

Validate webhook handler is idempotent and never calls confirm.

Sample 20 failures; confirm pattern.

Fixes (code + process)

Generate one idempotency key per logical op, store it, and reuse only if the body is bit-for-bit identical.

Confirm on client only once; disable duplicate clicks; debounce.

Server-side guard:

Keep a table: (idempotency_key, sha256(body), status, result_reference).

On reuse with different sha: respond 409 conflict with stored canonical params.

Backoff & visibility: exponential backoff on 5xx/timeouts; alert on >0.5% conflicts.

NestJS-style patch (sketch)

// middleware: enforce idempotency on writes
@Injectable()
export class IdempotencyMiddleware implements NestMiddleware {
  async use(req: Request, res: Response, next: NextFunction) {
    const key = req.header('Idempotency-Key');
    if (!key) return res.status(428).json({ error: { code: 'missing_idempotency_key' }});

    const bodyHash = sha256(JSON.stringify(req.body));
    const existing = await this.repo.findOne({ where: { key } });

    if (!existing) {
      await this.repo.insert({ key, body_hash: bodyHash, status: 'pending' });
      return next();
    }
    if (existing.body_hash !== bodyHash) {
      return res.status(409).json({ error: { code: 'idempotency_conflict' }});
    }
    if (existing.status === 'succeeded') {
      return res.json(existing.cached_response);
    }
    return next(); // pending or failed → allow retry
  }
}

5) Behavioral — Stripe Values (sample STAR answers you can tailor)
Users first — “Making a painful task effortless”

S: Our EKS-based event API processed ~200 req/s; finance team struggled to reconcile failures.
T: Reduce user pain: add clear failure reasons and self-serve retries.
A: Added structured error codes and an “idempotent replay” endpoint; surfaced status in UI with filters.
R: Reduced support tickets by 48%, median time-to-resolution from 2h → 15m. Shows empathy + measurable impact.

High standards — “Preventing regressions at scale”

S: Memory heap/CPU spikes every 8 hours on a NestJS service.
T: Make the system stable under continuous load.
A: Added p99 profiling, capped concurrency, optimized ORM N+1, introduced circuit breakers, and right-sized pod requests/limits; added load test in CI.
R: Eliminated OOMKills, cut p95 latency 40%, stabilized autoscaling. Demonstrates craftsmanship.

Think rigorously — “Choosing storage for event sourcing”

S: Designing event store in DynamoDB with Postgres read models.
T: Guarantee write throughput + query performance.
A: Benchmarked partition strategies, wrote a simulator for hot partition risk, chose aggregateId as PK, version as SK, introduced outbox → CDC to Postgres, and defined eventually-consistent read patterns.
R: Sustained >50k events/min with zero hot partitions during peak; clean, defensible tradeoffs.

Collaboration — “Unblocking cross-team integration”

S: Auth0 signup metadata missing in post-registration actions; team friction over ownership.
T: Align on a fix without blocking release.
A: Reproduced issue, found SDK field mismatch; proposed a minimal schema change; paired with platform team to deploy.
R: Restored metadata flow, unblocked onboarding, prevented similar issues with a contract test.

Why Stripe? (concise)

“I love Stripe’s focus on developer tools for real businesses. Payments and fintech demand correctness, idempotency, and thoughtful APIs—exactly the kind of rigor I enjoy. I’ve shipped high-throughput services, event-sourced systems, and idempotent APIs; I want to apply that craft where it multiplies global economic access.”

Practice rubric (self-grading)

Coding (45m)

✅ Correct & complete (handles unmatched + discrepant)

✅ O(n + m), minor units, clean code, tests

✅ Mentions streaming/sorting for huge data

System design (60m)

✅ Clear APIs, schema, flows, idempotency

✅ Webhooks, queues, outbox, retries, DLQ

✅ Observability, security, multi-region story

API/Product (45m)

✅ States, trials, proration, dunning

✅ Versioning & DX (errors, SDKs)

✅ Tradeoffs clearly articulated

Debugging (30m)

✅ Root cause via logs & request hashing

✅ Precise, low-risk fixes (key scoping, conflicts)

✅ Rollout + alerting plan

Behavioral (30m)

✅ STAR, measurable outcomes

✅ Values alignment

✅ Tight “Why Stripe”
