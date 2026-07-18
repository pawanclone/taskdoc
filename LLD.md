# Low-Level Design (LLD) — User Payout Management System

> **Author:** Harsh Kumar Sharma  
> **Language:** JavaScript (Node.js)  
> **Database:** SQLite  
> **Purpose:** Technical Assessment — SDE Intern

---

## 1. Problem Statement Summary

Build a system to manage affiliate sale payouts with:
- 10% advance payouts on pending sales (idempotent)
- Admin reconciliation (approve/reject) with advance adjustment
- 24-hour withdrawal rate limiting
- Failed payout recovery (credit back to balance)

---

## 2. High-Level Architecture

```mermaid
graph TD
    Client["🖥️ REST Client"]

    Client -->|HTTP/JSON| API

    subgraph API["API Layer"]
        Router --> UsersCtrl["UsersController"]
        Router --> SalesCtrl["SalesController"]
        Router --> PayoutCtrl["PayoutController"]
    end

    subgraph BIZ["Service Layer"]
        AdvanceSvc["AdvancePayoutService"]
        ReconSvc["ReconciliationService"]
        WithdrawSvc["WithdrawalService"]
    end

    subgraph DATA["Model Layer"]
        UserModel["User"]
        SaleModel["Sale"]
        PayoutModel["Payout"]
        BalanceModel["UserBalance"]
        AuditModel["PayoutAdjustment"]
    end

    subgraph DB["SQLite Database"]
        direction LR
        T1[("users")]
        T2[("sales")]
        T3[("payouts")]
        T4[("user_balances")]
        T5[("payout_adjustments")]
    end

    PayoutCtrl --> AdvanceSvc
    PayoutCtrl --> ReconSvc
    PayoutCtrl --> WithdrawSvc

    AdvanceSvc --> SaleModel
    AdvanceSvc --> BalanceModel
    ReconSvc --> SaleModel
    ReconSvc --> BalanceModel
    WithdrawSvc --> PayoutModel
    WithdrawSvc --> BalanceModel

    DATA --> DB
```

### Layer Responsibilities

| Layer | Role | Principle |
|-------|------|-----------|
| **Routes** | URL → Controller mapping | Single Responsibility |
| **Controllers** | Input validation, HTTP status codes, response shaping | Thin — no business logic |
| **Services** | Business rules, transaction orchestration, multi-model coordination | All domain logic lives here |
| **Models** | Single-table CRUD, SQL queries, data retrieval | No cross-table awareness |
| **Database** | Schema, constraints, indexes, referential integrity | Data integrity at DB level |

**Why this layering?**  
Controller doesn't know about transactions. Service doesn't know about HTTP. Model doesn't know about business rules. Each layer can be tested and replaced independently.

---

## 3. Entity-Relationship Diagram

```mermaid
erDiagram
    users ||--o{ sales : has
    users ||--o{ payouts : requests
    users ||--|| user_balances : has
    users ||--o{ payout_adjustments : receives
    sales ||--o{ advance_payouts : receives
    sales ||--o{ payout_adjustments : triggers
    payouts ||--o{ payout_adjustments : triggers

    users {
        TEXT id PK
        TEXT name
        TEXT email UK
        TEXT created_at
        TEXT updated_at
    }

    sales {
        TEXT id PK
        TEXT user_id FK
        TEXT brand
        TEXT status
        REAL earning
        REAL advance_paid
        INT advance_transferred
        INT reconciled
        TEXT created_at
        TEXT updated_at
    }

    advance_payouts {
        TEXT id PK
        TEXT sale_id FK
        TEXT user_id FK
        REAL amount
        TEXT transferred_at
    }

    payouts {
        TEXT id PK
        TEXT user_id FK
        REAL amount
        TEXT status
        TEXT failure_reason
        TEXT initiated_at
        TEXT completed_at
        TEXT updated_at
    }

    payout_adjustments {
        TEXT id PK
        TEXT user_id FK
        TEXT sale_id FK
        TEXT payout_id FK
        TEXT type
        REAL amount
        TEXT description
        TEXT created_at
    }

    user_balances {
        TEXT user_id PK
        REAL withdrawable_balance
        REAL total_earned
        REAL total_advance_paid
        REAL total_withdrawn
        TEXT last_withdrawal_at
        TEXT updated_at
    }
```

**Column constraints** (enforced at DB level):

| Table | Column | Constraint |
|-------|--------|------------|
| sales | brand | `CHECK IN (brand_1, brand_2, brand_3)` |
| sales | status | `CHECK IN (pending, approved, rejected)` |
| sales | earning | `CHECK >= 0` |
| sales | advance_transferred | `0` (no) or `1` (yes) — idempotency flag |
| payouts | amount | `CHECK > 0` |
| payouts | status | `CHECK IN (initiated, processing, completed, failed, cancelled, rejected)` |
| payout_adjustments | type | `CHECK IN (advance_credit, reconciliation_credit, reconciliation_debit, failed_payout_credit)` |
| user_balances | all REAL columns | `DEFAULT 0` |

---

## 4. Database Schema Details

### Indexes (for query performance)

```sql
CREATE INDEX idx_sales_user_status    ON sales(user_id, status);
CREATE INDEX idx_sales_status         ON sales(status);
CREATE INDEX idx_advance_payouts_sale ON advance_payouts(sale_id);
CREATE INDEX idx_payouts_user         ON payouts(user_id);
CREATE INDEX idx_payouts_status       ON payouts(status);
CREATE INDEX idx_adjustments_user     ON payout_adjustments(user_id);
```

### Why These Indexes?

| Index | Query It Speeds Up |
|-------|--------------------|
| `(user_id, status)` on sales | Advance job: `WHERE user_id = ? AND status = 'pending'` |
| `(status)` on sales | Batch job scanning all pending sales |
| `(sale_id)` on advance_payouts | Checking if advance already paid |
| `(user_id)` on payouts | 24-hour restriction check per user |
| `(user_id)` on adjustments | Fetching audit trail per user |

---

## 5. State Machine Diagrams

### 5.1 Sale Status Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Pending: Sale Created

    Pending --> Approved: Admin Approves
    Pending --> Rejected: Admin Rejects

    Approved --> [*]
    Rejected --> [*]

    note right of Pending
        Eligible for 10% advance.
        Flag prevents duplicate advances.
    end note

    note right of Approved
        Payout = earning - advance
    end note

    note right of Rejected
        Clawback = -advance_paid
    end note
```

### 5.2 Payout (Withdrawal) Status Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Initiated: User Withdraws

    Initiated --> Processing: Gateway picks up
    Initiated --> Completed: Direct success

    Processing --> Completed: Transfer done

    Initiated --> Failed: Error
    Processing --> Failed: Error
    Initiated --> Cancelled: User cancels
    Initiated --> Rejected: Gateway rejects

    Failed --> [*]: Balance restored
    Cancelled --> [*]: Balance restored
    Rejected --> [*]: Balance restored
    Completed --> [*]: Final
```

---

## 6. Core Workflow Diagrams

### 6.1 Advance Payout Flow

```mermaid
flowchart TD
    A["🔄 Advance Job Triggered"] --> B["Find pending sales where advance_transferred = 0"]
    B --> C{"Any eligible sales?"}
    C -->|No| D["✅ Done — nothing to process"]
    C -->|Yes| E["🔒 BEGIN TRANSACTION"]
    E --> F["For each sale: calculate 10% of earning"]
    F --> G["Set advance_transferred = 1 on sale"]
    G --> H["Credit amount to user balance"]
    H --> I["Log in payout_adjustments"]
    I --> J{"More sales?"}
    J -->|Yes| F
    J -->|No| K["COMMIT TRANSACTION"]
    K --> L["✅ Return summary"]

    style A fill:#4a90d9,color:#fff
    style D fill:#27ae60,color:#fff
    style L fill:#27ae60,color:#fff
    style E fill:#e67e22,color:#fff
    style K fill:#e67e22,color:#fff
```

**Idempotency:** If this job runs again, step B filters out already-processed sales (where `advance_transferred = 1`). No double-payment ever.

### 6.2 Reconciliation Flow

```mermaid
flowchart TD
    A["👨‍💼 Admin Reconciles Sale"] --> B["Fetch sale by ID"]
    B --> C{"Sale status = pending?"}
    C -->|No| D["❌ Error: Already reconciled"]
    C -->|Yes| E["🔒 BEGIN TRANSACTION"]

    E --> F{"New Status?"}

    F -->|Approved| G["Calculate: remainder = earning - advance_paid"]
    G --> H["Credit remainder to user balance"]
    H --> I["Log: reconciliation_credit"]

    F -->|Rejected| J["Calculate: clawback = advance_paid"]
    J --> K["Debit clawback from user balance"]
    K --> L["Log: reconciliation_debit"]

    I --> M["Update sale.status"]
    L --> M
    M --> N["COMMIT TRANSACTION"]
    N --> O["✅ Return adjustment details"]

    style A fill:#4a90d9,color:#fff
    style D fill:#e74c3c,color:#fff
    style O fill:#27ae60,color:#fff
    style G fill:#27ae60,color:#fff
    style J fill:#e74c3c,color:#fff
```

### 6.3 Withdrawal Flow

```mermaid
flowchart TD
    A["💰 User Requests Withdrawal"] --> B{"Withdrew in last 24h?"}
    B -->|Yes| C["❌ Denied: 24-hour cooldown"]
    B -->|No| D{"Balance >= amount?"}
    D -->|No| E["❌ Denied: Insufficient balance"]
    D -->|Yes| F["🔒 BEGIN TRANSACTION"]
    F --> G["Create payout record — status: initiated"]
    G --> H["Debit amount from withdrawable_balance"]
    H --> I["COMMIT TRANSACTION"]
    I --> J["✅ Return payout details"]

    style A fill:#4a90d9,color:#fff
    style C fill:#e74c3c,color:#fff
    style E fill:#e74c3c,color:#fff
    style J fill:#27ae60,color:#fff
```

### 6.4 Failed Payout Recovery (Question 2)

```mermaid
flowchart TD
    A["⚠️ Payout Failed / Cancelled / Rejected"] --> B["Fetch payout by ID"]
    B --> C{"Status = initiated or processing?"}
    C -->|No| D["❌ Error: Cannot reverse a completed payout"]
    C -->|Yes| E["🔒 BEGIN TRANSACTION"]
    E --> F["Update payout status to failed"]
    F --> G["Credit amount back to user balance"]
    G --> H["Reverse total_withdrawn counter"]
    H --> I["Log: failed_payout_credit in audit trail"]
    I --> J["COMMIT TRANSACTION"]
    J --> K["✅ User can re-withdraw the amount"]

    style A fill:#e67e22,color:#fff
    style D fill:#e74c3c,color:#fff
    style K fill:#27ae60,color:#fff
```

---

## 7. Sequence Diagrams (Detailed Interactions)

### 7.1 Advance Payout — Component Interaction

```mermaid
sequenceDiagram
    participant Job as Cron Job
    participant Svc as AdvancePayoutService
    participant Sale as Sale Model
    participant Bal as UserBalance
    participant Audit as PayoutAdjustment

    Job->>Svc: processForUser(userId)

    Svc->>Sale: findPendingWithoutAdvance(userId)
    Sale-->>Svc: eligible sales list

    loop Each eligible sale
        Svc->>Sale: markAdvanceTransferred(saleId, 10%)
        Svc->>Bal: recordAdvance(userId, amount)
        Svc->>Audit: create(advance_credit)
    end

    Svc-->>Job: processedCount, totalAdvance
```

### 7.2 Reconciliation — Component Interaction

```mermaid
sequenceDiagram
    participant Admin
    participant Svc as ReconciliationService
    participant Sale as Sale Model
    participant Bal as UserBalance

    Admin->>Svc: reconcileSale(saleId, status)
    Svc->>Sale: findById(saleId)
    Sale-->>Svc: sale data

    alt Approved
        Svc->>Bal: credit(earning - advance_paid)
    else Rejected
        Svc->>Bal: debit(advance_paid)
    end

    Svc->>Sale: updateStatus(saleId, newStatus)
    Svc-->>Admin: finalAdjustment
```

### 7.3 Failed Payout Recovery — Component Interaction

```mermaid
sequenceDiagram
    participant GW as Payment Gateway
    participant Svc as WithdrawalService
    participant Pay as Payout Model
    participant Bal as UserBalance
    participant Audit as PayoutAdjustment

    GW->>Svc: handleFailedPayout(payoutId, failed)
    Svc->>Pay: findById(payoutId)
    Pay-->>Svc: payout data

    Svc->>Pay: updateStatus(failed)
    Svc->>Bal: creditFailedPayout(amount)
    Svc->>Audit: create(failed_payout_credit)

    Svc-->>GW: balanceCredited = true
```

---

## 8. Complete Example Walkthrough (from Problem Statement)

### Sale A Journey (REJECTED)

```mermaid
flowchart LR
    A1["Sale A Created\nEarning: Rs.40\nStatus: PENDING"] --> A2["Advance Paid\n10% = Rs.4\nBalance: +Rs.4"] --> A3["Admin: REJECTED\nClawback: -Rs.4\nNet: Rs.0"]

    style A1 fill:#3498db,color:#fff
    style A2 fill:#f39c12,color:#fff
    style A3 fill:#e74c3c,color:#fff
```

### Sale B Journey (APPROVED)

```mermaid
flowchart LR
    B1["Sale B Created\nEarning: Rs.40\nStatus: PENDING"] --> B2["Advance Paid\n10% = Rs.4\nBalance: +Rs.4"] --> B3["Admin: APPROVED\nRemainder: Rs.40 - Rs.4 = Rs.36\nNet: Rs.40"]

    style B1 fill:#3498db,color:#fff
    style B2 fill:#f39c12,color:#fff
    style B3 fill:#27ae60,color:#fff
```

### Sale C Journey (APPROVED)

```mermaid
flowchart LR
    C1["Sale C Created\nEarning: Rs.40\nStatus: PENDING"] --> C2["Advance Paid\n10% = Rs.4\nBalance: +Rs.4"] --> C3["Admin: APPROVED\nRemainder: Rs.40 - Rs.4 = Rs.36\nNet: Rs.40"]

    style C1 fill:#3498db,color:#fff
    style C2 fill:#f39c12,color:#fff
    style C3 fill:#27ae60,color:#fff
```

### Combined Result

```mermaid
flowchart TD
    T["Total Pending Earnings = Rs.120"]
    T --> ADV["Advance Payout = 10% of Rs.120 = Rs.12"]
    ADV --> R{"Reconciliation"}
    R --> REJ["Sale A REJECTED\nAdjustment: -Rs.4"]
    R --> AP1["Sale B APPROVED\nAdjustment: +Rs.36"]
    R --> AP2["Sale C APPROVED\nAdjustment: +Rs.36"]
    REJ --> FINAL["Final Payout = -Rs.4 + Rs.36 + Rs.36 = Rs.68"]
    AP1 --> FINAL
    AP2 --> FINAL
    FINAL --> BAL["Withdrawable Balance = Rs.80\n(Rs.12 advance + Rs.68 final)"]

    style T fill:#3498db,color:#fff
    style ADV fill:#f39c12,color:#fff
    style REJ fill:#e74c3c,color:#fff
    style AP1 fill:#27ae60,color:#fff
    style AP2 fill:#27ae60,color:#fff
    style FINAL fill:#8e44ad,color:#fff
    style BAL fill:#2c3e50,color:#fff
```

### Step-by-Step Balance Trace

| Step | Action | Change | Balance |
|:----:|--------|:------:|:-------:|
| 0 | User created | — | **Rs.0** |
| 1 | Advance on Sale A (Rs.40 x 10%) | +Rs.4 | **Rs.4** |
| 2 | Advance on Sale B (Rs.40 x 10%) | +Rs.4 | **Rs.8** |
| 3 | Advance on Sale C (Rs.40 x 10%) | +Rs.4 | **Rs.12** |
| 4 | Sale A **REJECTED** → clawback advance | -Rs.4 | **Rs.8** |
| 5 | Sale B **APPROVED** → credit remainder (Rs.40 - Rs.4) | +Rs.36 | **Rs.44** |
| 6 | Sale C **APPROVED** → credit remainder (Rs.40 - Rs.4) | +Rs.36 | **Rs.80** |

**Final Payout (post-reconciliation) = -Rs.4 + Rs.36 + Rs.36 = Rs.68**  
**Total Withdrawable Balance = Rs.80** (includes Rs.12 advance already credited)

> The Rs.68 "Final Payout" from the problem statement is what remains AFTER accounting for the Rs.12 advance already paid. Our system's Rs.80 balance is correct — it represents the total the user can withdraw if they haven't withdrawn anything yet.

---

## 9. Class Design

### 9.1 Models (Data Access Layer)

```mermaid
classDiagram
    class User {
        +String id
        +String name
        +String email
        +create(name, email)
        +findById(id)
        +findAll()
    }

    class Sale {
        +String id
        +String user_id
        +String brand
        +String status
        +Number earning
        +Number advance_paid
        +Boolean advance_transferred
        +create(userId, brand, earning)
        +findPendingWithoutAdvance(userId)
        +markAdvanceTransferred(saleId, amt)
        +updateStatus(saleId, status)
    }

    class Payout {
        +String id
        +String user_id
        +Number amount
        +String status
        +create(userId, amount)
        +updateStatus(id, status, reason)
        +hasRecentWithdrawal(userId)
    }

    class UserBalance {
        +Number withdrawable_balance
        +Number total_earned
        +Number total_withdrawn
        +getBalance(userId)
        +credit(userId, amount)
        +debit(userId, amount)
        +recordWithdrawal(userId, amount)
        +creditFailedPayout(userId, amount)
    }

    class PayoutAdjustment {
        +String id
        +String type
        +Number amount
        +create(params)
        +findByUser(userId)
    }
```

### 9.2 Service → Model Dependency Map

Each service orchestrates multiple models inside a single database transaction.

```mermaid
flowchart LR
    subgraph Services
        APS["AdvancePayoutService"]
        RS["ReconciliationService"]
        WS["WithdrawalService"]
    end

    subgraph Models
        S["Sale"]
        UB["UserBalance"]
        P["Payout"]
        PA["PayoutAdjustment"]
    end

    APS --> S
    APS --> UB
    APS --> PA

    RS --> S
    RS --> UB
    RS --> PA

    WS --> P
    WS --> UB
    WS --> PA
```

### 9.3 Data Ownership (User → Entities)

```mermaid
flowchart TD
    U["User"] --> S["Sales (many)"]
    U --> P["Payouts (many)"]
    U --> UB["UserBalance (one)"]
    S --> PA1["PayoutAdjustments"]
    P --> PA2["PayoutAdjustments"]
```

**Key observations:**
- All 3 services depend on `UserBalance` — it is the central model
- All 3 services log to `PayoutAdjustment` — the immutable audit trail
- `AdvancePayoutService` and `ReconciliationService` share access to `Sale`
- Only `WithdrawalService` accesses `Payout`

---

## 10. API Endpoints — Complete Reference

### Users
| Method | Endpoint | Body | Response |
|--------|----------|------|----------|
| POST | `/api/users` | `{ name, email }` | `201: { success, data: User }` |
| GET | `/api/users` | — | `200: { success, data: User[] }` |
| GET | `/api/users/:id` | — | `200: { success, data: User }` |

### Sales
| Method | Endpoint | Body | Response |
|--------|----------|------|----------|
| POST | `/api/sales` | `{ userId, brand, earning }` | `201: { success, data: Sale }` |
| GET | `/api/sales/:id` | — | `200: { success, data: Sale }` |
| GET | `/api/users/:userId/sales` | — | `200: { success, data: { sales, summary } }` |

### Advance Payouts
| Method | Endpoint | Body | Response |
|--------|----------|------|----------|
| POST | `/api/advance-payouts/process/:userId` | — | `200: { processedCount, totalAdvance, details }` |
| POST | `/api/advance-payouts/process-all` | — | `200: { usersProcessed, totalAdvances }` |

### Reconciliation
| Method | Endpoint | Body | Response |
|--------|----------|------|----------|
| POST | `/api/reconciliation/sale` | `{ saleId, status }` | `200: { finalAdjustment, ... }` |
| POST | `/api/reconciliation/batch` | `{ reconciliations: [{saleId, status}] }` | `200: { processed, failed, results }` |
| GET | `/api/reconciliation/summary/:userId` | — | `200: { salesBreakdown, totalFinalPayout }` |

### Withdrawals
| Method | Endpoint | Body | Response |
|--------|----------|------|----------|
| POST | `/api/withdrawals` | `{ userId, amount }` | `201: { success, data: Payout }` |
| POST | `/api/withdrawals/:id/complete` | — | `200: { success, data: Payout }` |
| POST | `/api/withdrawals/:id/fail` | `{ status, reason? }` | `200: { balanceCredited: true }` |
| GET | `/api/withdrawals/:userId/history` | — | `200: { success, data: Payout[] }` |

### Balance
| Method | Endpoint | Body | Response |
|--------|----------|------|----------|
| GET | `/api/balance/:userId` | — | `200: { withdrawable_balance, total_earned, ... }` |

---

## 11. Edge Cases & Failure Handling

| # | Edge Case | Handling Strategy |
|---|-----------|-------------------|
| 1 | Advance job runs multiple times | `advance_transferred` flag — idempotent |
| 2 | Sale reconciled twice | Status guard: only `pending` can transition |
| 3 | Withdrawal within 24 hours | Timestamp check on recent payouts |
| 4 | Insufficient balance | Pre-debit balance check with error |
| 5 | Completed payout marked as failed | Status guard: only `initiated/processing` can fail |
| 6 | Negative balance after clawback | Allowed as debt; withdrawals blocked at 0 |
| 7 | Reconciliation before advance runs | No clawback needed; full amount credited |
| 8 | Zero-earning sale | 10% of 0 = 0; recorded but no money moves |
| 9 | Batch reconciliation partial failure | Per-sale error collection; successes still commit |
| 10 | Server crash mid-operation | DB transaction auto-rollback; no partial state |

---

## 12. Design Decisions & Trade-offs

### 12.1 Materialized Balance vs. Computed-on-Read

| Approach | Read Cost | Write Cost | Consistency Risk |
|----------|-----------|------------|-----------------|
| **Materialized (chosen)** | O(1) | O(1) per update | Drift possible |
| Computed from adjustments | O(n) | O(1) insert only | Always consistent |

**Decision:** Materialized. Payout dashboards check balance on every page load. O(1) reads are critical. The `payout_adjustments` audit trail exists as a fallback to reconstruct the true balance if drift is suspected.

### 12.2 Per-Sale vs. Aggregate Advance Tracking

**Decision:** Per-sale (`advance_transferred` flag + `advance_paid` amount on each sale).  
**Why:** During reconciliation, we need to know exactly how much advance was paid for *this specific sale* to calculate the correct remainder or clawback. Aggregate tracking would require proportional splitting, which is error-prone.

### 12.3 Transaction Boundaries

**Decision:** Each service method wraps its entire operation in a single database transaction.  
**Why:** Atomicity. If crediting the balance succeeds but the audit record fails, the system is inconsistent. The transaction ensures all-or-nothing.

### 12.4 Immutable Audit Trail

**Decision:** `payout_adjustments` is append-only. No UPDATE or DELETE ever.  
**Why:** Financial systems require full traceability. Every balance change has a traceable record. Enables dispute resolution, balance reconstruction, and forensic auditing.

### 12.5 SQLite for Demonstration

**Decision:** SQLite instead of PostgreSQL/MongoDB.  
**Why:** Reviewer can `npm install && npm start` with zero external dependencies.  
**Production migration:** Replace `src/config/database.js` with a PostgreSQL connection pool. All SQL is standard and portable. The layered architecture ensures no other file needs to change.

---

## 13. Production Considerations

If this were production, I would add:

| Concern | Solution |
|---------|----------|
| **Database** | PostgreSQL with connection pooling |
| **Concurrency** | Row-level locking (`SELECT ... FOR UPDATE`) |
| **Advance Job** | Cron scheduler or message queue (BullMQ) |
| **Authentication** | JWT-based auth middleware |
| **Rate Limiting** | express-rate-limit per IP/user |
| **Monitoring** | Structured logging (pino), APM |
| **Failed Payout Retry** | Exponential backoff with dead-letter queue |
| **Balance Reconciliation** | Periodic job: materialized vs. sum of adjustments |
| **API Versioning** | `/api/v1/` prefix for backward compatibility |
