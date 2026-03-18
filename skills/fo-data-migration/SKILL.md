---
name: fo-data-migration
description: >
  Orchestrate D365 F&O data migration using the Data Management Framework (DMF).
  Covers migration strategy, entity sequencing, DMF performance optimization,
  opening balance migration, subledger reconciliation, cutover planning, and
  common error resolution. Use when: planning data migration, executing DMF
  imports/exports, troubleshooting import errors, performing mock cutover, or
  migrating opening balances. This is a cross-cutting skill used throughout
  implementation phases.
  NOT for: module-specific configuration (use the module skill), integration
  setup (use fo-integration-patterns), ongoing data sync (use dual-write).
metadata:
  platform:
    category: "fo-data-migration"
    riskLevel: "high"
    configSequence: 0
    configLevel: "cross-cutting"
    requires:
      products: ["F&O"]
    bpcProcess: "99 - Administer to Operate (data management)"
    msLearnPath: "https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/data-entities-data-packages"
    sbdPhase: "Implement → Prepare"
---

# Data Migration Framework (DMF) Operations

## Microsoft Learn References
See `references/ms-learn-sources.md` for verified URLs covering DMF overview, data entities, configuration templates, optimization guide, error descriptions, general journal entity, financial dimension config, posting profiles, account reconciliation, FastTrack sessions, MCP server, and Success by Design methodology.

## Core Concepts

### Entity Categories
| Category | Description | Examples | Migration Stage |
|---|---|---|---|
| **Parameter** | Single-record settings tables | GL Parameters, AP Parameters | Stage 1 (Configuration) |
| **Reference** | Small-quantity reference data | Units, tax codes, currencies | Stage 1 (Configuration) |
| **Master** | Core business nouns — high volume | Customers, vendors, products | Stage 2 (Master Data) |
| **Document** | Worksheet/transaction data | Open POs, open SOs, journals | Stage 3 (Opening Balances) |
| **Transaction** | Posted historical data | Posted invoices, GL entries | Usually SUMMARY not detail |

**Critical rule from Microsoft:** *Posted transactions are typically excluded during migration. They are non-idempotent and migrating them in detail creates referential integrity complexity. Migrate in summary via journal entries instead.*

### Three-Stage Migration Sequence

```
STAGE 1: CONFIGURATION (Setup entities)
├── System parameters, GAB setup, number sequences
├── Fiscal calendars, currencies, exchange rates
├── Chart of accounts, main accounts, dimensions
├── ⚠️ Financial dimension config for integrating apps (CRITICAL)
├── Tax codes/groups, payment terms, cash discounts
├── Module parameters (GL, AP, AR, FA, Inventory)
├── Posting profiles (link modules to GL accounts)
└── Bank groups, bank accounts

STAGE 2: MASTER DATA (Reference/Master entities)
├── Global address book parties + addresses
├── Vendors + vendor bank accounts
├── Customers + customer bank accounts
├── Products (released products, variants, attributes)
├── Fixed assets + asset books
├── Units of measure + conversions
├── Sites, warehouses, locations
├── BOM headers + lines, routes
└── Resources, operations

STAGE 3: OPENING BALANCES (Document entities)
├── GL opening balances (general journal)
├── AP open invoices (via AP journal → posts to AP control + subledger)
├── AR open invoices (via AR journal → posts to AR control + subledger)
├── Inventory on-hand (via movement journal → posts to inventory control)
├── Fixed asset acquisitions (via FA journal → posts to FA control)
├── Bank balances (via bank journal → posts to bank control)
└── RECONCILE: subledger totals MUST equal GL control accounts
```

## ⚠️ CRITICAL: Opening Balance Migration Logic

**Control account balances CANNOT be posted directly to the GL. They MUST be posted through their respective subledgers.**

### Correct Sequence
1. **Post subledger opening balances FIRST:**
   - AP: Import open vendor invoices via AP journal → creates AP subledger entries AND posts to AP control account
   - AR: Import open customer invoices via AR journal → creates AR subledger entries AND posts to AR control account
   - FA: Import fixed asset acquisitions → creates FA books AND posts to FA control accounts
   - Inventory: Import via inventory movement journal → creates item ledger AND posts to inventory control accounts
   - Bank: Import via bank journal → creates bank transactions AND posts to bank GL accounts

2. **Post GL opening balances EXCLUDING control accounts:**
   - Use General Journal entity (`LedgerJournalEntity`)
   - Import remaining GL account balances (equity, retained earnings, other BS accounts)
   - **EXCLUDE:** AP control, AR control, FA control, Inventory control, Bank — already populated by step 1

3. **Reconcile:**
   - AP subledger total = AP control account in GL
   - AR subledger total = AR control account in GL
   - Inventory subledger = Inventory control in GL
   - FA net book value = FA control accounts in GL
   - Bank subledger = Bank GL accounts
   - Total GL trial balance = 0 (debits = credits)
   - Use D365's Account Reconciliation feature (v10.0.44+) for automated reconciliation

## DMF Performance Optimization

### Settings to Configure

| Optimization | Menu Path | Impact |
|---|---|---|
| Disable change tracking | Data management > Data entities > [entity] > Disable Change Tracking | Significant speed improvement |
| Enable set-based processing | Data management > Data entities > [entity] > Set-based processing = Yes | 5-10x faster (supported entities only) |
| Configure batch threads | System admin > Setup > Server configuration > Maximum batch threads | Default 8, can increase to 12-16 |
| Import threshold record count | Framework parameters > Entity settings > Configure entity execution parameters | Controls records per thread |
| Import task count | Same location | Controls parallel threads per entity |
| Disable business validations | Entity structure > Run business validations = No | Faster but DANGEROUS — mature data only |
| Disable validateField | Modify target mapping > Call validate Field method = unchecked | Skip field-level validation |
| Clean staging tables | Job history cleanup tile | Prevents table bloat |
| Update statistics | LCS/PPAC or direct SQL `sp_updatestats` | Critical before large imports on sandbox |
| Break large files | Split source files | Gives SQL optimizer time to replan |

### General Journal Entity Tips (LedgerJournalEntity)
- `JOURNALBATCHNUMBER`: Use ~10,000 lines per batch number
- Physical vouchers: Keep small (~10 lines or less)
- Set-based processing: Voucher numbers MUST be provided in import file when enabled
- `ACCOUNTDISPLAYVALUE`: Requires financial dimension configuration for integrating applications
- Same `VOUCHER` + different `TRANSDATE` = creates separate physical vouchers (usually unintended)
- High cardinality dimensions (many new values) slow imports significantly
- Disabling set-based processing: Let system auto-generate voucher numbers (slower but easier)

### Multi-threading Gotchas
- Some entities don't support multi-threading (e.g., `CustomersV3` — "Custom sequence is defined, more than one task is not supported")
- Test in Tier 2+ environment — Tier 1 performance is NOT representative
- Run migrations during off-peak hours with dedicated batch group

## Common DMF Errors & Resolution

| Error | Cause | Resolution |
|---|---|---|
| "Number sequence not set" | Auto-generate enabled, no sequence configured | Configure number sequence in module parameters |
| "Fields not mapped to Entity" | Template exported without headers | Re-export template with "First row header = Yes" |
| "Update conflict — multiple processes" | Parallel processing + duplicate records | Deduplicate files, enable sequential processing |
| "Column has incorrect data" | Null/empty/duplicate in mandatory fields | Pre-validate data, fix specific rows |
| "XML not in correct format" (0xC0010009) | Mapping includes columns not in file | Remove unmapped fields from mapping |
| Excel "0xC02020E8" | Unmapped fields in Excel sheet | Remove unmapped columns |
| "Confidential" Excel label | Excel file has Confidential label | Remove label, or use CSV instead |
| "Schema mismatch" (BYOD) | Entity schema updated but not refreshed | Refresh entity list, republish to BYOD |
| "Entity Execution Sequence" error | Wrong import order | Use `GetEntitySequence` OData action |
| Dimension import errors | Financial dimension config not set up | Configure GL > CoA > Dimensions > Financial dimension config for integrating applications |
| Enum value errors (DMF021) | Enum mappings missing | Remove affected enum column or fix mapping |
| Import task count > 1 error | Entity doesn't support multi-threading | Set task count to 1 |

## Microsoft's Configuration Template Sequence (Levels)

```
Level 10:  System Setup
Level 15:  Global Address Book
Level 20:  General Ledger (Shared)
Level 22:  Workflow
Level 25:  General Ledger (Company)
Level 30-90: Cross-module dependency bands
Level 100: Bank
Level 120: Accounts Payable
Level 130: Tax
Level 140: Accounts Receivable
Level 150: Fixed Assets
Level 160: Budgeting
Level 300: Inventory Management
Level 310: Product Management
Level 320: Procurement
Level 330: Sales & Marketing
Level 395: Quality Management
Level 400: Warehouse Management
Level 405: Transportation Management
Level 410: Production Control
Level 412: Process Manufacturing
Level 420: Costing
Level 500: Retail/Commerce
Level 600: Expense Management
Level 650: Project Accounting
```

## MCP-Powered DMF Operations

Using the D365 MCP server or the community d365fo-client:

### Create DMF Export Project
```
1. data_find_entity_type("DataProject") → discover project entity
2. data_create_entities → create export project definition
3. GetEntitySequence OData action → get correct entity ordering
4. Add entities to project with correct sequence
5. Execute export → generates data package
6. GetExportedPackageUrl → retrieve download URL
```

### Create DMF Import Project
```
1. Upload package to blob storage
2. data_create_entities → create import project
3. ImportFromPackage OData action → start import
4. GetExecutionSummaryStatus → monitor progress
5. GetExecutionErrors → retrieve error details
6. GenerateImportErrors → download error file
```

## Cutover Planning

### Mock Go-Live Checklist (from Microsoft Success by Design)
- [ ] Configuration data package tested and validated
- [ ] Master data import tested — all entities load without errors
- [ ] Opening balances imported and reconciled (subledger = GL)
- [ ] Mock period close performed and reconciled
- [ ] Cutover runbook defined with tasks, owners, durations, dependencies
- [ ] Go/no-go criteria defined
- [ ] Rollback plan documented
- [ ] Data migration environment sized appropriately (Tier 2+)
- [ ] All databases and environments in same Azure region (latency)
- [ ] Timing tested: total cutover window fits within acceptable downtime

## Related Skills
- **Module config skills:** `fo-org-admin`, `fo-gl-chart-of-accounts`, `fo-accounts-payable`, `fo-accounts-receivable`, etc.
- **Validation:** `fo-validation-framework`
- **Integration:** `fo-integration-patterns`
