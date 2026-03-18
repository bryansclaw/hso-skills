---
name: fo-gl-chart-of-accounts
description: >
  Design and configure D365 F&O chart of accounts, main accounts, financial
  dimensions, account structures, advanced rules, ledger setup, and financial
  dimension configuration for integrating applications. Use when: setting up a
  new chart of accounts, modifying account structures, adding financial dimensions,
  configuring main account categories, or setting up the ledger for a legal entity.
  DEPENDS ON: fo-org-admin (legal entities, fiscal calendars must exist first).
  This is Level 20-25 in Microsoft's config template sequence — the foundation
  for ALL financial modules. Every other module posts to GL.
  NOT for: journal operations (use fo-gl-journals), period close (use fo-period-close),
  tax configuration (use fo-tax-configuration).
metadata:
  platform:
    category: "fo-general-ledger"
    riskLevel: "high"
    configSequence: 2
    configLevel: "20-25"
    requires:
      products: ["F&O"]
      skills: ["fo-org-admin"]
    bpcProcess: "13.10 - Record to Report > Define accounting policies"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-use-general-ledger-dyn365-finance/"
    mbExam: "MB-310 (40-45% weight: Implement financial management)"
    sbdPhase: "Implement"
---

# General Ledger — Chart of Accounts & Ledger Configuration

## Microsoft Learn References
See `references/ms-learn-sources.md` for verified URLs and detailed guidance from:
- [Plan chart of accounts](https://learn.microsoft.com/en-us/dynamics365/finance/general-ledger/plan-chart-of-accounts)
- [Configure ledger](https://learn.microsoft.com/en-us/dynamics365/finance/general-ledger/configure-ledger)
- [Configure account structures](https://learn.microsoft.com/en-us/dynamics365/finance/general-ledger/configure-account-structures)
- [Main account types](https://learn.microsoft.com/en-us/dynamics365/finance/general-ledger/main-account-types)
- [Financial dimensions](https://learn.microsoft.com/en-us/dynamics365/finance/general-ledger/financial-dimensions)
- [Financial dimension config for integration](https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/financial/financial-dimension-configuration-integration)
- [Posting profiles overview](https://learn.microsoft.com/en-us/dynamics365/finance/general-ledger/pstg-prfles-ovrvw)

## Prerequisites
- `fo-org-admin` completed (legal entities exist, fiscal calendars created)
- System Administrator or GL Administrator security role

## ⚠️ CRITICAL PRE-STEP: Financial Dimension Configuration for Integrating Applications

**This MUST be configured before ANY data import involving dimensions. This is the #1 missed prerequisite that causes import failures.**

- **Menu:** General ledger > Chart of accounts > Dimensions > Financial dimension configuration for integrating applications
- **MCP approach:** Form tool (requires navigating tabs and activating formats)

### Required Format Types
| Format Type | Required For | Includes Main Account? |
|---|---|---|
| **Ledger dimension format** | Journal imports, transaction data | Yes (first segment) |
| **Default dimension format** | Customer/vendor/product default dimensions | No |
| **Budget register dimension format** | Budget entries | Context-dependent |
| **Budget planning dimension format** | Budget plans | Context-dependent |

### Rules
- Only ONE format per type can be active at a time
- This is a GLOBAL setting — applies to ALL legal entities
- Format must include ALL dimensions across ALL legal entities (unused ones left blank in import files)
- If not configured → import fails with cryptic error messages
- Must match the segment order in your import files exactly

## Configuration Sequence

### Phase 1: Shared GL Setup (Level 20)

#### Step 1: Fiscal Calendars
- **Menu:** General ledger > Calendars > Fiscal calendars
- **Data entity:** `FiscalCalendars`
- **MCP:** Data tool
- **Then:** Create fiscal years and periods within each calendar
- **Entities:** `FiscalCalendarYears`, `FiscalCalendarPeriods`
- **Supports:** Standard (12-month), 4-4-5, 13-period, custom structures

#### Step 2: Currency Codes
- **Menu:** General ledger > Currencies > Currencies
- **Data entity:** `Currencies`
- **MCP:** Data tool
- **Note:** At minimum, define accounting currency and reporting currency

#### Step 3: Exchange Rate Types & Rates
- **Menu:** General ledger > Currencies > Exchange rate types
- **Data entity:** `ExchangeRateTypes`, `ExchangeRates`
- **MCP:** Data tool for types, Data tool for rates
- **Note:** D365 includes APIs to auto-import rates from OANDA or other providers

#### Step 4: Chart of Accounts (Structure)
- **Menu:** General ledger > Chart of accounts > Accounts > Chart of accounts
- **Data entity:** `ChartOfAccounts`
- **MCP:** Data tool
- **Design decision:** Shared CoA across legal entities vs unique per entity
- **Note:** Chart of accounts is a container — main accounts are added separately

#### Step 5: Main Account Categories
- **Menu:** General ledger > Chart of accounts > Accounts > Main account categories
- **Data entity:** `MainAccountCategories`
- **MCP:** Data tool
- **Categories:** Revenue, Expense, Asset, Liability, Equity (mapped to financial reporting)
- **Tip:** Align to GAAP/IFRS reporting requirements

#### Step 6: Financial Dimensions (Definitions)
- **Menu:** General ledger > Chart of accounts > Dimensions > Financial dimensions
- **Data entity:** `FinancialDimensions`
- **MCP:** Data tool
- **Two types:**
  - **Custom dimensions** — standalone value lists
  - **Entity-backed dimensions** — backed by existing tables (Customer, Vendor, Project, etc.)
- **Common dimensions:** Department, Cost Center, Business Unit, Project

### Phase 2: Company-Specific GL Setup (Level 25)

#### Step 7: Main Accounts
- **Menu:** General ledger > Chart of accounts > Accounts > Main accounts
- **Data entity:** `MainAccounts`
- **MCP:** Data tool (high volume — can be hundreds/thousands)
- **Critical fields:** Account number, name, main account type, main account category, posting type, currency restrictions
- **Main account types (from MS Learn):**
  - **Profit and loss / Revenue / Expense** — P&L posting accounts. Revenue+Expense have same function as P&L.
  - **Balance sheet / Asset / Liability / Equity** — balance sheet transaction accounts
  - **Total** — display account interval totals (configured via Account interval page). Cannot be suspended; simulate with Active from/to dates.
  - **Reporting** — Brazil financial statement reporting only
- **Do NOT use delimiter characters in account numbers** (e.g., if delimiter is "-", don't name accounts "1000-01" — causes parsing issues)
- **Legal entity overrides:** Can suspend accounts per company or set active periods

#### Step 8: Financial Dimension Values
- **Menu:** General ledger > Chart of accounts > Dimensions > Financial dimension values
- **Data entity:** `FinancialDimensionValues` (or entity-backed source tables)
- **MCP:** Data tool
- **Note:** Entity-backed dimensions (like Department) are populated by creating records in the backing table (OperatingUnits with type=Department)

#### Step 9: Account Structures
- **Menu:** General ledger > Chart of accounts > Structures > Configure account structures
- **MCP:** Form tool (complex visual UI — cannot use data tools)
- **Purpose:** Define which main account + dimension combinations are valid for posting
- **From MS Learn:**
  - Maximum 11 segments per structure. If >11 needed, evaluate if dimension can be derived via hierarchy instead.
  - Advanced rules can add up to 16 segments total.
  - Main account is REQUIRED in every structure but doesn't have to be first segment.
  - Each main account value can exist in ONLY ONE structure assigned to the ledger (no overlapping ranges).
  - **Best practice (from Microsoft):** Separate structures for Balance Sheet vs P&L:
    - Structure 1: BS accounts (100000-399999) + Business Unit
    - Structure 2: P&L accounts (400000-999999) + Business Unit + Department + Cost Center
    - Advanced rule: When Main Account 400000-499999 → also require Customer dimension
  - If budgeting against a dimension → MUST include in account structure (budgeting doesn't use advanced rules)
- **Critical:** Must be ACTIVATED after configuration (status changes from Draft → Active). Activation can take several minutes and triggers sync of unposted transactions.
- **Cannot change CoA after transactions posted.** System blocks with detailed error listing every affected posting profile table and field. Must clear all posting profile references first.

#### Step 10: Advanced Rules (Optional)
- **Menu:** General ledger > Chart of accounts > Structures > Advanced rule structures
- **MCP:** Form tool
- **Purpose:** Allow additional dimensions for specific main account ranges

#### Step 11: Ledger Setup (per Legal Entity)
- **Menu:** General ledger > Ledger setup > Ledger
- **MCP:** Form tool
- **From MS Learn (configure-ledger):**
  - Select Chart of accounts from dropdown (can share same CoA across entities)
  - Select Account structures — can use multiple, but DON'T overlap main account ranges between structures
  - Select Fiscal calendar for the entity
  - Set Accounting currency (home currency) and optional Reporting currency
  - Set Exchange rate type
  - **Changing CoA after posting:** BLOCKED if transactions exist. Error message lists every affected posting profile table+field. Must clear all posting profile references to switch. Bank account table may require Microsoft support to clear.
  - Adding/changing account structures triggers synchronization of unposted transactions — must wait for completion before saving
- **Must run per legal entity** — switch company context before configuring

#### Step 12: GL Parameters
- **Menu:** General ledger > Ledger setup > General ledger parameters
- **MCP:** Form tool (7 tabs: Ledger, Sales tax, Inventory dimensions, Number sequences, Batch transfer rules, Chart of accounts and dimensions, Ledger settlements)
- **Data entity:** `GeneralLedgerParameters` (partial — some settings require form tools)

## Validation Checks
- [ ] Every main account has a type (Balance sheet, P&L, Revenue, Expense, Total, Reporting)
- [ ] Every main account has a category assigned
- [ ] Account structures are ACTIVATED (not Draft)
- [ ] Account structures include all required dimensions
- [ ] Ledger is configured for each legal entity (CoA + calendar + currencies assigned)
- [ ] Financial dimension configuration for integrating applications has active formats
- [ ] At least one ledger dimension format is active
- [ ] At least one default dimension format is active
- [ ] Fiscal calendar has all required periods for the implementation timeline
- [ ] Exchange rates exist for all transaction currencies
- [ ] Number sequences are assigned in GL parameters

## MCP Tool Priority
- **Data tools:** Currencies, exchange rates, fiscal calendars, chart of accounts, main accounts, main account categories, dimension definitions, dimension values
- **Form tools:** Account structures (visual tree), advanced rules, ledger setup (per-company assignment), GL parameters (multi-tab settings), financial dimension configuration for integrating applications

## Common Errors
- "Account structure validation failed" → Structure not activated or missing required dimensions
- Dimension import errors → Financial dimension configuration for integrating applications not set up
- "Fiscal period not open" → Period status not set to Open for posting date
- "Exchange rate not found" → Missing rate for transaction currency on transaction date
- "Main account does not exist" → Account not created or wrong legal entity context

## BPC Process Mapping
| BPC Level | Process | This Skill Covers |
|---|---|---|
| 13 | Record to Report | ✅ |
| 13.10 | Define accounting policies | ✅ Primary |
| 13.10.010 | Configure chart of accounts | ✅ |
| 13.10.020 | Configure financial dimensions | ✅ |
| 13.10.030 | Configure currencies and exchange rates | ✅ |

## Related Skills
- **Prerequisite:** `fo-org-admin` (Level 10-15)
- **Next:** `fo-gl-journals` (Level 25), `fo-tax-configuration` (Level 130), `fo-gl-posting-profiles` (Level 25)
- **All financial modules depend on this skill**
