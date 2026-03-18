# Period Close Process

Source: Microsoft Learn + verified research — 2026-03-18

## Monthly Close Sequence (Typical)

### 1. Complete Subledger Posting
- All AP invoices posted and matched
- All AR invoices posted
- All inventory transactions financially updated
- All production orders ended (cost calculated)
- All bank transactions posted
- All project transactions posted (if applicable)

### 2. Run Inventory Close
- **Menu:** Cost management > Periodic > Closing and adjustment > Inventory close
- Required for all costing methods EXCEPT moving average
- Settles issue transactions to receipt transactions
- Posts adjustments to GL
- **Run during off-peak hours**
- After close: can't post inventory transactions before the closing date

### 3. Foreign Currency Revaluation (per module)
Restates open balances at period-end exchange rates:

| Module | Menu Path | What It Revalues |
|---|---|---|
| GL | General ledger > Periodic tasks > Foreign currency revaluation | GL balance sheet accounts in foreign currency |
| AP | Accounts payable > Periodic tasks > Foreign currency revaluation | Open vendor invoices in foreign currency |
| AR | Accounts receivable > Periodic tasks > Foreign currency revaluation | Open customer invoices in foreign currency |
| Bank | Cash and bank management > Periodic tasks > Foreign currency revaluation | Bank accounts in foreign currency |

Each creates unrealized gain/loss journal entries.

### 4. Subledger-to-GL Reconciliation
Verify each subledger ties to its GL control account:

| Subledger | Report | GL Account |
|---|---|---|
| AP | Vendor aging/balance | AP control (summary account from posting profile) |
| AR | Customer aging/balance | AR control |
| Inventory | Inventory valuation | Inventory control accounts |
| FA | Fixed asset NBV | FA asset - accumulated depreciation |
| Bank | Bank balance | Bank GL accounts |

**Automated:** Use Account Reconciliation feature (v10.0.44+):
Menu: General ledger > Periodic tasks > Account reconciliation

### 5. Post Accruals and Deferrals
- Periodic journals for recurring entries
- Deferral schedule processing
- Manual accrual entries

### 6. Close the Period
- **Menu:** General ledger > Calendars > Fiscal calendars > [period] > Period status
- Set status: **On hold** (temporarily blocks posting) or **Permanently closed** (irreversible)
- Can set per-module holds (e.g., close GL while AP stays open for late invoices)

## Year-End Close

### Prerequisites
- All 12 monthly closes complete
- All subledgers reconciled
- FA depreciation run through year-end
- Retained earnings account(s) exist in chart of accounts
- Year-end close parameters configured (Accounts for automatic transactions)

### Process
Navigation: **General ledger > Period close > Year end close**

1. Select the fiscal year
2. Select which companies to close
3. Choose closing parameters:
   - **Create opening transactions:** Detailed (per-dimension) or Summary
   - **Close P&L to:** One or multiple retained earnings accounts
   - **Transfer P&L dimensions:** Which dimensions carry forward to opening balance
4. Run the close
5. System creates:
   - Closing entries: transfer P&L balances to retained earnings
   - Opening entries: carry forward balance sheet balances to new year
6. Review closing/opening vouchers

### Common Issues
- "Retained earnings account not found" → Accounts for automatic transactions not configured
- Opening balance mismatch → Close run multiple times without reversal
- Dimension not transferring → Transfer P&L dimensions setting not checked for that dimension

## Financial Period Close Workspace

Navigation: **General ledger > Period close > Financial period close workspace**

Provides a structured task list for period close:
- Define close tasks with owners, due dates, and dependencies
- Track completion across legal entities
- Visualize close status on a calendar
- Enforce task sequencing (Task B can't start until Task A is complete)
