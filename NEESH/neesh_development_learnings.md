# Building a Personal Wealth Manager with AI — Learnings, Mistakes, and Things I'd Do Differently

*A product manager's honest account of building a real financial application with AI tools.*

---

## Who Am I and Why Does This Matter

I'm a technical product manager. I started my career as a software engineer, then moved into product management — where I've been for the last five years. I understand how software is built. I can read code, review architecture decisions, and have technical conversations with engineering teams. But I'm not actively writing code day-to-day.

For this project, I used AI tools to build the entire product — from generating the initial architecture to implementing every sprint. No co-developer. No engineering team. Just me directing AI through planning, design, implementation, and testing.

This is the story of how I built **NEESH**, a personal wealth management application that tracks 11 asset classes, manages family portfolios, imports broker statements using AI, and computes live net worth across currencies. I made a lot of mistakes, learned a lot of lessons, and I'm documenting everything here — not because it's a success story, but because the journey was more instructive than the destination.

If you're a technical PM curious about building with AI, or a developer interested in what a PM-led AI-assisted build actually looks like from the inside, this is for you.

---

## Part 1 — Planning: The Work Before the Work

### The First Mistake

I had a problem. My family's financial life was scattered across too many places — mutual fund apps, stock brokers, FD spreadsheets, PF statements, RSUs vesting in USD, a home loan EMI that just keeps going. No single view of net worth. Tax season was a scramble every year.

I gave this vague concept to the AI and expected it to figure out the rest. The AI was supportive — it said yes, this sounds like a good idea, here are some things you might consider. But it was lost. Because I was lost. I hadn't thought through what the application *actually needed to do*.

> **The AI is not a mind reader. It works with what you give it. Vagueness in, plausible-sounding-but-wrong out.**

### The Four Planning Steps That Changed Everything

Once I understood that I needed to do the thinking first, I broke the planning into four steps.

**Step 1 — Unconstrained Brainstorming.** I wrote every feature idea I had. No filtering. Investment tracking, expense categorisation, AI-powered statement import, loan simulation, tax projections, WhatsApp notifications, family accounts, Zerodha integration. Everything.

The rule was: no filtering during this phase.

**Step 2 — Ruthless Prioritisation.** Every item evaluated against a single filter: does this serve the core product vision? Anything that didn't pass was cut. This wasn't about what was cool or technically impressive — it was about what the app needed to *be*.

**Step 3 — Must Have vs. Nice to Have.** Impact × effort matrix. High impact, reasonable effort → Phase 1. Lower impact or high complexity → Phase 2/3. This gave me a clear Phase 1 that was ambitious enough to be useful but scoped enough to be buildable.

**Step 4 — User Journeys.** For every feature in Phase 1, I drafted the full user journey — what a user does when they first open the app, how they add a transaction, what they see on the dashboard, what happens when a price refresh fails. Every scenario, every edge case I could imagine.

Then I gave this consolidated document to the AI and asked it to *evaluate* — not build. To check coherence, cover gaps, and ask clarifying questions.

**This was the step where everything changed.** Because now the AI had real material to work with. It came back with genuinely good questions — questions I hadn't considered. I answered all of them, and it produced a well-structured PRD. The same process followed for an architecture document — clarifying questions first, document second.

> **You have to clear your own head before you can direct an AI.** Think of AI as highly capable but completely dependent on your guidance. It is not a creative partner who will rescue a vague idea. It is a powerful executor that produces excellent output when given excellent input.

### Document Discipline

Three types of documents, kept strictly separate:

| Document | Contains | Does NOT Contain |
|:---------|:---------|:-----------------|
| **PRD** | What the product does, who it's for, what each phase covers | Library names, schema design, code patterns |
| **Architecture** | Database schema, models, data flow, dependencies, technical decisions | Product strategy, user personas |
| **Sprint Design** | Implementation plan, test scenarios, acceptance criteria | Architectural philosophy |

The AI itself recognised that the implementation scope was too large for a single design document and broke it into 8 sprint design documents — from project bootstrap through Zerodha integration.

What surprised me was that the AI started auto-managing its own context during this step. It recognised it was running out of context window, compressed the conversation, and continued. It asked for my sign-off at key decision points. When the documents were done, it gave me a structured summary of everything it had produced.

### The Iteration Rule

After the PRD was drafted, I asked the AI to review it against the architecture document and flag gaps. It found gaps. After the architecture doc, I asked it to review both documents together. More gaps. Each round produced fewer and smaller issues.

By the third pass, there was one minor inconsistency. That's when I knew we were ready to move forward.

> **Two to three rounds of iteration per stage, not one and not infinite.** One pass leaves too many errors. Infinite passes waste time and confuse the AI's context. Two or three passes done thoughtfully is the sweet spot.

Another thing I learned: **refine early, not late.** If you wait until Sprints 6, 7, and 8 are implemented to do your first serious review pass, the AI has to hold an enormous amount of context to evaluate everything. If you refine at each stage — after the PRD, after the architecture, after each sprint design — the AI is working with manageable scope and the errors stay small.

### Phase 2/3 Guidelines: Planning Without Over-Committing

The PRD covered Phases 1, 2, and 3. But I asked the AI to draft Phase 2/3 as lightweight guidelines — just enough to preserve direction, not rigid enough to lock anything down. The key instruction was: *don't be rigid, because Phase 1 development may change some of these decisions.*

This was the right call. Some Phase 1 decisions shifted during development, and having Phase 2/3 as loose guidelines meant there was nothing to undo. Phase 1 code includes hooks for Phase 2 features (like the `as_of_date` parameter on every tax handler) without actually building those features.

---

## Part 2 — Architecture: Decisions That Compound

### Transactions as Source of Truth

The single most impactful architectural decision: **transactions are the source of truth, holdings are always derived.**

Every financial event is a transaction. Holdings are computed from the full transaction history — never stored independently. Edit a past transaction → holding recomputes. Delete a transaction → holding adjusts. No data can ever be "out of sync."

This eliminates an entire class of bugs. In a financial application where accuracy over decades matters, this is non-negotiable.

### Layered Architecture — Strict Separation

| Layer | Does | Does NOT Do |
|:------|:-----|:------------|
| **Routes** | Accept HTTP requests, return HTML | Compute anything |
| **Services** | All business logic, validation, orchestration | Touch the database directly |
| **Repositories** | Read/write database | Make decisions or compute |
| **Templates** | Render HTML | Calculate values |

This sounds textbook, but the discipline of maintaining it paid off repeatedly. When Phase 2 needs a tax simulation service, it can compose with existing holding and transaction services without refactoring routes or templates. When business logic leaks into routes or templates, new services can't reuse it — and that debt compounds across phases.

### Asset Class Handler Pattern

11 asset classes with fundamentally different behaviours — but every one implements the same interface with the same methods. Tax rates and thresholds are class-level constants, not buried in if/else logic.

When the Indian budget changes LTCG rates, it's a one-line config update per handler. When Phase 2 adds tax simulation, it calls the same `get_tax_info()` method with a future date. The interface was designed for this from day one.

### Database Choices for Longevity

SQLite in WAL mode. No daemon process. The entire database is a single file. Backups = copy one file. Schema changes through Alembic migrations — versioned, auditable, reversible.

A deliberate tradeoff: no concurrent writes, but with 2-3 family members and infrequent writes, this is never a bottleneck. The simplicity buys us decades of maintenance-free operation.

Holdings summary is a **rebuildable cache** — it can be dropped and recomputed entirely from transactions. This means Phase 2/3 can change what's cached without risk.

### Key Uniqueness Constraints

Two constraints that, when violated, produced subtle and hard-to-debug errors:

**Transaction duplicate detection** must match on ALL of: `asset_class`, `symbol`, `transaction_date`, `quantity`, `price_per_unit`, and `instrument_name`. Missing any one field (we missed `instrument_name` initially) causes false duplicate warnings for things like different SGB series bought on the same date.

**Holdings uniqueness** is by `(user_id, asset_class, symbol, platform)`. You can hold TCS on both Zerodha and Groww — they're separate holdings. Forgetting to filter by platform when computing XIRR or portfolio values silently produces wrong results.

---

## Part 3 — Development: Where Theory Meets Reality

### The Feedback Discipline

I'm a product manager, not an active developer. When I found bugs, I could sometimes read the code and guess where things went wrong — but more often, what worked better was describing exactly what I was experiencing.

That specificity turned out to be the entire skill.

**Vague feedback:** *"The date field doesn't work properly."*

**Specific feedback:** *"There is a calendar icon inside the date input field. Clicking the calendar icon opens the date picker. However, clicking anywhere else on the input field does not open the date picker. Expected: clicking anywhere on the input field should open the date picker."*

The vague version leaves the AI guessing. It might fix the wrong thing, or partially fix the right thing, or ask you four clarifying questions before it can proceed. The specific version gives it everything it needs to act immediately and correctly.

> **The most important skill in AI-assisted development is not deep coding expertise. It is knowing how to describe precisely what you observe and what you expect.**

I learned to structure feedback like a product bug report:
- What I did
- What happened
- What I expected to happen instead
- Any relevant context (which screen, which input, which action triggered it)

And always in small batches — 3-4 specific points per cycle. This structure, applied consistently, reduced the back-and-forth dramatically. Ten points at once exceeds what the AI can reliably track end-to-end before the context shifts.

### Sprint Strategy: Plans vs. Reality

**Original plan:** Single Sprint 2 for all transaction CRUD across 11 asset classes.
**Reality:** Broke into 6 sub-sprints (2A through 2F).

Each asset class has unique complexity — FIFO lot tracking for equities, perquisite tax for RSUs, accrued interest for FDs, fund flow linking for bank accounts. Trying to do all 11 at once would have been chaotic.

| Sub-Sprint | Focus |
|:-----------|:------|
| 2A | Core transaction model + FIFO lots |
| 2B | Asset handlers (Equity, MF, ETF) |
| 2C | Fixed income (FD, SGB) |
| 2D | RSU/ESOP/ESPP |
| 2E | Bank accounts + Fund flow |
| 2F | Provident Fund + Loan schema |

Better incremental testing, clearer deliverables, and more accurate estimates at each step. The lesson: **when reality exceeds the plan, break the plan down — don't push through.**

### Test Scenarios: The Thing I Almost Forgot

Before handing sprint designs to the development AI, I made one more pass: I asked it to check whether every design document included test scenarios.

This came from a mistake I had already made in an earlier pass. I assumed the AI would auto-generate test cases as it built things. It didn't. It built what the design document described. If the document didn't say "run these tests and report results," the AI wasn't going to run tests.

> **You have to tell the AI everything about your expectations, including what success looks like after the work is done.**

Once test scenarios were embedded in each sprint document, I added one more instruction: *after implementing each sprint, run all tests, report outcomes, and wait for my feedback before proceeding.* This small addition fundamentally changed the quality of the development loop.

### Test Isolation

A bug pattern that cost me hours: the dev server and test suite sharing the same database. Combined with Windows file locking (Windows holds file handles longer than Linux), this produced intermittent "Database is locked" errors and test pollution.

**The fix:**

| Environment | Port | Database | Script |
|:------------|:-----|:---------|:-------|
| Development | 8000 | `data/neesh.db` | `start_dev.bat` |
| Testing | 8765 | `test_e2e.db` | `run_e2e_tests.bat` |
| Production | 8000 | `data/neesh.db` | `start_user.bat` |

Separate databases. Separate ports. Automated cleanup before each test run. Scripts handle virtual environment activation and file handle closure. The cognitive load dropped immediately — no more "is the dev server still running?" before tests.

### The AI Import Pipeline: Designing for Accuracy

The statement import flow deserves its own section because it captures several design principles at once.

**The user flow:**
1. Upload Excel/CSV → extract transactions (local parser or AI)
2. Show ALL extracted transactions (all selected by default)
3. Attempt import of selected transactions
4. Failures (duplicates, validation errors) shown in a table with inline edit
5. User fixes and re-imports, or rejects

**Key design decisions and why:**
- All entries selected by default — no confidence-score-based filtering. The user reviews everything. This avoids the AI silently dropping valid transactions because it wasn't confident enough.
- Three-tier parsing pipeline: local regex parser first (instant, known formats), Gemini Flash second (extraction), Gemini Pro third (verification). Same "right model for the right task" principle applied within the product itself.
- Files parsed locally first (openpyxl/csv) and sent as text to Gemini — never as binary files. This avoids Gemini's File API quotas and improves reliability.

### Market Data Search: Iterating on Performance

**Problem:** Searching the AMFI file (5000+ mutual funds) was too slow for a responsive dropdown.

**Evolution:**
1. First attempt: search the full AMFI text file on every keystroke → unusable.
2. Cache AMFI file to disk (`data/amfi_nav_cache.txt`) → better but still sluggish.
3. Use mfapi.in API for instant search → fast, with disk cache as fallback if API fails.
4. Handle numeric input separately — searching by scheme code (e.g., "120716") needed to match the scheme code field, not the fund name.

**UI lesson:** Use solid black backgrounds for search dropdowns — alpha transparency made text unreadable against certain page backgrounds. Small thing, big usability impact. Also: Indian number formatting (lakh, crore) instead of international notation (million, billion).

---

## Part 4 — Common Bug Patterns

Building across 11 asset classes, I ran into recurring patterns. Documenting them here because they're the kind of bugs that waste hours the first time and seconds every time after — if you know the pattern.

### Incomplete Duplicate Detection

**Symptom:** False duplicate warnings for distinct transactions.
**Root cause:** Duplicate check only matched `(symbol, date, qty, price)` — missed `instrument_name` for things like different SGB series, or different mutual fund schemes with the same AMFI code.
**Fix:** Match on all distinguishing fields: `asset_class`, `symbol`, `transaction_date`, `quantity`, `price_per_unit`, and `instrument_name`.
**Lesson:** You CAN buy the same stock multiple times on the same day with different quantities and prices. The duplicate check must be precise enough to not block legitimate transactions.

### Holdings Lookup Without Platform Filter

**Symptom:** XIRR calculation used wrong holdings, or P&L didn't match.
**Root cause:** Holdings are unique by `(user_id, asset_class, symbol, platform)` but lookups only filtered by symbol.
**Fix:** Always filter by platform. You can hold TCS on both Zerodha and Groww — they're separate holdings with separate cost bases.
**Lesson:** Composite keys need to be respected everywhere, not just in the schema definition. Every query, every lookup.

### Windows File Locking

**Symptom:** "Database is locked" errors, or file deletion failures during test cleanup.
**Root cause:** Windows holds file handles longer than Linux. SQLite connections not being properly closed before cleanup scripts ran.
**Fix:** Dedicated cleanup scripts with proper file handle closure. Stop the dev server before running E2E tests.

### Both Test and Code Can Be Wrong

This one matters more than it sounds.

When a test fails, the instinct is to fix the code. But sometimes the test expectation is wrong — especially with financial calculations where the correct answer isn't always obvious. Is the XIRR for a specific set of cash flows 12.3% or 12.7%? Is the LTCG threshold 365 days or 366 days (leap year)?

> **When debugging, question both the implementation and the test expectations. Fix the underlying logic, not just what makes the test pass.**

Tests should verify correct behaviour, not just current behaviour. The distinction matters when you're building a financial application where "close enough" isn't acceptable.

---

## Part 5 — Working With AI: The Meta-Lessons

### The Context Window Drives Everything

If there is one technical constraint that anyone building with AI must understand, it is the context window.

The AI has a limit to how much it can hold in memory at once. As conversations grow long, as documents get larger, as more code gets written — the AI starts to forget things. Not because it's careless, but because it has literally run out of space to hold everything at once.

This is not a flaw you can fix. It is a property of the tool you have to design around.

How this affected the build:
- **Documents were missed or incomplete** because the AI ran out of context while generating them. I'd ask it to review its own output and it would find gaps it had no memory of creating.
- **Feedback I gave was only partially implemented**, because 10-point feedback lists exceeded what the AI could track end-to-end before the context shifted.
- **Architecture decisions made early in the conversation were forgotten** by implementation time.

**Strategies that helped:**

1. **Keep documents as the source of truth, not the conversation.** If a decision is made, it goes into a document. The AI references documents, not memory.
2. **Break feedback into small focused batches.** 3-4 specific points per feedback cycle is more reliable than 10 points at once. After each batch is implemented and verified, move to the next.
3. **After each implementation, explicitly ask the AI to update the relevant design documents, PRD, and architecture.** Don't assume it remembers to do this. Tell it every time.
4. **Refine documents iteratively, not all at once at the end.** Each stage produces a document. Review it. Fix it. Then move forward. Don't defer all review to the end.

### The Detour That Cost Me Time

Midway through Sprint testing, I ran a command I didn't need to run. I was curious about something and asked the AI to execute an exploratory task that had nothing to do with the current sprint.

The AI did it. It was happy to do it. It used a significant chunk of context doing it.

The result: the AI's context was partially consumed by work that wasn't relevant to what I was building. It became slightly harder to stay on the main trajectory. I had to do extra work to re-establish context.

> **Just because an AI can do something does not mean you should ask it to.** Every token you spend on unnecessary work is a token not spent on your actual goal. Irrelevant tasks don't just waste time — they degrade the quality of everything that follows by crowding out working context.

### Model Selection: The Cost and Quality Decision Nobody Talks About

Building this application was not cheap. Not in time, and not in money.

I used two AI tools across this project. One was a high-capability reasoning model which I used for everything that required deep thinking: writing and refining PRDs, producing the architecture document, designing sprint plans, reviewing documents for gaps, and asking the clarifying questions that shaped the product. The other was an AI coding agent, which did the actual code implementation sprint by sprint.

This taught me one of the most overlooked practical lessons of working with AI tools: **model selection is not a minor detail. It is a cost and quality decision that matters enormously.**

There is a wide spectrum of AI models available today. They differ in capability, in speed, in context window size, and in cost per token. Using a high-powered model for a task that doesn't require it is like hiring a senior architect to write meeting notes. The notes will be excellent. But you have massively overpaid.

**The mental model I now use:**

| Task Type | Right Model Tier | Examples |
|:----------|:----------------|:---------|
| Complex reasoning | High-capability (Sonnet, GPT-4o, Gemini Pro) | PRD review, identifying logical gaps, architecture decisions, non-trivial code |
| Simple execution | Lighter models (Haiku, Flash, GPT-4o Mini) | Grammar cleanup, reformatting, summarisation, simple factual questions |

**The analogy that clicks:** Imagine you need to both design a bridge and paint it. You would hire a structural engineer to design it — because the design requires deep expertise and the cost of getting it wrong is catastrophic. But once the design is done, you would hire a painter to paint it. You would not pay the structural engineer's day rate to apply paint.

AI models work the same way. If I want to refine the phrasing of an already-approved PRD — fix sentence flow, correct grammar — I do not need a high-capability model. A lighter model handles it competently at a fraction of the cost. But if I want to identify whether my PRD has logical gaps, contradicts itself across sections, or is missing critical edge cases — that is a task that benefits from deeper reasoning.

Before starting any AI-assisted task, I now ask myself:
1. What is the actual complexity of what I'm asking?
2. Does this require deep reasoning, or is this execution of a clear and simple instruction?
3. What context window does this task need — is my content large enough to stress a cheaper model's limits?
4. What does it cost if the output is mediocre — and does that justify a more capable model?

NEESH's own AI pipeline follows this principle at the product level: Gemini Flash for fast statement extraction, Gemini Pro for verification that requires reasoning. The Pro verification step caught ~15% of extraction errors — wrong transaction types, inverted amounts, misidentified currencies. Same concept, applied to the product itself.

> **The goal is not to always use the best model. The goal is to always use the right model.**

### Git Safety: Protecting Work From the AI (and Yourself)

When an AI coding agent writes, modifies, and sometimes deletes files — all at speed, often across multiple files in a single operation — you are not watching every individual change. You are reviewing outcomes.

This is fine when things go right. When things go wrong, it can be catastrophic.

**Scenario 1 — The wrong prompt.** You give the AI an instruction that is slightly off. It interprets it in a way you didn't intend and starts rewriting or removing things that were working. You don't notice until the damage is done.

**Scenario 2 — System failure.** Your machine crashes. Your session terminates unexpectedly. Hours of AI-generated code, gone.

Both have the same solution: **Git from day one. Push after every sprint.**

Git gives you a complete safety net. Every sprint is a snapshot. If something goes wrong — bad prompt, accidental deletion, system crash — you roll back to the last good state and lose nothing.

> **AI writes code fast. Mistakes happen fast too. Git ensures that fast mistakes don't become permanent ones.**

**Restricting the AI's own Git access** was another lesson learned. I configured hooks to block the AI from running destructive Git commands (`git commit`, `git push`, `git rm`, `git reset`). The AI can view status, diffs, and logs — but it cannot push, commit, or discard. I control what gets committed. This single guardrail prevented at least two incidents where the AI would have committed incomplete work.

---

## Part 6 — Documentation & Knowledge Management

### Keep Your AI Context File Minimal

When working with AI coding tools, you often maintain a context file that the AI reads at the start of every session. The temptation is to dump everything into this file — sprint history, completed work, detailed logs, full architecture.

Don't. The AI's context window is finite, and every line of this file consumes space that could be used for actual work.

**What belongs in the context file:**
- Quick start commands
- Current sprint focus
- Active issues or blockers
- Essential context only

**What does NOT belong:**
- State information or progress logs
- Detailed logging of completed work
- Extensive history
- Full architecture (link to doc instead)

### Document Hierarchy

Over 10 sprints, the document structure that emerged:

```
docs/
├── DEVELOPER_GUIDE.md         # Comprehensive setup & testing
├── prd.md                     # Product requirements (what)
├── phase1_arch.md             # Architecture (how)
├── phase23_guidelines.md      # Future phase direction (loose)
├── sprint0.md - sprint9.md    # Implementation guides per sprint
├── flow_review.md             # Transaction flow diagrams
└── ai_context.md (root)       # Minimal quick reference for AI
```

The key discipline: **each document has one job.** The PRD doesn't contain architecture. The architecture doesn't contain product strategy. Sprint docs don't contain philosophy. When documents stay clean, the AI can reference the right one without context pollution.

### Stale Document Management

Over months of development, documents accumulate. Some become stale. A scoring system helped:

| Score | Action | Examples |
|:------|:-------|:---------|
| 8-10 (very stale) | Safe to remove | Empty files, one-time checklists, superseded meta-notes |
| 5-7 (somewhat stale) | Consider archiving | Session-specific summaries, duplicate analyses |
| 0-4 (still relevant) | Keep as reference | Sprint guides, architecture decisions, PRD, test scenarios |

### Change Request Management

When requirements changed mid-development (they always do), I tracked them in separate CR files. The rule: **archive, don't delete.** Move to `docs/archive/` with an incorporation note after the changes are merged into sprint docs. This preserves the decision trail without cluttering the working document set.

---

## Part 7 — Engineering Notes

*These are the specific technical patterns, configurations, and reference material that helped during development. Less narrative, more actionable.*

### Environment Configuration

```ini
# Core Settings
APP_NAME=NEESH
DEBUG=true                    # Enable API docs and detailed errors
DEV_MODE=true                 # Use password login instead of OTP

# Database
DATABASE_URL=sqlite+aiosqlite:///./data/neesh.db

# Security
JWT_SECRET=<generate-with-python>   # python -c "import secrets; print(secrets.token_hex(32))"
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=43200   # 30 days
```

Three environment configs: `.env.prod` (real data), `.env.dev` (test data for review), `.env.test` (E2E testing). Never mix them.

### Market Data Providers

| Source | Assets Covered | Fallback |
|:-------|:--------------|:---------|
| yfinance | NSE/BSE stocks, ETFs | Google Finance |
| AMFI NAV API | Mutual funds | Disk-cached AMFI file |
| RBI Reference Rates | Forex (USD, EUR, etc.) | — |
| Gold price providers | MCX Gold, SGB valuation | — |

Price cache in SQLite. Background scheduler refreshes at 6:30 PM IST (after Indian market close). During market hours (9:15 AM – 3:30 PM), prices considered stale after 1 hour.

### FIFO Lot Management

Implemented at the asset handler level, not as a generic utility. Each asset class has its own FIFO logic because the rules differ:
- Equities: standard FIFO, holding period starts per lot
- Mutual funds: same as equities but NAV-based
- RSUs: holding period starts from vest date, not grant date
- FDs: no lots — single principal amount

Critical for tax classification: when selling, the oldest lots are matched first. This determines per-lot whether the gain is STCG or LTCG.

### Testing Approach

| Type | Location | Database | Purpose |
|:-----|:---------|:---------|:--------|
| Unit tests | `tests/unit/` | In-memory SQLite | Fast, isolated, business logic focus |
| E2E tests | `tests/e2e/` | `test_e2e.db` | Full integration with Playwright, real user workflows |

```bash
# After E2E tests, inspect the database
sqlite3 test_e2e.db
SELECT * FROM transactions LIMIT 10;
SELECT * FROM holdings_summary;
.tables
.schema transactions
```

---

## Part 8 — The Consolidated Playbook

### On Planning
- Clear your own head before directing the AI. It amplifies your clarity, it does not replace it.
- Brainstorm freely, then filter ruthlessly. Two separate steps, not one.
- Write your user journeys before writing any specification.
- Build complete documents before starting development. Incomplete specs produce incomplete software.
- Ask the AI to evaluate and ask clarifying questions before it writes anything.
- Two to three iterations per document stage. Not one and not infinite.
- Refine early, at every stage. Do not defer review to the end.

### On Architecture
- Transactions as source of truth — no independent state for derived data.
- Strict layer separation — business logic only in services. Routes don't compute. Repositories don't decide.
- Design interfaces for extension — embed hooks for future phases without building the features.
- Tax rules are configuration, not code. Rates are class-level constants.
- Schema changes through Alembic migrations. Always. No manual `ALTER TABLE`.
- JSON columns are for display-only data. Anything queryable gets a proper column.
- Holdings summary is a rebuildable cache. Drop it, recompute from transactions, lose nothing.

### On Development
- Break large sprints when reality exceeds the plan.
- Embed test scenarios in design documents before development starts.
- Tell the AI explicitly what you expect after each sprint is done.
- Separate databases, ports, and cleanup scripts for dev vs. test.
- Both test and code can be wrong — question both. Fix the underlying logic, not just what makes the test pass.
- Scripts over manual steps. Reduces cognitive load, prevents errors, handles environment differences.

### On AI Collaboration
- Documents are the source of truth, not conversations.
- Small, specific feedback batches. Structure them like bug reports.
- After each implementation, explicitly ask the AI to update documents. Never assume it remembers.
- Match model capability to task complexity. Don't overpay for simple work.
- Don't give AI unnecessary work — context is a finite resource.
- Restrict the AI's destructive capabilities (Git hooks, file protection).
- Git after every sprint. Non-negotiable.

### On Cost
- Not all AI models are equal in capability, context limit, or price. Know the difference.
- Use high-capability models for complex reasoning: reviewing specs, identifying gaps, architecture.
- Use lighter, cheaper models for execution: grammar, reformatting, summarisation.
- Match the model to the task. Overpaying for simple work and under-investing in complex work are equally costly mistakes.
- Before starting any AI task, ask: what is the actual complexity, and what does it cost if the output is mediocre?

---

## Part 9 — Don't Rush to Check a Box

I want to close with something that resonates deeply with me, and that captures the risk most people and organisations face right now with AI.

There is a very real sense of urgency in the air. A fear that if you don't act immediately, you'll miss the moment. That fear is understandable. I felt it too when I started this project.

But here is the danger: when urgency turns into panic, it produces a particular kind of mistake — the *checkbox mistake*. You rush into something fast and shallow just so you can say, "Yes, we're using AI." You pick the first tool you see, apply it to the nearest problem, and declare success. The box is ticked. Nothing of lasting value is built.

The right framing, and the one that changed how I approached this project, is the difference between being an **AI value consumer** and an **AI value creator**.

A value consumer plugs AI into an existing workflow as a shortcut. They save a little time. They produce slightly better-looking output. But they are entirely dependent on whatever the AI gives them out of the box, and the moment the wind changes — a new model, a new tool, a new paradigm — they have to start over from nothing.

A value creator does something harder. They think first. They are deliberate and strategic about what they are building, how they are building it, and what problem it actually solves. They invest in understanding the tools, not just using them. They build foundations — good documents, clear processes, a feedback discipline, version control — that survive model upgrades and tool changes. When the wind shifts, they don't start over. They adapt, because what they built was grounded in real thinking, not just borrowed capability.

This project pushed me to be a value creator, even though I didn't have the vocabulary for it at the time. The months of planning, the document iterations, the precise feedback, the sprint discipline — none of that was the AI doing magic. All of that was me doing work, using AI as a powerful instrument to extend what I was capable of.

The extraordinary thing about where we are right now is that the tools are available to everyone. You don't need to be an engineer. You don't need a large organisation behind you. You don't need to have the full picture before you start. What you need is to be thoughtful, to be specific, to be willing to do the hard cognitive work that separates a meaningful outcome from a ticked box.

> **The opportunity is not to use AI. The opportunity is to use AI to build something that matters.**

The people who do that thoughtful, deliberate work — the value creators — are the ones who will make the biggest impact with this technology. Not because they moved fastest, but because they built on real foundations.

That is what this whole journey has been about for me. And I'm only getting started.

---

*Building alongside AI — a product manager's notes.*
