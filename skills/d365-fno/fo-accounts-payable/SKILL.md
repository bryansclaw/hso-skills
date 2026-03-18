---
name: fo-accounts-payable
description: >
  Configure D365 F&O Accounts Payable module including AP parameters, vendor
  groups, vendor posting profiles, payment methods, invoice validation policies,
  1099 setup (US), vendor approval workflows, and charge codes. Also covers
  vendor master data migration and open AP balance migration.
  Use when: setting up AP module, configuring vendor payments, payment formats
  (NACHA/ACH, SEPA), invoice matching, or managing vendor workflows.
  DEPENDS ON: fo-gl-chart-of-accounts (main accounts for posting profiles),
  fo-tax-configuration (tax groups for vendors), fo-cash-bank-management (bank
  accounts for payment methods).
  NOT for: procurement/purchasing (use fo-procurement), expense management.
metadata:
  platform:
    category: "fo-accounts-payable"
    riskLevel: "medium"
    configSequence: 6
    configLevel: "120"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts", "fo-tax-configuration", "fo-cash-bank-management"]
    bpcProcess: "05 - Source to Pay"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-use-accounts-payable-dyn365-finance/"
    mbExam: "MB-310 (10-15% weight: Implement and manage AP and expenses)"
    sbdPhase: "Implement"
---

# Accounts Payable Configuration

## Microsoft Learn References
See `references/ms-learn-sources.md` for verified URLs covering AP setup, vendor posting profiles, invoice automation, three-way matching, NACHA/ER payment formats, and entity documentation.

## Prerequisites
- GL chart of accounts configured with AP-related main accounts (AP control, purchase discount, purchase variance, etc.)
- Tax configuration completed (tax codes, groups for vendors)
- Bank accounts set up (required for payment methods)
- Financial dimension configuration for integrating applications set up (for vendor default dimensions)

## Configuration Sequence

### Step 1: AP Parameters
- **Menu:** Accounts payable > Setup > Accounts payable parameters
- **MCP:** Form tool (multi-tab parameter form)
- **Key tabs:** General, Payment, Settlement, Number sequences, Invoice validation
- **Critical settings:**
  - Invoice matching: 2-way or 3-way matching
  - Posting date controls
  - Cash discount administration
  - Vendor invoice automation settings

### Step 2: Vendor Groups
- **Menu:** Accounts payable > Setup > Vendor groups
- **Data entity:** `VendorGroups`
- **MCP:** Data tool
- **Purpose:** Classify vendors (trade, service, intercompany, one-time)
- **Note:** Groups link to posting profiles — set up groups before posting profiles

### Step 3: Vendor Posting Profiles
- **Menu:** Accounts payable > Setup > Vendor posting profiles
- **Data entity:** `VendorPostingProfiles`
- **MCP:** Data tool, but validate with Form tool (complex account assignments)
- **Critical accounts:** Summary account (AP control), purchase expenditure, cash discount
- **Note:** Can be set at All/Group/Table level. Validate main accounts exist first.

### Step 4: Methods of Payment (Vendor)
- **Menu:** Accounts payable > Payment setup > Methods of payment
- **Data entity:** `VendorPaymentMethods`
- **MCP:** Data tool for basic setup, Form tool for payment format assignment
- **Common methods:** Check, Electronic (ACH/NACHA), Wire, Credit card
- **ER Configuration required:** For electronic payments, must assign Electronic Reporting format:
  - US: NACHA/ACH format (Organization admin > Electronic reporting > import from Microsoft repository)
  - Europe: SEPA Credit Transfer (ISO 20022)
  - Set "Generic electronic export format = Yes" on File format FastTab

### Step 5: Payment Terms & Cash Discounts
- **Menu:** Accounts payable > Payment setup > Terms of payment / Cash discounts
- **Data entities:** `PaymentTerms`, `CashDiscounts`
- **MCP:** Data tool
- **Note:** Shared with AR — may already exist. Check before creating duplicates.

### Step 6: Charges Codes
- **Menu:** Accounts payable > Charges setup > Charges code
- **Data entity:** `MarkupTable` (shared charges entity)
- **MCP:** Data tool
- **Purpose:** Define freight, handling, insurance and other charges

### Step 7: Invoice Validation Policies
- **Menu:** Accounts payable > Setup > Invoice validation > Invoice matching policies
- **MCP:** Form tool
- **Settings:** Price tolerance, quantity tolerance, charges matching
- **Related:** Invoice totals tolerances, line-level matching

### Step 8: 1099 Setup (US only)
- **Menu:** Accounts payable > Setup > Tax 1099 > 1099 fields
- **MCP:** Form tool
- **Purpose:** US tax reporting for independent contractors

### Step 9: AP Workflows
- **Menu:** Accounts payable > Setup > Accounts payable workflows
- **MCP:** Form tool (visual workflow designer)
- **Common workflows:** Vendor invoice approval, payment journal approval, vendor change approval

## Vendor Master Data Migration

### Entity Sequence
| Seq | Entity | Data Entity Name | Dependencies |
|---|---|---|---|
| 1 | Vendor groups | `VendorGroups` | GL main accounts |
| 2 | Posting profiles | `VendorPostingProfiles` | Vendor groups, main accounts |
| 3 | Payment terms | `PaymentTerms` | None (shared with AR) |
| 4 | Cash discounts | `CashDiscounts` | Main accounts |
| 5 | Methods of payment | `VendorPaymentMethods` | Bank accounts, ER configs |
| 6 | Vendors | `VendorsV2` | Groups, payment terms, tax groups, currencies, addresses |
| 7 | Vendor bank accounts | `VendorBankAccounts` | Vendors |
| 8 | Vendor contacts | `VendorContacts` | Vendors, GAB |

### Pre-Migration Validation
Before importing vendors, the agent must verify:
```
1. data_find_entities("VendorGroups") → all referenced groups exist
2. data_find_entities("PaymentTerms") → all referenced terms exist
3. data_find_entities("SalesTaxGroups") → all referenced tax groups exist
4. data_find_entities("Currencies") → all referenced currencies exist
5. Check for duplicate vendor names/tax IDs across source data
6. Validate address data (country/state/city hierarchy complete)
```

### Opening AP Balances
- Import open vendor invoices via AP journal (NOT directly to GL)
- This creates both AP subledger entries AND posts to AP control account
- After import: AP aging report total must equal AP control account in GL
- Use Account Reconciliation feature (v10.0.44+) for automated verification

## ISV Awareness
- **Tax:** If Avalara or Vertex is used, tax calculation may override native tax engine on vendor invoices
- **AP Automation:** Kofax, ABBYY, or Tungsten Network may handle invoice capture — AP agent needs awareness of ISV-created pending invoices
- **EDI:** TrueCommerce/SPS Commerce may auto-create vendor invoices from EDI 810 documents

## Power Automate Integration
- **Business Event:** "Vendor invoice approved" → trigger notification flow
- **Business Event:** "Payment journal posted" → trigger bank file generation
- **Approval flow:** Vendor change requests → Power Automate approval → Teams adaptive card

## Validation Checks
- [ ] AP parameters configured (all tabs reviewed)
- [ ] Vendor posting profiles have valid GL accounts
- [ ] At least one payment method configured with valid bank account
- [ ] Electronic payment format (ER) imported and assigned (if using electronic payments)
- [ ] Invoice matching policy set (2-way or 3-way)
- [ ] Vendor number sequence active
- [ ] 1099 fields configured (US implementations only)
- [ ] Test: Create a vendor → create PO → receive → invoice → match → post → pay

## Common Errors
- "Posting profile not found" → Vendor group not mapped to posting profile
- "Summary account not specified" → Posting profile missing AP control account
- "Payment method not valid" → Bank account not assigned to payment method
- "Invoice matching failed" → Tolerance percentages too tight for initial testing
- "Number sequence exhausted" → Vendor number sequence range too small

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`, `fo-tax-configuration`, `fo-cash-bank-management`
- **Related:** `fo-procurement` (purchase orders), `fo-accounts-receivable` (shared setup)
- **Integration:** `fo-integration-patterns` (dual-write vendor sync to CE/Dataverse)
