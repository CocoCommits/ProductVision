# NEESH — Never Ending Earnest Savings Hustle

### A self-hosted family wealth tracker built to last 50 years.

> *"What is my family's complete financial picture — across every investment, every platform, every currency — right now?"*

NEESH answers that question. It's a personal wealth management application I built for my family — consolidating 11 asset classes, multiple brokers, two currencies, and up to 5 families into a single private dashboard. No cloud dependencies for data storage. No subscription fees. Just a Python app running on whatever hardware you have at home.

---

## What's Inside

| Section | |
|:--------|:--|
| [Product Vision](#product-vision) | Why this exists and what it solves |
| [What It Tracks](#what-it-tracks) | 11 asset classes, income, tax classification |
| [How It Works](#how-it-works) | Architecture and transaction flows |
| [AI-Powered Import](#ai-powered-import) | Statement parsing with Gemini |
| [Family Accounts](#family-accounts) | Multi-user, role-based access |
| [Roadmap](#roadmap) | Phase 1 (done) → Phase 2 & 3 (planned) |
| [Technical Decisions](#technical-decisions) | Key architecture choices and why |
| [Development Learnings](#development-learnings) | What I learned building this |

---

<a name="product-vision"></a>

## 🎯 Product Vision

Most Indian families have money scattered across a dozen platforms — mutual funds on Groww, stocks on Zerodha, FDs at three different banks, RSUs from work vesting in USD, a PF that nobody checks, and a home loan EMI that just... keeps going. Tax season arrives and everyone scrambles.

NEESH brings all of that into one place.

### Goals

| Goal | Target |
|------|--------|
| **Consolidation** | 11 asset classes, 3+ broker platforms, multi-currency |
| **Accuracy** | > 95% net worth match vs. manual calculation |
| **Timeliness** | < 1 hour price staleness during market hours |
| **Ease of Import** | < 5 minutes to upload and process a broker statement |
| **Family Coverage** | Up to 4 members per family, 5 families (15 users) |
| **Data Longevity** | Designed for 50–60 years of continuous use |

### Platform Support

Runs on anything that runs Python 3.11+:

- **Raspberry Pi 4** (4GB) — the original target. Always-on home server, ~₹5K.
- **BeagleBone Black** — similar ARM SBC, tested and working.
- **Any Linux box** — Ubuntu, Debian, Raspberry Pi OS. systemd service included.
- **Windows** — full support with batch scripts for dev/test workflows.

The entire stack — web server, background jobs, database — runs in a **single Python process**. No Docker. No Redis. No PostgreSQL daemon. Just SQLite in WAL mode and a FastAPI server.

### What's Live Today (Phase 1) — and What's Coming

```mermaid
flowchart TB
    subgraph phase1["✅ Phase 1 (Sprints 0–17 Done · Sprint 18 In Progress)"]
        direction LR
        A["📊 11 Asset Classes"] ~~~ B["💰 Income Tracking"] ~~~ C["🤖 AI Import"] ~~~ D["📈 Live Market Data"] ~~~ E["👨‍👩‍👧‍👦 Family Accounts"] ~~~ F["🔐 Admin & Auth"]
        G["🏛️ Corporate Actions"] ~~~ H["🌐 Remote Access"] ~~~ I["💾 Backup & Restore"]
    end

    subgraph phase2["📋 Phase 2 — PLANNED"]
        direction LR
        L["💸 Expense Tracking"] ~~~ M["🏠 Loan Simulator"] ~~~ N["🧾 Tax Simulator"]
    end

    subgraph phase3["🔮 Phase 3 — FUTURE"]
        direction LR
        J["🧠 AI Investment Insights"] ~~~ K["📱 WhatsApp & Email Alerts"]
    end

    phase1 --> phase2 --> phase3

    style phase1 fill:#065f46,color:#d1fae5
    style phase2 fill:#1e3a5f,color:#bfdbfe
    style phase3 fill:#4c1d95,color:#ddd6fe
```

---

<a name="what-it-tracks"></a>

## 📦 What It Tracks

### 11 Asset Classes, Grouped by Behavior

The system handles five fundamentally different types of financial instruments. Each group shares a transaction model but has unique tax rules, valuation logic, and lifecycle patterns.

```mermaid
graph LR
    subgraph A["Tradeable Securities"]
        A1["Direct Equity"]
        A2["Mutual Funds"]
        A3["ETFs"]
        A4["ESPP"]
    end

    subgraph B["Employee Compensation"]
        B1["RSUs"]
        B2["ESOPs"]
    end

    subgraph C["Fixed Income"]
        C1["Fixed Deposits"]
        C2["Gold Bonds (SGB)"]
    end

    subgraph D["Balance-Based"]
        D1["Bank Accounts"]
        D2["Provident Fund"]
    end

    subgraph E["Liabilities"]
        E1["Loans"]
    end

    style A fill:#3b82f6,color:#fff
    style B fill:#8b5cf6,color:#fff
    style C fill:#f59e0b,color:#fff
    style D fill:#10b981,color:#fff
    style E fill:#ef4444,color:#fff
```

| Group | Asset Classes | Valuation Model | Key Complexity |
|:------|:-------------|:----------------|:---------------|
| **Tradeable** | Stocks, MF, ETF, ESPP | Qty × Market Price | FIFO lot tracking, live prices, multi-exchange |
| **Employee Stock** | RSU, ESOP | Vest/Exercise events | Perquisite tax, FX conversion, auto-creates equity on vest |
| **Fixed Income** | FD, SGB | Principal + Accrued Interest | Maturity lifecycle, penalty on early exit |
| **Balance-Based** | Bank, PF | Running balance | Fund flow linking, contribution tracking |
| **Liability** | Loans | Outstanding principal | Negative asset, EMI tracking |

### Multi-Currency & Tax Awareness

Every holding carries its tax classification — computed automatically from asset class rules and holding period. Foreign investments (RSUs, ESPPs, US stocks) track both foreign currency values and INR equivalents using RBI reference rates.

| Asset Class | STCG Threshold | LTCG Rate | Special Rules |
|:------------|:--------------|:----------|:--------------|
| Equity / MF / ETF | 12 months | 12.5% (above ₹1.25L exempt) | FIFO cost basis |
| Debt MF | 24 months | Slab rate | No indexation post-2023 |
| Fixed Deposit | N/A | Slab rate | Interest = income, TDS applicable |
| SGB | 12 months | 12.5% | **Tax-free on 8-year maturity** |
| RSU / ESOP / ESPP | 12 months from vest | 12.5% | Perquisite tax at vest/exercise |
| Provident Fund | N/A | **Exempt** | EEE regime |

### Income Tracking

Income isn't just salary. NEESH tracks every money inflow:
- **Manual entries** — salary, bonuses, freelance, rental
- **Auto-generated income** — dividends from equity holdings, interest from FDs, SGB coupon payments
- **TDS tracking** — tax deducted at source on FDs, dividends
- **Financial year views** — income summaries aligned to India's April–March FY

When a dividend transaction is recorded on a stock holding, the system automatically creates an income entry. No double entry.

---

<a name="how-it-works"></a>

## ⚙️ How It Works

### The Core Principle: Transactions Are the Source of Truth

Every financial event is recorded as a transaction. Holdings are never stored independently — they're always **computed** from the transaction history. This means:

- Edit a past transaction → holding automatically recomputes
- Delete a transaction → holding adjusts
- No data can ever be "out of sync"

The `holdings_summary` table is a **cache** — it can be dropped and rebuilt from scratch at any time.

```mermaid
flowchart LR
    T["📝 Transactions\n(source of truth)"] --> R["⚙️ Recompute\nEngine"]
    R --> H["📊 Holdings Summary\n(derived cache)"]
    R --> X["📈 XIRR\nCalculation"]
    R --> TC["🧾 Tax\nClassification"]

    M["📡 Market Data\n(yfinance, AMFI, RBI)"] --> H

    H --> D["🖥️ Dashboard"]
    X --> D
    TC --> D
```

### System Architecture

```mermaid
flowchart TB
    subgraph Client["Browser (Any Device)"]
        UI["HTMX + Tailwind CSS + Chart.js\n(Server-rendered, no SPA)"]
    end

    subgraph Server["Single Python Process"]
        direction TB
        Routes["FastAPI Routes\n(Jinja2 Templates)"]
        Services["Service Layer\n(All Business Logic)"]
        Repos["Repositories\n(Database Access)"]
        Jobs["APScheduler\n(Background Jobs)"]
        AI["AI Module\n(Local Parser → Gemini → LiteLLM)"]
        Market["Market Data\n(yfinance, AMFI, RBI)"]
    end

    subgraph Storage["SQLite (WAL Mode)"]
        DB[("Single .db File\n~50-100MB over 20 years")]
    end

    subgraph External["External APIs (Outbound Only)"]
        Gemini["Google Gemini"]
        YF["yfinance"]
        AMFI_API["AMFI NAV"]
        RBI["RBI Rates"]
        WA["WhatsApp Cloud API\n(OTP primary)"]
        SMTP["Email SMTP\n(OTP fallback)"]
        Zerodha["Zerodha Kite"]
        GDrive["Google Drive\n(encrypted backup)"]
        NSE_CA["NSE CA API\n(corporate actions)"]
        ZROK["ZROK + Cloudflare Worker\n(remote access tunnel)"]
    end

    UI <--> Routes
    Routes --> Services
    Services --> Repos
    Repos --> DB
    Jobs --> Services
    AI --> Gemini
    Market --> YF
    Market --> AMFI_API
    Market --> RBI
    Services --> AI
    Services --> Market

    style Server fill:#1e293b,color:#e2e8f0
    style Storage fill:#064e3b,color:#d1fae5
```

**Layered architecture** — every layer has one job:

| Layer | Responsibility | Rule |
|:------|:--------------|:-----|
| **Routes** | Accept requests, return HTML | No business logic. Ever. |
| **Services** | All computation, validation, orchestration | The only place logic lives |
| **Repositories** | Database reads and writes | No decisions, no calculations |
| **Templates** | Display data | No computation |

This strict separation means Phase 2/3 features can compose with existing services without refactoring.

---

> 👇 *Click any section below to expand the detailed flow diagrams.*

<details>
<summary><h3>Transaction Flow — Tradeable Securities (Stocks, MF, ETF)</h3></summary>

The most common flow. Covers buying, selling, corporate actions, and automatic holding recomputation.

```mermaid
flowchart TB
    Input["User submits transaction\n(symbol, qty, price, date, type)"]

    Input --> Route["POST /transactions/add"]
    Route --> Validate["Validation Layer"]

    Validate --> TypeCheck{"Transaction\nType?"}

    TypeCheck -->|"buy / opening_balance"| Create
    TypeCheck -->|"sell"| SellCheck{"Sufficient\nquantity?"}
    TypeCheck -->|"dividend"| DivFlow["Record dividend\n+ auto-create income entry"]
    TypeCheck -->|"bonus / split"| CorpAction["Adjust quantity\n(zero cost basis)"]

    SellCheck -->|"No"| Error["❌ Error:\nCannot sell more\nthan you hold"]
    SellCheck -->|"Yes"| Create

    DivFlow --> Create
    CorpAction --> Create

    Create["Save Transaction to DB"]

    Create --> Recompute["Recompute Holding"]

    subgraph Recompute["⚙️ Holding Recomputation"]
        direction TB
        Fetch["Fetch ALL transactions\nfor this symbol + platform"]
        Fetch --> CalcQty["total_qty = bought + bonus − sold"]
        CalcQty --> CalcAvg["avg_cost = total_invested / total_qty"]
        CalcAvg --> FetchPrice["Fetch current market price"]
        FetchPrice --> CalcPL["P&L = current_value − invested"]
        CalcPL --> TaxClass["Classify: STCG / LTCG\nbased on holding days"]
        TaxClass --> XIRR["Compute XIRR\nfrom cash flow history"]
        XIRR --> Upsert["Upsert holdings_summary"]
    end

    Upsert --> Done["✅ Redirect to holdings page"]
```

Mutual funds use AMFI scheme codes for NAV lookup. ETFs and stocks use yfinance. The recomputation engine is the same for all three — only the price source differs.

</details>

---

<details>
<summary><h3>RSU Vest → Auto-Creates Direct Equity</h3></summary>

When RSU shares vest, the system automatically creates a corresponding `direct_equity` BUY transaction. This avoids double-entry and ensures vested shares appear in the equity portfolio with the correct cost basis (fair market value at vest, converted to INR).

```mermaid
sequenceDiagram
    participant User
    participant System as Transaction Service
    participant DB as Database

    User->>System: Record RSU Vest\n(100 shares, FMV $150, FX ₹83)

    System->>DB: Save RSU vest transaction
    System->>System: Recompute RSU holding\n(track unvested grants)

    Note over System: Auto-creation trigger

    System->>DB: Create direct_equity BUY\n(100 shares, cost = $150 × ₹83)
    System->>System: Recompute equity holding\n(vested shares now tradeable)

    System-->>User: ✅ Both holdings updated

    Note over DB: RSU holding: tracks grant/vest history\nEquity holding: tracks market value + STCG/LTCG
```

**Tax implications captured:**
- **At vest:** Perquisite tax = (FMV − $0) × quantity × FX rate → taxed as salary income
- **At sale:** Capital gains computed from vest-date FMV (not $0), with holding period starting from vest date

</details>

---

<details>
<summary><h3>Fixed Deposit Lifecycle</h3></summary>

FDs follow a state machine — they're opened, accrue interest daily, and close via maturity or premature withdrawal.

```mermaid
stateDiagram-v2
    [*] --> Active: opening_balance\n(principal, rate, maturity date)

    Active --> Active: interest\n(periodic payout for\nnon-cumulative FDs)
    Active --> Closed: maturity\n(full principal + interest)
    Active --> Closed: premature_withdrawal\n(principal − penalty)

    Closed --> [*]: Holdings row deleted\nTransactions preserved for history

    note right of Active
        Current value computed daily:
        P × (1 + r × days/365)
    end note
```

On maturity or premature withdrawal, the holdings row is **deleted** (the FD no longer exists as a holding), but all transactions remain for historical record and tax computation.

</details>

---

<details>
<summary><h3>Bank Account Fund Flow</h3></summary>

Bank accounts track running balances and link to investment transactions — when you buy stocks, the money has to come from somewhere.

```mermaid
flowchart LR
    subgraph Investments
        Buy["Buy 100 HDFC\n₹1,50,000"]
        Sell["Sell 50 TCS\n₹2,00,000"]
        FDMat["FD Maturity\n₹1,07,000"]
    end

    subgraph Bank["Bank Account Balance"]
        direction TB
        Out["transfer_out\n−₹1,50,000"]
        In1["transfer_in\n+₹2,00,000"]
        In2["transfer_in\n+₹1,07,000"]
        Int["interest\n+₹850"]
    end

    Buy -->|"fund source"| Out
    Sell -->|"fund destination"| In1
    FDMat -->|"fund destination"| In2
```

Balance computation:
- If a `balance_update` transaction exists → use it directly (from bank statement)
- Otherwise → accumulate: `opening_balance + transfers_in − transfers_out + interest`

</details>

---

<details>
<summary><h3>Holding Recomputation Engine</h3></summary>

This is the heart of the system. Runs after every transaction create/edit/delete.

```mermaid
flowchart TB
    Start(["recompute_holding()"])
    Fetch["Fetch all transactions\nfor user + asset_class + symbol + platform"]

    NoTxns{"Any\ntransactions?"}
    Delete["Delete holdings_summary row"]

    AssetType{"Asset\ntype?"}

    BankCalc["Compute running balance\nfrom transfers + interest"]
    PFCalc["Sum contributions + interest\nor use latest balance_update"]
    FDCalc["Principal + accrued interest\nat current date"]

    StandardCalc["Standard: qty × price model"]
    Qty["total_qty = bought + bonus − sold"]
    ZeroCheck{"qty = 0?"}
    Avg["avg_cost = total_invested / qty"]
    Tax["Tax classify from holding days"]
    XIRR["XIRR from cash flows"]
    Upsert["Upsert holdings_summary"]

    Start --> Fetch --> NoTxns
    NoTxns -->|"None"| Delete
    NoTxns -->|"Has txns"| AssetType

    AssetType -->|"bank_account"| BankCalc --> Upsert
    AssetType -->|"provident_fund"| PFCalc --> Upsert
    AssetType -->|"fixed_deposit / sgb"| FDCalc --> Upsert
    AssetType -->|"equity / mf / etf\nrsu / esop / espp"| StandardCalc

    StandardCalc --> Qty --> ZeroCheck
    ZeroCheck -->|"Yes"| Delete
    ZeroCheck -->|"No"| Avg --> Tax --> XIRR --> Upsert
```

XIRR is computed using `pyxirr` (Rust-based, 100× faster than scipy). Returns annualized return accounting for irregular cash flow timing — critical for SIP investments where money goes in monthly.

</details>

---

<details>
<summary><h3>Corporate Actions — Detection, Approval, and Application</h3></summary>

The NSE CA API is queried daily at 6:00 AM IST for five action types: **bonus**, **split**, **demerger**, **merger**, and **symbol change**. Detected actions are matched to user holdings and surfaced for explicit user approval — nothing is auto-applied.

```mermaid
flowchart TB
    NSESync["NSE CA API\n(daily sync 6:00 AM IST)"]
    Fetch["Fetch: bonus · split\ndemerger · merger · symbol change"]
    SymMatch{"Matches any\nuser holding?"}
    NoMatch["Skip — no users affected"]
    DupeCheck{"Already\napplied?"}
    NewUA["Create pending UserAction\n(one per user per CA)"]
    NavBadge["🔔 Nav badge updates\n(pending count)"]
    ActionsPage["User opens Actions & Notifications\n(sorted: symbol → ex_date, oldest first)"]
    OrderCheck{"Earlier unapplied CA\nfor same symbol?"}
    Blocked["⛔ Apply button disabled\nTooltip shows which CA must go first"]
    ApplyBtn["User clicks Apply"]
    ServerGuard["Server-side ordering check\n(guard cannot be bypassed via UI)"]
    ApplyOp["Apply CA:\nUpdate transaction quantities\nInherit cost basis from parent"]
    SoftMark["Soft-mark consumed transactions\n(not deleted — audit trail preserved)"]
    Recompute["Recompute holdings_summary"]
    Done["✅ Holding updated\nQty · avg price · XIRR recalculated"]
    Dismiss["User clicks Dismiss\n(CA rejected — no change to holdings)"]

    NSESync --> Fetch --> SymMatch
    SymMatch -->|"No match"| NoMatch
    SymMatch -->|"Matched"| DupeCheck
    DupeCheck -->|"Already applied"| NoMatch
    DupeCheck -->|"New"| NewUA --> NavBadge --> ActionsPage
    ActionsPage --> OrderCheck
    OrderCheck -->|"Earlier pending"| Blocked
    OrderCheck -->|"Clear"| ApplyBtn --> ServerGuard --> ApplyOp --> SoftMark --> Recompute --> Done
    ActionsPage --> Dismiss
```

| CA Type | Effect on Holdings | Zerodha Tradebook Entry |
|:--------|:------------------|:------------------------|
| **Bonus** | New shares at zero cost; avg price dilutes | Zero-price buy entry |
| **Split** | Qty × ratio; face value divides | No separate buy — quantities updated |
| **Symbol Change** | All transactions migrated to new symbol | Old symbol retired in DB |
| **Demerger** | Child company shares created; parent unchanged | Zero-price buy for child symbol |
| **Merger** | Target shares swapped for acquirer shares | Target transactions marked consumed |

**Import-time CA detection** — Zerodha tradebook already contains CA-derived entries (zero-cost demerger allocations, post-split sell volumes that exceed pre-split holdings). The import pipeline detects and handles these before committing:

```mermaid
flowchart TB
    Upload["Upload tradebook\n(CSV / Excel)"]
    Scan["Scan for CA indicators:\nzero-cost buys · sells exceeding holdings"]
    HybridSearch{"Parent symbol found\nin batch or existing DB?"}
    Transform["Inherit cost basis from parent\nCreate CA-correct transactions"]
    SellExceeds["Sell qty exceeds available holding\n(CA not yet applied)"]
    ImportWarn["Create import_warning UserAction\n(visible on Actions page with badge)"]
    ReviewPage["Import review page\n(CA detections and warnings listed)"]
    Commit["User confirms → transactions saved"]

    Upload --> Scan --> HybridSearch
    HybridSearch -->|"Found"| Transform --> ReviewPage
    HybridSearch -->|"Not found"| SellExceeds --> ImportWarn --> ReviewPage
    ReviewPage --> Commit
```

**CA math rules:**
- **Ex-date eligibility**: buy date must be strictly `<` ex_date — buying *on* the ex_date means you paid the adjusted price but receive no extra shares
- **Fractional shares**: quantities are always floored (`int(qty × ratio)`); the decimal remainder is paid as cash by the broker and does **not** carry forward to future CA calculations
- **Ordering**: CAs for the same symbol must be applied chronologically; the server enforces this regardless of UI state

</details>

---

<details>
<summary><h3>Income Management — Manual, Auto-Generated, and FY Views</h3></summary>

Income flows from three sources: manual entries, automatic creation triggered by transactions, and a background job for recurring schedules.

```mermaid
flowchart TB
    subgraph Manual["Manual Income Entry"]
        Sal["Salary · Bonus\n(one-time or recurring)"]
        Free["Freelance · Rental · Other\n(any source with optional TDS)"]
    end

    subgraph AutoCreate["Auto-Created from Transactions"]
        DivTxn["Dividend transaction\non direct_equity holding"]
        FDInt["FD interest payout\n(non-cumulative FDs)"]
        SGBCoup["SGB coupon payment\n(2.5% p.a., semi-annual)"]
        RSUPerq["RSU vest event\nPerquisite = FMV × qty × FX rate"]
    end

    subgraph RecurringJob["Recurring (Daily Background Job)"]
        RecSal["Scheduled salary entry\n(generates on due date)"]
        RecIntAccr["FD interest accrual\n(cumulative FDs — computed daily)"]
    end

    IncTable[("income table\nsource · type · amount\ntds_amount · financial_year")]

    subgraph FYDashboard["Financial Year View (Apr–Mar)"]
        FYBreak["Income by category\nsalary · dividends · interest · other"]
        TDSSumm["TDS credited by deductors\nvs. gross income"]
        NetInc["Net income = gross − TDS"]
    end

    Manual --> IncTable
    AutoCreate -->|"auto-creates entry\n(no double-entry)"| IncTable
    RecurringJob --> IncTable
    IncTable --> FYDashboard
```

**No double-entry:** when a dividend transaction is recorded on a holding, the income entry is created automatically. The user never enters dividend income separately.

| Income Source | Auto-Created? | TDS Applicable | Notes |
|:-------------|:-------------|:--------------|:------|
| Salary / Bonus | No (manual or recurring) | Yes | Employer TDS stored per entry |
| FD Interest | Yes (from FD transaction) | Yes (10% above ₹40K) | Auto-recorded on payout or accrual |
| Dividends | Yes (from equity transaction) | Yes (10% above ₹5K) | Single entry auto-created |
| SGB Coupon | Yes (background job) | No | Semi-annual, auto-scheduled |
| RSU Perquisite | Yes (on vest event) | Yes (employer TDS) | FX-converted to INR at vest date |
| Freelance / Rental | No (manual) | Depends | Optional TDS field |

**TDS tracking** is per-entry — each income record stores gross amount and TDS separately. The FY dashboard aggregates both so users can reconcile against Form 26AS: how much was deducted by employers/banks vs. how much is still owed.

</details>

---

<details>
<summary><h3>Admin Dashboard</h3></summary>

The first registered user is automatically assigned the Admin role. Admins have system-wide visibility across all users and families, with a dedicated panel for operations, audit, and data integrity.

```mermaid
flowchart TB
    AdminPanel["🔑 Admin Panel\n(system-wide access, bypasses family scope)"]

    subgraph UserAdmin["User & Family Management"]
        AllUsers["View all registered users\nacross all families"]
        RoleMgmt["Assign / change roles\nadmin · owner · editor · viewer"]
        WriteToggle["Toggle write access\n(per user or system-wide lock)"]
        FamilyMgmt["View and manage all families\n(members, roles, combined data)"]
    end

    subgraph AuditOps["Audit & Monitoring"]
        AuditLog["Audit log viewer\nall write operations with user + timestamp"]
        SysHealth["System health check\nDB status · job schedules · API connectivity"]
    end

    subgraph BackupOps["Backup Operations"]
        BackupNow["On-demand backup\n(local + Google Drive)"]
        BackupHistory["Backup history\ndate · size · SHA-256 checksum"]
    end

    subgraph OnDemandSync["On-Demand Sync"]
        ForcePrice["Force price refresh\n(bypass 3:45 PM daily schedule)"]
        ForceCA["Force CA sync\n(bypass 6:00 AM daily schedule)"]
    end

    subgraph DataIntegrity["Holdings Baseline Check (Sprint 17)"]
        ZerodhaUpload["Upload Zerodha portfolio CSV\n(fresh export from Zerodha)"]
        DiffView["Diff view:\nNEESH qty vs Zerodha qty per symbol"]
        FixBtn["One-click fix\napply discrepancies from Zerodha"]
    end

    AdminPanel --> UserAdmin
    AdminPanel --> AuditOps
    AdminPanel --> BackupOps
    AdminPanel --> OnDemandSync
    AdminPanel --> DataIntegrity
    ZerodhaUpload --> DiffView --> FixBtn
```

**Write access enforcement** (Sprint 9): all write operations (transaction create/edit/delete, income changes, CA applications) check a system-wide write-access flag before executing. The Admin can lock the entire system (e.g. before a backup/restore) or lock a specific user's account. All write attempts while locked are rejected at the service layer.

**Holdings Baseline Check** (Sprint 17) is the end-to-end integrity check — it catches discrepancies that could arise from any combination of missed corporate actions, import errors, or manual entry mistakes. Upload today's Zerodha portfolio CSV, NEESH shows a side-by-side quantity comparison per symbol, and offers a one-click fix for any gaps.

</details>

---

<a name="ai-powered-import"></a>

## 🤖 AI-Powered Import

Manually entering every transaction is a non-starter when you have years of investment history. NEESH uses a **three-tier parsing pipeline** with a LiteLLM fallback to extract transactions from broker statements.

```mermaid
flowchart LR
    Upload["📄 Upload\nCSV / Excel / PDF / Image"]

    Upload --> T1

    subgraph Pipeline["Parsing Pipeline"]
        direction LR
        T1["🔧 Tier 1\nLocal Regex Parser\n(known formats)"]
        T2["⚡ Tier 2\nGemini Flash\n(extraction)"]
        T3["🧠 Tier 3\nGemini Pro\n(verification)"]
        T4["🔁 Fallback\nLiteLLM\n(OpenAI-compatible)"]
    end

    T1 -->|"Unknown format"| T2
    T2 -->|"Extracted JSON"| T3
    T2 -->|"Gemini unavailable"| T4
    T1 -->|"Parsed ✅"| Review
    T3 -->|"Verified ✅"| Review
    T4 -->|"Extracted JSON"| Review

    Review["👤 User Review\n& Confirm"]

    style T1 fill:#10b981,color:#fff
    style T2 fill:#3b82f6,color:#fff
    style T3 fill:#8b5cf6,color:#fff
    style T4 fill:#f59e0b,color:#fff
```

| Tier | Engine | Purpose | Speed |
|:-----|:-------|:--------|:------|
| **Tier 1** | Regex parser | Known formats (Zerodha, CAMS, NSDL) — zero API cost | Instant |
| **Tier 2** | Gemini Flash | Extract structured data from unknown formats | ~2-3 sec |
| **Tier 3** | Gemini Pro | Re-verify extracted data for accuracy | ~5 sec |
| **Fallback** | LiteLLM | OpenAI-compatible endpoint when Gemini is unavailable | ~2-5 sec |

**Why multiple tiers?** The local parser handles known formats at zero cost. Flash is fast and great at structured extraction from unknown formats. Pro catches reasoning errors — wrong date formats, misidentified transaction types, currency confusion (~15% of extraction errors). LiteLLM is the fallback when Gemini is unavailable or rate-limited (e.g. using an internal corporate gateway).

Excel and CSV files are parsed locally first (openpyxl/csv) and sent as text to Gemini — avoiding File API quotas. Images (JPG/PNG) are sent directly to Gemini Vision (Sprint 18).

---

<a name="family-accounts"></a>

## 👨‍👩‍👧‍👦 Family Accounts

NEESH is designed for families, not just individuals. A family of four can each have their own portfolio while seeing a combined family dashboard.

```mermaid
flowchart TB
    subgraph Family["Family Group (max 4 members)"]
        Owner["👑 Owner\nFull control + member management"]
        Editor["✏️ Editor\nCan add/edit for all members"]
        Viewer["👀 Viewer\nRead-only access"]
        Viewer2["👀 Viewer\nRead-only access"]
    end

    subgraph Dashboards
        Individual["Individual Dashboard\nEach member's own portfolio"]
        Combined["Combined Dashboard\nFamily net worth + allocation"]
    end

    Owner --> Individual
    Owner --> Combined
    Editor --> Individual
    Editor --> Combined
    Viewer --> Individual
    Viewer --> Combined

    Admin["🔑 System Admin\nCan see everything\nManage all users & families"] -.-> Family
```

### Three-Tier Permission Model

| Role | Own Data | Family Members' Data | Family Settings |
|:-----|:---------|:--------------------|:---------------|
| **Owner** | Full CRUD | Full CRUD | Add/remove members, manage roles |
| **Editor** | Full CRUD | Full CRUD | View only |
| **Viewer** | View only | View only | View only |

The **Admin** role sits above families — system-wide access for user management, audit logs, and troubleshooting. First registered user is auto-assigned Admin.

---

<a name="roadmap"></a>

## 🗺️ Roadmap

### Phase 1  Done ·

| Sprint | Deliverable |
|:-------|:-----------|
| Sprint 0 | Project setup, auth base, models, health check |
| Sprint 1 | Registration (phone/name/DOB), WhatsApp OTP, JWT |
| Sprint 2 | Transaction CRUD, holdings computation, tax classification |
| Sprint 3 | Market data pipeline (yfinance, AMFI NAV, RBI rates) |
| Sprint 4 | Dashboard with Chart.js, net worth, XIRR, snapshots |
| Sprint 5 | AI Import Pipeline (local parser + Gemini + LiteLLM) |
| Sprint 6 | Income Tracking (salary, dividends, auto-interest, TDS, recurring) |
| Sprint 7 | Family Accounts (three-tier permissions, combined dashboard) |
| Sprint 8 | Settings page, Zerodha Kite Connect OAuth, CSV fallback |
| Sprint 9 | Admin Panel, audit logging, write access enforcement |
| Sprint 10 | Remote Access (ZROK static URL + Cloudflare Worker) |
| Sprint 11 | Email OTP Authentication (WhatsApp → Email → Console priority) |
| Sprint 12 | Corporate Actions Auto-Detection (NSE daily sync, bonus/split) |
| Sprint 13 | Demergers & Mergers (cost-basis splitting, merger swap ratios) |
| Sprint 14 | Symbol Fetch Tracking & CA cleanup (daily sync optimization) |
| Sprint 15 | Secure Backup (local + Google Drive + on-demand from admin) |
| Sprint 16 | Historical CA Reconciliation (full NSE history, ⚠️ indicators) |
| Sprint 17 | Holdings Baseline Check (Zerodha CSV vs NEESH, one-click fix) |
| Sprint 18 | AI-Powered Holdings & Salary Import — **In Progress** |

### Phase 2 — Planned

Three independent modules that extend Phase 1 without restructuring it.

> 👇 *Click to expand each module's details.*

<details>
<summary><b>Phase 2A — Expense Tracking</b></summary>

Syncs expense data from a user-maintained Google Sheet (no OAuth — just a public CSV export URL). Weekly background job pulls new rows, deduplicates, and stores in SQLite.

**Dashboard gets:** monthly expense summary, category breakdown, savings rate (income − expenses).

**Key design:** Expenses and income remain separate tables. Net savings is computed, never stored. Family-linked sheets aggregate into the family dashboard.

</details>

<details>
<summary><b>Phase 2B — Loan Simulation Engine</b></summary>

The `loans` table exists in Phase 1 but the UI is deferred. Phase 2 implements the full loan module **plus** a simulation engine.

Users can model:
- Extra EMI payments (which months)
- EMI step-up strategies (annual % increase)
- Lump-sum prepayments (reduce tenure or reduce EMI)

Output: original vs. optimized tenure, total interest saved, full month-by-month amortization schedule. Pure computation — no external APIs.

**Architecture note:** Phase 1 stores prepayment/rate history as JSON columns. Phase 2 migrates these to proper normalized tables via Alembic — planned from day one.

</details>

<details>
<summary><b>Phase 2C — Tax Simulation</b></summary>

Extends Phase 1's display-only tax classification into interactive "what-if" scenarios:

- *"If I sell TCS today, what's my tax?"*
- *"Which losing investments should I sell to offset gains?"* (tax harvesting)
- *"What's my total capital gains tax this FY?"*

**The hook was built in Phase 1:** every asset class handler accepts an `as_of_date` parameter. Phase 1 defaults it to today. Phase 2 passes future dates. That's the entire integration point — no refactoring needed.

For RSU/ESOP/ESPP: DTAA computation showing tax liability under both Indian and US jurisdictions.

</details>

### Phase 3 — Future

<details>
<summary><b>AI Investment Planning Assistant</b></summary>

The most ambitious module. Uses Google Gemini Pro to analyze holdings against user-defined investment strategies.

```mermaid
flowchart LR
    subgraph Input
        Strategy["User's Strategy\n(rationale, objective,\ntarget price, stop loss)"]
        Market["Market Data\n(RSI, P/E, moving averages,\nvolume trends)"]
        Holdings["Current Holdings\n(cost basis, P&L,\nholding period)"]
    end

    subgraph Analysis["Gemini Pro Analysis\n(scheduled nightly)"]
        Prompt["Context-rich Prompt\n(strategy + market + holdings)"]
        Result["AI Recommendation\n(buy/hold/sell +\nconfidence + reasoning)"]
    end

    subgraph Output
        Deviation{"Strategy\nDeviation?"}
        Alert["📱 WhatsApp Alert\n'AI recommends SELL,\nyour strategy is HOLD'"]
        Report["📧 Email Report\nFull analysis with\ntechnical indicators"]
        Dashboard2["🖥️ AI Dashboard\nAll analyses,\ndeviation trends"]
    end

    Strategy --> Prompt
    Market --> Prompt
    Holdings --> Prompt
    Prompt --> Result
    Result --> Deviation
    Deviation -->|"High/Critical"| Alert
    Deviation -->|"Any"| Report
    Deviation -->|"Any"| Dashboard2
```

**Deviation detection:** If you said "long-term hold" but the AI recommends "sell" → HIGH deviation → WhatsApp alert. If you said "short-term trade" but the AI says "hold" → MEDIUM deviation → email report.

**Scale consideration:** 15 users × ~20 holdings = ~300 analyses. Gemini Pro at 2 RPM = ~2.5 hours. Scheduled at 2 AM.

Reuses existing infrastructure:
- Gemini client (same module, different prompts)
- WhatsApp Cloud API (same integration, strategy deviation alerts instead of OTP)
- yfinance (extended with `get_technical_indicators()`)

</details>

---

<a name="technical-decisions"></a>

## 🏗️ Technical Decisions

> 👇 *Click to expand the reasoning behind each decision.*

<details>
<summary><b>Why SQLite for a 50-year application?</b></summary>

SQLite is the most deployed database in the world. It's not going away. A single `.db` file is trivially backed up, moved, and restored. Over 20 years, our data might reach ~100MB — SQLite supports up to 281 TB.

WAL (Write-Ahead Logging) mode gives concurrent reads while writes happen one at a time. With 2-3 simultaneous users and infrequent writes, this is never a bottleneck.

Weekly backups with SHA-256 checksums. Schema evolution through Alembic migrations — versioned, auditable, reversible.

</details>

<details>
<summary><b>Why server-rendered HTML (HTMX) instead of React/Vue?</b></summary>

This runs on a Raspberry Pi. No room for a Node.js build pipeline, 200MB of `node_modules`, or a separate frontend server.

HTMX gives us dynamic interactions (partial page updates, inline editing, async form submissions) with zero JavaScript framework overhead. Tailwind CSS handles styling. Chart.js handles charts. The browser gets plain HTML — fast to render, fast to load.

</details>

<details>
<summary><b>Why single process (no Celery, no Redis)?</b></summary>

Seven background jobs cover all scheduled work:

| Job | Schedule | What It Does |
|:----|:---------|:------------|
| Price refresh | Daily 3:45 PM IST | Update all held instruments from yfinance/AMFI |
| Corporate actions sync | Daily 6:00 AM IST | Fetch NSE CA data, match to user holdings |
| Recurring income | Daily | Generate scheduled salary/dividend/interest entries |
| Recurring transactions | Daily | Generate scheduled SIP/RD/PPF transactions |
| Symbol cleanup | Weekly | Mark closed positions inactive, prune stale CA rows |
| Local backup | Weekly | Compressed SQLite snapshot with retention policy |
| Google Drive backup | Monthly | AES-256 encrypted backup to Google Drive |

APScheduler's `AsyncIOScheduler` runs inside the same event loop as FastAPI. Jobs are async functions — they don't block web requests. If the process crashes, systemd restarts it. No message broker, no worker process, no extra 200-300MB of RAM.

</details>

<details>
<summary><b>Why FIFO for cost basis?</b></summary>

Indian tax law uses FIFO (First In, First Out) for equity capital gains. When you sell shares, the oldest lots are sold first. This determines whether a sale is STCG or LTCG.

The system stores every buy transaction as a separate lot with its own date and price. On sell, it matches against the oldest lots first, computing gains per-lot with the correct holding period.

</details>

<details>
<summary><b>Why three parsing tiers (local + Gemini Flash + Gemini Pro + LiteLLM fallback)?</b></summary>

Extraction is a different task than verification, and known formats need neither. Flash is fast (15 RPM free tier) and great at pulling structured data from messy text. Pro is slow (2 RPM) but catches reasoning errors.

In testing, the Pro verification step flagged ~15% of extraction errors — wrong transaction types, inverted buy/sell amounts, misidentified currencies. The two-model pipeline costs nothing (free tier) and significantly improves accuracy.

LiteLLM is the fallback when Gemini is unavailable or rate-limited. It accepts any OpenAI-compatible endpoint — including a corporate internal gateway (used during development for this project). Services call `get_ai_provider(type)` and never import provider code directly, so swapping the backend requires only a config change.

</details>

<details>
<summary><b>Cross-phase architectural invariants</b></summary>

Eight rules that every phase must follow:

1. **Transactions are the source of truth.** Holdings are always derivable from transactions.
2. **Every asset class handler implements the full interface**, including `as_of_date` for future tax simulation.
3. **Tax rules are configuration, not code.** Rates are class-level constants, not buried in if/else.
4. **Business logic lives only in the service layer.** Routes don't compute. Repositories don't decide.
5. **External clients have no business logic.** Gemini, WhatsApp Cloud, yfinance are generic wrappers.
6. **All schema changes go through Alembic.** No manual `ALTER TABLE`.
7. **JSON columns are for display-only data.** Anything queryable gets a proper column.
8. **`holdings_summary` is a rebuildable cache.** Drop it, recompute from transactions, lose nothing.

These constraints exist because Phase 2 and 3 need to compose with Phase 1 services. If business logic leaks into routes or templates, new features can't reuse it.

</details>

<details>
<summary><b>Why must corporate actions be user-approved, never auto-applied?</b></summary>

The system auto-detects bonus shares, splits, symbol changes, demergers, and mergers via the NSE CA API (daily sync at 6:00 AM IST). Detected actions are flagged as pending — users review and approve before application. No silent mutation of holdings.

Consumed transactions are soft-marked (`consumed_by_demerger`, `consumed_by_merger`) rather than deleted, so the full audit trail is preserved. This matters for a 50-year application where tracing a cost-basis decision back years later must be possible.

</details>

<details>
<summary><b>How does historical CA reconciliation work?</b></summary>

When a user imports 5 years of Zerodha tradebook, holdings are silently wrong until all historical corporate actions are also applied.

```mermaid
flowchart TD
    Import["Import Tradebook\n(5 years of Zerodha CSV)"]
    Holdings["Holdings Computed\n(raw — possibly wrong)"]
    Fetch["Full CA History Fetch\nNSE API from first_entry_date → today"]
    Compare["Compare expected vs actual quantities\nper symbol"]
    Warn["⚠️ Discrepancy indicators\non Holdings page"]
    Apply["User reviews & one-click applies\neach missing CA"]

    Import --> Holdings --> Fetch --> Compare --> Warn --> Apply
```

`symbol_fetch_tracking.first_entry_date` defines the start of the CA fetch range per symbol. The Holdings Baseline Check (Sprint 17) provides a further manual catch-all by comparing NEESH quantities against a Zerodha CSV export.

</details>

<details>
<summary><b>How does remote access work without port forwarding?</b></summary>

ZROK provides a static share URL that tunnels to `localhost:8000` on the Pi. When the Pi restarts, a Cloudflare Worker is notified with the new tunnel URL and updates its KV store. Family members bookmark `neesh.pages.dev` — that page fetches the current URL from KV and redirects. The stable landing URL never changes regardless of Pi restarts.

```mermaid
flowchart LR
    User["👨‍👩‍👧 Family\n(bookmark neesh.pages.dev)"]
    Pages["Cloudflare Pages\n(stable URL)"]
    Worker["Cloudflare Worker + KV\n(stores current ZROK URL)"]
    Tunnel["ZROK Tunnel\n(static share URL)"]
    Pi["Raspberry Pi\nlocalhost:8000"]

    User --> Pages --> Worker --> Tunnel --> Pi
```

No port forwarding. No static IP. ZROK and the Cloudflare Worker are both free tier.

</details>

---

## 📊 Data Model Overview

The database has 15+ tables, but the core data flow is simple:

```mermaid
erDiagram
    USERS ||--o{ TRANSACTIONS : "owns"
    USERS ||--o{ INCOME : "earns"
    USERS }o--o{ FAMILIES : "belongs to"

    TRANSACTIONS ||--|| HOLDINGS_SUMMARY : "derives"

    TRANSACTIONS {
        string asset_class
        string transaction_type
        decimal quantity
        decimal price
        date transaction_date
        string symbol
        string platform
        json metadata
    }

    HOLDINGS_SUMMARY {
        decimal total_quantity
        decimal average_cost
        decimal current_price
        decimal current_value
        decimal total_pnl
        decimal xirr
        string tax_classification
    }

    INCOME {
        string source
        string income_type
        decimal amount
        decimal tds_amount
        string financial_year
    }

    FAMILIES {
        string name
        int owner_id
    }
```

**Key design choices:**
- `metadata` JSON column stores asset-specific fields (FD interest rate, RSU grant date, loan EMI details) — keeps the transaction table universal without 50 nullable columns
- `holdings_summary` is denormalized for dashboard speed — but fully rebuildable
- Income entries can be auto-created from dividend/interest transactions — single source, no duplication

---

## 📐 Asset Class Handler Pattern

Every asset class implements a common interface. This is what makes the system extensible — adding a new asset class means implementing one handler, not touching the core engine.

```mermaid
classDiagram
    class AssetClassHandler {
        <<interface>>
        +get_tax_info(holding_days, as_of_date)
        +get_value_after_tax(value, cost, days)
        +get_display_fields()
        +validate_transaction(data)
    }

    class DirectEquityHandler {
        STCG_THRESHOLD = 365
        STCG_RATE = 0.20
        LTCG_RATE = 0.125
        LTCG_EXEMPTION = 125000
    }

    class MutualFundHandler {
        STCG_THRESHOLD = 365
        +equity vs debt sub-classification
    }

    class FixedDepositHandler {
        +accrued_interest_calc()
        +maturity_check()
    }

    class SGBHandler {
        MATURITY_YEARS = 8
        +maturity = tax exempt
    }

    class RSUHandler {
        +perquisite_calculation()
        +auto_create_equity()
    }

    AssetClassHandler <|.. DirectEquityHandler
    AssetClassHandler <|.. MutualFundHandler
    AssetClassHandler <|.. FixedDepositHandler
    AssetClassHandler <|.. SGBHandler
    AssetClassHandler <|.. RSUHandler
```

Tax rates and thresholds are **class-level constants**, not embedded in logic. When the Indian budget changes LTCG rates, it's a one-line config update per handler — not a code refactor.

The `as_of_date` parameter on `get_tax_info()` is Phase 1 code that exists solely for Phase 2's tax simulation. Today it defaults to `date.today()`. Phase 2 will pass future dates to compute hypothetical tax scenarios.

---

<a name="development-learnings"></a>

## 📖 Development Learnings

Building a 50-year financial application as a non-developer using AI tools taught me as much about planning and communication as it did about architecture and testing.

The full story is documented separately: **[Development Learnings — Building a Personal Wealth Manager with AI](neesh_development_learnings.md)**

Highlights:

- **Planning before prompting** — the four-step process (brainstorm → prioritise → categorise → user journeys) that turned a vague idea into a buildable spec
- **Transactions as source of truth** eliminates an entire class of data consistency bugs across 11 asset classes
- **Phase-aware architecture** — embedding hooks for Phase 2/3 (like `as_of_date` on every tax handler) without building the features
- **Three-tier AI parsing** (local regex → Gemini Flash → Gemini Pro) — same "right model for the right task" principle applied at the product level
- **The feedback discipline** — structured, specific, small-batch feedback is more effective than lengthy bug lists
- **Context window as a design constraint** — documents as source of truth, not conversations
- **Test isolation** with separate databases, ports, and cleanup scripts prevented hours of debugging
- **Git after every sprint** — non-negotiable when an AI agent is writing code at speed

---

## 📸 Screenshots

*Coming soon — screenshots of the dashboard, holdings view, transaction flows, and AI import pipeline.*

---

## 📚 Glossary

| Term | What It Is | What It Does in NEESH |
|:-----|:-----------|:---------------------|
| **SQLite** | A file-based relational database — no separate server process needed. The entire database is a single `.db` file. | Stores all transactions, holdings, income, user data. Trivially backed up by copying one file. |
| **WAL (Write-Ahead Logging)** | A mode for SQLite where writes go to a separate log file first, so readers aren't blocked. | Allows multiple family members to view dashboards simultaneously while one person is saving a transaction. |
| **FastAPI** | A modern Python web framework with built-in async support and automatic API documentation. | Serves all web pages, handles form submissions, processes API requests. Runs the entire backend. |
| **HTMX** | A small JavaScript library that lets HTML elements make HTTP requests and swap page content — no full-page reloads needed. | Powers dynamic interactions (search dropdowns, inline edits, partial page updates) without a React/Vue frontend. |
| **Tailwind CSS** | A utility-first CSS framework where you style elements with small class names (`bg-blue-500`, `text-lg`) instead of writing custom CSS. | Handles all UI styling including dark mode and responsive layouts. |
| **Jinja2** | A Python templating engine that turns HTML templates with placeholders into final HTML pages on the server. | Renders every page — dashboards, forms, tables — by injecting data from the service layer into HTML templates. |
| **JWT (JSON Web Token)** | A compact, signed token issued after login. The browser sends it with every request to prove identity. | Two tokens: a short-lived access token (30 min) and a long-lived refresh token (30 days). Users stay logged in without re-entering OTP daily. |
| **OTP (One-Time Password)** | A 6-digit code sent via WhatsApp or Email for login. Valid for a few minutes, single use. | Primary authentication method — no passwords to remember or leak. Delivery priority: WhatsApp Cloud API → Email SMTP → console (dev only). |
| **Alembic** | A database migration tool for SQLAlchemy. Each schema change is a numbered script that can be applied or rolled back. | Manages all database structure changes across app versions. Critical for a 50-year data lifespan. |
| **SQLAlchemy** | A Python ORM (Object-Relational Mapper) that lets you work with database tables as Python objects instead of raw SQL. | Defines all data models (User, Transaction, Holding) and handles async database queries. |
| **APScheduler** | A Python library for scheduling background tasks (like cron jobs) inside your application process. | Runs daily price refresh, forex rate updates, net worth snapshots, and weekly backups — all without a separate worker process. |
| **XIRR** | Extended Internal Rate of Return — a financial metric that computes annualized returns accounting for irregular cash flow timing. | Calculates true investment returns for SIPs, partial buys/sells, and multi-date investments. Computed using `pyxirr` (Rust-based, very fast). |
| **FIFO (First In, First Out)** | A cost basis method where the oldest purchase lots are matched first when selling. | Required by Indian tax law for equity. Determines whether each sold lot is STCG or LTCG based on when it was originally bought. |
| **STCG / LTCG** | Short-Term and Long-Term Capital Gains — Indian tax categories based on how long you held an investment before selling. | Auto-classified per holding using asset-class-specific thresholds (e.g., 12 months for equity, 24 months for debt). |
| **TDS (Tax Deducted at Source)** | Tax pre-deducted by banks/brokers on interest and dividends before paying you. | Tracked per income entry so users know how much tax is already paid vs. still owed. |
| **DTAA (Double Tax Avoidance Agreement)** | A treaty between countries (e.g., India-US) to prevent the same income from being taxed twice. | Used for RSU/ESOP/ESPP — foreign tax paid in the US can be credited against Indian tax liability. |
| **NAV (Net Asset Value)** | The per-unit price of a mutual fund, published daily by the fund house. | Fetched from AMFI's public API to value mutual fund holdings. |
| **AMFI** | Association of Mutual Funds in India — maintains a public database of all mutual fund NAVs and scheme codes. | Primary data source for mutual fund prices and fund search/lookup. |
| **yfinance** | A Python library that fetches stock/ETF prices, historical data, and metadata from Yahoo Finance. | Provides live and historical prices for stocks, ETFs, and SGBs (via gold price). |
| **Gemini (Flash / Pro)** | Google's AI models. Flash is fast and good at structured extraction. Pro is slower but better at reasoning and verification. | Flash extracts transaction data from uploaded statements. Pro re-verifies the extraction for accuracy. Both on free tier. Multiple API keys supported for quota rotation. |
| **LiteLLM** | An OpenAI-compatible API proxy that routes requests to many backend LLMs. | Fallback AI endpoint when Gemini is unavailable or rate-limited. Configured via `LITELLM_*` env vars to point at any OpenAI-compatible gateway (e.g. a corporate internal LLM). |
| **WhatsApp Cloud API** | Meta's official WhatsApp Business API, free tier. | Primary OTP delivery channel for login — the highest-priority channel in the fallback chain (WhatsApp Cloud → Email SMTP → Console). |
| **Twilio** | A legacy cloud communications API previously used for OTP delivery via WhatsApp. | **Deprecated for OTP** — replaced by WhatsApp Cloud API. Config still accepted for backward compatibility. |
| **systemd** | Linux's service manager — starts, stops, and auto-restarts background processes. | Runs the NEESH server as a system service that starts on boot and auto-restarts on crash. |
| **Cloudflare Tunnel** | A secure tunnel that exposes a local server to the internet via Cloudflare's network — no port forwarding or static IP needed. | Enables accessing NEESH from outside the home network over HTTPS. |
| **Upsert** | A database operation: insert a row if it doesn't exist, or update it if it does. | Used for `holdings_summary` — after recomputation, the holding row is created or updated in one atomic operation. |
| **EEE (Exempt-Exempt-Exempt)** | An Indian tax regime where contributions, growth, and withdrawals are all tax-free. | Applies to Provident Fund — no tax at any stage (within limits). |

---

<sub>Built with FastAPI, SQLite, HTMX, Tailwind CSS, Google Gemini, and a lot of family financial spreadsheets that needed to be consolidated.</sub>
