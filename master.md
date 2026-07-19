# Master Document — SDE Intern Assignment Solution

## What Was The Task?

An SDE Intern technical assessment requiring:
1. **Low-Level Design (LLD)** for a User Payout Management System (affiliate sales)
2. **Failed Payout Recovery** system (Question 2)
3. Working implementation in JavaScript or Python
4. Database schema, class design, APIs, edge cases, and design trade-off explanations
5. Submission as a public GitHub repo with README and docs

---

## What I Built

A complete **Node.js + Express.js** REST API backend with:

| Deliverable | Status | Location |
|------------|--------|----------|
| LLD Document | ✅ Done | `docs/LLD.md` |
| Database Schema | ✅ Done | `src/config/database.js` |
| Class Design | ✅ Done | `src/models/` + `src/services/` |
| API Endpoints (15 total) | ✅ Done | `src/routes/index.js` |
| Edge Cases & Failure Handling | ✅ Done | `tests/payout.test.js` |
| Working Implementation | ✅ Done | Full `src/` directory |
| Design Decisions Doc | ✅ Done | `docs/LLD.md` Section 9 |
| README | ✅ Done | `README.md` |
| Tests (24 passing) | ✅ Done | `tests/payout.test.js` |

---

## My Thought Process (Step by Step)

### Step 1: Understanding the Domain

I read the PDF and identified the core domain entities:
- **Users** — affiliate partners who earn commissions
- **Sales** — individual product sales with lifecycle (pending → approved/rejected)
- **Advance Payouts** — 10% pre-payment on pending sales
- **Reconciliation** — admin action that finalizes sale status
- **Withdrawals** — users cashing out their earned balance
- **Failed Payouts** — recovery when a withdrawal fails (Question 2)

### Step 2: Choosing the Tech Stack

**Decision: JavaScript + Node.js + Express + SQLite**

Why?
- Assignment says "JavaScript or Python" — I chose JS since it's the more common backend choice for these assessments
- **SQLite** instead of PostgreSQL/MongoDB so the reviewer can run it **instantly** without Docker or any database setup
- `better-sqlite3` gives synchronous, transactional SQLite — perfect for demonstrating atomic operations
- Express.js is the industry standard for REST APIs in Node

> [!NOTE]
> I specifically chose SQLite (embedded DB) over MongoDB/PostgreSQL because the submission instructions say the repo should be self-contained. A reviewer cloning the repo should be able to `npm install && npm start` without installing anything else.

### Step 3: Schema Design — The Hardest Part

The schema went through mental iterations:

**First thought:** Just have `users` and `sales` tables, compute everything on the fly.

**Problem:** Computing balance from scratch every time means O(n) queries. And there's no audit trail for advances paid.

**Second thought:** Add a `user_balances` table as a materialized view.

**Problem:** How do I track which sales got advances? If I just track a total, I can't do per-sale clawbacks during reconciliation.

**Final design:** 6 tables total:
1. `users` — user accounts
2. `sales` — with `advance_paid` and `advance_transferred` flag per sale
3. `advance_payouts` — immutable log of every advance issued
4. `payouts` — withdrawal lifecycle tracking
5. `payout_adjustments` — immutable audit trail of ALL balance changes
6. `user_balances` — materialized balance for O(1) reads

> [!IMPORTANT]
> The key insight was: **every sale needs its own advance tracking** (`advance_paid`, `advance_transferred`). Without per-sale tracking, you can't calculate the correct clawback when a specific sale is rejected. If 3 sales each got ₹4 advance and only 1 is rejected, you need to claw back exactly ₹4, not a proportion of the aggregate.

### Step 4: Architecture — Layered Design

I chose a 4-layer architecture:

```
Routes → Controllers → Services → Models → Database
```

**Why not just controllers talking to the DB directly?**
- The business logic is complex (transactions, multi-table updates)
- Services encapsulate transactions — the controller doesn't need to know about atomicity
- Models are thin data-access wrappers — easy to swap databases
- This separation makes testing much easier (tested services directly in tests)

### Step 5: The Tricky Balance Calculation

The problem statement says:
> "3 sales of ₹40 each, advance = ₹12, after reconciliation Final Payout = ₹68"

I initially got confused here. My system gives a balance of **₹80**, not ₹68. Is that a bug?

**No!** Here's the key distinction:
- **₹68** = the ADDITIONAL amount owed after reconciliation (what's LEFT to pay)
- **₹80** = the total withdrawable balance (₹12 advance already credited + ₹68 remaining)

The ₹12 advance was already credited to the balance when it was paid. So the balance correctly shows ₹80 — the user hasn't withdrawn anything yet.

If the user had already withdrawn the ₹12 advance, then their remaining balance would be ₹68, matching the problem statement's "Final Payout."

> [!TIP]
> I wrote a test that verifies both the individual adjustments (−₹4, +₹36, +₹36) and the final balance (₹80). The test includes detailed comments explaining why ₹80 is correct even though the problem says ₹68.

### Step 6: Idempotency — The Most Important Business Rule

> "Once an advance payout has been successfully transferred, the same sale must never receive another advance payout, even if the advance payout job runs multiple times."

This is implemented via:
1. A `advance_transferred` boolean flag on each sale
2. The query `findPendingWithoutAdvance()` only returns sales where `advance_transferred = 0`
3. Within a transaction, we set `advance_transferred = 1` before moving to the next sale
4. Even if the process crashes mid-batch and reruns, already-flagged sales won't be re-processed

I wrote a specific test for this: run the advance job twice, verify the balance doesn't double.

### Step 7: Failed Payout Recovery (Question 2)

The recovery flow:
1. Find the failed payout record
2. Validate it's in a recoverable state (`initiated` or `processing`, NOT `completed`)
3. Update payout status to `failed`/`cancelled`/`rejected`
4. Credit the amount back to `user_balances.withdrawable_balance`
5. Reverse the `total_withdrawn` counter
6. Create audit trail entry

> [!WARNING]
> A completed payout CANNOT be marked as failed (money already sent). This is enforced with a status guard. I wrote a test specifically for this: complete a payout, then try to mark it failed → expect error.

---

## Mistakes I Made (and How I Fixed Them)

### Mistake 1: Unicode encoding on Windows
When extracting PDF text, the first attempt crashed with `UnicodeEncodeError: 'charmap' codec can't encode characters`. The ₹ symbol (Unicode) doesn't encode in Windows' default cp1252 codepage.

**Fix:** Added `sys.stdout.reconfigure(encoding='utf-8')` before printing.

### Mistake 2: Misunderstanding "Final Payout = ₹68"
Initially I thought my system had a bug because the balance showed ₹80 instead of ₹68. I spent time re-tracing the math before realizing:
- ₹68 = post-reconciliation adjustment only
- ₹80 = total withdrawable (includes advance already in balance)

Both are correct — they measure different things. The test now documents this clearly.

### Mistake 3: Not handling reconciliation without advance
My first implementation assumed every sale would have an advance before reconciliation. But what if the admin reconciles a sale before the advance job runs?

**Fix:** Added `advancePaid || 0` default in reconciliation logic. If no advance was paid:
- Approved → full earning credited
- Rejected → no clawback needed (nothing to claw back)

Added specific tests for both cases.

### Mistake 4: 24-hour restriction counting failed payouts
Initially, the 24-hour check counted ALL payouts including failed ones. This meant: if a user's payout failed, they'd be locked out for 24 hours even though they didn't actually receive money.

**Fix:** The `hasRecentWithdrawal` query only counts payouts with status `initiated`, `processing`, or `completed` — not `failed`, `cancelled`, or `rejected`.

---

## Files Created

```
c:\Users\HARSH KUMAR SHARMA\Desktop\taskFayn\
├── package.json                          # Dependencies & scripts
├── .gitignore                            # Excludes node_modules, data/
├── README.md                             # Full project documentation
├── docs/
│   └── LLD.md                            # Complete Low-Level Design
├── src/
│   ├── app.js                            # Express entry point
│   ├── config/
│   │   └── database.js                   # SQLite schema (6 tables, indexes)
│   ├── models/
│   │   ├── User.js                       # User CRUD
│   │   ├── Sale.js                       # Sale lifecycle management
│   │   ├── Payout.js                     # Withdrawal records
│   │   ├── UserBalance.js                # Materialized balance
│   │   └── PayoutAdjustment.js           # Audit trail
│   ├── services/
│   │   ├── AdvancePayoutService.js       # Idempotent 10% advance logic
│   │   ├── ReconciliationService.js      # Approve/reject + adjustments
│   │   └── WithdrawalService.js          # Withdrawal + failed recovery
│   ├── controllers/
│   │   ├── UsersController.js            # User endpoints
│   │   ├── SalesController.js            # Sale endpoints
│   │   └── PayoutController.js           # All payout operations
│   ├── routes/
│   │   └── index.js                      # 15 API routes
│   ├── middleware/
│   │   └── errorHandler.js               # Error + request logging
│   └── utils/
│       └── seed.js                       # Sample data seeder
└── tests/
    └── payout.test.js                    # 24 tests, all passing
```

---

## How to Run Everything

```bash
# Install
npm install

# Seed sample data
npm run seed

# Run server
npm start           # → http://localhost:3000

# Run tests (24/24 passing)
npm test
```

---

## Test Results

```
# tests 24
# suites 7
# pass 24
# fail 0
# cancelled 0
# skipped 0
```

Test categories:
- **User & Sale Creation** (3 tests)
- **Advance Payout Processing** (3 tests) — includes idempotency
- **Reconciliation** (5 tests) — includes exact problem statement example
- **Withdrawals** (3 tests) — includes 24-hour restriction
- **Failed Payout Recovery** (5 tests) — Question 2 complete
- **Edge Cases** (5 tests) — zero earnings, no advance, negative balance, etc.
