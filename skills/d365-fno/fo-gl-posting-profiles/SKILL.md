---
name: fo-gl-posting-profiles
description: >
  Configure D365 F&O posting profiles that control how subledger transactions
  flow to the general ledger. Covers vendor posting profiles, customer posting
  profiles, inventory posting profiles, fixed asset posting profiles, and
  accounts for automatic transactions. This is the critical bridge between
  every module and GL — misaligned posting profiles are the #1 cause of
  subledger-to-GL reconciliation failures.
  Use when: setting up how AP/AR/Inventory/FA/Bank transactions post to GL,
  troubleshooting GL reconciliation issues, or reviewing posting configuration.
  DEPENDS ON: fo-gl-chart-of-accounts, plus the module being configured.
metadata:
  platform:
    category: "fo-general-ledger"
    riskLevel: "high"
    configSequence: 4
    configLevel: "25-300"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "13.10 - Record to Report > Define accounting policies"
    msLearnPath: "https://learn.microsoft.com/en-us/dynamics365/finance/general-ledger/pstg-prfles-ovrvw"
    mbExam: "MB-310"
---

# Posting Profiles — GL Bridge Configuration

## Why This Matters

Every module in D365 generates financial transactions that must post to the correct GL accounts. Posting profiles are the mapping layer that determines WHICH GL accounts receive WHICH transaction types. Get this wrong and:
- Subledger balances won't match GL control accounts
- Financial reports will be incorrect
- Period close will reveal unexplained differences
- Audit findings will result

**Microsoft's guidance:** *"Do a mock period close and reconcile each of your subledgers to the ledger before your go-live."*

## Posting Profile Types

### Vendor Posting Profiles
- **Menu:** Accounts payable > Setup > Vendor posting profiles
- **Data entity:** `VendorPostingProfiles`
- **Key accounts:** Summary (AP control), purchase expenditure for un-invoiced, cash discount, realized gain/loss
- **Levels:** All vendors / Vendor group / Specific vendor (Table)

### Customer Posting Profiles
- **Menu:** Accounts receivable > Setup > Customer posting profiles
- **Data entity:** `CustomerPostingProfiles`
- **Key accounts:** Summary (AR control), sales revenue, cash discount, write-off, interest
- **Levels:** All customers / Customer group / Specific customer (Table)

### Inventory Posting Profiles
- **Menu:** Inventory management > Setup > Posting > Posting
- **MCP:** Form tool (complex multi-tab matrix)
- **Transaction types and accounts:**
  - **Physical:** Physical inventory receipt, Physical inventory issue
  - **Financial:** Financial inventory receipt, Financial inventory issue, Cost of goods sold
  - **Production:** Pick list (issue), Report as finished (receipt), WIP
  - **Purchase:** Purchase expenditure for product, Purchase expenditure un-invoiced, Purchase accrual
  - **Sales:** Revenue, Cost of goods sold
  - **Variance:** Purchase price variance, Production price variance, Inventory cost revaluation
- **Grouped by:** Item group, Item group + Category, or All
- **⚠️ This is the most complex posting configuration and the most common source of reconciliation errors**

### Fixed Asset Posting Profiles
- **Menu:** Fixed assets > Setup > Fixed asset posting profiles
- **Key accounts:** Acquisition, Depreciation, Disposal (gain/loss), Net book value
- **Levels:** All / Fixed asset group / Specific asset

### Tax Ledger Posting Groups
- **Menu:** Tax > Setup > Sales tax > Ledger posting groups
- **Key accounts:** Sales tax payable, Sales tax receivable, Use tax expense, Settlement
- (Covered in `fo-tax-configuration` skill)

### Bank Transaction Posting
- **Menu:** Cash and bank management > Bank accounts > [account] > General tab > Main account
- Each bank account maps directly to a GL main account
- (Covered in `fo-cash-bank-management` skill)

### Accounts for Automatic Transactions
- **Menu:** General ledger > Posting setup > Accounts for automatic transactions
- **MCP:** Form tool
- **Purpose:** Default GL accounts for system-generated transactions:
  - Penny difference
  - Year-end close (profit/loss to retained earnings)
  - Intercompany due-to/due-from
  - Currency revaluation gain/loss
  - Rounding differences
  - Error account (catch-all for unresolved postings)

## Configuration Best Practices

### Before You Start
1. Have a complete chart of accounts with ALL required accounts
2. Map each module's transaction types to GL accounts in a spreadsheet FIRST
3. Get sign-off from the customer's controller/CFO on the GL account mapping
4. Configure ONE legal entity completely, test, then copy to others

### Posting Profile Design Rules
- **Never leave account assignments blank** — D365 will either fail or use an error account
- **Be consistent** — same GL accounts for the same transaction types across modules
- **Document everything** — posting profile changes after go-live cause reconciliation nightmares
- **Test with a full transaction cycle** — PO receipt → vendor invoice → payment → bank reconciliation

### Reconciliation Points (Must Balance)
| Subledger | GL Control Account | Reconciliation Report |
|---|---|---|
| AP subledger total | AP control account(s) | Vendor balance report vs GL trial balance |
| AR subledger total | AR control account(s) | Customer balance report vs GL trial balance |
| Inventory valuation | Inventory control account(s) | Inventory valuation report vs GL |
| Fixed asset NBV | FA control accounts | FA net book value vs GL |
| Bank subledger | Bank GL accounts | Bank balance vs GL |

### Account Reconciliation Feature (v10.0.44+)
- **Menu:** General ledger > Periodic tasks > Account reconciliation
- Automated reconciliation of GL with AP, AR, tax, and bank subledgers
- Our Validation Agent should run this after every migration stage

## MCP Approach
- **Data tools:** Vendor posting profiles, customer posting profiles (basic structure)
- **Form tools:** Inventory posting profiles (complex multi-tab matrix), FA posting profiles, accounts for automatic transactions
- **Validation:** After configuration, use `data_find_entities` to verify posting profile accounts match expected GL accounts

## Validation Checks
- [ ] Vendor posting profiles: summary account (AP control) assigned for all/each group
- [ ] Customer posting profiles: summary account (AR control) assigned for all/each group
- [ ] Inventory posting: accounts assigned for ALL transaction types (physical, financial, purchase, sales, production, variance)
- [ ] FA posting profiles: acquisition, depreciation, disposal accounts assigned
- [ ] Tax posting groups: payable, receivable, settlement accounts assigned
- [ ] Automatic transaction accounts: year-end close, rounding, error accounts assigned
- [ ] **RECONCILIATION TEST:** Post a full transaction cycle → verify subledger = GL control

## Common Errors
- "Posting type X does not have an account specified" → Missing account in posting profile
- Subledger-GL mismatch → Posting profile changed after transactions posted
- Unexpected variance amounts → Inventory posting profile accounts inconsistent between purchase and sales
- Year-end close failure → Retained earnings account not configured in automatic transactions
- Intercompany imbalance → Due-to/due-from accounts not configured or mismatched

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`
- **Used by:** ALL module skills (AP, AR, Inventory, FA, Production, etc.)
- **Validation:** `fo-data-migration` (opening balance reconciliation depends on correct posting profiles)
