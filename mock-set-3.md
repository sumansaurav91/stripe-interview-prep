# 📘 Stripe Interview Practice Workbook - Mock Set 3

This comprehensive workbook is structured into 4 major sections following Stripe's interview loop format:
- **Coding** (10 Q&A)
- **System Design** (8 Q&A)
- **API/Product** (6 Q&A)
- **Debugging & Behavioral** (6 Q&A)

**Total:** 30 curated Stripe-style questions with detailed answers.

---

## Section 1: Coding Challenges (10)

### 1. Settlement Ordering by Merchant (Heap)

**Question:** How would you implement fair settlement ordering?

**Answer:**
👉 Already covered above — fairness + chronological integrity.

---

### 2. Circular Transfers Detection

**Question:** How do you detect circular transfers in a payment system?

**Answer:**
👉 Already covered — DFS cycle detection.

---

### 3. Credit Card Masking

**Question:** Write a function that takes a card number "4242424242424242" and returns "************4242".

**Answer:**
```typescript
function maskCard(card: string): string {
  return card.slice(0, -4).replace(/\d/g, '*') + card.slice(-4);
}
```

**Stripe-specific insight:** Card data must be tokenized/PCI safe — real code never logs full PAN.

---

### 4. Currency Normalization

**Question:** Convert amounts into minor units (USD: 2 decimals, JPY: 0, KWD: 3).

**Answer:**
```typescript
function toMinorUnits(amount: number, currency: string): number {
  const map: Record<string, number> = { 
    USD: 2, 
    EUR: 2, 
    JPY: 0, 
    KWD: 3 
  };
  return Math.round(amount * Math.pow(10, map[currency] ?? 2));
}
```

**Stripe-specific insight:** Stripe always stores integers in DB to avoid float rounding errors.

---

### 5. Refund Allocation

**Question:** Given a charge with 3 line items, refund $50 across them proportionally.

**Answer:**
- Compute ratio of each line item to total
- Allocate refund = ratio * 50, round carefully to avoid cent mismatch
- Stripe does this to ensure math always balances

---

### 6. Transaction Stream Aggregation

**Question:** Given a stream of events `{ accountId, amount }`, compute rolling balance per account.

**Answer:**
- HashMap: `balance[accountId] += amount`
- Complexity O(n)
- Stripe uses similar approach in ledger streaming pipelines

---

### 7. Detect Duplicate Charges

**Question:** Given a list of charges `{ idempotencyKey, amount }`, remove duplicates.

**Answer:**
- Use HashSet on `idempotencyKey`
- **Stripe rule:** All APIs must be idempotent-safe

---

### 8. Rate Limiting Window

**Question:** Implement per-user 100 requests per 15 minutes.

**Answer:**
- Store `(userId, windowStart, count)`
- Reset after 15 minutes
- **Production:** use Redis TTL + atomic increment

---

### 9. Reconciliation of Events

**Question:** Stripe ledger has 1,000 events/day. Some events missing in merchant DB. Detect differences.

**Answer:**
- Use `event.id` → full outer join
- Merchant should ack every event; missing = alert

---

### 10. Matching Transfers with Refunds

**Question:** Given payouts and refunds, ensure net zero.

**Answer:**
- Group by merchant → `sum(payouts) - sum(refunds) = balance`
- Stripe always enforces ledger consistency

---

## Section 2: System Design (8)

### 11. Rate Limiter

**Question:** Design a scalable rate limiting system.

**Answer:**
👉 Already covered above (Redis + token bucket).

---

### 12. Multi-Currency Ledger

**Question:** Design a ledger system supporting multiple currencies.

**Answer:**
👉 Already covered above (double-entry, minor units).

---

### 13. Real-Time Fraud Detection System

**Question:** How to catch suspicious transactions (e.g., $10k charge from new card)?

**Answer:**
- **Architecture:** Stream events → Kafka → ML scoring service
- **Features:** geolocation, device fingerprint, velocity of charges
- **Latency budget:** <200ms inline decision
- **Stripe insight:** Prioritizes blocking fraud before authorization completes

---

### 14. Idempotent Payment API

**Question:** Ensure client retries don't double-charge.

**Answer:**
- API requires `Idempotency-Key` header
- Store result in Redis keyed by `(key, endpoint)`
- Return same response if retried

---

### 15. Webhook Delivery Service

**Question:** How to design webhook delivery at scale?

**Answer:**
- **Pattern:** Push → retry with exponential backoff
- **Deduplication:** by `event.id`
- **Merchant-facing:** must handle at-least-once delivery

---

### 16. Invoicing System

**Question:** Build a scalable invoice engine.

**Answer:**
- **Tables:** `invoices`, `invoice_items`, `payments`
- **Immutability:** Generate invoice snapshot at time of issue
- **Features:** Support retries + dunning (email, retries)

---

### 17. Real-Time Dashboard for 10M Payments

**Question:** Show merchant dashboard with metrics in <1s.

**Answer:**
- Use pre-aggregated OLAP store (ClickHouse/BigQuery)
- Periodic ETL from ledger
- Serve queries via cached materialized views

---

### 18. Dispute Resolution System

**Question:** Design chargeback handling.

**Answer:**
- Store `disputes` table linked to charges
- Merchant uploads evidence → async workflow
- **SLA tracking:** must respond within 7 days
- **Stripe approach:** Uses workflow orchestration (Airflow/Step Functions)

---

## Section 3: API & Product (6)

### 19. Bulk Payouts API

**Question:** Design an API for bulk payouts.

**Answer:**
👉 Already covered above.

---

### 20. Onboarding API for Connect Accounts

**Question:** Design API to onboard new connected accounts.

**Answer:**
- `POST /accounts` with business info
- Return `account_id`
- Support KYC → async verification → `verified` flag
- **Stripe insight:** Must handle global compliance laws

---

### 21. Subscription API

**Question:** Design API to create subscriptions.

**Answer:**
- `POST /subscriptions` → `{ customer_id, plan_id }`
- Store next invoice date, billing cycle
- Support proration when switching plans

---

### 22. Refund API

**Question:** API to refund a payment.

**Answer:**
- `POST /refunds { charge_id, amount }`
- Must support partial refunds
- Idempotent by `(charge_id, idempotency_key)`

---

### 23. Idempotency Explained to Non-Tech Merchant

**Question:** How would you explain "idempotency" to a merchant?

**Answer:**
"If you click 'refund' 3 times, we guarantee it happens only once. Idempotency means no accidental duplicates."

---

### 24. API Error Design

**Question:** How should Stripe structure errors?

**Answer:**
- **JSON format:** `{ error: { type, code, message, param } }`
- Always actionable messages
- **Stripe value:** Developer empathy

---

## Section 4: Debugging & Behavioral (6)

### 25. Webhook Duplicates

**Question:** How do you handle webhook duplicate delivery?

**Answer:**
👉 Already covered.

---

### 26. Slow Dashboard Queries

**Question:** How do you optimize slow dashboard queries?

**Answer:**
👉 Already covered.

---

### 27. Memory Leak in Node.js API

**Question:** EKS pod OOMKilled every 8h. Debug?

**Answer:**
- Use heap dump → check event listeners/objects not GC'd
- **Common bug:** caching unbounded user sessions
- **Fix:** LRU cache or Redis external store

---

### 28. Charge Failing in Production

**Question:** Merchant reports "all charges failing."

**Answer:**
- Check Stripe status page → gateway outage?
- Debug logs → invalid API key or expired secret?
- **Stripe's approach:** Prioritize blast radius containment

---

### 29. Behavioral: Disagreement with Teammate

**Question:** Tell me about a time you disagreed with a peer.

**Answer (STAR):**
- **Situation:** Disagreement over database choice (SQL vs NoSQL)
- **Task:** Needed reliable ledger system
- **Action:** Collected metrics, ran spike tests, showed why Postgres met ACID needs
- **Result:** Team aligned, decision accepted

**Shows Stripe value:** Seek the Truth, Not Consensus.

---

### 30. Behavioral: Users First

**Question:** Tell me about a time you put users first.

**Answer (STAR):**
- **Situation:** Developer onboarding was slow
- **Task:** Reduce integration pain
- **Action:** Built SDK + examples, wrote better docs
- **Result:** Integration time dropped 70%

**Stripe insight:** Looks for developer-first empathy.

---

## ✅ How to Use This Workbook

### Study Tips
- **Practice coding Qs** in LeetCode-style editor
- **Whiteboard design Qs** (draw diagrams)
- **Roleplay API/product Qs** out loud (simulate PM convo)
- **Reflect on behavioral Qs** with STAR method

### Success Strategy
- Focus on Stripe-specific insights and values
- Emphasize developer experience and empathy
- Demonstrate understanding of payment systems complexity
- Show problem-solving approach with real-world context

---

*Good luck with your Stripe interview! 🚀*
