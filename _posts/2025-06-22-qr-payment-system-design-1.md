---
layout: post
title: "System Design: QR Code Payment System (Synchronous Version)"
date: 2025-06-22
tags: system-design
categories: system-design
---

## 🎯 Key Design Goals for a Financial System

In any financial system — especially one handling wallet balances and real-time payments — we must ensure:
1. Idempotency

    Every client request must be processed **once and only once**.

    Retried requests (due to network issues or timeouts) **must not result in double charges**.

2. Recoverability

    The system **must be resilient to partial failures** (e.g., crash before commit).

    State must be **reconstructable from durable records** like transactions and ledgers.

3. Robustness

    No inconsistent state even under high concurrency.

    Must guard against race conditions, double-spends, phantom reads, etc.

## 🧱 System Components

```
[Client]
   ↓
[API Gateway]
   ↓
[Payment Service]
   ├──→ Idempotency Store
   ├──→ Transaction Store
   ├──→ Ledger Store
   ├──→ Wallet Store (row-level locks)
   └──→ Notification Queue (optional)
```

## 💽 Why Choose an RDB for QR Code Payment Systems?

### ✅ 1. Strong ACID Guarantees

| Property        | Why It Matters in Finance                                                              |
| --------------- | -------------------------------------------------------------------------------------- |
| **Atomicity**   | Ensures either all updates (wallet, ledger, tx) happen, or none — no partial writes.   |
| **Consistency** | Enforces rules (e.g., non-negative balance, foreign keys).                             |
| **Isolation**   | Prevents race conditions under concurrent transactions (important for wallet updates). |
| **Durability**  | Once committed, all writes are safe — even after power failure or crash.               |

🧠 In a payment system, you can’t afford half-complete operations — ACID is non-negotiable.

### ✅ 2. Transactional Integrity for Money Movement

You need to update:

- Wallet balances
- Transaction statuses
- Ledger entries
- Idempotency records

→ all in one coordinated transaction.

🛠️ RDBs allow this out-of-the-box using BEGIN ... COMMIT.

### ✅ 3. Relational Modeling Fits the Domain

You have:
- users, wallets, transactions, ledger, idempotency_keys, qr_codes, payment_methods

These are highly interrelated, and RDBs let you model and enforce:

- Foreign key constraints
- Unique keys
- Indexes
- Joins (for queries, audits, reports)

🧩 Perfect match for structured, interdependent financial data.

### ✅ 4. Mature Tooling for Auditing, Backups, Replication

- Point-in-time recovery
- Read replicas
- WAL-based backups
- SQL-based integrity checks
- Mature monitoring tools (e.g., pg_stat, MySQL slow query log)

📊 These are essential for financial observability and recovery.

### ✅ 5. Battle-tested at Scale

- Companies like Stripe, PayPal, Square, and Mercari itself use RDBs (e.g., PostgreSQL or MySQL) for core financial systems.

- Proven at millions of transactions per day, when properly partitioned and scaled.

🔁 Alternatives — and Why RDB Still Wins

| Option       | Why It's Risky Alone                             |
| ------------ | ------------------------------------------------ |
| NoSQL        | Weak consistency; can't do ACID across documents |
| KV Stores    | No schema, no joins, harder for audits           |
| Event Store  | Great for logging, but complex for balance logic |
| In-memory DB | Not durable enough for financial systems         |

### 🔒 Bottom Line

For a core payment ledger, an RDB gives you:

✅ ACID
✅ Reliability
✅ Auditability
✅ Schema enforcement
✅ Recovery options
✅ Fast joins & transactional updates

🔧 It's still the most trustable, explainable, and recoverable option for moving money safely.

## ☀️ Happy Path (Synchronous Flow)

### 🎬 1. Client sends request

```
POST /payments
{
  "payerId": "u123",
  "payeeId": "m456",
  "amount": 100,
  "requestId": "REQ-abc123"
}
```

### 🔁 2. Insert Idempotency Key and Transaction

```
SELECT * FROM idempotency_keys WHERE request_id = 'REQ-abc123';
```

- If exists → return stored result.
- If not → insert:
```
BEGIN;

-- Step 2a: Insert idempotency key (must be first for deduplication)
INSERT INTO idempotency_keys (request_id, status)
VALUES ('REQ-abc123', 'PENDING');

-- Step 2b: Insert transaction record
INSERT INTO transactions (txn_id, payer, payee, amount, status)
VALUES ('txn-001', 'u123', 'm456', 100, 'PENDING');
```

#### 💡 Why Insert Both Early?

| Reason                        | Benefit                                                                 |
| ----------------------------- | ----------------------------------------------------------------------- |
| **Crash safety**              | Partial progress is visible; no hidden side effects                     |
| **Deduplication**             | Prevents duplicate request from executing again                         |
| **Debugging & observability** | Operators can observe in-flight or failed transactions                  |
| **Retry support**             | Allows clean recovery: continue where it left off or return cached data |
| **Auditability**              | You have a durable footprint of what was *about to happen*              |

### 🔐 3. Begin transaction

All the following actions are wrapped in a single ACID DB transaction.


### 💳 4. Insert initial transaction record

```
INSERT INTO transactions (txn_id, payer, payee, amount, status)
VALUES ('txn-001', 'u123', 'm456', 100, 'PENDING');
```

### 🔒 5. Lock payer wallet row

```
SELECT balance FROM wallets WHERE user_id = 'u123' FOR UPDATE;
```

### ➖ 6. Check and deduct balance

```
UPDATE wallets SET balance = balance - 100 WHERE user_id = 'u123';
```

### 📜 7. Insert ledger records

```
INSERT INTO ledger (...) VALUES
  ('u123', 100, 'txn-001', 'DEBIT'),
  ('m456', 100, 'txn-001', 'CREDIT');
```

### ✅ 8. Mark transaction as SUCCESS

```
UPDATE transactions SET status = 'SUCCESS' WHERE txn_id = 'txn-001';
```

### 🗂️ 9. Update idempotency record

```
UPDATE idempotency_keys
SET status = 'SUCCESS', response_json = '{...}'
WHERE request_id = 'REQ-abc123';
```

### 📦 10. Commit transaction

After commit, response to client the transaction is success

### 📣 11. Optional: push notifications to payer/payee

## 🛠️ Unhappy Path - Example Scenario: Failure During Wallet Deduction

### Suppose this flow:

1. Request comes in → requestId = REQ-abc123

2. Inserted into:

    2.1 idempotency_keys (PENDING)

    2.2 transactions (PENDING)

3. Locked payer wallet

💥 App crashes before updating balance or inserting ledger

### 🧰 How System Handles It

| Concern                     | Mitigation                                                                                                                 |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **No partial side effects** | Since all actions were inside a single DB transaction and it was never committed, **everything rolls back automatically**. |
| **Client retry**            | The next request will hit `idempotency_keys` → status = `PENDING`.<br>You can return "in-progress" or retry the full flow. |
| **No double spend**         | Since nothing was committed, no wallet deduction or ledger entry happened.                                                 |
| **Safe to resume**          | The retry can safely **reuse the existing transaction record**, continue from known safe point, and commit the full flow.  |

### 💾 Recovery Principles

#### Use Idempotency Key as the Anchor

Any retry uses the same requestId, which either:

- Returns existing response (if SUCCESS)
- Detects PENDING and waits or resumes

#### Store durable intent early

Ensures that either:

- Wallet + ledger + transaction + idempotency key are all updated
- Or none of them are

#### Use atomic DB transactions

- Insert transactions and idempotency_keys first.
- These act like a crash-safe bookmark for recovery tools or retries.

### Summary

| Failure Point       | Outcome                       | Why It's Safe               |
| ------------------- | ----------------------------- | --------------------------- |
| Before transaction  | Nothing persisted             | No action needed            |
| During transaction  | Rolled back on crash          | ACID rollback               |
| After commit        | Success is visible to retries | No side effects repeated    |
| Retry after failure | Detected via idempotency key  | Ensures exactly-once effect |

## ✅ How to Use the Ledger for Auditing

1. Transaction Verification
- Every txn_id must result in a net-zero effect.
- For every DEBIT entry, there should be a corresponding CREDIT entry of the same amount.
- You can run integrity checks like:

```
SELECT txn_id, SUM(CASE type WHEN 'DEBIT' THEN -amount ELSE amount END) as net
FROM ledger
GROUP BY txn_id
HAVING net != 0;
```

2. User History / Statement Generation

- The ledger is an append-only journal.
- You can reconstruct a user’s balance by replaying entries over time.
```
SELECT * FROM ledger WHERE user_id = 'u123' ORDER BY created_at;
```

3. Balance Verification (Snapshot Check)
- Each ledger entry can store a balance_snapshot at the moment of mutation.
- Periodically, you can validate:
```
SELECT MAX(balance_snapshot) FROM ledger WHERE user_id = 'u123';
```
vs.
```
SELECT balance FROM wallets WHERE user_id = 'u123';
```

4. Audit Trail
- Who was paid, how much, and when — all verifiable.
- Required for compliance, fraud investigations, customer disputes.

5. Immutable Record
- Unlike wallets (which store current state), ledger is immutable and append-only.
- It’s your source of truth if you ever need to rebuild balances after a crash or inconsistency.

🛡️ In summary:

| Use Case               | Ledger Role                                     |
| ---------------------- | ----------------------------------------------- |
| Recover from crash     | Replay entries to rebuild balances              |
| Detect inconsistencies | Compare ledger-derived balance vs. wallet table |
| Prove transactions     | Immutable record of all changes                 |
| Generate statements    | Use ordered ledger events                       |
| Ensure correctness     | Check that DEBIT + CREDIT are always paired     |

## ✅ How This Design Satisfies the 3 Key Requirements

| Requirement        | How It's Handled                                                                  |
| ------------------ | --------------------------------------------------------------------------------- |
| **Idempotency**    | `idempotency_keys` table with unique `requestId` ensures safe retries.            |
| **Recoverability** | All state changes are durable and atomic. `PENDING` status helps detect failures. |
| **Robustness**     | Pessimistic locking (`FOR UPDATE`) + ACID transactions avoid race conditions.     |


## 📊 Observability & Monitoring

| Signal Type    | Examples                                                                         |
| -------------- | -------------------------------------------------------------------------------- |
| **Logs**       | Request/response traces, txn state transitions                                   |
| **Metrics**    | `wallet_lock_timeout_total`, `payments_success_total`, `duplicate_request_total` |
| **Traces**     | End-to-end spans from API → Wallet → Ledger → Commit                             |
| **Dashboards** | Wallet latency, txn throughput, balance errors                                   |
| **Alerts**     | Long lock waits, transaction failures, inconsistent ledger balance               |

## ⚠️ Bottlenecks and Limitations

| Bottleneck                   | Description                                                                       |
| ---------------------------- | --------------------------------------------------------------------------------- |
| **Pessimistic locking**      | `SELECT ... FOR UPDATE` serializes access — can lead to contention on hot wallets |
| **Single DB transaction**    | All logic in one DB → risks long transactions & lock contention under load        |
| **Idempotency table growth** | Needs periodic pruning or TTL management                                          |
| **Retry behavior**           | Poor retry strategies can cause thundering herd effects                           |

## 🚀 Future Improvements

Use logical sharding by user_id to reduce lock contention.

Add timeouts and monitoring on long-lived locks.

Consider outbox pattern or saga pattern for decoupling ledger, notifications.

Consider event-driven PSP integration for multi-provider support.

