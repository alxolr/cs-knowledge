# Payment System Design

**Track:** Design Concepts
**Difficulty Tier:** Advanced
**Prerequisites:** [Microservices Architecture](../microservices-architecture/README.md), [Notification System Design](../notification-system-design/README.md)

## Concept Overview

Payment systems are among the most demanding distributed systems to design because they must guarantee correctness of financial transactions above all else. A single lost, duplicated, or incorrectly processed payment can erode user trust and create regulatory liability. Systems like Stripe, PayPal, and Square process millions of transactions daily, coordinating between merchants, acquiring banks, card networks, and issuing banks — all while maintaining sub-second latency for the end user.

At the core of every payment system is the concept of idempotency: ensuring that retrying a failed request never results in a double charge. Beyond idempotency, designers must reason about eventual consistency across ledgers, the two-phase nature of authorization and capture, and the complex state machines that govern a payment's lifecycle from initiation through settlement to potential refund or chargeback. Fraud detection, PCI compliance boundaries, and multi-currency support add further layers of complexity.

This module explores the design of payment systems through scenario-based problems that cover high-level architecture, idempotency and exactly-once processing, ledger design, refund and chargeback workflows, and fraud detection pipelines. Each problem exercises a different aspect of building a reliable, scalable payment platform.

---

## Problem 1 — High-Level Architecture of a Payment Processing Platform

**Difficulty:** Easy
**Estimated Time:** 35 minutes

### Problem Statement

Design the high-level architecture for an online payment processing platform that allows merchants to charge customers via credit cards. The design should identify the core services, external integrations, and the end-to-end flow for processing a single payment from the moment a customer clicks "Pay" to the moment the merchant receives confirmation.

### Scenario

**Context:** A startup is building a payment platform for small e-commerce merchants. Merchants integrate via a REST API. When a customer checks out, the merchant's server calls the payment platform to charge the customer's credit card. The platform must communicate with external card networks (Visa, Mastercard) through an acquiring bank. The system handles 500 transactions per second at peak and must respond to the merchant within 2 seconds. The team needs a clear architectural blueprint before writing any code.

**Requirements:** Identify the major internal services and their responsibilities. Show how the platform integrates with external parties (acquiring bank, card networks). Describe the end-to-end flow for a successful card payment. Explain what information is stored and where. State what happens when the acquiring bank is temporarily unreachable.

**Expected Approach:** Introduce an API Gateway that authenticates merchant requests, a Payment Service that orchestrates the transaction lifecycle, a Payment Gateway Adapter that communicates with the acquiring bank, and a Ledger Service that records all financial events. Use asynchronous notifications to inform merchants of final payment status. When the acquiring bank is unreachable, queue the request and retry with exponential backoff.

<details>
<summary>Hints</summary>

1. The payment flow typically has two phases: authorization (verifying the card has sufficient funds and placing a hold) and capture (actually moving the money). Some systems combine these into a single "charge" call for simplicity, but separating them gives merchants flexibility to authorize at checkout and capture at shipment.
2. An API Gateway handles authentication (API keys), rate limiting, and request validation before forwarding to internal services. This keeps payment logic decoupled from cross-cutting concerns.
3. The Payment Gateway Adapter is an abstraction layer over external bank APIs. It translates internal payment requests into the acquiring bank's protocol (e.g., ISO 8583) and handles retries, timeouts, and circuit breaking. This isolation means switching acquiring banks requires changing only the adapter, not the core payment logic.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Design a service-oriented architecture with clear separation between API handling, payment orchestration, external bank communication, and financial record-keeping.

1. **Core services:**
   - **API Gateway:** Authenticates merchant API keys, validates request payloads (amount, currency, card token), applies rate limiting, and routes requests to the Payment Service.
   - **Payment Service:** The orchestrator. Receives payment requests, creates a payment record with status `PENDING`, coordinates with the Payment Gateway Adapter to authorize and capture funds, updates payment status, and triggers notifications.
   - **Payment Gateway Adapter:** Translates internal payment requests into the acquiring bank's wire protocol. Manages connection pooling, timeouts (2-second deadline), retries, and circuit breaking for external calls.
   - **Ledger Service:** Append-only log of all financial events (authorization, capture, refund, chargeback). Each entry is immutable and includes a timestamp, payment ID, event type, amount, and currency. Used for reconciliation and auditing.
   - **Notification Service:** Sends asynchronous webhooks to merchants when payment status changes (success, failure, refund). Uses the notification infrastructure from the prerequisite module.

2. **External integrations:**
   - **Acquiring Bank:** The financial institution that processes card transactions on behalf of the payment platform. The adapter sends authorization and capture requests via a secure channel (TLS + mutual authentication).
   - **Card Networks (Visa, Mastercard):** The acquiring bank communicates with card networks, which route the transaction to the customer's issuing bank. The payment platform does not interact with card networks directly.

3. **End-to-end flow for a successful payment:**
   ```
   1. Customer clicks "Pay" → Merchant server calls POST /v1/payments with amount, currency, and card token.
   2. API Gateway authenticates the merchant, validates the request, and forwards to Payment Service.
   3. Payment Service creates a payment record (status: PENDING) in the database and generates a unique payment_id.
   4. Payment Service calls Payment Gateway Adapter with the payment details.
   5. Adapter sends an authorization request to the acquiring bank.
   6. Acquiring bank routes through the card network to the issuing bank, which approves and places a hold on the customer's funds.
   7. Acquiring bank returns an authorization code to the adapter.
   8. Adapter returns success to Payment Service.
   9. Payment Service updates the payment record (status: AUTHORIZED) and sends a capture request to the adapter.
   10. Adapter sends a capture request to the acquiring bank, which confirms the fund transfer.
   11. Payment Service updates the payment record (status: CAPTURED) and writes a ledger entry.
   12. Payment Service returns the payment_id and status to the merchant via the API Gateway.
   13. Notification Service sends a webhook to the merchant confirming the successful payment.
   ```

4. **Data storage:**
   - **Payment records:** Stored in a relational database (PostgreSQL) with columns for payment_id, merchant_id, amount, currency, status, card_token_hash, created_at, updated_at. Indexed by payment_id and merchant_id.
   - **Ledger entries:** Append-only table with columns for entry_id, payment_id, event_type, amount, currency, timestamp. Never updated or deleted.
   - **Card tokens:** Raw card numbers are never stored. A tokenization service (or third-party vault) replaces card numbers with opaque tokens before they reach the Payment Service, maintaining PCI compliance.

5. **Handling acquiring bank unavailability:**
   - If the adapter receives a timeout or connection error from the acquiring bank, it retries up to 3 times with exponential backoff (1s, 2s, 4s).
   - If all retries fail, the Payment Service updates the payment record to status `FAILED_TEMPORARY` and enqueues the payment for background retry (up to 1 hour).
   - The merchant receives an immediate response indicating the payment is pending, and a webhook is sent when the final status is determined.
   - A circuit breaker in the adapter opens after 10 consecutive failures, rejecting new requests for 30 seconds to avoid overwhelming the recovering bank.

</details>

---

## Problem 2 — Idempotency and Exactly-Once Payment Processing

**Difficulty:** Medium
**Estimated Time:** 45 minutes

### Problem Statement

Design an idempotency mechanism for a payment processing platform that guarantees every payment is processed exactly once, even when clients retry requests due to network timeouts, server crashes, or load balancer failovers. The mechanism must handle concurrent duplicate requests, work across multiple stateless API server instances, and not degrade throughput under normal (non-retry) traffic.

### Scenario

**Context:** The payment platform from Problem 1 processes 500 transactions per second across 10 stateless API server instances behind a load balancer. Merchants occasionally experience network timeouts and retry their payment requests. Without idempotency protection, a retry can result in a double charge — the customer is billed twice for the same purchase. Last month, 0.3% of transactions were duplicates, resulting in costly manual refunds and merchant complaints. The team needs a robust idempotency layer that prevents double charges without adding significant latency to normal requests.

**Requirements:** Design an idempotency key scheme that merchants include with each payment request. Explain how the system detects and handles duplicate requests. Address the race condition where two identical requests arrive simultaneously at different API servers. Define the storage and lifecycle of idempotency records (how long they are kept, when they are purged). Analyze the impact on latency and throughput for both first-time and duplicate requests.

**Expected Approach:** Require merchants to include a unique `Idempotency-Key` header with each payment request. Store idempotency records in a shared database with the key as a unique constraint. On each request, attempt to insert the idempotency record — if it already exists, return the stored response. Use database-level uniqueness constraints to handle concurrent duplicates. Set a TTL on idempotency records (e.g., 24 hours) and purge expired records in a background job.

<details>
<summary>Hints</summary>

1. The idempotency key should be generated by the merchant (client-side), not the server. This ensures that retries of the same logical payment carry the same key. A common choice is a UUID v4 generated at the point of sale, or a deterministic key derived from the order ID (e.g., `order_12345_payment`).
2. Use a database unique constraint on the `(merchant_id, idempotency_key)` pair. When two concurrent requests with the same key hit different API servers, both attempt to INSERT the idempotency record. The database guarantees that exactly one INSERT succeeds; the other receives a unique constraint violation and knows it is a duplicate.
3. The idempotency record should store the full response (status code, body) of the original request. When a duplicate is detected, the system returns the stored response without re-processing the payment. This means the merchant sees the same response regardless of how many times they retry.
4. Be careful about the window between inserting the idempotency record and completing the payment. If the server crashes after inserting the record but before processing the payment, the record exists but has no response. Subsequent retries must detect this "in-progress" state and either wait or re-process.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a database-backed idempotency layer with unique constraints, in-progress state handling, and TTL-based cleanup.

1. **Idempotency key scheme:**
   - Merchants include an `Idempotency-Key` header with each POST request to `/v1/payments`. The key is a string up to 64 characters, typically a UUID or a deterministic identifier like `order_{order_id}_payment`.
   - The key is scoped to the merchant: `(merchant_id, idempotency_key)` is the unique identifier. Different merchants can use the same key without collision.
   - If the header is missing, the API returns a 400 error requiring the key. This forces merchants to adopt idempotent practices.

2. **Idempotency record schema:**
   ```sql
   CREATE TABLE idempotency_records (
       id BIGSERIAL PRIMARY KEY,
       merchant_id BIGINT NOT NULL,
       idempotency_key VARCHAR(64) NOT NULL,
       status VARCHAR(20) NOT NULL,  -- 'IN_PROGRESS', 'COMPLETED', 'FAILED'
       request_hash VARCHAR(64),      -- SHA-256 of request body (to detect mismatched retries)
       response_status_code INT,
       response_body JSONB,
       created_at TIMESTAMP NOT NULL DEFAULT NOW(),
       completed_at TIMESTAMP,
       UNIQUE (merchant_id, idempotency_key)
   );
   CREATE INDEX idx_idempotency_created ON idempotency_records (created_at);
   ```

3. **Request processing flow:**
   ```
   1. API server receives POST /v1/payments with Idempotency-Key header.
   2. Compute request_hash = SHA-256(request_body).
   3. Attempt to INSERT into idempotency_records:
      INSERT INTO idempotency_records (merchant_id, idempotency_key, status, request_hash, created_at)
      VALUES (?, ?, 'IN_PROGRESS', ?, NOW())
      ON CONFLICT (merchant_id, idempotency_key) DO NOTHING
      RETURNING id;
   4. If INSERT succeeds (new record created):
      a. Process the payment (call Payment Service, etc.).
      b. UPDATE idempotency_records SET status = 'COMPLETED', response_status_code = ?, response_body = ?, completed_at = NOW() WHERE id = ?
      c. Return the response to the merchant.
   5. If INSERT fails (record already exists):
      a. SELECT status, request_hash, response_status_code, response_body FROM idempotency_records WHERE merchant_id = ? AND idempotency_key = ?
      b. If status = 'COMPLETED': return the stored response (idempotent replay).
      c. If status = 'IN_PROGRESS': the original request is still processing. Return 409 Conflict with a "Retry-After: 5" header, asking the merchant to retry in 5 seconds.
      d. If request_hash differs from the stored hash: return 422 Unprocessable Entity — the merchant is reusing an idempotency key with different request parameters, which is an error.
   ```

4. **Handling server crashes (in-progress recovery):**
   - If the server crashes after inserting the `IN_PROGRESS` record but before completing the payment, the record is left in `IN_PROGRESS` state indefinitely.
   - A background reaper job runs every 5 minutes and finds `IN_PROGRESS` records older than 10 minutes. It marks them as `FAILED` and sets a response indicating the payment should be retried.
   - When the merchant retries, the system sees the `FAILED` status, deletes the old record, and processes the request as new.
   - **Alternative:** Use a row-level lock with a timeout. The second request waits up to 10 seconds for the first to complete. If the lock times out, it assumes the first request failed and takes over.

5. **Concurrent duplicate handling:**
   - Two identical requests arrive at different API servers within milliseconds. Both attempt the INSERT. The database unique constraint ensures exactly one succeeds.
   - The losing request reads the existing record and follows the duplicate handling logic (step 5 above).
   - No application-level distributed locking is needed — the database provides the serialization point.

6. **TTL and cleanup:**
   - Idempotency records are retained for 24 hours. After 24 hours, the merchant can no longer replay the same key (they must generate a new one).
   - A background job runs hourly: `DELETE FROM idempotency_records WHERE created_at < NOW() - INTERVAL '24 hours'`. Deletes are batched (1,000 rows per transaction) to avoid long-running transactions.
   - The 24-hour window is a balance: long enough for merchants to retry after extended outages, short enough to keep the table small.

7. **Performance impact:**
   - **First-time requests:** One additional INSERT (typically <1 ms with an index) and one UPDATE after processing. Total overhead: ~2 ms.
   - **Duplicate requests:** One failed INSERT + one SELECT. No payment processing occurs. Total time: ~1 ms.
   - **Table size:** At 500 TPS, 24-hour retention produces ~43 million rows. With the unique index, lookups remain fast. Partitioning by `created_at` (daily partitions) makes cleanup efficient via partition drops.

</details>

---

## Problem 3 — Double-Entry Ledger Design for Financial Accuracy

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design a double-entry accounting ledger for a payment platform that tracks all money movement with guaranteed accuracy. The ledger must ensure that every transaction is balanced (debits equal credits), support multiple currencies, enable real-time balance queries for merchants, and provide an audit trail for regulatory compliance. The design must handle high write throughput while maintaining strict consistency guarantees.

### Scenario

**Context:** The payment platform holds funds on behalf of merchants between the time a customer pays and the time the merchant withdraws. Regulators require the platform to demonstrate that it can account for every dollar at any point in time. The current system uses a simple `balance` column on the merchant table, updated with `UPDATE merchants SET balance = balance + amount`. This approach has led to discrepancies: the sum of merchant balances does not match the sum of incoming payments minus outgoing payouts. Auditors have flagged this as a critical risk. The team needs a proper double-entry ledger that makes discrepancies mathematically impossible.

**Requirements:** Explain the double-entry accounting principle and why it prevents discrepancies. Design the ledger schema with accounts, journal entries, and line items. Show how a customer payment, a merchant payout, and a refund are each recorded as balanced journal entries. Design a mechanism for real-time merchant balance queries without scanning the entire ledger. Address how the system handles multi-currency transactions (e.g., a customer pays in EUR, the merchant receives USD). Explain how the ledger supports auditing and reconciliation.

**Expected Approach:** Implement a double-entry ledger where every transaction creates a journal entry with at least two line items that sum to zero (debits = credits). Maintain running balance caches per account, updated atomically with each journal entry. Use a separate exchange rate service for multi-currency transactions, recording both the original and converted amounts. Provide immutable, append-only storage for audit compliance.

<details>
<summary>Hints</summary>

1. In double-entry accounting, every financial event is recorded as a journal entry with multiple line items. Each line item debits or credits an account. The sum of all debits must equal the sum of all credits within each journal entry. This invariant, enforced at write time, makes it impossible for money to appear or disappear.
2. Accounts represent buckets of money: a merchant's balance, the platform's operating account, a customer's refund liability, etc. Each account has a type (asset, liability, revenue, expense) that determines whether a debit increases or decreases the balance.
3. For real-time balance queries, maintain a `current_balance` column on each account, updated atomically within the same transaction that inserts the journal entry. This avoids scanning millions of line items to compute a balance.
4. Multi-currency transactions require recording the original amount and currency, the exchange rate used, and the converted amount. The journal entry should balance in a single "base currency" (e.g., USD) for consistency, with the original currency stored for reference.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement an append-only double-entry ledger with account-level balance caching, multi-currency support via a base-currency normalization strategy, and immutable records for auditability.

1. **Double-entry principle:**
   - Every movement of money is recorded as a journal entry containing two or more line items. Each line item either debits or credits an account. Within every journal entry, the total debits must equal the total credits.
   - Example: A $100 customer payment to a merchant creates a journal entry with two line items: debit the "Customer Payments Receivable" account by $100 and credit the "Merchant X Payable" account by $100. The entry sums to zero.
   - This invariant is enforced by a database CHECK constraint or application-level validation before committing the transaction. If debits ≠ credits, the transaction is rejected.

2. **Ledger schema:**
   ```sql
   CREATE TABLE accounts (
       account_id BIGSERIAL PRIMARY KEY,
       account_name VARCHAR(255) NOT NULL,
       account_type VARCHAR(20) NOT NULL,  -- 'ASSET', 'LIABILITY', 'REVENUE', 'EXPENSE'
       currency VARCHAR(3) NOT NULL,        -- ISO 4217 code (USD, EUR, etc.)
       current_balance BIGINT NOT NULL DEFAULT 0,  -- in smallest currency unit (cents)
       created_at TIMESTAMP NOT NULL DEFAULT NOW()
   );

   CREATE TABLE journal_entries (
       entry_id BIGSERIAL PRIMARY KEY,
       payment_id BIGINT,                   -- reference to the originating payment
       entry_type VARCHAR(50) NOT NULL,      -- 'PAYMENT', 'PAYOUT', 'REFUND', 'FEE', 'FX_CONVERSION'
       description TEXT,
       created_at TIMESTAMP NOT NULL DEFAULT NOW()
   );

   CREATE TABLE line_items (
       line_item_id BIGSERIAL PRIMARY KEY,
       entry_id BIGINT NOT NULL REFERENCES journal_entries(entry_id),
       account_id BIGINT NOT NULL REFERENCES accounts(account_id),
       direction VARCHAR(6) NOT NULL,       -- 'DEBIT' or 'CREDIT'
       amount BIGINT NOT NULL,              -- positive integer in smallest currency unit
       currency VARCHAR(3) NOT NULL,
       original_amount BIGINT,              -- amount in original currency (for FX)
       original_currency VARCHAR(3),
       exchange_rate DECIMAL(18,8),
       created_at TIMESTAMP NOT NULL DEFAULT NOW()
   );
   ```
   - All amounts are stored as integers in the smallest currency unit (cents for USD, pence for GBP) to avoid floating-point errors.
   - Tables are append-only: no UPDATE or DELETE operations are permitted on journal_entries or line_items.

3. **Recording a customer payment ($100 to Merchant X):**
   ```
   Journal Entry: { entry_type: 'PAYMENT', payment_id: 12345 }
   Line Items:
     DEBIT  "Platform Funds Receivable"  $100.00  (asset increases — money coming in)
     CREDIT "Merchant X Payable"         $97.00   (liability increases — owed to merchant)
     CREDIT "Platform Revenue - Fees"    $3.00    (revenue increases — platform's 3% fee)
   Sum of debits: $100.00 = Sum of credits: $97.00 + $3.00 ✓
   ```

4. **Recording a merchant payout ($97 to Merchant X's bank):**
   ```
   Journal Entry: { entry_type: 'PAYOUT', payment_id: NULL }
   Line Items:
     DEBIT  "Merchant X Payable"         $97.00   (liability decreases — debt settled)
     CREDIT "Platform Bank Account"      $97.00   (asset decreases — money leaving)
   Sum: $97.00 = $97.00 ✓
   ```

5. **Recording a refund ($100 back to customer):**
   ```
   Journal Entry: { entry_type: 'REFUND', payment_id: 12345 }
   Line Items:
     DEBIT  "Merchant X Payable"         $97.00   (reduce merchant's balance)
     DEBIT  "Platform Revenue - Fees"    $3.00    (reverse the fee)
     CREDIT "Platform Funds Receivable"  $100.00  (money going back out)
   Sum: $100.00 = $100.00 ✓
   ```

6. **Real-time balance queries:**
   - Each account has a `current_balance` column updated atomically within the same database transaction that inserts the journal entry and line items:
     ```sql
     BEGIN;
     INSERT INTO journal_entries (...) VALUES (...) RETURNING entry_id;
     INSERT INTO line_items (...) VALUES (...);
     UPDATE accounts SET current_balance = current_balance + 9700 WHERE account_id = <merchant_account>;
     UPDATE accounts SET current_balance = current_balance + 300 WHERE account_id = <fee_account>;
     UPDATE accounts SET current_balance = current_balance + 10000 WHERE account_id = <receivable_account>;
     -- Verify sum of line items = 0 (application check before COMMIT)
     COMMIT;
     ```
   - Querying a merchant's balance is a single indexed read: `SELECT current_balance FROM accounts WHERE account_id = ?`.
   - A nightly reconciliation job verifies that `current_balance` matches the sum of all line items for each account, catching any drift.

7. **Multi-currency handling:**
   - When a customer pays €85 and the merchant receives USD, the system uses the current exchange rate (e.g., 1 EUR = 1.18 USD → €85 = $100.30).
   - The journal entry records both currencies:
     ```
     DEBIT  "Platform EUR Receivable"  €85.00  (original_amount: 8500, original_currency: EUR)
     CREDIT "Merchant X Payable (USD)" $100.30 (converted at 1.18)
     ```
   - A separate FX conversion journal entry moves the EUR to USD in the platform's accounts when the platform settles with its bank.
   - Exchange rates are fetched from an external rate service and cached for 60 seconds. The rate used is recorded on each line item for auditability.

8. **Auditing and reconciliation:**
   - The ledger is append-only — corrections are made by adding new journal entries (reversal entries), never by modifying existing records.
   - Every journal entry includes a timestamp, the originating payment_id, and the entry type, providing a complete audit trail.
   - Daily reconciliation compares the ledger totals against external bank statements and card network settlement reports. Discrepancies trigger alerts for manual investigation.
   - Regulatory reports are generated directly from the ledger by querying journal entries within a date range, grouped by entry type and account.

</details>

---

## Problem 4 — Refund and Chargeback Workflow Orchestration

**Difficulty:** Hard
**Estimated Time:** 55 minutes

### Problem Statement

Design the refund and chargeback handling subsystem for a payment platform. The system must support merchant-initiated full and partial refunds, bank-initiated chargebacks (disputes), and the complex state transitions that a payment undergoes during these processes. The design must ensure that refunds never exceed the original payment amount, chargebacks are responded to within the card network's deadline, and all state transitions are auditable and recoverable from failures.

### Scenario

**Context:** The payment platform processes 2 million transactions per day. Approximately 1% of transactions are refunded by merchants, and 0.1% result in chargebacks initiated by customers through their banks. The current refund implementation is a simple API call that reverses the original payment, but it has several problems: (1) merchants have accidentally issued multiple full refunds for the same payment, refunding more than the original amount, (2) chargebacks arrive as asynchronous notifications from the acquiring bank and are sometimes missed or processed too late, causing the platform to lose disputes by default, and (3) when a refund fails midway (e.g., the bank call times out), the payment is left in an inconsistent state — the ledger shows the refund but the money was never returned to the customer. The team needs a robust workflow engine for refunds and chargebacks.

**Requirements:** Design the state machine for a payment's lifecycle, covering all states from creation through capture, refund, and chargeback. Implement safeguards that prevent over-refunding (total refunds exceeding the original capture amount). Design the chargeback workflow including receiving the dispute, gathering evidence, submitting a response to the card network, and handling the final ruling. Address failure recovery — what happens when a refund is partially processed and the system crashes. Explain how the notification system informs merchants of chargeback events and deadlines.

**Expected Approach:** Model the payment lifecycle as an explicit state machine with well-defined transitions. Track cumulative refunded amounts per payment to enforce the refund cap. Use a saga pattern for refund execution — if the bank reversal fails, compensate by rolling back the ledger entry. Process chargebacks via a dedicated queue with SLA-based prioritization (disputes closest to their deadline are processed first). Integrate with the notification system to alert merchants immediately upon chargeback receipt.

<details>
<summary>Hints</summary>

1. A payment state machine might include states: PENDING → AUTHORIZED → CAPTURED → PARTIALLY_REFUNDED → FULLY_REFUNDED, with a parallel chargeback track: CAPTURED → DISPUTE_OPENED → DISPUTE_UNDER_REVIEW → DISPUTE_WON / DISPUTE_LOST. Each transition has preconditions (e.g., can only refund a CAPTURED payment) and side effects (e.g., create a ledger entry).
2. To prevent over-refunding, maintain a `total_refunded` column on the payment record. Before processing a refund, check that `total_refunded + refund_amount <= captured_amount`. Use a database-level constraint or a SELECT FOR UPDATE to prevent race conditions between concurrent refund requests.
3. The saga pattern for refunds: Step 1 — create a refund record and ledger entry (debit merchant, credit customer). Step 2 — call the acquiring bank to reverse the charge. If Step 2 fails, the compensating action is to reverse the ledger entry (debit customer, credit merchant) and mark the refund as failed.
4. Chargebacks have strict deadlines imposed by card networks (typically 7–21 days to respond). A priority queue ordered by deadline ensures the most urgent disputes are handled first. Missing a deadline means the platform automatically loses the dispute and must absorb the cost.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement an explicit state machine for the payment lifecycle, a saga-based refund executor with over-refund protection, and a deadline-driven chargeback processing pipeline.

1. **Payment state machine:**
   ```
   PENDING ──→ AUTHORIZED ──→ CAPTURED ──→ PARTIALLY_REFUNDED ──→ FULLY_REFUNDED
      │             │              │                │
      │             │              │                └──→ DISPUTE_OPENED ──→ ...
      │             │              │
      │             │              └──→ DISPUTE_OPENED ──→ DISPUTE_UNDER_REVIEW
      │             │                                          │
      │             └──→ AUTH_EXPIRED                   ┌──────┴──────┐
      │                                                 ▼             ▼
      └──→ FAILED                                 DISPUTE_WON   DISPUTE_LOST
   ```
   - **Transition rules:** Each state transition is guarded by preconditions:
     - `CAPTURED → PARTIALLY_REFUNDED`: refund_amount < captured_amount
     - `CAPTURED → FULLY_REFUNDED`: refund_amount == captured_amount (or cumulative refunds reach captured_amount)
     - `CAPTURED → DISPUTE_OPENED`: only triggered by an external chargeback notification
     - `DISPUTE_OPENED → DISPUTE_UNDER_REVIEW`: evidence submitted to card network
     - `DISPUTE_UNDER_REVIEW → DISPUTE_WON / DISPUTE_LOST`: card network ruling received
   - State transitions are persisted in a `payment_state_transitions` audit table: `(payment_id, from_state, to_state, triggered_by, timestamp)`.

2. **Over-refund protection:**
   ```sql
   -- Payment record includes:
   ALTER TABLE payments ADD COLUMN captured_amount BIGINT NOT NULL;
   ALTER TABLE payments ADD COLUMN total_refunded BIGINT NOT NULL DEFAULT 0;

   -- Refund processing (within a transaction):
   BEGIN;
   SELECT captured_amount, total_refunded FROM payments WHERE payment_id = ? FOR UPDATE;
   -- Application check: total_refunded + refund_amount <= captured_amount
   -- If check fails, ROLLBACK and return error "Refund exceeds captured amount"
   UPDATE payments SET total_refunded = total_refunded + refund_amount, status = ... WHERE payment_id = ?;
   INSERT INTO journal_entries (...);
   INSERT INTO line_items (...);
   COMMIT;
   ```
   - `SELECT FOR UPDATE` locks the payment row, preventing concurrent refund requests from both passing the check and causing an over-refund.
   - Partial refunds are supported: a $100 payment can be refunded $30, then $50, then $20 (three separate refunds totaling $100).

3. **Saga-based refund execution:**
   ```
   Step 1: Create refund record (status: INITIATED)
   Step 2: Create ledger journal entry (debit merchant payable, credit customer refund payable)
   Step 3: Call acquiring bank to reverse the charge
     → If SUCCESS:
         Update refund record (status: COMPLETED)
         Update payment total_refunded and status
         Notify merchant via webhook
     → If FAILURE:
         Compensating action: Create reversal ledger entry (undo Step 2)
         Update refund record (status: FAILED)
         Notify merchant of failed refund
     → If TIMEOUT:
         Mark refund as PENDING_BANK_CONFIRMATION
         Schedule a reconciliation check in 1 hour to query the bank for the refund status
   ```
   - Each step is logged in a `refund_saga_steps` table for recovery. If the system crashes between steps, a recovery worker reads the saga state and resumes from the last completed step.

4. **Chargeback processing pipeline:**
   ```
   1. RECEIVE: Acquiring bank sends a chargeback notification (via webhook or batch file).
      → Parse the notification, create a dispute record (status: OPENED, deadline: T+14 days).
      → Immediately notify the merchant via push notification and email with the dispute details and deadline.
   
   2. GATHER EVIDENCE: The merchant uploads evidence (receipts, delivery confirmation, communication logs) via the platform's dashboard.
      → Evidence is stored in object storage, linked to the dispute record.
      → If the merchant does not respond within T+10 days, send a reminder notification.
   
   3. SUBMIT RESPONSE: The platform compiles the evidence into the card network's required format and submits it to the acquiring bank.
      → Update dispute status: UNDER_REVIEW.
      → If submission fails, retry up to 3 times. If all retries fail, alert the operations team.
   
   4. RECEIVE RULING: The card network issues a ruling (typically within 30–60 days).
      → DISPUTE_WON: The chargeback is reversed. No financial impact. Notify merchant.
      → DISPUTE_LOST: The platform absorbs the chargeback amount. Create a ledger entry debiting the merchant's account. Notify merchant.
   ```

5. **Deadline-driven prioritization:**
   - Disputes are stored in a priority queue ordered by `deadline ASC`. The chargeback processing service always picks the dispute with the nearest deadline.
   - SLA tiers:
     - Red (< 3 days remaining): Immediate escalation to operations team.
     - Yellow (3–7 days remaining): High priority in the queue.
     - Green (> 7 days remaining): Normal priority.
   - A background job runs hourly to check for disputes approaching their deadline without a merchant response, triggering escalation notifications.

6. **Failure recovery for chargebacks:**
   - Each chargeback processing step is recorded in a `dispute_workflow_steps` table.
   - If the system crashes during evidence submission, the recovery worker detects the incomplete step and retries.
   - Idempotency keys are used for bank API calls to prevent duplicate submissions.

7. **Merchant notifications (integration with Notification System):**
   - Chargeback events trigger notifications through the Notification System (prerequisite module):
     - `DISPUTE_OPENED`: Immediate push notification + email with dispute details, amount, and response deadline.
     - `EVIDENCE_REMINDER`: Sent at T+7 and T+10 days if no evidence uploaded.
     - `DISPUTE_RULING`: Push notification + email with the outcome and any financial impact.
   - Notifications include deep links to the dispute dashboard where merchants can view details and upload evidence.

</details>

---

## Problem 5 — Real-Time Fraud Detection Pipeline

**Difficulty:** Hard
**Estimated Time:** 60 minutes

### Problem Statement

Design a real-time fraud detection pipeline for a payment platform that evaluates every transaction for fraud risk before it is authorized. The pipeline must make a decision (approve, reject, or flag for manual review) within 100 milliseconds, incorporate both rule-based checks and machine-learning model scores, adapt to evolving fraud patterns, and minimize false positives that block legitimate customers while catching as many fraudulent transactions as possible.

### Scenario

**Context:** The payment platform processes 1,000 transactions per second. Fraud losses have been increasing — last quarter, $2.3 million was lost to fraudulent transactions that were not detected until customers filed chargebacks weeks later. The current fraud prevention is a simple set of hardcoded rules (e.g., block transactions over $5,000, block transactions from certain countries). These rules are too coarse: they block many legitimate high-value purchases (12% false positive rate) while missing sophisticated fraud patterns like card testing attacks (many small transactions in rapid succession) and account takeover (legitimate-looking transactions from a compromised account). The team needs a layered fraud detection system that combines fast rule checks with ML-based scoring, all within the 100 ms latency budget.

**Requirements:** Design the pipeline architecture showing how a transaction flows through multiple detection stages. Define the rule engine layer and give examples of rules that catch common fraud patterns. Describe how an ML fraud scoring model is integrated, including feature extraction, model serving, and score interpretation. Design the feedback loop that uses chargeback data to retrain the model and update rules. Address how the system handles the latency constraint — what happens if the ML model is slow to respond. Explain the manual review queue for borderline transactions.

**Expected Approach:** Implement a multi-stage pipeline: Stage 1 — fast rule checks (blocklists, velocity checks) that can immediately reject obvious fraud in <10 ms. Stage 2 — ML model scoring using pre-computed features and a low-latency model serving infrastructure. Stage 3 — decision engine that combines rule results and ML score to produce a final verdict. Transactions flagged for review enter a manual review queue. A feedback loop ingests chargeback data to label transactions and periodically retrain the model.

<details>
<summary>Hints</summary>

1. The pipeline should be sequential with early termination: if Stage 1 (rules) definitively rejects a transaction (e.g., card is on a known stolen card list), there is no need to invoke the ML model. This saves latency and compute for clear-cut cases.
2. Feature extraction for the ML model should use pre-computed features stored in a low-latency cache (e.g., Redis). Features like "number of transactions from this card in the last hour" or "average transaction amount for this merchant" are computed incrementally as transactions arrive, not calculated on-the-fly during scoring.
3. The ML model should be served via a dedicated inference service with strict latency SLAs (e.g., p99 < 50 ms). If the model times out, the system should fall back to a rule-only decision rather than blocking the transaction indefinitely.
4. The feedback loop has two timescales: (a) near-real-time — when a chargeback is filed, the transaction is labeled as fraudulent and added to the training dataset; (b) periodic — the model is retrained weekly on the updated dataset and deployed via a blue-green deployment to avoid downtime.

</details>

<details>
<summary>Solution Outline</summary>

**Approach:** Implement a three-stage sequential pipeline with early termination, pre-computed feature caching, a dedicated ML inference service with fallback, and a chargeback-driven feedback loop.

1. **Pipeline architecture overview:**
   ```
   Transaction ──→ [Stage 1: Rule Engine] ──→ [Stage 2: ML Scoring] ──→ [Stage 3: Decision Engine] ──→ Verdict
                        │ (< 10 ms)              │ (< 50 ms)                │ (< 5 ms)
                        │                         │                          │
                        ▼                         ▼                          ▼
                   REJECT (obvious fraud)    Score 0.0–1.0            APPROVE / REJECT / REVIEW
                   APPROVE (allowlisted)
   ```
   - Total latency budget: 100 ms. Stage 1: 10 ms, Stage 2: 50 ms, Stage 3: 5 ms, overhead: 35 ms.
   - Early termination: If Stage 1 produces a definitive REJECT or APPROVE, Stages 2 and 3 are skipped.

2. **Stage 1 — Rule engine (< 10 ms):**
   - Rules are evaluated in priority order. Each rule returns REJECT, APPROVE, or CONTINUE (pass to next stage).
   - Example rules:
     - **Blocklist check:** If the card hash or IP address is on the known-fraud blocklist → REJECT. Blocklist is stored in Redis for O(1) lookup.
     - **Velocity check:** If this card has attempted > 10 transactions in the last 5 minutes → REJECT (card testing attack). Velocity counters are maintained in Redis with TTL-based expiry.
     - **Amount threshold:** If the transaction amount exceeds the merchant's configured maximum → REJECT.
     - **Geographic anomaly:** If the transaction originates from a country where the cardholder has never transacted before, and the amount exceeds $200 → CONTINUE with a risk flag (let ML decide).
     - **Allowlist:** If the card + merchant pair has 50+ successful transactions with no chargebacks → APPROVE (trusted relationship).
   - Rules are stored in a configuration service and can be updated without redeploying the application. Changes take effect within 30 seconds via a config push mechanism.

3. **Stage 2 — ML fraud scoring (< 50 ms):**
   - **Feature extraction:** Features are pre-computed and stored in Redis, updated incrementally as transactions arrive:
     - `card_txn_count_1h`: Number of transactions from this card in the last hour.
     - `card_avg_amount_30d`: Average transaction amount for this card over 30 days.
     - `merchant_chargeback_rate`: Chargeback rate for this merchant over 90 days.
     - `card_country_mismatch`: Boolean — does the transaction country differ from the card's issuing country?
     - `time_since_last_txn`: Seconds since the last transaction from this card.
     - `device_fingerprint_age`: How long this device fingerprint has been associated with this card.
   - Features are fetched in a single Redis MGET call (< 2 ms).
   - **Model serving:** A dedicated inference service hosts the fraud detection model (e.g., gradient-boosted trees or a small neural network). The service runs on GPU-equipped instances with model weights loaded in memory. Inference latency: p50 = 15 ms, p99 = 40 ms.
   - **Score interpretation:** The model returns a fraud probability between 0.0 (legitimate) and 1.0 (fraudulent). This score is passed to Stage 3.
   - **Timeout fallback:** If the ML service does not respond within 50 ms, the pipeline proceeds to Stage 3 with a default score of 0.5 (neutral) and a `ml_timeout` flag. The decision engine uses rule results alone in this case.

4. **Stage 3 — Decision engine (< 5 ms):**
   - Combines rule engine flags and ML score to produce a final verdict:
     ```
     if ml_score >= 0.85:
         verdict = REJECT  (high confidence fraud)
     elif ml_score >= 0.50:
         verdict = MANUAL_REVIEW  (borderline — needs human judgment)
     elif ml_score < 0.50 and no_rule_flags:
         verdict = APPROVE  (low risk)
     elif ml_timeout and rule_flags_present:
         verdict = MANUAL_REVIEW  (ML unavailable, rules flagged risk)
     elif ml_timeout and no_rule_flags:
         verdict = APPROVE  (ML unavailable, but rules see no risk)
     ```
   - Thresholds (0.85, 0.50) are configurable per merchant based on their risk tolerance. High-value merchants may set a lower review threshold.
   - The verdict, ML score, rule results, and all features are logged for every transaction (used for model retraining and auditing).

5. **Manual review queue:**
   - Transactions with verdict `MANUAL_REVIEW` are held in a pending state (authorization is not completed) and placed in a review queue.
   - Fraud analysts see the transaction details, ML score, triggered rules, and customer history. They approve or reject within a configurable SLA (e.g., 15 minutes).
   - If the SLA expires without a decision, the transaction is auto-approved (to avoid blocking the customer indefinitely) and flagged for post-transaction monitoring.
   - Queue prioritization: higher ML scores and larger amounts are reviewed first.

6. **Feedback loop and model retraining:**
   - **Near-real-time labeling:** When a chargeback is filed (from Problem 4's chargeback pipeline), the original transaction is labeled as `FRAUDULENT` in the training dataset. Transactions that are 90+ days old without a chargeback are labeled as `LEGITIMATE`.
   - **Weekly retraining:** Every week, the ML model is retrained on the updated dataset (last 12 months of labeled transactions). The new model is evaluated against a holdout set. If precision and recall meet thresholds (e.g., precision > 0.90, recall > 0.70), the model is promoted.
   - **Blue-green deployment:** The new model is deployed to a shadow inference service. For 24 hours, both old and new models score every transaction, but only the old model's score is used for decisions. If the new model's performance metrics are acceptable in production, traffic is switched to the new model.
   - **Rule updates:** Fraud analysts review weekly reports of false positives and false negatives. New rules (e.g., a newly discovered fraud pattern) are added to the rule engine configuration. Obsolete rules are retired.

7. **Monitoring and alerting:**
   - Track metrics: approval rate, rejection rate, manual review rate, ML model latency (p50, p99), rule engine latency, false positive rate (legitimate transactions rejected), false negative rate (fraudulent transactions approved).
   - Alert if the rejection rate spikes above 5% (may indicate a rule misconfiguration blocking legitimate traffic).
   - Alert if ML model latency p99 exceeds 50 ms (risk of timeouts degrading fraud detection accuracy).
   - Dashboard showing real-time fraud detection performance, accessible to the fraud operations team.

</details>
