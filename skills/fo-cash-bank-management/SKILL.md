---
name: fo-cash-bank-management
description: >
  Configure D365 F&O Cash and Bank management including bank groups, bank
  accounts, bank transaction types, reconciliation rules, payment formats
  (NACHA, SEPA, Positive Pay), Electronic Reporting configurations, advanced
  bank reconciliation, and cash flow forecasting. Use when: setting up bank
  accounts, configuring payment file formats, bank reconciliation, or cash
  flow forecasting. DEPENDS ON: fo-gl-chart-of-accounts (bank GL accounts).
  Level 100 in Microsoft's config template sequence.
  NOT for: customer/vendor payment method setup (use fo-accounts-payable or
  fo-accounts-receivable — though they depend on this skill).
metadata:
  platform:
    category: "fo-cash-bank"
    riskLevel: "medium"
    configSequence: 5
    configLevel: "100"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "13.20 - Record to Report > Manage cash"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-cash-bank-management-dyn365-finance/"
    mbExam: "MB-310 (within financial management 40-45%)"
    sbdPhase: "Implement"
---

# Cash and Bank Management Configuration

## Prerequisites
- GL main accounts for bank accounts (asset accounts)
- Legal entity currency configured in ledger setup

## Configuration Sequence

### Step 1: Bank Groups
- **Menu:** Cash and bank management > Setup > Bank groups
- **Data entity:** `BankGroups`
- **MCP:** Data tool
- **Purpose:** Classify bank accounts (domestic, international, payroll)

### Step 2: Bank Transaction Types
- **Menu:** Cash and bank management > Setup > Bank transaction types
- **Data entity:** `BankTransactionTypes`
- **MCP:** Data tool
- **Purpose:** Classify bank transaction categories for reconciliation matching

### Step 3: Bank Accounts
- **Menu:** Cash and bank management > Bank accounts > Bank accounts
- **Data entity:** `BankAccountsV2`
- **MCP:** Data tool for creation, Form tool for advanced settings
- **Critical fields:** Bank account number, routing number, SWIFT/BIC, IBAN, currency, main GL account, bank group
- **Note:** Each bank account links to a GL main account — verify account exists

### Step 4: Electronic Reporting (ER) Payment Formats
- **Menu:** Organization administration > Electronic reporting > Configurations
- **MCP:** Form tool (must import from Microsoft's Dataverse repository)
- **Common formats:**

| Region | Format Name | Use |
|---|---|---|
| US | NACHA (Generic electronic format) | ACH vendor payments |
| US | Positive Pay | Check fraud prevention |
| Europe | ISO20022 Credit Transfer | SEPA vendor payments |
| Europe | ISO20022 Direct Debit | SEPA customer collections |
| Global | MT940 | Bank statement import |
| Global | BAI2 | Bank statement import (US) |
| Global | camt.053 | Bank statement import (ISO 20022) |

**Setup for ACH/NACHA (US):**
1. Import NACHA ER configuration from Dataverse repository
2. On payment method: File format FastTab → "Generic electronic export format = Yes"
3. Assign ER configuration in "Export format configuration" field

### Step 5: Check Layout (if using checks)
- **Menu:** Cash and bank management > Setup > Check layout
- **MCP:** Form tool
- **Purpose:** Define check printing format

### Step 6: Advanced Bank Reconciliation Setup
- **Menu:** Cash and bank management > Setup > Advanced bank reconciliation setup
- **MCP:** Form tool
- **Components:**
  - Reconciliation matching rules (auto-match criteria)
  - Bank statement format (for import)
  - Statement import mapping
- **Modern Bank Reconciliation (v10.0.44+):**
  - Automated transaction matching
  - Direct subledger postings
  - Configurable GL posting descriptions

### Step 7: Cash Flow Forecasting
- **Menu:** Cash and bank management > Setup > Cash flow forecasting
- **MCP:** Form tool
- **Features:**
  - AI-driven cash flow forecasts (included in D365 Finance)
  - Compare forecasts to snapshots and actuals
  - Integrates with BPA (Business Performance Analytics) for MCP Analytics queries

## Validation Checks
- [ ] Bank groups created
- [ ] Bank accounts have valid GL main accounts assigned
- [ ] Bank account currencies match or are supported
- [ ] ER payment format imported and assigned to payment methods
- [ ] Bank reconciliation matching rules defined
- [ ] Bank statement format configured for statement import
- [ ] Number sequences configured for bank transactions
- [ ] Test: Create a payment journal → generate payment file → verify file format

## Common Errors
- "Bank account not found" → Bank account not set up or wrong legal entity context
- "Export format not configured" → ER configuration not imported or not assigned to payment method
- "Reconciliation matching rule failed" → Rules too strict for initial data
- Bank statement import errors → Statement format mapping incorrect

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`
- **Used by:** `fo-accounts-payable` (vendor payment methods), `fo-accounts-receivable` (customer payment methods)
- **Analytics:** Cash flow forecasting data available through BPA MCP Analytics server
