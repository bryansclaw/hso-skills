# Configure Ledger (per Legal Entity)

Source: Microsoft Learn — verified 2026-03-18
Navigation: **General ledger > Ledger setup > Ledger**

## What the Ledger Page Does

For each legal entity, the Ledger page assigns:
- Chart of accounts
- Account structures
- Fiscal calendar
- Accounting currency (home currency)
- Reporting currency (optional second currency)
- Exchange rate type

## Steps

### Select Chart of Accounts
1. On the Ledger page, select **Chart of accounts** field
2. Choose the CoA from the dropdown
3. Multiple legal entities can share the same CoA
4. If legal entities need different CoAs, create separate ones

**Warning: You CANNOT change the CoA after transactions have been posted.** The system blocks the change and shows a detailed error listing every affected posting profile (table, field, company). You must clear all posting profile references first. If the bank account table is listed as a blocker, contact Microsoft support — the main account field on bank accounts is required and can't be cleared in the application.

### Select Account Structures
1. In the **Account structures** section, click **Add**
2. Select account structure(s) from the list
3. Click **Select**
4. Saving triggers synchronization of unposted transactions — wait for completion

**Rules:**
- Can assign multiple account structures to one ledger
- Account structures must NOT have overlapping main account ranges
- Example of invalid overlap: Structure A covers 100000-399999, Structure B covers 200000-499999 — ranges overlap at 200000-399999

### Select Fiscal Calendar
- Choose the calendar that defines fiscal years and periods for this legal entity
- Each legal entity can use a different calendar
- Changing the calendar after transactions requires careful recalculation

### Set Currencies
- **Accounting currency:** The primary currency for all GL postings in this entity
- **Reporting currency:** Optional secondary currency for dual-currency reporting
- Select Exchange rate type for automatic rate lookup

## Legal Entity Overrides
Not all main accounts are valid for all legal entities. Use the Legal entity overrides section on main accounts to:
- Suspend an account for a specific company
- Set the owner of an account
- Define the period when the account is active (Active from / Active to dates)
