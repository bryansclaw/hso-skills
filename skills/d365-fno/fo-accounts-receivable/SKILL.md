---
name: fo-accounts-receivable
description: >
  Configure D365 F&O Accounts Receivable module including AR parameters,
  customer groups, customer posting profiles, payment methods, credit
  management, collections setup, interest codes, collection letters,
  aging definitions, and free text invoice templates. Also covers customer
  master data migration and open AR balance migration.
  Use when: setting up AR module, configuring customer payments, credit limits,
  collections processes, or subscription billing.
  DEPENDS ON: fo-gl-chart-of-accounts, fo-tax-configuration, fo-cash-bank-management.
  Level 140 in Microsoft's config template sequence.
  NOT for: sales order processing (use fo-sales-marketing), project invoicing.
metadata:
  platform:
    category: "fo-accounts-receivable"
    riskLevel: "medium"
    configSequence: 7
    configLevel: "140"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts", "fo-tax-configuration", "fo-cash-bank-management"]
    bpcProcess: "02 - Order to Cash"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-use-accounts-receivable-dyn365-finance/"
    mbExam: "MB-310 (15-20% weight: AR, credit, collections, subscription billing)"
    sbdPhase: "Implement"
---

# Accounts Receivable Configuration

## Microsoft Learn References
See `references/ms-learn-sources.md` for verified URLs covering AR setup, customer posting profiles, credit management, collections, aging, account reconciliation, and entity documentation.

## Prerequisites
- GL chart of accounts with AR-related main accounts (AR control, sales revenue, cash discount, write-off)
- Tax configuration completed (tax codes/groups for customers)
- Bank accounts configured (for customer payment methods)
- Financial dimension configuration for integrating applications set up

## Configuration Sequence

### Step 1: AR Parameters
- **Menu:** Accounts receivable > Setup > Accounts receivable parameters
- **MCP:** Form tool
- **Key tabs:** General, Settlement, Collections, Number sequences, Credit management

### Step 2: Customer Groups
- **Menu:** Accounts receivable > Setup > Customer groups
- **Data entity:** `CustomerGroups`
- **MCP:** Data tool

### Step 3: Customer Posting Profiles
- **Menu:** Accounts receivable > Setup > Customer posting profiles
- **Data entity:** `CustomerPostingProfiles`
- **MCP:** Data tool + Form tool validation
- **Critical accounts:** Summary (AR control), revenue, cash discount, write-off

### Step 4: Methods of Payment (Customer)
- **Menu:** Accounts receivable > Payment setup > Methods of payment
- **Data entity:** `CustomerPaymentMethods`
- **MCP:** Data tool
- **Common:** Check, Electronic, Credit card, Direct debit (SEPA DD)

### Step 5: Credit Management Setup
- **Menu:** Credit and collections > Setup > Credit management parameters
- **MCP:** Form tool
- **Key settings:** Credit limit rules, blocking rules, scoring groups, risk groups
- **Business Event:** Credit hold applied → Power Automate notification to credit manager

### Step 6: Collections Setup
- **Menu:** Credit and collections > Setup > Collections
- **MCP:** Form tool
- **Components:** Aging period definitions, collection letter sequences, interest codes, write-off journal

### Step 7: Aging Period Definitions
- **Menu:** Credit and collections > Setup > Aging period definitions
- **MCP:** Form tool
- **Purpose:** Define aging buckets (Current, 1-30, 31-60, 61-90, 91+)

### Step 8: Collection Letters
- **Menu:** Accounts receivable > Setup > Collections > Collection letter sequence
- **MCP:** Form tool
- **Purpose:** Automated dunning letter escalation

### Step 9: Interest Codes
- **Menu:** Accounts receivable > Setup > Interest > Interest codes
- **Data entity:** `InterestCodes`
- **MCP:** Data tool

### Step 10: Free Text Invoice Templates
- **Menu:** Accounts receivable > Setup > Free text invoice templates
- **MCP:** Form tool
- **Purpose:** Recurring invoice templates

### Step 11: AR Workflows
- **Menu:** Accounts receivable > Setup > Accounts receivable workflows
- **MCP:** Form tool

## Customer Master Data Migration

| Seq | Entity | Data Entity Name | Dependencies |
|---|---|---|---|
| 1 | Customer groups | `CustomerGroups` | GL main accounts |
| 2 | Posting profiles | `CustomerPostingProfiles` | Groups, main accounts |
| 3 | Payment terms | `PaymentTerms` | Shared with AP |
| 4 | Methods of payment | `CustomerPaymentMethods` | Bank accounts |
| 5 | Customers | `CustomersV3` | Groups, terms, tax groups, currencies |
| 6 | Customer bank accounts | `CustomerBankAccounts` | Customers |
| 7 | Customer contacts | `CustomerContacts` | Customers, GAB |

**Note on CustomersV3:** This entity does NOT support multi-threading (Import task count must = 1). "Custom sequence is defined, more than one task is not supported."

## Opening AR Balances
- Import open customer invoices via AR journal (NOT directly to GL)
- Creates AR subledger entries AND posts to AR control account
- After import: AR aging report total must equal AR control account in GL
- Use Account Reconciliation feature (v10.0.44+)

## Dual-Write Awareness
- Customers sync to Dataverse: `CustomersV3` → `account` table
- Requires supporting maps: CDS Parties, CDS Party Postal Addresses, Party Contacts V3
- Customer changes in F&O propagate to Dataverse in near-real-time (and vice versa)

## Validation Checks
- [ ] AR parameters configured (all tabs)
- [ ] Customer posting profiles have valid GL accounts
- [ ] Payment methods configured with bank accounts
- [ ] Credit management parameters set (if using credit management)
- [ ] Aging period definitions created
- [ ] Customer number sequence active
- [ ] Test: Create customer → create free text invoice → post → collect payment

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`, `fo-tax-configuration`, `fo-cash-bank-management`
- **Related:** `fo-sales-marketing`, `fo-accounts-payable` (shared setup)
