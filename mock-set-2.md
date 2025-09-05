# 🎯 Stripe Mock Interview Set #2

## Table of Contents
- [1. Coding Challenge (45 min)](#1-coding-challenge-45-min)
- [2. Full-Stack / System Design (60 min)](#2-full-stack--system-design-60-min)
- [3. API / Product Design (45 min)](#3-api--product-design-45-min)
- [4. Debugging / Technical Deep Dive (30 min)](#4-debugging--technical-deep-dive-30-min)
- [5. Behavioral / Stripe Values (30 min)](#5-behavioral--stripe-values-30-min)

---

## 1. Coding Challenge (45 min)

### Question

You're building a dispute resolution system for Stripe. Given an array of payment events, return the longest streak of consecutive successful payments for a merchant.

#### Event Structure
Each event looks like:
```typescript
{
  id: string;
  merchantId: string;
  status: "succeeded" | "failed" | "pending";
  timestamp: number; // Unix epoch seconds
}
```

#### Requirements
- A streak means consecutive chronological events for that merchant with status = "succeeded"
- Return a map `{ merchantId: longestStreak }`
- Handle millions of events efficiently

### Answer / Approach

#### Algorithm Steps
1. **Group events by merchantId**
2. **Sort each group by timestamp**
3. **Scan sequentially to count streaks**

#### Complexity Analysis
- **Time Complexity**: O(n log n) due to sorting, O(n) scan
- **Optimization**: If upstream data is already time-ordered, drop sort → O(n)

### TypeScript Solution

```typescript
type Event = {
  id: string;
  merchantId: string;
  status: "succeeded" | "failed" | "pending";
  timestamp: number;
};

function longestStreak(events: Event[]): Record<string, number> {
  const grouped: Record<string, Event[]> = {};
  
  // Group events by merchantId
  for (const e of events) {
    (grouped[e.merchantId] ||= []).push(e);
  }

  const result: Record<string, number> = {};
  
  // Process each merchant's events
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
```

### Edge Cases
- **Merchant with no successes** → streak = 0
- **Multiple merchants** → each independent
- **Millions of events** → watch memory; could stream events sorted by merchant, timestamp (merge-sort style)

### Production Considerations
- Store events partitioned by `(merchant_id, timestamp)` in DB → avoid per-merchant sort
- Precompute streak counters in warehouse for dashboards

---

## 2. Full-Stack / System Design (60 min)

### Question

**"Design a refund service for Stripe merchants."**

Merchants can request refunds for charges.

#### Requirements
- Support partial & full refunds
- Refunds should be idempotent (same refund request can be retried)
- Must handle asynchronous processing (refunds can take minutes)
- Provide API + dashboard view
- Scale to tens of millions of refunds/day

### Answer (Stripe-level)

#### Core APIs

```http
POST /v1/refunds
Body: {
  charge_id,
  amount?, 
  reason?, 
  idempotency_key
}

GET /v1/refunds/:id

GET /v1/refunds?charge_id=...

POST /v1/webhooks/refunds (for status updates)
```

#### Refund Lifecycle
```
pending → processing → succeeded | failed
```

#### DB Schema

```sql
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
```

#### Flow

1. **Request**: Merchant POSTs refund with idempotency key
2. **Validation**:
   - Ensure charge exists & refundable
   - Check idempotency (same merchant + key → return existing record)
3. **Persist**: Insert refund row with pending
4. **Async Worker**: Picks refund, calls external payment rails (ACH, card network)
5. **Update**: Refund becomes processing, then webhook updates → succeeded or failed
6. **Notify**: Merchant can GET or receive webhook

#### Scale
- Partition refunds by `(merchant_id, created_at)` for queries
- Kafka/Kinesis queue for async refund jobs
- DLQ for failed attempts
- Ensure at-least-once delivery but with idempotency guard

#### Frontend

**Dashboard React UI**:
- "Request refund" modal
- Refund list view with filters (status/date)
- Status updates via polling or WebSocket → real-time

---

## 3. API / Product Design (45 min)

### Question

**"Design an API for webhook management at Stripe."**

Merchants want to register, test, and manage webhooks.

### Answer

#### Endpoints

```http
POST /v1/webhook_endpoints
Body: {
  url,
  events: ["payment_intent.succeeded", "charge.refunded"],
  secret?
}

GET /v1/webhook_endpoints

GET /v1/webhook_endpoints/:id

POST /v1/webhook_endpoints/:id/test

DELETE /v1/webhook_endpoints/:id
```

#### Features

- **Multiple endpoints per merchant**
- **Event filtering** → less noise for merchants
- **Secret rotation** → security best practices
- **Retry policies + DLQ** → handle endpoint downtime

#### Developer Experience

- **Dashboard UI** for quick setup
- **Test webhook** with example payload
- **Clear error messages** for misconfigured URLs (timeouts, 500s)

#### Versioning

- Webhooks tied to **Stripe API version** of account
- **Breaking changes** → require opt-in per endpoint

---

## 4. Debugging / Technical Deep Dive (30 min)

### Scenario

A merchant complains: **"Refund requests sometimes succeed, sometimes fail with duplicate refund error."**

#### Logs

```http
POST /v1/refunds
Idempotency-Key: refund-123
Response: 200 OK {
  refund_id: r1,
  status: "succeeded"
}

POST /v1/refunds
Idempotency-Key: refund-123
Body differs:
  { charge_id: c1, amount: 2000 } vs. { charge_id: c1, amount: 1500 }
Response: 409 Conflict "duplicate refund"
```

### Answer

#### Root Cause
- **Same idempotency key** used with **different payloads**
- Stripe guarantees idempotency only if request body is identical
- Different amount = conflict

#### Debugging Steps

1. **Compare stored payload hash** vs. new request
2. **Look at merchant integration code** → are they reusing a static key?
3. **Check client retries** → is the library setting keys properly?

#### Fix

1. **Educate merchant**:
   - Idempotency key must be **unique per refund attempt**
   - But **consistent across retries** of that same attempt

2. **Add middleware**:
   - Hash request body on first use
   - Reject mismatched reuse

3. **Improve error docs**:
   - Clarify "duplicate refund" vs. "conflict on idempotency"

---

## 5. Behavioral / Stripe Values (30 min)

### Q1: Tell me about a time you simplified a complex system for end users

#### Answer (STAR Format)

**Situation**: Our EKS-based pipeline produced opaque errors → users confused

**Task**: Improve developer experience

**Action**: Built error mapper → mapped low-level infra errors to clear categories (auth, quota, network)

**Result**: Ticket volume dropped 30%, dev onboarding much smoother. Shows **"Users First."**

### Q2: Describe a time you identified a scaling bottleneck before it became a production issue

#### Answer

**Situation**: High API traffic (200/s) → Postgres read spikes

**Task**: Anticipate failure before peak season

**Action**: Ran load tests, spotted missing index on `(account_id, created_at)`. Added composite index + read replicas

**Result**: Sustained 10x load in production without incident. Shows **"High Standards & Rigor."**

### Q3: Why Stripe?

#### Answer

"I'm excited by Stripe's mission of **growing the GDP of the internet**. My experience building scalable APIs, idempotent systems, and event-driven pipelines aligns directly with Stripe's challenges — **correctness and developer experience**. Stripe's culture of **rigor and empathy** is exactly what I want to contribute to."

---

*This document contains comprehensive mock interview questions and answers designed to prepare for Stripe's technical interview process. Each section focuses on different aspects of the role while demonstrating technical depth and alignment with Stripe's values.*
