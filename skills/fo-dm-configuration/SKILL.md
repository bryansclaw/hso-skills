---
name: fo-dm-configuration
description: >
  Migrate D365 F&O configuration data (Stage 1) — system parameters, shared
  setup, module parameters, posting profiles, and all setup entities using DMF
  configuration data templates. Covers golden configuration environment strategy,
  configuration plan management, entity sequencing within data packages, and
  environment-to-environment config promotion.
  Use when: migrating configuration from golden config to target environments,
  creating configuration data packages, or managing multi-company config rollout.
  This is the FIRST migration stage — must complete before master data.
metadata:
  platform:
    category: "fo-data-migration"
    riskLevel: "high"
    configSequence: 0
    configLevel: "cross-cutting"
    requires:
      products: ["F&O"]
    bpcProcess: "99 - Administer to Operate"
    msLearnPath: "https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/configuration-data-templates"
---

# Configuration Data Migration (Stage 1)

## Strategy: Golden Configuration Environment

Microsoft recommends a **dedicated golden configuration environment** — a separate environment used ONLY for configuration data. No transactions, no test data. Restricted access. This is your source of truth.

**Flow:**
```
Golden Config Environment
    │
    ├──→ Export config data packages (DMF)
    │
    ├──→ Import to DEV environment (for development)
    ├──→ Import to UAT environment (for testing)
    ├──→ Import to TRAINING environment
    └──→ Import to PRODUCTION (at cutover)
```

## Configuration Plan

A **configuration plan** is a structured list of ALL configuration data needed per phase:

| Column | Purpose |
|---|---|
| Entity name | DMF entity |
| Module | Which F&O module |
| Sequence level | Microsoft's template level (10-650) |
| Shared vs Company-specific | Does it apply across legal entities? |
| Status | Not started / In progress / Complete / Validated |
| Owner | Who is responsible |
| Dependencies | What must be configured first |
| Data source | Manual entry / Import file / Copy from template |
| Notes | Special handling, known issues |

Maintain in Excel or Azure DevOps work items.

## DMF Configuration Templates

### Microsoft's Default Templates

Load via: Data management workspace > Templates tile > Load default templates

**Merged templates available:**
- **System and Shared** — system setup, GAB, shared GL, workflow (Levels 10-22)
- **Financials** — GL, bank, AP, tax, AR, FA, budgeting (Levels 25-160)
- **Supply Chain Management** — inventory, products, procurement, sales, WMS, production, costing (Levels 300-420)

### Template Sequencing Rules

Templates use three levels of ordering:
1. **Unit** — entities in different units process in PARALLEL
2. **Level** — within same unit, same level = PARALLEL
3. **Sequence** — within same unit+level, sequence determines ORDER

**Microsoft uses Unit=1 for everything** to ensure dependency ordering is respected.

### Default Template Levels (verified from Microsoft docs)

```
Unit 1, Level 10:  System Setup
Unit 1, Level 15:  Global Address Book
Unit 1, Level 20:  General Ledger (Shared)
Unit 1, Level 22:  Workflow
Unit 1, Level 25:  General Ledger (Company)
Unit 1, Level 30:  Band 1 — no dependencies beyond GL
                   (Payment terms, cash discounts, delivery terms/modes)
Unit 1, Level 40:  Band 2 — depends on Band 1
                   (Sales tax codes, sales tax groups, item tax groups)
Unit 1, Level 50:  Band 3 — depends on Band 2
                   (Vendor groups, customer groups, charge codes)
Unit 1, Level 60-90: Bands 4-7 (progressively dependent)
Unit 1, Level 100: Bank
Unit 1, Level 120: Accounts Payable
Unit 1, Level 130: Tax
Unit 1, Level 140: Accounts Receivable
Unit 1, Level 150: Fixed Assets
Unit 1, Level 160: Budgeting
Unit 1, Level 300: Inventory Management
Unit 1, Level 310: Product Management
Unit 1, Level 320: Procurement
Unit 1, Level 330: Sales and Marketing
Unit 1, Level 395: Quality Management
Unit 1, Level 400: Warehouse Management
Unit 1, Level 405: Transportation Management
Unit 1, Level 410: Production Control
Unit 1, Level 412: Process Manufacturing
Unit 1, Level 420: Costing
Unit 1, Level 500: Retail/Commerce
Unit 1, Level 600: Expense Management
Unit 1, Level 650: Project Accounting
```

## Configuration Entity Catalog (Stage 1)

### System & Shared (Levels 10-22)

| Entity Name | Data Entity | Level | Shared? | Notes |
|---|---|---|---|---|
| System parameters | N/A (form) | 10 | Yes | Use form tools or copy DB |
| Address formats | `AddressFormatHeaders` | 15 | Yes | |
| Countries/regions | `AddressCountryRegions` | 15 | Yes | |
| States | `AddressStates` | 15 | Yes | Depends on countries |
| Cities | `AddressCities` | 15 | Yes | Depends on states |
| Postal codes | `AddressZipCodes` | 15 | Yes | Depends on cities |
| Contact types | `ContactTypes` | 15 | Yes | |
| Address purposes | `AddressPurposes` | 15 | Yes | |
| Fiscal calendars | `FiscalCalendars` | 20 | Yes | |
| Fiscal years | `FiscalCalendarYears` | 20 | Yes | |
| Fiscal periods | `FiscalCalendarPeriods` | 20 | Yes | |
| Currencies | `Currencies` | 20 | Yes | |
| Exchange rate types | `ExchangeRateTypes` | 20 | Yes | |
| Exchange rates | `ExchangeRates` | 20 | Yes | |
| Chart of accounts | `ChartOfAccounts` | 20 | Yes (can share) | |
| Main account categories | `MainAccountCategories` | 20 | Yes | |
| Financial dimensions | `FinancialDimensions` | 20 | Yes | Definitions only |
| Number sequences (shared) | `NumberSequences` | 22 | Scope-dependent | |

### General Ledger — Company (Level 25)

| Entity Name | Data Entity | Shared? | Notes |
|---|---|---|---|
| Main accounts | `MainAccounts` | Per CoA | Can be shared if CoA shared |
| Financial dimension values | `FinancialDimensionValues` | Depends | Custom dims = shared; Entity-backed = per company |
| Account structures | N/A (form) | Per ledger | Must use form tools — complex visual config |
| Ledger setup | N/A (form) | Per company | Assigns CoA + calendar + currencies |
| GL parameters | `GeneralLedgerParameters` (partial) | Per company | Some settings need form tools |
| Journal names | `LedgerJournalNames` | Per company | |
| Default descriptions | `DefaultDescriptions` | Per company | |
| Posting definitions | N/A (form) | Per company | |

### Cross-Module Bands (Levels 30-90)

| Entity Name | Data Entity | Level | Notes |
|---|---|---|---|
| Payment terms | `PaymentTerms` | 30 | Shared AP/AR |
| Payment days | `PaymentDays` | 30 | |
| Payment schedules | `PaymentSchedules` | 30 | |
| Cash discount codes | `CashDiscounts` | 30 | |
| Terms of delivery | `TermsOfDelivery` | 30 | Incoterms |
| Modes of delivery | `ModesOfDelivery` | 30 | |
| Sales tax authorities | `SalesTaxAuthorities` | 40 | Creates vendor auto |
| Settlement periods | `SalesTaxSettlementPeriods` | 40 | |
| Tax ledger posting groups | `TaxLedgerPostingGroups` | 40 | |
| Sales tax codes | `SalesTaxCodes` | 40 | |
| Sales tax code values | `SalesTaxCodeValues` | 40 | |
| Sales tax groups | `SalesTaxGroups` | 40 | |
| Item sales tax groups | `ItemSalesTaxGroups` | 40 | |
| Vendor groups | `VendorGroups` | 50 | |
| Customer groups | `CustomerGroups` | 50 | |
| Charges codes | `MarkupTable` | 50 | |

### Module Parameters & Posting Profiles (Levels 100-420)

| Entity Name | Data Entity | Level | Notes |
|---|---|---|---|
| Bank groups | `BankGroups` | 100 | |
| Bank transaction types | `BankTransactionTypes` | 100 | |
| Bank accounts | `BankAccountsV2` | 100 | Needs GL main accounts |
| AP parameters | `VendParameters` (partial) | 120 | Multi-tab — form tools for full config |
| Vendor posting profiles | `VendorPostingProfiles` | 120 | Needs vendor groups + GL accounts |
| Vendor payment methods | `VendorPaymentMethods` | 120 | Needs bank accounts + ER formats |
| AR parameters | `CustParameters` (partial) | 140 | Multi-tab |
| Customer posting profiles | `CustomerPostingProfiles` | 140 | |
| Customer payment methods | `CustomerPaymentMethods` | 140 | |
| FA parameters | N/A (form) | 150 | |
| FA groups | `FixedAssetGroups` | 150 | |
| Depreciation profiles | `DepreciationProfiles` | 150 | |
| FA posting profiles | N/A (form) | 150 | Complex matrix |
| Budget models | N/A (form) | 160 | |
| Inventory parameters | N/A (form) | 300 | Multi-tab |
| Item model groups | `ItemModelGroups` | 300 | |
| Item groups | `ItemGroups` | 300 | |
| Storage dimension groups | `StorageDimensionGroups` | 300 | |
| Tracking dimension groups | `TrackingDimensionGroups` | 300 | |
| Inventory posting | N/A (form) | 300 | Complex matrix — most critical |
| Sites | `Sites` | 300 | |
| Warehouses | `Warehouses` | 300 | |
| Units of measure | `Units` | 310 | |
| Unit conversions | `UnitConversions` | 310 | |
| Product categories | `ProductCategories` | 310 | |
| Procurement parameters | N/A (form) | 320 | |
| Procurement categories | N/A (form) | 320 | Hierarchical — form tools |
| Sales parameters | N/A (form) | 330 | |
| Production parameters | N/A (form) | 410 | |
| Route groups | `RouteGroups` | 410 | |
| Cost categories | `CostCategories` | 420 | |

## ⚠️ Critical Pre-Step

**Before ANY config import:** Configure "Financial dimension configuration for integrating applications"
- Menu: GL > Chart of accounts > Dimensions > Financial dimension configuration for integrating applications
- Activate: Ledger dimension format, Default dimension format
- This is NOT importable — must be done manually via form tools on each target environment

## Multi-Company Config Promotion

### Data Sharing (for shared config)
- Use cross-company data sharing for entities that should be identical across legal entities
- Menu: System administration > Setup > Configure cross-company data sharing
- Common shared entities: payment terms, delivery terms, currencies, tax codes

### Company-Specific Config
- Switch company context before import
- Set "Legal entity" filter on DMF project
- Module parameters are always company-specific
- Posting profiles can be company-specific even within shared CoA

## Data Package Management

### Creating Config Packages
```
1. Data management workspace > Export
2. Add entities from template (or select individually)
3. Verify entity sequence (critical for dependency order)
4. Set format (Excel recommended for review, XML for precision)
5. Export → generates downloadable package (.zip)
6. Package contains: manifest + entity data files
```

### Importing Config Packages
```
1. Data management workspace > Import
2. Upload package
3. Review entity list and sequence
4. Verify mappings (View map for each entity)
5. Check for conflicts (entity already has data?)
6. Import → staging tables → target tables
7. Review execution log for errors
```

### Config Package Numbering Convention
```
[Module#].[DataType#].[Sequence#]

Module: 00=System, 01=GL, 02=Bank, 03=AP, 04=Tax, 05=AR, 
        06=FA, 07=Budget, 10=Inventory, 11=Products, 
        12=Procurement, 13=Sales, 17=Production
DataType: 1=Setup/Config, 2=Master, 3=Transactions
Sequence: Sequential within module+type
```

## Validation After Config Import

### Per-Module Validation
- [ ] System: Parameters accessible, number sequences active
- [ ] GL: Chart of accounts populated, account structures activated, ledger assigned
- [ ] Tax: Tax codes have rates, groups have codes, posting groups have accounts
- [ ] Bank: Bank accounts linked to GL, payment formats assigned
- [ ] AP: Parameters set, posting profiles have GL accounts, payment methods configured
- [ ] AR: Parameters set, posting profiles have GL accounts
- [ ] FA: Groups configured, depreciation profiles created
- [ ] Inventory: Item model groups, item groups, posting matrix complete
- [ ] Production: Resources, route groups, operations defined

### Cross-Module Validation
- [ ] Financial dimension config for integrating apps is active on target
- [ ] All posting profiles reference valid GL main accounts
- [ ] Number sequences active for all modules
- [ ] Workflows configured and active (if applicable)
- [ ] ER configurations imported (for electronic payments)

## Common Errors
- "Entity not found in package" → Entity name changed between versions; regenerate package
- "Mapping mismatch" → Target environment has different customizations; remap
- "Dependency not met" → Sequence wrong; reorder entities in project
- Import succeeds but data wrong → Company context wrong; verify legal entity on import
- "Number sequence exhausted" → Sequence range too small; extend range
- Account structure not active → Must manually activate after import (can't be imported in active state)

## Related Skills
- **Next stage:** `fo-dm-master-data` (Stage 2)
- **Module skills:** All fo-* skills define what needs to be configured
- **Tools:** `fo-integration-patterns` (DMF details)
