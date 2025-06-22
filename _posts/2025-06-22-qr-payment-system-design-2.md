---
layout: post
title: "System Design: Scale the QR Code Payment System using the Saga Pattern"
date: 2025-06-22
tags: system-design
categories: system-design
---

# System Design: Scale the QR Code Payment System using the Saga Pattern

Scale the QR Code Payment System using the Saga Pattern, which is ideal when:

- You want to loosen coupling between services
- Avoid long-lived DB locks
- Support asynchronous or event-driven workflows
- Prepare for horizontal scaling and resilience

## 🧱 Objective:

Use the Saga Pattern to orchestrate a distributed payment flow that’s:

- Eventually consistent
- Fault-tolerant
- Scalable
- Still safe for financial operations

## 🎯 When to Use Sagas

- Services span multiple domains (wallet, PSP, fraud, ledger, notifications)
- DB transactions can’t span services or modules
- Need graceful failure handling and compensation logic
- Want non-blocking workflows (no waiting for external services in a DB txn)

## 🧩 High-Level Components

```
+------------------+
|   API Gateway    |
+--------+---------+
         |
         v
+--------------------------+
|   Payment Service        | ← HTTP layer + idempotency check
|   (IdempotencyKey)       |
+-----------+--------------+
            |
            v   emits event: PaymentRequested
+--------------------------+
| Transaction Service      | ← owns saga state & orchestrates steps
| (Saga + Recovery logic)  |
+-----------+--------------+
            |
            | emits commands (via Event Queue):
            |
            ├── DeductWalletCommand ─────────────▶ Wallet Service
            ├── WriteLedgerCommand ──────────────▶ Ledger Service
            └── SendNotificationCommand ─────────▶ Notification Service
                      ↑                                ↑
                      | emits events                   | emits events
              WalletDebited / WalletFailed    NotificationSent / Failed
                      ↑                                ↑
                      └────────── receives  LedgerWritten event
                                   from Ledger Service
```

### 📦 Components and Responsibilities

| Component                | Responsibility                                                                                               |
| ------------------------ | ------------------------------------------------------------------------------------------------------------ |
| **API Gateway**          | Accepts requests, forwards to Payment Service                                                                |
| **Payment Service**      | Handles `requestId`, writes `IdempotencyKey`, emits `PaymentRequested`                                       |
| **Transaction Service**  | Orchestrates the Saga:<br>Tracks state (`PENDING → COMPLETED/FAILED`), emits commands, compensates if needed |
| **Wallet Service**       | Listens to `DeductWalletCommand` / `RefundWalletCommand`, updates balance, emits result events               |
| **Ledger Service**       | Listens to `WriteLedgerCommand`, writes double ledger records, emits confirmation                            |
| **Notification Service** | Sends payment result to users, listens to `SendNotificationCommand`                                          |
| **Event Queue**          | Asynchronous, reliable event delivery (Kafka, SQS, Pub/Sub, etc.)                                            |

### 🧠 Event Topics (example with Kafka-style)

| Topic Name                | Publisher           | Subscribers                   |
| ------------------------- | ------------------- | ----------------------------- |
| `PaymentRequested`        | PaymentService      | TransactionService            |
| `DeductWalletCommand`     | TransactionService  | WalletService                 |
| `WalletDebited`           | WalletService       | TransactionService            |
| `WalletFailed`            | WalletService       | TransactionService            |
| `WriteLedgerCommand`      | TransactionService  | LedgerService                 |
| `LedgerWritten`           | LedgerService       | TransactionService            |
| `SendNotificationCommand` | TransactionService  | NotificationService           |
| `NotificationSent`        | NotificationService | TransactionService (optional) |

### ✅ Benefits of This Design

- Scalable: services run and scale independently
- Fault tolerant: one component failure won’t cascade
- Retryable: use idempotency keys and DLQs to recover
- Auditable: each step is traceable via ledger and event logs
- Resilient: transaction flow can resume after partial failure

## ✅ Happy Path Flow

Step 1: Initiate Payment (HTTP)

- Client → API Gateway → Payment Service
- PaymentService:
  - Checks or inserts IdempotencyKey[requestId] = PENDING
  - Emits PaymentRequested event

Step 2: Start Saga

- Transaction Service receives PaymentRequested
  - Creates Transaction[txnId] with status = PENDING
  - Emits DeductWalletCommand

Step 3: Deduct Wallet

- Wallet Service receives DeductWalletCommand
- Validates balance and locks wallet
  - Deducts amount in a local DB transaction
  - Emits WalletDebited event

Step 4: Write Ledger

- Transaction Service receives WalletDebited
- Emits WriteLedgerCommand

  - Ledger Service:
    - Appends DEBIT (payer) and CREDIT (payee) entries
    - Emits LedgerWritten event

Step 5: Finalize Transaction

- Transaction Service receives LedgerWritten
- Updates Transaction.status = SUCCESS
- Emits SendNotificationCommand

Step 6: Notify Users

- Notification Service receives SendNotificationCommand
- Sends push/email/in-app notification to payer and payee
- Optionally emits NotificationSent

 Step 7: Respond to Client

- (Optional depending on sync or async design)
- Payment Service listens for success event or polls TransactionService
- Returns 200 OK with txnId, status = SUCCESS

### 🔐 Final States After Success

| Table / Service    | State / Record                                      |
| ------------------ | --------------------------------------------------- |
| `idempotency_keys` | requestId → `SUCCESS` + cached response             |
| `transactions`     | txnId → `SUCCESS`, with payer/payee/amount metadata |
| `wallets`          | updated balances                                    |
| `ledger_entries`   | debit & credit rows                                 |
| `notifications`    | message sent/logged                                 |

### 🔄 Compare: Sync vs. Async Design

| Aspect                  | **Sync API**                               | **Async API**                              |
| ----------------------- | ------------------------------------------ | ------------------------------------------ |
| **Client experience**   | Waits for result (success/failure)         | Gets accepted immediately, polls/gets push |
| **PaymentService role** | Needs to *wait* for events or poll status  | Just emits event and returns immediately   |
| **Response status**     | `200 OK`, `400 Bad Request`, etc.          | `202 Accepted` with `requestId/txnId`      |
| **Coupling**            | Slightly more coupled (waits on result)    | Fully decoupled                            |
| **Latency**             | Higher (depends on saga duration)          | Low latency                                |
| **Use cases**           | POS, e-commerce checkout (UX critical)     | Top-ups, batch jobs, QR payments           |
| **Retries/Timeouts**    | Must handle waiting/timeout inside service | External retry logic possible              |

#### Sync Examples

> "The user scans QR and expects to see 'Payment Success' right away on screen."

- You’ll wait for TransactionService to finish processing
- Either listen to success events or poll txn.status
- Risk: delay/timeout if downstream is slow

#### Async Examples

> "We show a spinner and notify user when payment finishes (via push or polling)."

- Just return 202 Accepted with requestId
- All processing happens behind the scenes
- Easier to scale and monitor

#### Final Guiding Principle

- The level of coupling and response behavior should be determined by the UX/business requirement, not technical preference.
- If real-time confirmation is required → sync
- If eventual confirmation is acceptable → async

## 🔁 Retry Strategy Overview

| Retry Layer            | Purpose                                        | Example                                    |
| ---------------------- | ---------------------------------------------- | ------------------------------------------ |
| **Producer Retry**     | If publish to Kafka/SQS fails                  | Retry publishing `DeductWalletCommand`     |
| **Consumer Retry**     | If handler fails to process the message        | Retry consuming `WalletDebited`            |
| **Business Retry**     | If downstream dependency fails (e.g., DB down) | Retry debit wallet DB update               |
| **Orchestrator Retry** | Retry full saga step after timeout/failure     | Re-publish command from TransactionService |

### 🧠 Common Retry Tactics

1. Immediate Retry
  - Useful for transient errors (e.g., race conditions, network blips)
  - Example: Retry DB write 3 times with 100ms delay

2. Exponential Backoff

  - Waits longer between retries (e.g., 1s → 2s → 4s...)
  - Helps reduce load on recovering services

3. Circuit Breaker

  - Temporarily stops retrying after too many failures
  - Prevents cascading failures

## 💀 Dead Letter Queues (DLQ)

> A DLQ stores messages that fail permanently after all retries are exhausted

| Component           | When to DLQ                                               |
| ------------------- | --------------------------------------------------------- |
| WalletService       | DeductWalletCommand fails 5 times                         |
| LedgerService       | WriteLedgerCommand consistently fails (e.g., invalid txn) |
| NotificationService | Email/push provider unreachable or malformed msg          |
| TransactionService  | Cannot progress saga due to missing events                |

### What Happens in DLQ?

- The message is stored along with:
    - Error reason
    - Retry count
    - Timestamps
- Triggers an alert / monitoring dashboard
- May trigger manual review or automated retry batch job

### 📦 Sample DLQ Architecture

```
+----------------------+
| Event Queue (Kafka)  |
+----------+-----------+
           |
     +-----v----------------------+
     | WalletService              |
     |  - Retry handler           |
     |  - OnFail → DLQ Producer   |
     +-----+----------------------+
           |
           v
     +--------------------+
     | Wallet.DLQ Topic   | <-- Store JSONs of failed messages
     +--------------------+

     (repeat for ledger, notify, etc.)
```

### 🔐 Idempotency + Retry

Because all retries must be safe to repeat, each service must:

- Use idempotent keys
- Only update state if not already done
- Ensure side-effects (wallet, ledger) aren't repeated

### 🔭 Monitoring & Alerting

| What to Monitor                  | Tool/Action                          |
| -------------------------------- | ------------------------------------ |
| DLQ size growing                 | Alert (PagerDuty, Slack)             |
| Message processing latency       | Metrics dashboard (e.g., Prometheus) |
| Retry count per message          | Metrics + DLQ tagging                |
| Saga stuck in `PENDING` too long | Auto-recovery or ops investigation   |

### Summary

| Mechanism       | Why It Matters in Fintech                           |
| --------------- | --------------------------------------------------- |
| **Retries**     | Handle transient failures gracefully                |
| **DLQ**         | Prevent infinite retry loops, ensure recoverability |
| **Idempotency** | Ensures retry won't corrupt state                   |
| **Monitoring**  | Enables ops teams to intervene fast                 |

## 🔄 Compensation Logic

In a saga-based system, if one step in the payment process fails irrecoverably, the system cannot roll back using ACID transactions. Instead, it uses compensation actions to reverse the effects of the previous steps.

### Examples of Compensation:

| Failed Step            | Compensation Action                        |
| ---------------------- | ------------------------------------------ |
| Ledger writing fails   | Emit `RefundWalletCommand` to return funds |
| Notification fails     | Retry only; no compensation needed         |
| Wallet deduction fails | Mark the transaction as `FAILED`           |

All compensation commands are:

- Emitted by the TransactionService based on saga state
- Handled asynchronously
- Must be idempotent and retry-safe

## 🔄 Transaction State Machine

```
   +------------------+
   |     PENDING      |
   +--------+---------+
            |
     WalletDebited
            |
            v
   +------------------+
   |   WALLET_OK      |
   +--------+---------+
            |
     LedgerWritten
            |
            v
   +------------------+
   |   SUCCESS        |
   +------------------+

            |
     WalletFailed / LedgerFailed
            v
   +------------------+
   |     FAILED       |
   +------------------+
```

## 🧯 Recovery from Partial Failures

Even with retries and DLQs, services or brokers may crash. To recover from these cases, we ensure:

### 🛠 Durable State Tracking

- The TransactionService maintains a durable record of:

  - Saga steps completed
  - Emitted commands
  - Response events received

This allows resumption after failure or restart.

### ♻️ Automatic Saga Resumption

On service restart:

- Scan transactions where status = PENDING or WALLET_OK
- Determine the next missing step
- Re-emit the corresponding command (e.g., WriteLedgerCommand)

### ✅ Idempotency Ensures Safe Retries

- Every command includes a txnId or requestId
- Handlers must ignore duplicate commands/events that have already been processed

> Recovery logic is part of the TransactionService and is often run as a periodic background job or as part of service startup.