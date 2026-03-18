# D365 F&O Configuration & Data Migration Deep Dive

**Version:** 0.1 (Draft)
**Date:** 2026-03-18
**Companion to:** ARCHITECTURE.md

---

## Table of Contents

1. [D365 ERP MCP Server Integration](#1-d365-erp-mcp-server-integration)
2. [Business Process Catalog (BPC) Alignment](#2-business-process-catalog-bpc-alignment)
3. [Module Configuration Sequence & Dependencies](#3-module-configuration-sequence--dependencies)
4. [Module-by-Module Skills & Agent Design](#4-module-by-module-skills--agent-design)
5. [Data Migration Framework & Entity Catalog](#5-data-migration-framework--entity-catalog)
6. [Master Data Dependencies & Validation](#6-master-data-dependencies--validation)
7. [MCP-Powered Agent Prompts Per Module](#7-mcp-powered-agent-prompts-per-module)
8. [Microsoft Learn Integration](#8-microsoft-learn-integration)

---

## 1. D365 ERP MCP Server Integration

### What It Is

The **Dynamics 365 ERP MCP Server** is Microsoft's official Model Context Protocol server for F&O. It provides a standard interface for AI agents to access data and business logic. This is the foundation for our platform's D365 connector layer — instead of building custom APIs, we leverage MCP to interact with F&O environments.

**Key insight:** The MCP server provides three tool categories that map directly to our agent operations:

### MCP Tool Categories

#### Data Tools (OData-based CRUD)
Best for standard data operations — faster, fewer calls, better performance.

| Tool | Purpose | Our Use Case |
|---|---|---|
| `data_find_entity_type` | Discover OData entities by search | Agent discovers available entities for a module |
| `data_get_entity_metadata` | Get entity schema/fields | Agent understands field requirements before migration |
| `data_create_entities` | Create records via OData | Master data import, configuration record creation |
| `data_update_entities` | Update records via OData | Configuration updates, data corrections |
| `data_delete_entities` | Delete records via OData | Test data cleanup, migration rollback |
| `data_find_entities` | Query/read records via OData | Validation, pulling related master data, verification |

#### Form Tools (Headless UI Operations)
Best for operations that involve business logic, button clicks, or actions not available via simple CRUD. Works through server APIs — not screen scraping.

| Tool | Purpose | Our Use Case |
|---|---|---|
| `form_find_menu_item` | Find a menu item (by search) | Agent navigates to correct form for configuration |
| `form_open_menu_item` | Open a form | Agent opens configuration pages |
| `form_click_control` | Click buttons/actions on form | Execute actions like "Post", "Approve", "Validate" |
| `form_set_control_values` | Set field values on forms | Configure parameters, set up module settings |
| `form_filter_form` | Apply filters | Find specific records |
| `form_filter_grid` | Filter grids | Navigate large lists |
| `form_open_lookup` | Open lookup controls | Select related values (e.g., select a chart of accounts) |
| `form_select_grid_row` | Select grid rows | Target specific records for action |
| `form_save_form` | Save form | Persist configuration changes |
| `form_close_form` | Close form | Clean up after operations |
| `form_find_controls` | Find controls on form | Discover available fields/buttons |
| `form_open_or_close_tab` | Navigate tabs | Access different sections of setup forms |
| `form_sort_grid_column` | Sort grids | Organize data for review |

#### Action Tools (Custom X++ Logic)
For operations exposed through custom code — AI tools implementing `ICustomAPI`.

| Tool | Purpose | Our Use Case |
|---|---|---|
| `api_find_actions` | Discover available custom actions | Find available custom operations |
| `api_invoke_action` | Execute custom business logic | Run custom validation, execute complex operations |

### Platform Integration Strategy

```
Our Platform Agent
       │
       ▼
┌─────────────────────────────┐
│  Semantic Kernel Plugin     │
│  "D365FOMCPPlugin"          │
│                             │
│  Wraps MCP tools into       │
│  Semantic Kernel functions  │
│  with approval gates        │
└─────────┬───────────────────┘
          │
          ▼
┌─────────────────────────────┐
│  D365 ERP MCP Server        │
│  (Microsoft-provided)       │
│                             │
│  • Tier 2+ environment      │
│  • Version 10.0.47+         │
│  • Feature enabled           │
│  • Our platform registered   │
│    as Allowed MCP Client    │
└─────────┬───────────────────┘
          │
          ▼
┌─────────────────────────────┐
│  D365 F&O Environment       │
│                             │
│  Security role determines   │
│  what agent can see/do      │
└─────────────────────────────┘
```

**Critical design decision:** Data tools are preferred over form tools for CRUD operations. Form tools are for configuration steps that involve business logic (button clicks, parameter settings, posting). Our agent instructions must encode this preference explicitly.

### Agent Security Roles per D365

| Agent | D365 Security Role | Rationale |
|---|---|---|
| Config Agent (read) | System Administrator (read) | Needs to see all configuration forms |
| Config Agent (write-dev) | System Administrator | Full access on Dev environments only |
| Config Agent (write-uat) | Restricted custom role | Limited write on UAT, requires human approval |
| Migration Agent | Data Management Admin | DMF operations + entity CRUD |
| Validation Agent | Read-only custom role | Query-only access for validation |

---

## 2. Business Process Catalog (BPC) Alignment

### The 15 End-to-End Scenarios

Microsoft's BPC defines 15 end-to-end business process scenarios. Our platform's agents and skills align to these:

#### Primary Processes (Revenue/Delivery)
| # | End-to-End Process | F&O Modules | Our Agent Coverage |
|---|---|---|---|
| 1 | **Prospect to Quote** | Sales (limited in F&O, primarily CE) | CE Config Agent |
| 2 | **Order to Cash** | AR, Sales, Warehouse, Transportation | AR Agent, Inventory Agent |
| 3 | **Service to Deliver** | Field Service, Project Ops | Not Phase 1 |
| 4 | **Plan to Produce** | Production Control, Process Mfg, MRP | Production Agent |
| 5 | **Source to Pay** | AP, Procurement, Vendor mgmt | AP Agent |
| 6 | **Project to Profit** | Project Accounting, Project Ops | Not Phase 1 |
| 7 | **Case to Resolution** | Case management | Not Phase 1 |

#### Processes to Manage Work
| # | End-to-End Process | F&O Modules | Our Agent Coverage |
|---|---|---|---|
| 8 | **Acquire to Dispose** | Fixed Assets | Fixed Assets Agent |
| 9 | **Concept to Market** | Product management | Product Agent |
| 10 | **Design to Retire** | Engineering change mgmt | Not Phase 1 |
| 11 | **Forecast to Plan** | Master Planning, Demand forecast | Inventory Agent |

#### Processes to Support Work
| # | End-to-End Process | F&O Modules | Our Agent Coverage |
|---|---|---|---|
| 12 | **Inventory to Deliver** | Inventory, Warehouse, Transportation | Inventory Agent |
| 13 | **Record to Report** | GL, Budgeting, Financial reporting | GL Agent (primary) |
| 14 | **Hire to Retire** | HR, Payroll | Not Phase 1 |
| 15 | **Administer to Operate** | System admin, Org admin | Org Admin Agent |

### BPC Catalog Structure (6 Levels)

```
Level 1: End-to-End Process          → "Record to Report"
Level 2: Business Process Area       → "Define accounting policies"
Level 3: Business Process            → "Configure chart of accounts"
Level 4: Scenario/Pattern            → "Shared chart of accounts across legal entities"
Level 5: System Process              → "General ledger > Chart of accounts > Chart of accounts"
Level 6: Test Case                   → "Verify main account created with correct category"
```

**How our platform uses BPC:**
- **Discovery Agent** maps meeting requirements to BPC process areas
- **BPC Agent** generates/maintains the catalog for each project, tracking which processes are in scope
- **Config Agents** use BPC Level 5 (System Processes) as their menu path navigation reference
- **Validation Agents** use BPC Level 6 (Test Cases) as validation criteria
- The BPC Excel workbook (downloadable from Microsoft) is imported into our platform as a reference dataset

### Record to Report — Business Process Areas (GL Focus)

| Business Process Area | Key Activities | Config Agent Skill |
|---|---|---|
| **Define accounting policies** | Organization structure, legal entities, chart of accounts, currencies, exchange rates | `fo-org-admin`, `fo-gl-chart-of-accounts` |
| **Manage cash** | Bank accounts, reconciliation, cash management | `fo-cash-bank-management` |
| **Manage budgets** | Budget creation, control, apportionments, workflows | `fo-budgeting` |
| **Record financial transactions** | Journals, allocations, deferrals, accruals, intercompany | `fo-gl-journals`, `fo-intercompany` |
| **Close financial periods** | Period close, revaluations, consolidation, elimination, year-end | `fo-period-close` |
| **Analyze financial performance** | Cash flow forecasting, financial statements, variance analysis | `fo-financial-reporting` |

---

## 3. Module Configuration Sequence & Dependencies

### Microsoft's Official Configuration Template Sequence

This is the dependency-ordered sequence from Microsoft's Data Management configuration templates. **This is the definitive order your agents must follow:**

```
PHASE 0: FOUNDATION (Levels 10-22)
═══════════════════════════════════
Level 10: System Setup
  → System parameters, shared parameters, number sequence scopes
  → Server configuration, batch groups

Level 15: Global Address Book
  → Party types, name sequences, address formats
  → Countries/regions, states, counties, cities
  → Address purposes, contact types

Level 20: General Ledger (Shared)
  → Fiscal calendars, fiscal years, periods
  → Currency codes, exchange rate types
  → Chart of accounts (shared structure)
  → Account structures, financial dimension sets
  → Financial dimension formats

Level 22: Workflow
  → Workflow types, templates, configurations

PHASE 1: CORE FINANCIAL (Level 25)
════════════════════════════════════
Level 25: General Ledger (Company-specific)
  → Ledger configuration (CoA assignment, currencies, fiscal calendar)
  → Main accounts (with account category, type, posting type)
  → Financial dimensions and values
  → Account structures assignment
  → Journal names, voucher series
  → GL parameters (all tabs)
  → Posting definitions
  → Default descriptions
  → Ledger allocation rules

PHASE 2: CROSS-MODULE DEPENDENCIES (Levels 30-90)
═══════════════════════════════════════════════════
Level 30: Band 1 (No dependencies beyond GL)
  → Payment terms, payment days, payment schedules
  → Methods of payment
  → Cash discount codes
  → Tax exempt numbers
  → Terms of delivery
  → Modes of delivery

Level 40: Band 2 (Depends on Band 1)
  → Sales tax codes
  → Sales tax groups
  → Item sales tax groups
  → Tax ledger posting groups
  → Withholding tax setup

Level 50: Band 3 (Depends on Band 2)
  → Vendor groups, Customer groups
  → Payment fee setup
  → Charges codes

Level 60: Band 4 (Depends on Band 3)
  → (Module-specific shared setup)

Level 70-90: Bands 5-7
  → (Progressively dependent configurations)

PHASE 3: FINANCIAL MODULES (Levels 100-160)
════════════════════════════════════════════
Level 100: Bank
  → Bank groups
  → Bank accounts
  → Bank transaction types
  → Bank reconciliation matching rules
  → Check layout

Level 120: Accounts Payable
  → AP parameters
  → Vendor groups
  → Vendor posting profiles
  → Vendor payment methods
  → Vendor charge groups
  → Invoice validation policies
  → 1099 setup (US)

Level 130: Tax
  → Tax authorities
  → Settlement periods
  → Tax registration numbers
  → Tax configuration (advanced)

Level 140: Accounts Receivable
  → AR parameters
  → Customer groups
  → Customer posting profiles
  → Customer payment methods
  → Customer charge groups
  → Collection letter sequences
  → Interest codes
  → Credit management setup

Level 150: Fixed Assets
  → Fixed asset groups
  → Depreciation profiles
  → Value models / Depreciation books
  → FA parameters
  → FA posting profiles

Level 160: Budgeting
  → Budget models
  → Budget dimensions
  → Budget allocation terms
  → Budget control configuration

PHASE 4: SUPPLY CHAIN & OPERATIONS (Levels 300-420)
════════════════════════════════════════════════════
Level 300: Inventory Management
  → Inventory parameters
  → Item model groups (FIFO, LIFO, Std Cost, Moving Avg)
  → Item groups
  → Storage dimension groups
  → Tracking dimension groups
  → Inventory posting profiles
  → Inventory dimensions
  → Sites, Warehouses, Locations

Level 310: Product Management
  → Product categories
  → Product dimension groups
  → Released products
  → Product attributes
  → Unit conversions

Level 320: Procurement
  → Procurement categories
  → Procurement parameters
  → Purchase agreement classifications
  → Purchasing policies

Level 330: Sales & Marketing
  → Sales parameters
  → Sales agreement classifications
  → Commission setup

Level 395: Quality Management
  → Quality groups
  → Test instruments
  → Quality orders setup

Level 400: Warehouse Management
  → WMS parameters
  → Location types
  → Location profiles
  → Wave templates
  → Work templates

Level 405: Transportation Management
  → TMS parameters
  → Carrier setup
  → Rating profiles

Level 410: Production Control
  → Production parameters
  → Route groups
  → Resource types & resources
  → Operations
  → BOM/Formula versions

Level 412: Process Manufacturing
  → Batch attributes
  → Potency setup

Level 420: Costing
  → Costing versions
  → Cost groups
  → Cost categories
  → Overhead calculation

PHASE 5: SPECIALIZED (Levels 500+)
════════════════════════════════════
Level 500: Retail/Commerce
Level 600: Expense Management
Level 650: Project Accounting
```

### Visual Dependency Graph

```
                    ┌──────────────┐
                    │ System Setup │ (10)
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ Global Addr  │ (15)
                    │ Book         │
                    └──────┬───────┘
                           │
              ┌────────────▼────────────┐
              │ General Ledger (Shared) │ (20)
              │ • Fiscal Calendar       │
              │ • Currencies            │
              │ • Chart of Accounts     │
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │ General Ledger (Company)│ (25)
              │ • Ledger Config         │
              │ • Main Accounts         │
              │ • Dimensions            │
              │ • Journals              │
              └────────────┬────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
    ┌─────────┐     ┌──────────┐      ┌─────────┐
    │Pmt Terms│     │ Tax Codes│      │ Cash    │
    │Discounts│     │ Tax Grps │      │ Disc    │
    │(30)     │     │ (40)     │      │ (30)    │
    └────┬────┘     └────┬─────┘      └────┬────┘
         │               │                 │
         ├───────────────┬┘                 │
         ▼               ▼                  │
    ┌─────────┐   ┌──────────┐              │
    │Vendor   │   │Customer  │              │
    │Groups   │   │Groups    │              │
    │(50)     │   │(50)      │              │
    └────┬────┘   └────┬─────┘              │
         │             │                    │
    ┌────▼────┐   ┌────▼─────┐   ┌─────────▼──┐
    │Bank     │   │Bank      │   │Bank        │
    │(100)    │   │(100)     │   │(100)       │
    └────┬────┘   └────┬─────┘   └────────────┘
         │             │
    ┌────▼────┐   ┌────▼──────┐
    │  AP     │   │  AR       │
    │ (120)   │   │ (140)     │
    └────┬────┘   └────┬──────┘
         │             │
         │        ┌────▼──────┐
         │        │Fixed Asset│
         │        │(150)      │
         │        └───────────┘
         │
    ┌────▼──────────┐
    │Inventory Mgmt │
    │(300)          │
    └────┬──────────┘
         │
    ┌────▼──────────┐
    │Product Mgmt   │
    │(310)          │
    └────┬──────────┘
         │
    ┌────▼──────────┐     ┌──────────────┐
    │Procurement    │     │Sales         │
    │(320)          │     │(330)         │
    └────┬──────────┘     └──────┬───────┘
         │                       │
    ┌────▼──────────┐     ┌──────▼───────┐
    │Warehouse      │     │Production    │
    │(400)          │     │(410)         │
    └───────────────┘     └──────────────┘
```

---

## 4. Module-by-Module Skills & Agent Design

### 4.1 Organization Administration Agent

**BPC Alignment:** Administer to Operate (99)

**Skill: `fo-org-admin`**

```yaml
---
name: fo-org-admin
description: >
  Configure D365 F&O organization structure: legal entities, operating units,
  organizational hierarchies, number sequences, and system parameters.
  Use when: setting up a new implementation, creating legal entities,
  configuring organization hierarchy, or setting system-wide parameters.
  This is ALWAYS the first module configured.
metadata:
  platform:
    category: "fo-foundation"
    riskLevel: "high"
    configSequence: 1
    configLevel: 10
    requires:
      products: ["F&O"]
    bpcProcess: "99 - Administer to Operate"
---
```

**Configuration Steps (with menu paths):**

| Step | Menu Path | Data Entity | MCP Approach |
|---|---|---|---|
| 1. Legal entities | Organization administration > Organizations > Legal entities | `LegalEntities` | Data tool: `data_create_entities` |
| 2. Operating units | Organization administration > Organizations > Operating units | `OperatingUnits` | Data tool |
| 3. Organization hierarchies | Organization administration > Organizations > Organization hierarchies | `OrganizationHierarchies` | Form tool (tree structure) |
| 4. Number sequence groups | Organization administration > Number sequences > Number sequences | `NumberSequences` | Data tool + Form tool for scope |
| 5. Global address book | Organization administration > Global address book | `GlobalAddressBook` | Data tool |
| 6. Address setup | Organization administration > Addresses > Address setup | `AddressFormats`, `CountryRegions`, `States` | Data tool |
| 7. System parameters | System administration > Setup > System parameters | N/A (parameter table) | Form tool: `form_open_menu_item` + `form_set_control_values` |

**Data Migration Order:**
1. Countries/Regions
2. States/Provinces
3. Cities
4. Address formats
5. Legal entities
6. Operating units
7. Organization hierarchies
8. Number sequences (shared)
9. Global address book parties

---

### 4.2 General Ledger Agent

**BPC Alignment:** Record to Report (13)

**Skills:**

#### `fo-gl-chart-of-accounts`

```yaml
---
name: fo-gl-chart-of-accounts
description: >
  Design and configure D365 F&O chart of accounts, main accounts,
  financial dimensions, account structures, and advanced rules.
  Use when: setting up a new chart of accounts, modifying account structures,
  adding financial dimensions, or configuring main account categories.
  DEPENDS ON: fo-org-admin (legal entities must exist first).
metadata:
  platform:
    category: "fo-general-ledger"
    riskLevel: "high"
    configSequence: 2
    configLevel: 20-25
    requires:
      products: ["F&O"]
      skills: ["fo-org-admin"]
    bpcProcess: "13.10 - Define accounting policies"
---
```

**Configuration Steps:**

| Step | Menu Path | Data Entity | MCP Approach |
|---|---|---|---|
| 1. Fiscal calendars | General ledger > Calendars > Fiscal calendars | `FiscalCalendars` | Data tool |
| 2. Fiscal years & periods | General ledger > Calendars > Fiscal calendars > [calendar] | `FiscalCalendarYears`, `FiscalCalendarPeriods` | Data tool |
| 3. Chart of accounts | General ledger > Chart of accounts > Accounts > Chart of accounts | `ChartOfAccounts` | Data tool |
| 4. Main account categories | General ledger > Chart of accounts > Accounts > Main account categories | `MainAccountCategories` | Data tool |
| 5. Main accounts | General ledger > Chart of accounts > Accounts > Main accounts | `MainAccounts` | Data tool |
| 6. Financial dimensions | General ledger > Chart of accounts > Dimensions > Financial dimensions | `FinancialDimensions` | Data tool |
| 7. Financial dimension values | General ledger > Chart of accounts > Dimensions > Financial dimension values | `FinancialDimensionValues` | Data tool |
| 8. Account structures | General ledger > Chart of accounts > Structures > Configure account structures | `AccountStructures` | Form tool (complex UI) |
| 9. Advanced rules | General ledger > Chart of accounts > Structures > Advanced rule structures | N/A | Form tool |
| 10. Ledger setup | General ledger > Ledger setup > Ledger | N/A (parameter) | Form tool |
| 11. Financial dimension sets | General ledger > Chart of accounts > Dimensions > Financial dimension sets | `DimensionSets` | Data tool |

#### `fo-gl-journals`

```yaml
---
name: fo-gl-journals
description: >
  Configure and manage D365 F&O general ledger journals including
  journal names, voucher series, journal controls, posting restrictions,
  periodic journals, and intercompany accounting.
  Use when: setting up journal names, configuring approval workflows,
  or creating journal entries.
metadata:
  platform:
    category: "fo-general-ledger"
    riskLevel: "medium"
    configSequence: 3
    configLevel: 25
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "13.40 - Record financial transactions"
---
```

**Configuration Steps:**

| Step | Menu Path | Data Entity |
|---|---|---|
| 1. Journal names | General ledger > Journal setup > Journal names | `LedgerJournalNames` |
| 2. Voucher series | General ledger > Journal setup > Voucher series | Related to number sequences |
| 3. Journal controls | General ledger > Journal setup > Journal control | N/A (parameter form) |
| 4. Posting restriction rules | General ledger > Journal setup > Posting restriction rules | N/A |
| 5. GL parameters | General ledger > Ledger setup > General ledger parameters | `GeneralLedgerParameters` (partial) |
| 6. Default descriptions | General ledger > Journal setup > Default descriptions | `DefaultDescriptions` |
| 7. Financial tags | General ledger > Chart of accounts > Financial tags | Form tool |
| 8. Intercompany setup | General ledger > Intercompany accounting > Intercompany accounting | Form tool |

#### `fo-gl-posting-profiles`

```yaml
---
name: fo-gl-posting-profiles
description: >
  Configure posting profiles that control how subledger transactions
  (AP, AR, Inventory, FA) post to the general ledger. This includes
  ledger posting groups, accounts for automatic transactions, and
  posting definitions.
  Use when: setting up how transactions flow from subledgers to GL.
  CRITICAL: This bridges GL to all other modules.
metadata:
  platform:
    category: "fo-general-ledger"
    riskLevel: "high"
    configSequence: 4
    configLevel: 25
    bpcProcess: "13.10 - Define accounting policies"
---
```

---

### 4.3 Accounts Payable Agent

**BPC Alignment:** Source to Pay (05)

**Skill: `fo-accounts-payable`**

```yaml
---
name: fo-accounts-payable
description: >
  Configure D365 F&O Accounts Payable module including AP parameters,
  vendor groups, vendor posting profiles, payment methods, payment
  terms, charge codes, invoice validation, and 1099 setup.
  Use when: setting up AP module, configuring vendor payments,
  or managing vendor invoicing rules.
  DEPENDS ON: fo-gl-chart-of-accounts, fo-cash-bank-management (for payment methods).
metadata:
  platform:
    category: "fo-accounts-payable"
    riskLevel: "medium"
    configSequence: 6
    configLevel: 120
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts", "fo-tax-configuration"]
    bpcProcess: "05 - Source to Pay"
---
```

**Configuration Steps:**

| Step | Menu Path | Data Entity |
|---|---|---|
| 1. AP parameters | Accounts payable > Setup > Accounts payable parameters | `VendParameters` |
| 2. Vendor groups | Accounts payable > Setup > Vendor groups | `VendorGroups` |
| 3. Vendor posting profiles | Accounts payable > Setup > Vendor posting profiles | `VendorPostingProfiles` |
| 4. Methods of payment (vendor) | Accounts payable > Payment setup > Methods of payment | `VendorPaymentMethods` |
| 5. Payment fee setup | Accounts payable > Payment setup > Payment fee | `VendPaymentFee` |
| 6. Charges codes | Accounts payable > Charges setup > Charges code | `VendCharges` |
| 7. Invoice validation policies | Accounts payable > Setup > Invoice validation > Invoice matching policies | Form tool |
| 8. Invoice totals tolerance | Accounts payable > Setup > Invoice validation > Invoice totals tolerances | Form tool |
| 9. 1099 setup (US) | Accounts payable > Setup > Tax 1099 > 1099 fields | Form tool |
| 10. Vendor approval workflows | Accounts payable > Setup > Accounts payable workflows | Form tool |

**Master Data Migration (AP):**

| Sequence | Entity | Data Entity Name | Dependencies |
|---|---|---|---|
| 1 | Vendor groups | `VendorGroups` | GL main accounts (posting profiles) |
| 2 | Vendor posting profiles | `VendorPostingProfiles` | Vendor groups, main accounts |
| 3 | Payment terms | `PaymentTerms` | None (shared) |
| 4 | Cash discounts | `CashDiscounts` | None (shared) |
| 5 | Methods of payment | `VendorPaymentMethods` | Bank accounts |
| 6 | Vendors | `VendorsV2` | Vendor groups, payment terms, tax groups, addresses |
| 7 | Vendor bank accounts | `VendorBankAccounts` | Vendors |
| 8 | Vendor contacts | `VendorContacts` | Vendors, Global address book |

---

### 4.4 Accounts Receivable Agent

**BPC Alignment:** Order to Cash (02)

**Skill: `fo-accounts-receivable`**

```yaml
---
name: fo-accounts-receivable
description: >
  Configure D365 F&O Accounts Receivable module including AR parameters,
  customer groups, customer posting profiles, payment methods,
  credit management, collections, interest codes, and collection letters.
  Use when: setting up AR, configuring customer payments, credit limits,
  or collections processes.
  DEPENDS ON: fo-gl-chart-of-accounts, fo-cash-bank-management.
metadata:
  platform:
    category: "fo-accounts-receivable"
    riskLevel: "medium"
    configSequence: 7
    configLevel: 140
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts", "fo-tax-configuration"]
    bpcProcess: "02 - Order to Cash"
---
```

**Configuration Steps:**

| Step | Menu Path | Data Entity |
|---|---|---|
| 1. AR parameters | Accounts receivable > Setup > Accounts receivable parameters | `CustParameters` |
| 2. Customer groups | Accounts receivable > Setup > Customer groups | `CustomerGroups` |
| 3. Customer posting profiles | Accounts receivable > Setup > Customer posting profiles | `CustomerPostingProfiles` |
| 4. Methods of payment (customer) | Accounts receivable > Payment setup > Methods of payment | `CustomerPaymentMethods` |
| 5. Collection letter sequences | Accounts receivable > Setup > Collections > Collection letter sequence | Form tool |
| 6. Interest codes | Accounts receivable > Setup > Interest > Interest codes | `InterestCodes` |
| 7. Credit management setup | Credit and collections > Setup > Credit management parameters | Form tool |
| 8. Aging period definitions | Credit and collections > Setup > Aging period definitions | Form tool |
| 9. AR workflows | Accounts receivable > Setup > Accounts receivable workflows | Form tool |
| 10. Free text invoice templates | Accounts receivable > Setup > Free text invoice templates | Form tool |

**Master Data Migration (AR):**

| Sequence | Entity | Data Entity Name | Dependencies |
|---|---|---|---|
| 1 | Customer groups | `CustomerGroups` | GL main accounts (posting profiles) |
| 2 | Customer posting profiles | `CustomerPostingProfiles` | Customer groups, main accounts |
| 3 | Payment terms | `PaymentTerms` | Shared with AP |
| 4 | Methods of payment | `CustomerPaymentMethods` | Bank accounts |
| 5 | Customers | `CustomersV3` | Customer groups, payment terms, tax groups, addresses |
| 6 | Customer bank accounts | `CustomerBankAccounts` | Customers |
| 7 | Customer contacts | `CustomerContacts` | Customers, GAB |

---

### 4.5 Tax Configuration Agent

**BPC Alignment:** Record to Report (13) — Comply with tax

**Skill: `fo-tax-configuration`**

```yaml
---
name: fo-tax-configuration
description: >
  Configure D365 F&O tax setup including sales tax authorities,
  settlement periods, sales tax codes, sales tax groups,
  item sales tax groups, ledger posting groups, and withholding tax.
  Use when: setting up tax configuration for any jurisdiction.
  DEPENDS ON: fo-gl-chart-of-accounts (needs main accounts for posting).
metadata:
  platform:
    category: "fo-tax"
    riskLevel: "high"
    configSequence: 5
    configLevel: 130
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "13.10 - Define accounting policies (tax compliance)"
---
```

**Configuration Steps:**

| Step | Menu Path | Data Entity |
|---|---|---|
| 1. Tax authorities | Tax > Indirect taxes > Sales tax > Sales tax authorities | `SalesTaxAuthorities` |
| 2. Settlement periods | Tax > Indirect taxes > Sales tax > Sales tax settlement periods | `SalesTaxSettlementPeriods` |
| 3. Ledger posting groups | Tax > Setup > Sales tax > Ledger posting groups | `TaxLedgerPostingGroups` |
| 4. Sales tax codes | Tax > Indirect taxes > Sales tax > Sales tax codes | `SalesTaxCodes` |
| 5. Sales tax values | (within sales tax codes) | `SalesTaxCodeValues` |
| 6. Sales tax groups | Tax > Indirect taxes > Sales tax > Sales tax groups | `SalesTaxGroups` |
| 7. Item sales tax groups | Tax > Indirect taxes > Sales tax > Item sales tax groups | `ItemSalesTaxGroups` |
| 8. Tax exempt numbers | Tax > Setup > Sales tax > Tax exempt numbers | `TaxExemptNumbers` |
| 9. Withholding tax codes | Tax > Indirect taxes > Withholding tax > Withholding tax codes | `WithholdingTaxCodes` |
| 10. Withholding tax groups | Tax > Indirect taxes > Withholding tax > Withholding tax groups | `WithholdingTaxGroups` |

---

### 4.6 Cash & Bank Management Agent

**BPC Alignment:** Record to Report (13) — Manage Cash

**Skill: `fo-cash-bank-management`**

```yaml
---
name: fo-cash-bank-management
description: >
  Configure D365 F&O Cash and Bank management including bank groups,
  bank accounts, bank transaction types, reconciliation rules,
  payment formats, and cash flow forecasting.
  Use when: setting up bank accounts, configuring reconciliation,
  or cash flow forecasting.
metadata:
  platform:
    category: "fo-cash-bank"
    riskLevel: "medium"
    configSequence: 5
    configLevel: 100
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "13.20 - Manage cash"
---
```

**Configuration Steps:**

| Step | Menu Path | Data Entity |
|---|---|---|
| 1. Bank groups | Cash and bank management > Setup > Bank groups | `BankGroups` |
| 2. Bank transaction types | Cash and bank management > Setup > Bank transaction types | `BankTransactionTypes` |
| 3. Bank accounts | Cash and bank management > Bank accounts > Bank accounts | `BankAccountsV2` |
| 4. Check layout | Cash and bank management > Setup > Check layout | Form tool |
| 5. Bank reconciliation matching rules | Cash and bank management > Setup > Advanced bank reconciliation setup > Reconciliation matching rules | Form tool |
| 6. Cash flow forecast setup | Cash and bank management > Setup > Cash flow forecasting | Form tool |
| 7. Payment formats | (Electronic reporting configurations) | Form tool |

---

### 4.7 Inventory Management Agent

**BPC Alignment:** Inventory to Deliver (12)

**Skill: `fo-inventory-management`**

```yaml
---
name: fo-inventory-management
description: >
  Configure D365 F&O Inventory management including inventory parameters,
  item model groups, item groups, storage dimensions, tracking dimensions,
  sites, warehouses, locations, and inventory posting profiles.
  Use when: setting up inventory structure, costing methods, or
  warehouse configuration.
  DEPENDS ON: fo-gl-chart-of-accounts (inventory posting profiles link to GL).
metadata:
  platform:
    category: "fo-inventory"
    riskLevel: "medium"
    configSequence: 8
    configLevel: 300
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "12 - Inventory to Deliver"
---
```

**Configuration Steps:**

| Step | Menu Path | Data Entity |
|---|---|---|
| 1. Inventory parameters | Inventory management > Setup > Inventory and warehouse management parameters | `InventoryParameters` |
| 2. Item model groups | Inventory management > Setup > Inventory > Item model groups | `ItemModelGroups` |
| 3. Item groups | Inventory management > Setup > Inventory > Item groups | `ItemGroups` |
| 4. Storage dimension groups | Inventory management > Setup > Dimension > Storage dimension groups | `StorageDimensionGroups` |
| 5. Tracking dimension groups | Inventory management > Setup > Dimension > Tracking dimension groups | `TrackingDimensionGroups` |
| 6. Inventory posting profiles | Inventory management > Setup > Posting > Posting | `InventoryPostingProfiles` |
| 7. Sites | Inventory management > Setup > Inventory breakdown > Sites | `Sites` |
| 8. Warehouses | Inventory management > Setup > Inventory breakdown > Warehouses | `Warehouses` |
| 9. Locations | Inventory management > Setup > Inventory breakdown > Locations | `Locations` |
| 10. Inventory dimensions | (Defined by dimension groups) | Combined setup |

**Master Data Migration (Inventory):**

| Sequence | Entity | Data Entity Name | Dependencies |
|---|---|---|---|
| 1 | Sites | `Sites` | Legal entities |
| 2 | Warehouses | `Warehouses` | Sites |
| 3 | Locations | `WMSLocations` | Warehouses |
| 4 | Item model groups | `ItemModelGroups` | None |
| 5 | Item groups | `ItemGroups` | GL posting profiles |
| 6 | Dimension groups | `StorageDimensionGroups`, `TrackingDimensionGroups` | None |
| 7 | Unit of measure | `Units`, `UnitConversions` | None |
| 8 | Released products | `ReleasedProductsV2` | Item groups, dimension groups, units |
| 9 | Product variants | `ReleasedProductVariants` | Released products |
| 10 | BOM headers | `BillOfMaterialsHeaders` | Released products |
| 11 | BOM lines | `BillOfMaterialsLines` | BOM headers, released products |
| 12 | Inventory on-hand | `InventoryOnHand` | Products, sites, warehouses |

---

### 4.8 Production Control Agent

**BPC Alignment:** Plan to Produce (04)

**Skill: `fo-production-control`**

```yaml
---
name: fo-production-control
description: >
  Configure D365 F&O Production control including production parameters,
  route groups, resource types, resources, operations, production groups,
  BOM/Route setup, and production order lifecycle.
  Use when: setting up manufacturing/production capabilities.
  DEPENDS ON: fo-inventory-management (products must exist).
metadata:
  platform:
    category: "fo-production"
    riskLevel: "medium"
    configSequence: 10
    configLevel: 410
    requires:
      products: ["F&O"]
      skills: ["fo-inventory-management"]
    bpcProcess: "04 - Plan to Produce"
---
```

**Configuration Steps:**

| Step | Menu Path | Data Entity |
|---|---|---|
| 1. Production parameters | Production control > Setup > Production control parameters | Form tool |
| 2. Resource types | Organization administration > Resources > Resource types | `ResourceTypes` |
| 3. Resources | Organization administration > Resources > Resources | `Resources` |
| 4. Resource capabilities | Organization administration > Resources > Resource capabilities | `ResourceCapabilities` |
| 5. Route groups | Production control > Setup > Routes > Route groups | `RouteGroups` |
| 6. Operations | Production control > Setup > Routes > Operations | `Operations` |
| 7. Production groups | Production control > Setup > Production > Production groups | `ProductionGroups` |
| 8. Route headers | Production control > Routes > All routes | `RouteHeaders` |
| 9. Route operations | (within routes) | `RouteOperations` |

---

### 4.9 Fixed Assets Agent

**BPC Alignment:** Acquire to Dispose (08)

**Skill: `fo-fixed-assets`**

```yaml
---
name: fo-fixed-assets
description: >
  Configure D365 F&O Fixed assets including FA parameters, FA groups,
  depreciation profiles, value models, depreciation books,
  FA posting profiles, and FA acquisition processes.
  Use when: setting up fixed asset management and depreciation.
  DEPENDS ON: fo-gl-chart-of-accounts.
metadata:
  platform:
    category: "fo-fixed-assets"
    riskLevel: "medium"
    configSequence: 7
    configLevel: 150
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "08 - Acquire to Dispose"
---
```

---

### 4.10 Budgeting Agent

**BPC Alignment:** Record to Report (13) — Manage Budgets

**Skill: `fo-budgeting`**

```yaml
---
name: fo-budgeting
description: >
  Configure D365 F&O Budgeting including budget models, budget
  dimensions, allocation terms, budget control configuration,
  budget plans, and budget workflows.
  Use when: setting up budgeting and budget control.
  DEPENDS ON: fo-gl-chart-of-accounts (dimensions and accounts).
metadata:
  platform:
    category: "fo-budgeting"
    riskLevel: "medium"
    configSequence: 7
    configLevel: 160
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "13.30 - Manage budgets"
---
```

---

## 5. Data Migration Framework & Entity Catalog

### DMF Package Numbering Convention

Microsoft uses a structured numbering format for data packages:

```
Format: [Module#].[DataType#].[Sequence#]

Module Numbers:
  00 = Shared/System
  01 = General Ledger
  02 = Bank
  03 = Accounts Payable
  04 = Tax
  05 = Accounts Receivable
  06 = Fixed Assets
  07 = Budgeting
  10 = Inventory Management
  11 = Product Management
  12 = Procurement
  13 = Sales & Marketing
  15 = Warehouse Management
  16 = Transportation
  17 = Production Control
  18 = Process Manufacturing
  19 = Costing
  20 = Retail

Data Type Numbers:
  1 = Setup/Configuration
  2 = Master Data (reference)
  3 = Transaction Data (documents)
```

### Complete Data Migration Sequence

```
═══════════════════════════════════════════════════════════
STAGE 1: CONFIGURATION DATA (Setup entities)
═══════════════════════════════════════════════════════════

00.1.001  System parameters
00.1.002  Global address book setup
00.1.003  Number sequences (shared)
01.1.001  Fiscal calendars
01.1.002  Currency codes
01.1.003  Exchange rate types & rates
01.1.004  Chart of accounts
01.1.005  Main account categories
01.1.006  Financial dimensions
01.1.007  Account structures
01.1.008  Ledger setup
01.1.009  Journal names
01.1.010  GL parameters
01.1.011  Posting definitions
02.1.001  Bank groups
02.1.002  Bank transaction types
03.1.001  AP parameters
03.1.002  Vendor groups
03.1.003  Vendor posting profiles
04.1.001  Tax authorities
04.1.002  Settlement periods
04.1.003  Tax codes & values
04.1.004  Tax groups
04.1.005  Item tax groups
04.1.006  Tax ledger posting groups
05.1.001  AR parameters
05.1.002  Customer groups
05.1.003  Customer posting profiles
06.1.001  FA parameters
06.1.002  FA groups
06.1.003  Depreciation profiles
06.1.004  Value models
06.1.005  FA posting profiles
10.1.001  Inventory parameters
10.1.002  Item model groups
10.1.003  Item groups
10.1.004  Dimension groups
10.1.005  Inventory posting profiles
10.1.006  Sites
10.1.007  Warehouses

═══════════════════════════════════════════════════════════
STAGE 2: MASTER DATA (Reference entities)
═══════════════════════════════════════════════════════════

00.2.001  Parties (Global address book)
00.2.002  Addresses
00.2.003  Contact information
01.2.001  Main accounts
01.2.002  Financial dimension values
02.2.001  Bank accounts
03.2.001  Vendors
03.2.002  Vendor bank accounts
03.2.003  Vendor contacts
05.2.001  Customers
05.2.002  Customer bank accounts
05.2.003  Customer contacts
06.2.001  Fixed assets
06.2.002  Fixed asset books
10.2.001  Units of measure
10.2.002  Unit conversions
11.2.001  Product categories
11.2.002  Released products
11.2.003  Product variants
11.2.004  Product attributes
12.2.001  Vendor catalog items
13.2.001  Sales prices / Trade agreements
17.2.001  Resources
17.2.002  Routes
17.2.003  BOM headers & lines

═══════════════════════════════════════════════════════════
STAGE 3: OPENING BALANCES & TRANSACTIONS
═══════════════════════════════════════════════════════════

01.3.001  GL opening balances (journal import)
03.3.001  Vendor open transactions (invoices)
05.3.001  Customer open transactions (invoices)
06.3.001  Fixed asset opening balances
10.3.001  Inventory on-hand balances
17.3.001  Open production orders (if migrating)
```

### Validation Strategy Per Stage

**Our Migration Agent performs validation at each stage:**

```
STAGE 1 VALIDATION (Configuration)
├── All legal entities exist and are active
├── Chart of accounts is complete (all required main accounts)
├── All dimension values resolve
├── Account structures validate
├── Posting profiles have valid GL accounts
├── Tax codes/groups are complete
├── Number sequences are configured and active
└── Parameters are set (no null required fields)

STAGE 2 VALIDATION (Master Data)
├── All vendors/customers have valid groups
├── All vendors/customers have valid posting profiles
├── All products have valid item groups & dimension groups
├── Bank accounts link to valid GL accounts
├── No orphaned dimension values
├── Address validation (country/state/city hierarchy)
├── Duplicate detection (vendor/customer names, tax IDs)
└── Cross-reference validation (AP/AR to GL control accounts)

STAGE 3 VALIDATION (Opening Balances)
├── GL trial balance = 0 (debits = credits)
├── AP subledger = GL AP control account
├── AR subledger = GL AR control account
├── Inventory subledger = GL inventory control account
├── FA subledger = GL FA control accounts
├── Aging validation (AP/AR aging reports match)
└── Currency balance validation
```

### Pulling Related Master Data (Seamless Process)

When migrating an entity, the agent automatically identifies and validates related master data:

```
EXAMPLE: Migrating Vendors
═════════════════════════

Agent detects vendor file contains:
  • Vendor accounts
  • Vendor group codes
  • Payment terms codes
  • Currency codes
  • Tax group codes
  • Address data

Agent runs pre-migration validation:
  1. Query target: data_find_entities("VendorGroups")
     → Missing groups: "TRADE-20", "SERVICE-10"
     → Action: Flag for creation or notify consultant

  2. Query target: data_find_entities("PaymentTerms")
     → All payment terms exist ✓

  3. Query target: data_find_entities("SalesTaxGroups")
     → Missing: "EXEMPT-EXPORT"
     → Action: Flag — cannot migrate vendors using this group

  4. Query target: data_find_entities("CountryRegions")
     → All countries exist ✓

  5. Report to consultant:
     "Vendor migration pre-check: 2 blockers found.
      - 2 vendor groups missing: TRADE-20, SERVICE-10
      - 1 tax group missing: EXEMPT-EXPORT
      Create these before proceeding? [Approve/Modify/Skip]"
```

---

## 6. Master Data Dependencies & Validation

### Complete Dependency Matrix

```
Entity                    Depends On
══════════════════════════════════════════════════════════════
Legal Entity             → None (foundation)
Fiscal Calendar          → None (foundation)
Currency                 → None (foundation)
Chart of Accounts        → None (foundation)
Main Account             → Chart of Accounts, Main Account Category
Financial Dimension      → None (definition only)
Dimension Value          → Financial Dimension
Account Structure        → Chart of Accounts, Financial Dimensions
Ledger                   → Legal Entity, CoA, Fiscal Calendar, Currencies
Number Sequence          → Legal Entity (scope)
Address Format           → None
Country/Region           → Address Format
State/Province           → Country/Region
Party (GAB)              → Address Format, Country/Region
Tax Authority            → Vendor (creates auto-vendor), Address
Settlement Period        → Tax Authority
Tax Code                 → Settlement Period
Tax Group                → Tax Code(s)
Item Tax Group           → Tax Code(s)
Tax Ledger Posting Group → Main Account(s)
Bank Group               → None
Bank Account             → Bank Group, Currency, Main Account
Payment Terms            → None
Cash Discount            → Main Account
Method of Payment        → Bank Account, Payment Format
Vendor Group             → None (but posting profile needs Main Account)
Vendor Posting Profile   → Vendor Group, Main Account(s)
Vendor                   → Vendor Group, Payment Terms, Tax Group, Currency, Address
Customer Group           → None (but posting profile needs Main Account)
Customer Posting Profile → Customer Group, Main Account(s)
Customer                 → Customer Group, Payment Terms, Tax Group, Currency, Address
Item Model Group         → None (but inventory posting needs Main Account)
Item Group               → None (but inventory posting needs Main Account)
Dimension Group (Storage)→ None
Dimension Group (Track)  → None
Inventory Posting        → Item Group/Model Group, Main Account(s), Site
Unit of Measure          → None
Site                     → Legal Entity, Address
Warehouse                → Site
Released Product         → Item Model Group, Item Group, Dimension Groups, Unit
BOM Header               → Released Product
BOM Line                 → BOM Header, Released Product, Unit
Route                    → Resources, Operations
Production Order         → Released Product, BOM, Route, Site, Warehouse
FA Group                 → None
Depreciation Profile     → None
Value Model              → Depreciation Profile
FA Posting Profile       → FA Group, Main Account(s)
Fixed Asset              → FA Group, Value Model
Budget Model             → None
Budget Register          → Budget Model, Main Account, Dimension Values
```

---

## 7. MCP-Powered Agent Prompts Per Module

### Base System Prompt (for all F&O agents)

```
# Role
You are a D365 Finance & Operations configuration agent. You interact with
D365 F&O environments through the MCP server.

# Tool Selection Priority
1. For CRUD operations → ALWAYS use data tools first
2. For configuration that requires clicking buttons, navigating forms,
   or triggering business logic → use form tools
3. For custom X++ operations → use action tools

# Safety Rules
- NEVER modify production environments without explicit multi-person approval
- ALWAYS validate configuration before applying
- ALWAYS show the consultant what will change before executing
- Log every action to the audit trail
- When in doubt, READ first, then propose changes

# Data Tool Instructions
- Use data_find_entity_type before any CRUD operation
- Use data_get_entity_metadata to understand field requirements
- Use PLURAL entity names in OData paths
- Deep inserts NOT supported — create parent before child

# Form Tool Instructions
- Use menu item NAMES not labels
- Use control NAMES not labels
- Use tab NAMES not labels
- Always close forms when done
- If form state returns 25 rows, warn about truncation
```

### GL Configuration Agent Prompt

```
# Module: General Ledger
# BPC Process: 13 - Record to Report

You are configuring the General Ledger module. This is the FOUNDATION
of all financial modules — every other module posts to GL.

# Configuration Sequence (MUST follow this order):
1. Fiscal calendars and periods
2. Chart of accounts structure
3. Main account categories
4. Main accounts
5. Financial dimensions and values
6. Account structures
7. Ledger setup (assign CoA, calendar, currencies to legal entity)
8. Journal names and voucher series
9. GL parameters (all tabs)
10. Posting definitions and default descriptions

# Key Menu Paths:
- Chart of accounts: General ledger > Chart of accounts > Accounts > Main accounts
- Dimensions: General ledger > Chart of accounts > Dimensions > Financial dimensions
- Ledger setup: General ledger > Ledger setup > Ledger
- Parameters: General ledger > Ledger setup > General ledger parameters
- Journal names: General ledger > Journal setup > Journal names

# Validation Checks After Configuration:
- [ ] Every main account has a category and account type
- [ ] Account structures activate without errors
- [ ] At least one journal name exists per journal type needed
- [ ] Fiscal calendar has all required periods
- [ ] Ledger is assigned to legal entity with correct currencies
- [ ] Number sequences are active for journal vouchers

# Common Errors to Watch For:
- Account structure doesn't include required dimensions
- Main account missing from a posting profile
- Fiscal period not open for posting date
- Exchange rate missing for transaction currency
```

---

## 8. Microsoft Learn Integration

### Certification-Aligned Knowledge Sources

Our platform skills reference Microsoft Learn content as authoritative documentation. Each agent's skill references can be sourced from these certification paths:

#### MB-310: D365 Finance Functional Consultant

| Exam Topic (40-45%) | MS Learn Path | Our Skill |
|---|---|---|
| Design & configure chart of accounts | [Configure GL](https://learn.microsoft.com/en-us/training/paths/configure-use-general-ledger-dyn365-finance/) | `fo-gl-chart-of-accounts` |
| Configure ledgers & currencies | [Set up ledgers & journals](https://learn.microsoft.com/en-us/training/modules/configure-ledgers-journals-dyn365-finance/) | `fo-gl-chart-of-accounts` |
| Implement & manage journals | Same path | `fo-gl-journals` |
| Cash and bank management | [Configure cash & bank](https://learn.microsoft.com/en-us/training/paths/configure-cash-bank-management-dyn365-finance/) | `fo-cash-bank-management` |
| Periodic processes | [Perform periodic GL processes](https://learn.microsoft.com/en-us/training/paths/perform-periodic-processes/) | `fo-period-close` |
| Tax configuration | [Configure taxes](https://learn.microsoft.com/en-us/training/paths/work-tax-dyn365-finance/) | `fo-tax-configuration` |
| Cost management | [Configure cost management](https://learn.microsoft.com/en-us/training/paths/configure-use-cost-management-dyn365-finance/) | `fo-costing` |

| Exam Topic (15-20%) | MS Learn Path | Our Skill |
|---|---|---|
| AR setup | [Configure AR](https://learn.microsoft.com/en-us/training/paths/configure-use-accounts-receivable-dyn365-finance/) | `fo-accounts-receivable` |
| Credit & collections | Same path | `fo-credit-collections` |

| Exam Topic (10-15%) | MS Learn Path | Our Skill |
|---|---|---|
| AP setup & expenses | [Configure AP](https://learn.microsoft.com/en-us/training/paths/configure-use-accounts-payable-dyn365-finance/) | `fo-accounts-payable` |
| Budgeting (10-15%) | [Configure budgeting](https://learn.microsoft.com/en-us/training/paths/configure-use-budgeting-dyn365-finance/) | `fo-budgeting` |
| Fixed assets (10-15%) | [Configure FA](https://learn.microsoft.com/en-us/training/paths/configure-manage-fixed-assets-dyn365-finance/) | `fo-fixed-assets` |

#### MB-330: D365 Supply Chain Management

| Topic | MS Learn Path | Our Skill |
|---|---|---|
| Inventory management | [Configure inventory mgmt](https://learn.microsoft.com/en-us/training/paths/configure-manage-inventory-dyn365-supply-chain-mgmt/) | `fo-inventory-management` |
| Product management | [Configure product mgmt](https://learn.microsoft.com/en-us/training/paths/configure-manage-products-dyn365-supply-chain-mgmt/) | `fo-product-management` |
| Procurement | [Configure procurement](https://learn.microsoft.com/en-us/training/paths/configure-manage-procurement-dyn365-supply-chain-mgmt/) | `fo-procurement` |
| Production | [Configure production](https://learn.microsoft.com/en-us/training/paths/configure-manage-production-dyn365-supply-chain-mgmt/) | `fo-production-control` |
| Warehouse management | [Configure WMS](https://learn.microsoft.com/en-us/training/paths/configure-manage-warehouse-management-dyn365-supply-chain-mgmt/) | `fo-warehouse-management` |

### Skill Reference Architecture

Each skill includes a `references/` directory with curated content from Microsoft Learn:

```
skills/fo-gl-chart-of-accounts/
├── SKILL.md
├── references/
│   ├── ms-learn-gl-overview.md          ← Curated from MS Learn
│   ├── chart-of-accounts-guide.md       ← Curated from MS Learn
│   ├── financial-dimensions-guide.md    ← Curated from MS Learn
│   ├── account-structures-guide.md      ← Curated from MS Learn
│   ├── posting-profiles-reference.md    ← Curated from MS Learn
│   ├── data-entities-gl.md              ← Entity list with fields
│   └── bpc-record-to-report.md          ← BPC process mapping
├── scripts/
│   ├── validate-coa.py                  ← Validate chart of accounts
│   └── generate-coa-template.py         ← Generate CoA from template
└── assets/
    ├── coa-template.xlsx                ← Standard CoA template
    └── dimension-mapping.xlsx           ← Dimension mapping template
```

### MCP + Microsoft Learn Integration for Agent Intelligence

```
Agent receives: "Set up chart of accounts for manufacturing company"

Agent flow:
1. Load skill: fo-gl-chart-of-accounts
2. Load reference: ms-learn-gl-overview.md (context)
3. Load reference: chart-of-accounts-guide.md (how-to)
4. Check BPC: Record to Report > Define accounting policies
5. Use MCP data tools:
   a. data_find_entity_type("ChartOfAccounts") → discover entity
   b. data_get_entity_metadata(entity) → understand schema
   c. data_find_entities("MainAccountCategories") → check existing
6. Present plan to consultant:
   "Based on manufacturing best practices, I recommend:
    - 5-digit account numbers (10000-99999)
    - 3 financial dimensions: Department, CostCenter, Project
    - Account categories aligned to GAAP/IFRS
    Here's the proposed chart: [Adaptive Card with CoA structure]"
7. After approval → execute via MCP data tools
```

---

## Summary: Complete Agent & Skill Inventory for F&O

| Agent | Skills | Config Level | BPC Process |
|---|---|---|---|
| **Org Admin Agent** | `fo-org-admin` | 10-15 | 99 - Administer to Operate |
| **GL Agent** | `fo-gl-chart-of-accounts`, `fo-gl-journals`, `fo-gl-posting-profiles`, `fo-period-close`, `fo-financial-reporting` | 20-25 | 13 - Record to Report |
| **Tax Agent** | `fo-tax-configuration` | 130 | 13 - Record to Report (tax) |
| **Bank Agent** | `fo-cash-bank-management` | 100 | 13 - Record to Report (cash) |
| **AP Agent** | `fo-accounts-payable` | 120 | 05 - Source to Pay |
| **AR Agent** | `fo-accounts-receivable`, `fo-credit-collections` | 140 | 02 - Order to Cash |
| **FA Agent** | `fo-fixed-assets` | 150 | 08 - Acquire to Dispose |
| **Budget Agent** | `fo-budgeting` | 160 | 13 - Record to Report (budget) |
| **Inventory Agent** | `fo-inventory-management`, `fo-product-management` | 300-310 | 12 - Inventory to Deliver |
| **Procurement Agent** | `fo-procurement` | 320 | 05 - Source to Pay |
| **Sales Agent** | `fo-sales-marketing` | 330 | 02 - Order to Cash |
| **Production Agent** | `fo-production-control`, `fo-process-manufacturing` | 410-412 | 04 - Plan to Produce |
| **Costing Agent** | `fo-costing` | 420 | 13 - Record to Report (cost) |
| **Migration Agent** | `fo-data-migration`, `fo-dmf-operations`, `fo-data-validation` | All | Cross-cutting |
| **Validation Agent** | `fo-validation-framework` | All | Cross-cutting |

**Total: 14 specialized agents, 25+ skills, covering all core F&O modules**

---

*This document should be read alongside ARCHITECTURE.md for the full platform design.*
