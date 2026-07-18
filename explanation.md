# Explanation & Interview Preparation Guide

> This document explains every aspect of the User Payout Management System, breaks down the flows in simple terms, and prepares you for interview questions.

---

## Part 1: Understanding the Problem (In Simple Terms)

### What is this system?

Imagine you run an affiliate marketing platform like Amazon Associates. When a user (affiliate) refers a customer who buys something:

1. **Sale is created** → status = `pending` (we don't know yet if the customer will keep the product)
2. **We pay 10% advance** → to keep affiliates happy while they wait
3. **After some days**, the return period ends:
   - **Customer kept the product** → sale becomes `approved` → we pay the remaining 90%
   - **Customer returned the product** → sale becomes `rejected` → we take back the 10% advance
4. **User withdraws money** → but only once every 24 hours
5. **If the bank transfer fails** → we credit the money back so they can try again

---

## Part 2: Complete Flow Walkthrough

### Flow 1: Advance Payout (The 10% Pre-Payment)

**When does it happen?**  
A background job runs periodically (could be every hour, daily, etc.)

**What does it do?**
```
For each user:
  1. Find all sales that are:
     - status = "pending"  AND
     - advance_transferred = 0 (haven't received advance yet)
  2. For each such sale:
     - Calculate: advance = earning × 10%
     - Set advance_transferred = 1 on the sale (PREVENT DUPLICATES)
     - Add advance amount to user's withdrawable_balance
     - Log it in payout_adjustments (audit trail)
  3. All of this happens inside ONE database transaction
```

**Why is idempotency important?**  
If the job crashes halfway and reruns, sales that already have `advance_transferred = 1` are skipped. No double-payment ever.

**Real example:**
```
Sale: { earning: ₹40, advance_transferred: 0 }
  → advance = ₹40 × 0.10 = ₹4
  → sale.advance_paid = ₹4, sale.advance_transferred = 1
  → user.withdrawable_balance += ₹4
  → adjustment_log: { type: "advance_credit", amount: ₹4 }
```

---

### Flow 2: Reconciliation (Approve or Reject)

**When does it happen?**  
An admin manually reviews sales after the return period ends.

**What happens for APPROVED sale?**
```
Sale: { earning: ₹40, advance_paid: ₹4 }
  → remainder = ₹40 - ₹4 = ₹36
  → user.withdrawable_balance += ₹36
  → adjustment_log: { type: "reconciliation_credit", amount: ₹36 }
  → sale.status = "approved"
```
The user already got ₹4 as advance, so we only pay the remaining ₹36.

**What happens for REJECTED sale?**
```
Sale: { earning: ₹40, advance_paid: ₹4 }
  → The user already received ₹4 they shouldn't have
  → user.withdrawable_balance -= ₹4  (CLAWBACK)
  → adjustment_log: { type: "reconciliation_debit", amount: -₹4 }
  → sale.status = "rejected"
```
We take back the advance because the product was returned.

**Special case: Sale reconciled BEFORE advance job runs**
- `advance_paid = 0`
- Approved → full ₹40 credited (nothing to subtract)
- Rejected → nothing to claw back (nothing was advanced)

---

### Flow 3: The Problem Statement Example (Most Important!)

```
3 sales: ₹40 each → Total pending = ₹120

Step 1: Advance payout job
  - Sale 1: advance = ₹4
  - Sale 2: advance = ₹4
  - Sale 3: advance = ₹4
  - Total advance = ₹12
  - Balance = ₹12

Step 2: Reconciliation
  - Sale 1 → REJECTED:  clawback ₹4  → Balance = ₹12 - ₹4 = ₹8
  - Sale 2 → APPROVED:  credit ₹36   → Balance = ₹8 + ₹36 = ₹44
  - Sale 3 → APPROVED:  credit ₹36   → Balance = ₹44 + ₹36 = ₹80

Result:
  - "Final Payout" from problem = ₹68 (= -₹4 + ₹36 + ₹36)
  - This is the post-reconciliation adjustment only
  - Our system's balance = ₹80 (includes the ₹12 advance already credited)
  - If user already withdrew ₹12, remaining = ₹68 ← matches problem statement
```

**Key insight for interview:** The ₹68 and ₹80 are BOTH correct. ₹68 = remaining to pay. ₹80 = total withdrawable (advance + remaining).

---

### Flow 4: Withdrawal (24-Hour Rule)

```
User wants to withdraw ₹50:
  1. Check: has user withdrawn in last 24 hours? → Query payouts table
  2. Check: is balance >= ₹50? → Check user_balances table
  3. If both pass:
     - Create payout record (status: "initiated")
     - Debit ₹50 from withdrawable_balance
     - Return payout details
  4. If 24-hour check fails: → Return error with last withdrawal time
  5. If balance insufficient: → Return error with available balance
```

**Why do we count "initiated" and "processing" in the 24-hour check?**  
Because even if the transfer hasn't completed, the user already has a withdrawal in progress. We should wait for it to finish (or fail) before allowing another.

**Why DON'T we count "failed" payouts?**  
Because the money was returned to the user's balance. Blocking them after a failure would be unfair — they never actually received the money.

---

### Flow 5: Failed Payout Recovery (Question 2)

```
Scenario: User withdrew ₹50, bank transfer failed

Before failure:
  - Balance: ₹30 (was ₹80, withdrew ₹50)
  - Payout: { status: "initiated", amount: ₹50 }

After handleFailedPayout():
  - Payout: { status: "failed", reason: "Bank error" }
  - Balance: ₹30 + ₹50 = ₹80 (money returned)
  - total_withdrawn reversed: ₹50 → ₹0
  - Audit entry: { type: "failed_payout_credit", amount: ₹50 }
  
User can now withdraw again (subject to 24-hour rule from their
last SUCCESSFUL withdrawal, not the failed one).
```

**Critical guard:** A COMPLETED payout cannot be marked as failed. The status check ensures only `initiated` or `processing` payouts can fail. This prevents crediting money that was already successfully transferred.

---

## Part 3: Why I Made Each Design Decision

### Decision 1: Why SQLite instead of MongoDB/PostgreSQL?

**For this assignment:** Reviewer can `npm install && npm start`. No Docker, no MongoDB Atlas, no PostgreSQL server to install.

**For production:** I would use PostgreSQL because:
- Row-level locking for concurrent balance updates
- ACID transactions with full isolation levels
- Better query planner for complex joins
- Connection pooling for high-traffic

**How easy is migration?** Only `src/config/database.js` changes. Every model uses standard SQL. The layered architecture isolates the database concern.

### Decision 2: Why a `user_balances` table instead of computing from adjustments?

**Option A (chosen):** Pre-computed balance in `user_balances`
- Balance read = 1 query → O(1)
- Every write updates the balance → slight overhead
- Risk: balance can drift if a write is missed

**Option B:** Compute balance by summing all adjustments
- Balance read = aggregate query → O(n) where n = number of adjustments
- No drift risk — always consistent
- Gets slower as adjustments grow

**Why I chose A:** In a real payout system, balance is checked on every dashboard page load, every withdrawal attempt, and every advance calculation. O(1) reads are critical. The `payout_adjustments` table exists as a safety net — if we ever suspect drift, we can recompute the true balance from the audit trail.

### Decision 3: Why per-sale advance tracking?

If we only tracked "John has received ₹12 in total advances," we can't determine what happens when Sale 1 is rejected. Do we claw back ₹4? ₹6? ₹12?

By storing `advance_paid` on each sale, we know Sale 1 got ₹4, so we claw back exactly ₹4. This is unambiguous and correct.

### Decision 4: Why database transactions for every operation?

Consider what happens without transactions during withdrawal:
```
1. Create payout record ✓
2. --- SERVER CRASHES ---
3. Debit balance ✗ (never runs)
```
Result: Payout exists but balance was never debited. User effectively got free money.

With a transaction, step 1 and step 3 either BOTH succeed or BOTH fail. The database rolls back automatically on crash.

### Decision 5: Why an immutable audit trail?

Financial systems must answer: "Why does John have ₹47.50 in his balance?"

The `payout_adjustments` table can reconstruct the entire balance history:
```
1. advance_credit:         +₹4   (sale abc)
2. advance_credit:         +₹4   (sale def)
3. reconciliation_debit:   -₹4   (sale abc rejected)
4. reconciliation_credit:  +₹36  (sale def approved)
5. failed_payout_credit:   +₹7.50 (payout xyz failed)
  
Sum = ₹47.50 ← matches current balance
```

---

## Part 4: Code Architecture Explained

### How a request flows through the system

```
HTTP Request
    ↓
src/app.js (Express middleware: JSON parsing, logging)
    ↓
src/routes/index.js (maps URL to controller method)
    ↓
src/controllers/PayoutController.js (validates input, calls service)
    ↓
src/services/ReconciliationService.js (business logic + transaction)
    ↓
src/models/Sale.js + UserBalance.js (SQL queries)
    ↓
src/config/database.js (SQLite connection)
    ↓
Response flows back up the chain
```

### Why this layering matters

| If you put business logic in... | Problem |
|--------------------------------|---------|
| Controller | Can't reuse logic. Can't test without HTTP. |
| Model | Models become bloated. Cross-table logic is messy. |
| Service (correct) | Testable. Reusable. Clean boundaries. |

---

## Part 5: Potential Interview Questions & Answers

### Q1: "Walk me through what happens when a sale is created and then approved."

**Answer:**
1. `POST /api/sales` → Sale created with `status: pending`
2. Advance job runs → 10% of earning credited to user's balance, `advance_transferred = 1`
3. Admin calls `POST /api/reconciliation/sale` with `status: approved`
4. Service calculates `remainder = earning - advance_paid`
5. Remainder credited to `withdrawable_balance`
6. Sale status updated to `approved`
7. Audit entry created with type `reconciliation_credit`
8. All of steps 4-7 happen in one transaction

---

### Q2: "How do you prevent the advance payout from being paid twice?"

**Answer:**
Two-layer protection:
1. **Application layer:** `findPendingWithoutAdvance()` only returns sales where `advance_transferred = 0`
2. **Transaction:** Once we set `advance_transferred = 1`, even if another instance of the job runs simultaneously, the query won't return that sale

In production with PostgreSQL, I'd add a third layer: `SELECT ... FOR UPDATE` to lock the row during the transaction.

---

### Q3: "What happens if a rejected sale's clawback makes the balance negative?"

**Answer:**
We allow negative balances. This represents a debt the user owes the platform.

Example:
- User gets ₹10 advance, withdraws ₹10
- Sale gets rejected → clawback ₹10 → balance = -₹10
- The user cannot withdraw (balance < 0)
- Future approved sales will first offset this debt before the user can withdraw again

---

### Q4: "Why didn't you use MongoDB?"

**Answer:**
This system is inherently transactional — balance updates, audit logs, and status changes must happen atomically. SQL databases with ACID transactions are the natural fit.

MongoDB has multi-document transactions (since 4.0), but they're slower, have more edge cases, and don't support the CHECK constraints I use for status validation at the database level.

That said, the audit trail (`payout_adjustments`) could be in MongoDB if we needed flexible schema for different adjustment types in the future.

---

### Q5: "How would you scale this for 10 million users?"

**Answer:**
1. **Database:** PostgreSQL with read replicas. Balance reads go to replicas, writes to primary.
2. **Advance job:** Run as a distributed worker queue (BullMQ/SQS). Partition by user_id hash to prevent concurrent processing of the same user.
3. **Caching:** Redis for balance reads. Invalidate on any write.
4. **Rate limiting:** Redis-backed rate limiter (sliding window) instead of DB-based 24-hour check.
5. **Sharding:** Shard `payout_adjustments` by user_id (append-only, grows fastest).

---

### Q6: "What if the server crashes mid-transaction?"

**Answer:**
Database transactions are atomic. If the server crashes before `COMMIT`:
- All changes are rolled back automatically
- No partial state — balance and audit log stay consistent
- The operation can be safely retried

This is why I wrap every multi-step operation in `db.transaction()`.

---

### Q7: "How does the 24-hour withdrawal restriction work exactly?"

**Answer:**
```sql
SELECT COUNT(*) FROM payouts 
WHERE user_id = ? 
  AND status IN ('initiated', 'processing', 'completed')
  AND initiated_at >= datetime('now', '-24 hours')
```
If count > 0, withdrawal is denied.

Key detail: `failed`, `cancelled`, and `rejected` payouts are NOT counted. If your payout failed, you shouldn't be punished with a 24-hour lockout.

---

### Q8: "Why is `payout_adjustments` table necessary? Can't you just look at sales and payouts?"

**Answer:**
The adjustments table serves three purposes:
1. **Audit trail:** Every balance change is traceable with timestamp, type, and description
2. **Balance reconstruction:** If the materialized balance drifts, sum all adjustments to get the true balance
3. **Debugging:** When a user disputes their balance, customer support can see exactly what happened

Without it, you'd have to reverse-engineer the balance from sale statuses, advance payouts, and withdrawal records — much harder.

---

### Q9: "How do you handle failed payouts? Walk me through the complete recovery."

**Answer:**
1. Payout gateway reports failure (webhook/API call)
2. We validate the payout is in `initiated` or `processing` status (not `completed`)
3. In a single transaction:
   - Update payout status to `failed` with reason
   - Credit the amount back to `withdrawable_balance`
   - Reverse the `total_withdrawn` counter
   - Create `failed_payout_credit` audit entry
4. User can now see their restored balance and initiate a new withdrawal

The status guard is critical — if a payout is `completed`, it means money was successfully sent. We must NOT credit it back, or the user would effectively receive double payment.

---

### Q10: "What are the biggest risks in this system?"

**Answer:**
1. **Double advance payment:** Mitigated by `advance_transferred` flag + transactions
2. **Balance drift:** Mitigated by audit trail that can reconstruct true balance
3. **Race conditions on withdrawal:** Mitigated by transactions (SQLite serializes writes)
4. **Clawback exceeding balance:** Allowed — tracked as negative balance (debt)
5. **Failed payout not reported:** Would need a periodic job to check payment gateway status

---

### Q11: "If you had more time, what would you add?"

**Answer:**
1. **Authentication & authorization** — JWT tokens, admin vs. user roles
2. **Webhook receiver** for payment gateway callbacks (auto-handle failures)
3. **Pagination** on list endpoints (currently returns all records)
4. **Balance reconciliation job** — periodic check comparing materialized vs. computed balance
5. **API rate limiting** — prevent abuse beyond the 24-hour withdrawal rule
6. **Logging with correlation IDs** — trace a request across all service calls
7. **Input sanitization** — prevent SQL injection (though better-sqlite3 uses parameterized queries)

---

## Part 6: Common Mistakes to Avoid in the Interview

| Mistake | Why it's wrong | What to say instead |
|---------|---------------|-------------------|
| "I'd use MongoDB for everything" | Financial data needs ACID transactions | "SQL for transactional data, NoSQL for audit logs if needed" |
| "Balance is computed on every read" | O(n) doesn't scale | "Materialized balance with audit trail as safety net" |
| "Just use a boolean for advance tracking" | Need the amount for reconciliation math | "Track both the flag AND the amount per sale" |
| "24-hour check uses all payouts" | Failed payouts shouldn't block users | "Only count initiated/processing/completed in the window" |
| "Transactions aren't needed for single DB" | Crashes can happen between any two operations | "Transactions ensure atomicity regardless of DB type" |
