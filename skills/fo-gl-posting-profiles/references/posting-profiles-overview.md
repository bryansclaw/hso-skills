# Posting Profiles — Cross-Module Overview

Source: Microsoft Learn (pstg-prfles-ovrvw, recommended-practices-pstg-prfles) — verified 2026-03-18

## What Posting Profiles Control

Before configuring posting profiles, you must:
1. Configure the chart of accounts on the Ledger page
2. Have main accounts created for all required transaction types

A **posting type** defines a general category for a debit or credit. Each module has specific posting types that map to GL accounts via posting profiles.

## Module-Specific Posting Profiles

| Module | Navigation | Key Posting Types |
|---|---|---|
| **AP** | AP > Setup > Vendor posting profiles | Summary (AP control), Purchase expenditure un-invoiced, Cash discount, Arrival, Offset |
| **AR** | AR > Setup > Customer posting profiles | Summary (AR control), Revenue, Cash discount, Write-off, Interest, Collection letter fee |
| **Inventory** | Inventory management > Setup > Posting > Posting | Physical/Financial inventory, COGS, Purchase expenditure, Production issue/receipt, Variance |
| **FA** | Fixed assets > Setup > Fixed asset posting profiles | Acquisition, Depreciation, Accumulated depreciation, Disposal gain/loss |
| **Tax** | Tax > Setup > Sales tax > Ledger posting groups | Sales tax payable/receivable, Use tax expense/payable, Settlement |
| **Bank** | (On each bank account record) | Main account = GL account for the bank |
| **Production** | Production control > Setup > Production groups | WIP, Cost of goods manufactured |
| **Project** | Project management > Setup > Posting > Ledger posting setup | Revenue, Cost, WIP, Accrued revenue |

## Account Code Priority (Same Pattern for AP and AR)

| Account Code | Scope | Priority |
|---|---|---|
| **Table** | Specific vendor/customer | 1 (most specific — checked first) |
| **Group** | Vendor/customer group | 2 |
| **All** | All vendors/customers | 3 (fallback) |

## Recommended Practices (from Microsoft)

### Before Go-Live
1. **Mock period close:** Reconcile each subledger to the ledger before go-live
2. **Mock cutover:** Test all open balances and open transactions before initial go-live
3. **Validate posting profiles:** Every profile must have valid GL accounts for all transaction types

### Design Rules
- **Never leave account assignments blank** — system will either fail or post to an error account
- **Be consistent** — same accounts for same transaction types across related modules
- **Document all profiles** — posting profile changes after go-live cause reconciliation nightmares
- **Test with full transaction cycle** — PO receipt → invoice → payment → bank reconciliation

### Accounts for Automatic Transactions
Navigation: **General ledger > Posting setup > Accounts for automatic transactions**

System-generated transactions need default accounts:
| Transaction Type | Purpose |
|---|---|
| Penny difference | Round-off amounts |
| Year-end close | P&L to retained earnings transfer |
| Intercompany | Due-to/due-from |
| Currency revaluation | Unrealized gain/loss |
| Rounding | Rounding differences |
| Error account | Catch-all for unresolved postings |

## Reconciliation Checklist

After configuring all posting profiles:

| Check | How | Must Match |
|---|---|---|
| AP subledger | Vendor aging report total | = AP summary account in GL |
| AR subledger | Customer aging report total | = AR summary account in GL |
| Inventory | Inventory valuation report | = Inventory GL accounts |
| FA | FA net book value report | = FA GL accounts |
| Bank | Bank balance per account | = Bank GL accounts |
| GL | Trial balance | Debits = Credits |

**Automated:** Account Reconciliation feature (v10.0.44+):
Navigation: General ledger > Periodic tasks > Account reconciliation
