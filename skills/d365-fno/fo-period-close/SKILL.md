---
name: fo-period-close
description: >
  Configure and execute D365 F&O financial period close processes including
  period status management, subledger reconciliation, foreign currency
  revaluation, financial consolidation, elimination, year-end close, ledger
  settlements, and financial reporting. Use when: performing month-end close,
  year-end close, financial consolidation, or setting up close workspaces.
  DEPENDS ON: fo-gl-chart-of-accounts, fo-gl-posting-profiles (all posting
  must be correct before close).
metadata:
  platform:
    category: "fo-general-ledger"
    riskLevel: "high"
    configSequence: 11
    configLevel: "25"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts", "fo-gl-posting-profiles"]
    bpcProcess: "13.50 - Record to Report > Close financial periods"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/perform-periodic-processes/"
    mbExam: "MB-310 (within Implement financial management 40-45%)"
---

# Financial Period Close

## Period Close Workspace Setup

### Financial Period Close Workspace
- **Menu:** General ledger > Period close > Financial period close workspace
- **MCP:** Form tool
- **Components:**
  - Close schedules (define task lists per period)
  - Close calendar (track open/on-hold/closed periods)
  - Task assignments (who does what)
  - Dependencies (task B can't start until task A completes)

### Period Status Management
- **Menu:** General ledger > Calendars > Fiscal calendars > [calendar] > Period status
- **MCP:** Form tool
- **Statuses:**
  - **Open** — posting allowed
  - **On hold** — posting blocked temporarily
  - **Permanently closed** — no posting allowed (irreversible)
- **Best practice:** Close periods as soon as reconciliation is complete to prevent backdated postings
- **Module-level hold:** Can put periods on hold per module (GL, AP, AR, etc.) independently

## Monthly Close Process (Typical Sequence)

### 1. Subledger Posting Completion
- Ensure all AP invoices posted
- Ensure all AR invoices posted
- Ensure all inventory transactions financially updated
- Run: Inventory close/recalculation (Inventory management > Periodic > Closing and adjustment)
- Ensure all bank transactions posted

### 2. Foreign Currency Revaluation
- **GL revaluation:** General ledger > Periodic tasks > Foreign currency revaluation
- **AP revaluation:** Accounts payable > Periodic tasks > Foreign currency revaluation
- **AR revaluation:** Accounts receivable > Periodic tasks > Foreign currency revaluation
- **Bank revaluation:** Cash and bank management > Periodic tasks > Foreign currency revaluation
- **Purpose:** Restate open balances in foreign currencies at period-end exchange rates
- **Creates:** Unrealized gain/loss journal entries
- **MCP:** Form tool for each (set parameters and execute)

### 3. Subledger-to-GL Reconciliation
- AP aging report total = AP control account in GL
- AR aging report total = AR control account in GL
- Inventory valuation = Inventory control accounts in GL
- Bank balances = Bank GL accounts
- FA net book value = FA GL accounts
- **Account Reconciliation feature (v10.0.44+):** Automates this — General ledger > Periodic tasks > Account reconciliation

### 4. Accruals and Deferrals
- Post any period-end accrual journals
- Process periodic journals (General ledger > Periodic tasks > Periodic journals)
- Review and adjust deferral schedules

### 5. Consolidation (if multi-entity)
- **Menu:** General ledger > Periodic > Consolidate > Consolidate online
- **Purpose:** Combine financial results from multiple legal entities
- **Prerequisites:** Consolidation company configured, elimination rules defined
- **MCP:** Form tool

### 6. Elimination Entries (if multi-entity)
- Intercompany eliminations
- Minority interest adjustments
- **Menu:** General ledger > Periodic > Elimination

### 7. Ledger Settlements
- **Menu:** General ledger > Periodic tasks > Ledger settlements
- **Purpose:** Match and settle offsetting GL entries
- **MCP:** Form tool

### 8. Financial Reporting
- Generate financial statements (Balance sheet, Income statement, Cash flow)
- **Menu:** General ledger > Inquiries and reports > Financial reports
- **MCP:** Form tool (select report, set parameters, generate)
- **BPA alternative:** Use MCP Analytics server for AI-driven financial analysis

### 9. Close Period
- Set period status to "On hold" or "Permanently closed"
- Open next period if not already open

## Year-End Close

### Process
- **Menu:** General ledger > Period close > Year end close
- **MCP:** Form tool
- **What it does:**
  1. Transfers P&L account balances to retained earnings
  2. Creates opening balance entries for new fiscal year
  3. Posts closing/opening vouchers
- **Prerequisites:**
  - All monthly closes for the fiscal year must be complete
  - Retained earnings account(s) defined in chart of accounts
  - Year-end close parameters configured (Accounts for automatic transactions)
  - All subledgers reconciled

### Year-End Close Configuration
- **Opening transactions:** Configurable — create detailed or summary opening entries
- **Close P&L to:** One retained earnings account or multiple (by financial dimension)
- **Fiscal year closing sheet:** Review template before running

## Consolidation Setup (if needed)

### Consolidation Company
- Separate legal entity used only for consolidated reporting
- Must have same chart of accounts structure (or mapped)
- Fiscal calendar must align

### Consolidation Methods
- **Online consolidation** — real-time within D365
- **Financial reporting consolidation** — via report tree definitions (no data movement)

### Elimination Rules
- **Menu:** General ledger > Periodic > Elimination > Elimination rule
- Define intercompany transaction elimination patterns

## Validation Checks
- [ ] Period close workspace configured with task lists
- [ ] Close calendar has all fiscal periods
- [ ] Foreign currency revaluation parameters set (exchange rate type, posting accounts)
- [ ] Account reconciliation feature enabled (v10.0.44+)
- [ ] Retained earnings account(s) defined for year-end close
- [ ] Consolidation company configured (if multi-entity)
- [ ] Elimination rules defined (if intercompany transactions exist)
- [ ] Financial report definitions exist (Balance sheet, P&L at minimum)
- [ ] Test: Run mock period close → reconcile all subledgers → generate financial statements

## Common Errors
- Year-end close "retained earnings account not found" → Automatic transaction setup incomplete
- Consolidation imbalance → Currency translation differences not accounted for
- Revaluation entries reversed unexpectedly → Reversal date parameter set incorrectly
- Period can't be closed → Pending transactions exist (check all subledgers)
- Opening balance mismatch → Year-end close run multiple times without reversal

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`, `fo-gl-posting-profiles`
- **Used with:** `fo-financial-reporting`, `fo-budgeting` (budget vs actuals comparison)
