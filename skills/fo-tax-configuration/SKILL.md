---
name: fo-tax-configuration
description: >
  Configure D365 F&O tax setup including sales tax authorities, settlement
  periods, sales tax codes, sales tax groups, item sales tax groups, ledger
  posting groups, withholding tax, and tax registration numbers. Use when:
  setting up tax for any jurisdiction, configuring tax calculation, or
  managing tax settlement. DEPENDS ON: fo-gl-chart-of-accounts (needs main
  accounts for tax ledger posting groups). Level 130 in Microsoft's config
  template sequence — must be configured before AP and AR.
  NOT for: Avalara/Vertex ISV tax engine setup, e-invoicing (RCS), or
  country-specific localizations.
metadata:
  platform:
    category: "fo-tax"
    riskLevel: "high"
    configSequence: 5
    configLevel: "40-130"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "13.10 - Record to Report > Define accounting policies (tax)"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/work-tax-dyn365-finance/"
    mbExam: "MB-310 (within financial management 40-45%)"
    sbdPhase: "Implement"
---

# Tax Configuration

## Prerequisites
- GL chart of accounts configured with tax-related main accounts (sales tax payable, sales tax receivable, use tax expense, etc.)
- Legal entities configured with correct country/region (determines tax localization features)

## Configuration Sequence

### Cross-Module Band Level 40 (shared tax setup)

#### Step 1: Sales Tax Authorities
- **Menu:** Tax > Indirect taxes > Sales tax > Sales tax authorities
- **Data entity:** `SalesTaxAuthorities`
- **MCP:** Data tool
- **Note:** Creating an authority auto-creates a vendor record for tax payments
- **Key fields:** Authority name, rounding form, round-off amount, report layout

#### Step 2: Settlement Periods
- **Menu:** Tax > Indirect taxes > Sales tax > Sales tax settlement periods
- **Data entity:** `SalesTaxSettlementPeriods`
- **MCP:** Data tool for header, Form tool for period intervals
- **Purpose:** Define the calendar for tax filing (monthly, quarterly, annual)
- **Dependency:** Requires tax authority from Step 1

#### Step 3: Ledger Posting Groups
- **Menu:** Tax > Setup > Sales tax > Ledger posting groups
- **Data entity:** `TaxLedgerPostingGroups`
- **MCP:** Data tool
- **Critical accounts:** Sales tax payable, sales tax receivable, use tax expense, use tax payable, settlement account
- **Note:** Must have valid GL main accounts — verify before creating

#### Step 4: Sales Tax Codes
- **Menu:** Tax > Indirect taxes > Sales tax > Sales tax codes
- **Data entity:** `SalesTaxCodes`
- **MCP:** Data tool for codes, then add values
- **Key fields:** Tax code, name, settlement period, ledger posting group, percentage/amount
- **Dependencies:** Settlement period (Step 2), ledger posting group (Step 3)

#### Step 5: Sales Tax Code Values
- **Menu:** (within sales tax codes > Values button)
- **Data entity:** `SalesTaxCodeValues`
- **MCP:** Data tool
- **Defines:** Tax rate percentages per date range

#### Step 6: Sales Tax Groups
- **Menu:** Tax > Indirect taxes > Sales tax > Sales tax groups
- **Data entity:** `SalesTaxGroups`
- **MCP:** Data tool
- **Purpose:** Assign to customers, vendors, legal entities — determines WHICH tax codes apply to a transaction
- **Contains:** One or more tax codes

#### Step 7: Item Sales Tax Groups
- **Menu:** Tax > Indirect taxes > Sales tax > Item sales tax groups
- **Data entity:** `ItemSalesTaxGroups`
- **MCP:** Data tool
- **Purpose:** Assign to products/items — determines tax treatment of the ITEM
- **Tax calculation:** Intersection of sales tax group + item sales tax group = applicable tax codes

### Module-Level 130 (tax-specific)

#### Step 8: Tax Exempt Numbers
- **Menu:** Tax > Setup > Sales tax > Tax exempt numbers
- **Data entity:** `TaxExemptNumbers`
- **MCP:** Data tool
- **Purpose:** Track tax-exempt customer/vendor registrations

#### Step 9: Withholding Tax (if applicable)
- **Menu:** Tax > Indirect taxes > Withholding tax > Withholding tax codes/groups
- **Data entities:** `WithholdingTaxCodes`, `WithholdingTaxGroups`
- **MCP:** Data tool
- **Note:** India-specific withholding tax entities (TDS/TCS) have country-region restrictions — must have Indian legal entity address

#### Step 10: Tax Registration Numbers
- **Menu:** Organization administration > Global address book > Registration types
- **MCP:** Form tool
- **Purpose:** VAT registration, tax ID numbers per legal entity/customer/vendor

## Tax Calculation Logic
```
Transaction tax = Intersection of:
  ├── Sales Tax Group (on customer/vendor/header)
  └── Item Sales Tax Group (on product/line)
  = Tax codes that appear in BOTH groups
  → Rate from tax code values for the transaction date
```

## ISV Tax Engine Awareness
If the customer uses Avalara, Vertex, or Thomson Reuters:
- Native D365 tax calculation may be overridden by ISV engine
- Sales tax groups/codes still need to exist for GL posting
- ISV typically handles rate lookup and calculation
- D365 handles posting via ledger posting groups
- Agent should check: is ISV tax installed? If yes, defer rate configuration to ISV

## Validation Checks
- [ ] At least one tax authority exists per jurisdiction
- [ ] Settlement periods have period intervals defined
- [ ] Ledger posting groups have valid GL accounts
- [ ] Tax codes have rate values for the implementation date range
- [ ] Sales tax groups contain the correct tax codes
- [ ] Item sales tax groups exist for each product tax category (taxable, exempt, zero-rated)
- [ ] Tax calculation test: Create a test PO/SO → verify correct tax amount calculated
- [ ] Tax settlement test: Run settlement process for test period → verify GL posting

## Common Errors
- "Tax code not found for this combination" → Tax code not in both groups (intersection is empty)
- "Settlement period not defined" → Period intervals missing on settlement period
- "Ledger posting group not assigned" → Tax code missing posting group reference
- "Withholding tax registration" error → Entity country-region restriction (India-specific entities)
- Tax amount = 0 on transactions → Check tax code values date range vs transaction date

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`
- **Used by:** `fo-accounts-payable`, `fo-accounts-receivable`, `fo-procurement`, `fo-sales-marketing`
