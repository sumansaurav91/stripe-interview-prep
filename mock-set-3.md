This will be structured into 4 major sections (like Stripe’s loop):

Coding (10 Q&A)

System Design (8 Q&A)

API/Product (6 Q&A)

Debugging & Behavioral (6 Q&A)

Total: 30 curated Stripe-style questions with detailed answers.

📘 Stripe Interview Practice Workbook
Section 1: Coding Challenges (10)
1. Settlement Ordering by Merchant (Heap)

👉 Already covered above — fairness + chronological integrity.

2. Circular Transfers Detection

👉 Already covered — DFS cycle detection.

3. Credit Card Masking

Q: Write a function that takes a card number "4242424242424242" and returns "************4242".

A:

function maskCard(card: string): string {
  return card.slice(0, -4).replace(/\d/g, '*') + card.slice(-4);
}


Stripe-specific: Card data must be tokenized/PCI safe — real code never logs full PAN.

4. Currency Normalization

Q: Convert amounts into minor units (USD: 2 decimals, JPY: 0, KWD: 3).

A:

function toMinorUnits(amount: number, currency: string): number {
  const map: Record<string, number> = { USD: 2, EUR: 2, JPY: 0, KWD: 3 };
  return Math.round(amount * Math.pow(10, map[currency] ?? 2));
}


Stripe always stores integers in DB to avoid float rounding errors.

5. Refund Allocation

Q: Given a charge with 3 line items, refund $50 across them proportionally.

A:

Compute ratio of each line item to total.

Allocate refund = ratio * 50, round carefully to avoid cent mismatch.

Stripe does this to ensure math always balances.

6. Transaction Stream Aggregation

Q: Given a stream of events { accountId, amount }, compute rolling balance per account.

A:

HashMap: balance[accountId] += amount.

Complexity O(n).
Stripe uses similar approach in ledger streaming pipelines.

7. Detect Duplicate Charges

Q: Given a list of charges { idempotencyKey, amount }, remove duplicates.

A:

Use HashSet on idempotencyKey.

Stripe rule: All APIs must be idempotent-safe.

8. Rate Limiting Window

Q: Implement per-user 100 requests per 15 minutes.

A:

Store (userId, windowStart, count).

Reset after 15 minutes.

Production: use Redis TTL + atomic increment.

9. Reconciliation of Events

Q: Stripe ledger has 1,000 events/day. Some events missing in merchant DB. Detect differences.

A:

Use event.id → full outer join.

Merchant should ack every event; missing = alert.

10. Matching Transfers with Refunds

Q: Given payouts and refunds, ensure net zero.

A:

Group by merchant → sum(payouts) - sum(refunds) = balance.

Stripe always enforces ledger consistency.

Section 2: System Design (8)
11. Rate Limiter

👉 Already covered above (Redis + token bucket).

12. Multi-Currency Ledger

👉 Already covered above (double-entry, minor units).

13. Real-Time Fraud Detection System

Q: How to catch suspicious transactions (e.g., $10k charge from new card)?

A:

Stream events → Kafka → ML scoring service.

Features: geolocation, device fingerprint, velocity of charges.

Latency budget: <200ms inline decision.

Stripe prioritizes blocking fraud before authorization completes.

14. Idempotent Payment API

Q: Ensure client retries don’t double-charge.

A:

API requires Idempotency-Key header.

Store result in Redis keyed by (key, endpoint).

Return same response if retried.

15. Webhook Delivery Service

Q: How to design webhook delivery at scale?

A:

Push → retry with exponential backoff.

Deduplication by event.id.

Merchant-facing: must handle at-least-once delivery.

16. Invoicing System

Q: Build a scalable invoice engine.

A:

Tables: invoices, invoice_items, payments.

Generate invoice snapshot at time of issue (immutability).

Support retries + dunning (email, retries).

17. Real-Time Dashboard for 10M Payments

Q: Show merchant dashboard with metrics in <1s.

A:

Use pre-aggregated OLAP store (ClickHouse/BigQuery).

Periodic ETL from ledger.

Serve queries via cached materialized views.

18. Dispute Resolution System

Q: Design chargeback handling.

A:

Store disputes table linked to charges.

Merchant uploads evidence → async workflow.

SLA tracking (must respond within 7 days).

Stripe uses workflow orchestration (Airflow/Step Functions).

Section 3: API & Product (6)
19. Bulk Payouts API

👉 Already covered above.

20. Onboarding API for Connect Accounts

Q: Design API to onboard new connected accounts.

A:

POST /accounts with business info.

Return account_id.

Support KYC → async verification → verified flag.

Stripe: must handle global compliance laws.

21. Subscription API

Q: Design API to create subscriptions.

A:

POST /subscriptions → { customer_id, plan_id }.

Store next invoice date, billing cycle.

Support proration when switching plans.

22. Refund API

Q: API to refund a payment.

A:

POST /refunds { charge_id, amount }.

Must support partial refunds.

Idempotent by (charge_id, idempotency_key).

23. Idempotency Explained to Non-Tech Merchant

Q: How would you explain “idempotency” to a merchant?

A:

“If you click ‘refund’ 3 times, we guarantee it happens only once. Idempotency means no accidental duplicates.”

24. API Error Design

Q: How should Stripe structure errors?

A:

JSON format: { error: { type, code, message, param } }.

Always actionable messages.

Stripe values developer empathy.

Section 4: Debugging & Behavioral (6)
25. Webhook Duplicates

👉 Already covered.

26. Slow Dashboard Queries

👉 Already covered.

27. Memory Leak in Node.js API

Q: EKS pod OOMKilled every 8h. Debug?

A:

Use heap dump → check event listeners/objects not GC’d.

Common bug: caching unbounded user sessions.

Fix: LRU cache or Redis external store.

28. Charge Failing in Production

Q: Merchant reports “all charges failing.”

A:

Check Stripe status page → gateway outage?

Debug logs → invalid API key or expired secret?

Stripe’s approach: prioritize blast radius containment.

29. Behavioral: Disagreement with Teammate

Q: Tell me about a time you disagreed with a peer.

A (STAR):

S: Disagreement over database choice (SQL vs NoSQL).

T: Needed reliable ledger system.

A: Collected metrics, ran spike tests, showed why Postgres met ACID needs.

R: Team aligned, decision accepted.

Shows Stripe value: Seek the Truth, Not Consensus.

30. Behavioral: Users First

Q: Tell me about a time you put users first.

A (STAR):

S: Developer onboarding was slow.

T: Reduce integration pain.

A: Built SDK + examples, wrote better docs.

R: Integration time dropped 70%.

Stripe looks for developer-first empathy.

✅ How to Use This Workbook

Practice coding Qs in LeetCode-style editor.

Whiteboard design Qs (draw diagrams).

Roleplay API/product Qs out loud (simulate PM convo).

Reflect on behavioral Qs with STAR.
