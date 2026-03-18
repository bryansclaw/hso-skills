# F&O Deep Research Addendum — Verified & Expanded Findings

**Date:** 2026-03-18
**Status:** Research in progress — iterating
**Companion to:** ARCHITECTURE.md, FNO-CONFIG-MIGRATION-DEEP-DIVE.md

---

## Critical Platform Changes (2025-2026)

### LCS → PPAC Migration (CRITICAL for architecture)

**This changes our platform architecture significantly.**

- **Starting February 2026:** New customers CANNOT create projects in Microsoft Dynamics Lifecycle Services (LCS) for D365 Finance, SCM, or Project Operations
- **LCS environment management is being fully retired** — replaced by Power Platform Admin Center (PPAC)
- New environment types: **UDE** (Unified Developer Environment) and **USE** (Unified Sandbox Environment) replace traditional Tier-1/Tier-2+
- All new implementations must use PPAC for environment provisioning
- The "One Dynamics One Platform" convergence means F&O environments are now Dataverse-native

**Impact on our platform:**
- Our environment connector must target PPAC APIs, not LCS APIs
- Environment provisioning agents need to work with PPAC PowerShell/REST
- Configuration packages move from LCS Asset Library to PPAC/Dataverse solutions
- Dual-write becomes the default integration pattern (not optional)
- The D365 MCP server requires Tier 2+ or PPAC-deployed environments (NOT Cloud Hosted Environments)

### MCP Server Evolution (verified details)

**Static → Dynamic transition (from Microsoft Ignite 2025 blog):**

| Aspect | Static MCP (Build 2025) | Dynamic MCP (Ignite 2025+) |
|---|---|---|
| Tools | 13 fixed tools | 3 tool categories, dynamically generated |
| Coverage | Specific operations (check inventory, match invoices, etc.) | "Hundreds of thousands" of ERP functions across "tens of thousands of forms" |
| Extensibility | None — fixed tools | ISV extensions and customizations automatically available |
| Security | Fixed permissions | Dynamic per-user security context |
| Status | **Retiring in 2026** | Public preview (v10.0.47+) |
| Recommended model | N/A | **Claude Sonnet 4.5** (Microsoft's own recommendation) |

**Key MCP limitations and gotchas (from community + Microsoft docs):**
1. **Performance:** Community reports describe the preview as "slow to respond and behave inconsistently" depending on agent interface (source: MSDynamicsWorld review)
2. **Form state truncation:** Tool responses max at 25 rows — agent must warn about truncation
3. **No deep inserts:** Can't create nested entities in one call — parent first, then child
4. **Entity naming:** Must use PLURAL names in OData paths. For V2+ entities, plural before V (e.g., `SalesOrderHeadersV2`)
5. **Sidecar integration not supported:** Adding MCP to the Copilot for F&O sidecar chat panel isn't supported — errors may occur
6. **Enum filtering syntax:** Must use format `$filter=Style has Namespace.Pattern'Yellow'`
7. **CHE not supported:** Cloud Hosted Environments don't work with MCP — must be Tier 2+ or PPAC UDE
8. **No write operations on grid filters** — grid operations are read-only
9. **Menu items:** Use NAMES not labels (critical mistake that causes failures)
10. **Form tool vs Data tool selection:** Microsoft explicitly recommends data tools for CRUD, form tools for business logic only. Agents that use form tools for basic CRUD are 3-5x slower.

### MCP for Analytics (NEW — December 2025)

A **second** MCP server for analytics is now available:
- Provides governed access to **Business Performance Analytics (BPA)** data
- Includes measures, dimensions, reports, and semantic models
- Enables AI agents to reason over consistent financial definitions (revenue, margin, cash flow)
- This is directly relevant to our Meeting Intelligence pipeline and financial reporting agents

---

## Data Migration Framework — Verified Deep Dive

### Entity Categories (from Microsoft official docs)

| Category | Description | Example | Migration Relevance |
|---|---|---|---|
| **Parameter** | Single-record tables with settings | GL Parameters, AP Parameters | First to migrate, small volume |
| **Reference** | Small quantity reference data | Units, tax codes, currencies | Early migration, low risk |
| **Master** | Core business nouns — high volume | Customers, vendors, products | Bulk migration, validation-heavy |
| **Document** | Worksheet/transaction data | Sales orders, POs, open balances | Complex, dependency-heavy |
| **Transaction** | Posted historical transactions | Posted invoices, GL entries | Usually migrated in SUMMARY not detail |

**Critical insight from Microsoft guidance:** *"Posted transactions (non-idempotent items such as posted invoices and balances) are typically excluded during a full dataset copy. Migrating completed transactions can also lead to further complexity in trying to preserve the referential integrity. In general, transactions from a completed business process aren't migrated in detail but in summary."*

### DMF Performance Optimization (from Microsoft's FastTrack team)

| Optimization | How | Impact |
|---|---|---|
| **Disable change tracking** | Data management > Data entities > Disable Change Tracking | Significant import speed improvement |
| **Enable set-based processing** | Data entities tile > Set-based processing column | 5-10x faster for supported entities |
| **Configure batch threads** | System admin > Setup > Server configuration > Maximum batch threads | Default 8, can increase to 12-16 |
| **Import threshold record count** | Framework parameters > Entity settings > Configure entity execution parameters | Controls records per thread |
| **Import task count** | Same location | Controls parallel threads per entity |
| **Disable validations** | Entity structure > Run business validations = No | Faster but DANGEROUS — only for mature data |
| **Clean staging tables** | Job history cleanup tile | Prevents table bloat, improves SQL performance |
| **Update statistics** | LCS or direct SQL `sp_updatestats` | Critical before large imports on sandbox |
| **Break large files** | Split into smaller chunks | Gives SQL optimizer time to replan |

**General Journal entity specific tips (verified from MS Learn):**
- Values for `JOURNALBATCHNUMBER` should have ~10,000 lines each
- Physical vouchers should be kept small (~10 lines or less)
- When using set-based processing, voucher numbers MUST be provided in the import file
- `ACCOUNTDISPLAYVALUE` requires "Financial dimension configuration for integrating applications" to be configured first
- High cardinality dimensions (many new values) slow imports significantly
- TRANSDATE must be the same for all rows with the same VOUCHER — otherwise extra vouchers are created

### Opening Balances Migration Strategy (verified from community + Microsoft)

**The fundamental principle:** Control account balances CANNOT be posted directly to the GL. They MUST be posted through their respective subledgers.

**Correct sequence:**
```
1. Post subledger opening balances first:
   ├── AP: Import open vendor invoices via AP journal
   │   → Posts to: AP control account + vendor subledger
   ├── AR: Import open customer invoices via AR journal
   │   → Posts to: AR control account + customer subledger
   ├── FA: Import fixed asset acquisitions via FA journal
   │   → Posts to: FA control accounts + asset books
   ├── Inventory: Import via inventory movement journal
   │   → Posts to: Inventory control accounts + item ledger
   └── Bank: Import via bank journal
       → Posts to: Bank control accounts

2. Post GL opening balances EXCLUDING control accounts:
   ├── Use General Journal entity
   ├── Import remaining GL account balances
   ├── EXCLUDE: AP control, AR control, FA control, Inventory control, Bank
   └── These are already populated by step 1

3. Reconcile:
   ├── AP subledger total = AP control account in GL
   ├── AR subledger total = AR control account in GL
   ├── Inventory subledger = Inventory control in GL
   ├── FA net book value = FA control accounts in GL
   ├── Bank subledger = Bank GL accounts
   └── Total GL trial balance = 0 (debits = credits)
```

**Microsoft's recommended practice (from posting profiles docs):**
*"Do a mock period close and reconcile each of your subledgers to the ledger before your go-live. Also do a mock cutover of all open balances and open transactions before your initial go-live."*

**Account Reconciliation feature (new in v10.0.44):**
D365 Finance now has a built-in Account Reconciliation feature that can reconcile GL with AP, AR, tax, and bank subledgers. Our validation agent should use this.

---

## Success by Design Integration

### SbD Phases → Our Platform Mapping

| SbD Phase | Activities | Our Agent Role |
|---|---|---|
| **Discover** | Gather requirements, validate business processes, high-level solution approach | Discovery Agent processes meetings, extracts requirements, maps to BPC |
| **Initiate** | Solution Blueprint Review, define workstreams, project plan | BPC Agent generates catalog, Config Agent reads BPC to plan config sequence |
| **Implement** | Build solution per design, Implementation Reviews | Config/Migration/Solution Agents execute, Validation Agent checks |
| **Prepare** | UAT, training, cutover plan, mock go-lives, go/no-go criteria | Migration Agent runs mock cutover, Validation Agent reconciles |
| **Operate** | Go-live, stabilization, phase 2 planning | Monitoring agents, DevOps Agent tracks issues |

### Solution Blueprint Review → Our Platform Equivalent

The SbD Solution Blueprint Review asks:
1. Is the solution roadmap-aligned?
2. Are technical and project risks addressed?
3. Is the data model correct?
4. Is the security model appropriate?
5. Are integration patterns defined?

Our platform should automatically generate a "Blueprint Readiness Report" that answers these using data from:
- BPC catalog coverage (in-scope vs out-of-scope processes)
- Configuration completeness per module
- Data entity mapping coverage
- Security role assignments per agent
- Integration pattern documentation

### Fit-to-Standard / Fit-Gap Analysis

**Microsoft's recommended approach (process-focused implementation):**

1. Download BPC catalog → identify applicable processes
2. Don't delete out-of-scope processes — mark them as "manual" or "other system"
3. Use BPC as DevOps work items (importable to Azure DevOps)
4. Organize workshops by Level 2 business process areas
5. Configure solution to match standard processes FIRST
6. Only then identify gaps requiring customization
7. Evaluate gap resolution: config change vs extension vs ISV solution

**Platform integration opportunity:** Our Discovery Agent should automatically categorize meeting-extracted requirements as:
- **Fit** — matches BPC standard process
- **Config** — achievable through configuration (no code)
- **Gap** — requires extension or ISV
- **Process Change** — customer should adopt standard instead of customizing

---

## Integration Architecture Patterns

### When to Use What (verified from Microsoft guidance)

| Pattern | Use For | Not For |
|---|---|---|
| **Data entities (OData)** | Synchronous API calls, Excel add-in, lightweight integration | Bulk migration (too slow for large volumes) |
| **DMF (Data Management Framework)** | Bulk migration, config import/export, recurring integration | Real-time sync |
| **Dual-write** | Near-real-time bidirectional sync between F&O and Dataverse/CE | Bulk migration (NOT designed for high volume) |
| **Virtual entities** | Read access to F&O data from Dataverse without data movement | Write operations, complex queries |
| **MCP Server** | AI agent interactions, configuration automation, headless operations | Bulk data operations (data tools use OData underneath) |
| **Business Events** | Event-driven integration (triggers on business actions) | Data sync, migration |
| **Azure Data Factory** | Complex ETL, hybrid/cloud migration, data warehouse feeding | Real-time operations |

**Critical warnings from Microsoft:**
- **Dual-write Initial Sync is NOT a replacement for data migration** — it's designed for low-volume setup/reference data only
- Keep data migration SEPARATE from Dual-write infrastructure
- Dual-write uses F&O OData entities — not suitable for bulk import
- Don't compare Tier-1 performance to Tier-2+ (results are not representative)

---

## Common DMF Error Patterns (verified from Microsoft docs)

Our Migration Agent's error handling should recognize and remediate these:

| Error | Cause | Agent Response |
|---|---|---|
| "Update conflict — more than one process updating same record" | Parallel processing + duplicate records across files | Deduplicate source files, enable sequential processing |
| "Fields not mapped to Entity" | Template generated with "First row header = No" | Regenerate template with headers enabled |
| "XML not in correct format" | Mapping includes columns not in file | Remove unmapped fields or fix file |
| "Number sequence not set" | Auto-generate enabled but no sequence configured | Configure number sequence before import |
| "Column has incorrect data" | Null/empty/duplicate in mandatory columns | Pre-validate data, flag specific rows |
| "Schema mismatch in AxDb and BYOD" | Entity schema updated but not refreshed | Refresh entity list, republish to BYOD |
| "Entity Execution Sequence" error | Wrong import order, dependency not met | Use `GetEntitySequence` API to validate order |
| Excel "0xC02020E8" | Unmapped fields in Excel sheet | Remove unmapped columns from Excel |
| "Confidential" label on Excel | Excel labeled as confidential | Remove confidential label from file |
| Import task count > 1 for unsupported entities | Some entities don't support multithreading | Set task count to 1 (e.g., `CustomersV3`) |

---

## Microsoft Learn Training Paths (verified URLs)

### Finance Module Paths
| Path | URL | Skills Covered |
|---|---|---|
| Configure GL | https://learn.microsoft.com/en-us/training/paths/configure-use-general-ledger-dyn365-finance/ | CoA, dimensions, journals, ledger setup |
| Cash & Bank | https://learn.microsoft.com/en-us/training/paths/configure-cash-bank-management-dyn365-finance/ | Bank accounts, reconciliation, cash flow |
| Periodic GL | https://learn.microsoft.com/en-us/training/paths/perform-periodic-processes/ | Period close, revaluation, consolidation |
| Configure AP | https://learn.microsoft.com/en-us/training/paths/configure-use-accounts-payable-dyn365-finance/ | Vendor setup, invoices, payments |
| Configure AR | https://learn.microsoft.com/en-us/training/paths/configure-use-accounts-receivable-dyn365-finance/ | Customer setup, invoices, collections |
| Budgeting | https://learn.microsoft.com/en-us/training/paths/configure-use-budgeting-dyn365-finance/ | Budget models, control, planning |
| Fixed Assets | https://learn.microsoft.com/en-us/training/paths/configure-manage-fixed-assets-dyn365-finance/ | FA groups, depreciation, acquisition |
| Taxes | https://learn.microsoft.com/en-us/training/paths/work-tax-dyn365-finance/ | Tax codes, groups, settlement |
| Cost Management | https://learn.microsoft.com/en-us/training/paths/configure-use-cost-management-dyn365-finance/ | Costing methods, item model groups |
| Financial Management (comprehensive) | https://learn.microsoft.com/en-us/training/paths/set-up-configure-financial-management-work-general-ledger/ | All GL processes |

### MB-310 Exam Structure (verified July 2025 update)
- **Implement financial management: 40-45%** (GL, cash/bank, periodic, tax, cost mgmt)
- **AR, credit, collections, subscription billing: 15-20%**
- **AP and expenses: 10-15%**
- **Budgeting: 10-15%**
- **Fixed assets: 10-15%**

### Supply Chain Paths
| Path | URL | Skills Covered |
|---|---|---|
| Inventory Mgmt | https://learn.microsoft.com/en-us/training/paths/configure-manage-inventory-dyn365-supply-chain-mgmt/ | Item groups, posting, sites/warehouses |
| Product Mgmt | https://learn.microsoft.com/en-us/training/paths/configure-manage-products-dyn365-supply-chain-mgmt/ | Categories, dimensions, released products |
| Procurement | https://learn.microsoft.com/en-us/training/paths/configure-manage-procurement-dyn365-supply-chain-mgmt/ | Purchase setup, vendor selection |
| Production | https://learn.microsoft.com/en-us/training/paths/configure-manage-production-dyn365-supply-chain-mgmt/ | Routes, resources, BOM |
| Warehouse | https://learn.microsoft.com/en-us/training/paths/configure-manage-warehouse-management-dyn365-supply-chain-mgmt/ | WMS setup, waves, work templates |

---

## Key GitHub Repositories

| Repo | Purpose | Relevance |
|---|---|---|
| [D365-FastTrack-Architecture-Insights/Partner-Architect-Sessions](https://github.com/D365-FastTrack-Architecture-Insights/Partner-Architect-Sessions) | FastTrack team's implementation guidance PDFs | Reference material for agent skills |
| [mafzaal/d365fo-mcp-prompts](https://github.com/mafzaal/d365fo-mcp-prompts) | MCP prompt templates for DMF operations | Direct integration — DMF automation prompts |
| [srikanth-paladugula/mcp-dynamics365-server](https://github.com/srikanth-paladugula/mcp-dynamics365-server) | Community MCP server for D365 | Alternative/complementary to Microsoft's official server |
| [thedataguy.pro D365FO MCP Server](https://thedataguy.pro/d365fo-client/) | VS Code extension for D365FO MCP | One-click install for development scenarios |

---

---

## Business Performance Analytics (BPA) — Verified Deep Dive

### What BPA Is

BPA is Microsoft's built-in analytics layer for D365 Finance — **included in the Finance license at no extra cost**. It consolidates financial and operational data through pre-defined data models and measures.

**Key characteristics:**
- **Data refreshes:** Currently twice per day
- **History:** Rolling 4 years (3 years history + current year)
- **Built on:** Microsoft Dataverse + Power BI
- **Extensible via:** Microsoft Fabric (bring your own Fabric workspace)
- **Reports:** Pre-configured Power BI reports for finance, supply chain, projects, commerce

### BPA Data Model (relevant for MCP Analytics server)

The MCP for Analytics server connects to BPA's dimensional models, enabling:
- Natural language queries against BPA data
- Dynamic generation of DAX queries by AI agents
- Row-level security enforcement based on user roles
- Integration with VS Code and Copilot Studio

**This is critical for our platform:**
- Our Financial Reporting Agent should use MCP Analytics server, NOT raw GL queries
- BPA provides pre-computed measures that are more reliable than ad-hoc calculations
- Consistency guarantee: all agents reference the same definitions for revenue/margin/cash flow
- BPA's data quality assessment automatically logs issues — our validation agent should monitor these

### BPA Use Cases Aligned to Our Agents

| Use Case | BPA Capability | Our Agent |
|---|---|---|
| Financial reporting | Pre-built trial balance, income statement, balance sheet | Financial Reporting Agent |
| Budget vs actuals | Budget comparison measures | Budget Agent |
| Cash flow analysis | AI-driven cash flow forecasting | Cash Management Agent |
| Inventory analytics | Stock levels, turnover, aging | Inventory Agent |
| Project tracking | Milestones, budget adherence, utilization | Project Agent (Phase 2) |

---

## Unified Developer Environment (UDE) — Verified

### What's Changing

| Old Model | New Model |
|---|---|
| Tier-1 dev VMs (CHE via LCS) | **UDE** — Unified Developer Environment via PPAC |
| Tier-2+ sandbox via LCS | **USE** — Unified Sandbox Environment via PPAC |
| LCS project-based management | PPAC + Azure DevOps pipeline-based management |
| X++ compilation on local VM | Cloud-based compilation |
| Manual deployment packages | Enforced ALM through Azure DevOps pipelines |

### Impact on Our Platform Architecture

1. **Environment provisioning:** Our platform must use PPAC PowerShell/REST APIs, not LCS APIs
2. **Solution deployment:** All code changes deploy through Azure DevOps pipelines — no manual package upload
3. **Development workflow:** UDE eliminates the old model of downloading VMs. Development is cloud-native.
4. **Dataverse-native:** Every F&O environment is now a Dataverse environment. This means:
   - Power Platform capabilities are available by default
   - Dual-write is the integration mechanism
   - Power Automate flows can trigger on F&O events
   - Model-driven apps can access F&O data via virtual entities

### For Our Solution Agent

The Solution Agent that generates X++ code must account for:
- Code must be packaged as Dataverse solutions (not just model packages)
- Deployment goes through Azure DevOps pipelines with gated approvals
- UDE supports hot-swapping of code during development
- No more remote desktop to dev VMs — VS Code + Power Platform CLI is the new workflow

---

## Dual-Write Architecture (Critical for CE+F&O implementations)

### When Dual-Write Applies

If the D365 practice implements BOTH CE (Sales, Customer Service, Field Service) AND F&O (Finance, SCM), dual-write is the near-real-time bidirectional sync mechanism.

**Key warnings verified from Microsoft:**
- Dual-write Initial Sync is NOT for data migration — it's for low-volume reference data only
- Keep data migration completely SEPARATE from dual-write infrastructure
- Dual-write uses F&O OData entities — inherently slower than DMF for bulk operations
- Customer/vendor sync requires MULTIPLE supporting maps (CDS Parties, Party Postal Addresses, Party Contacts V3)

### Dual-Write Maps Relevant to Configuration

| Map | F&O Entity | Dataverse Table | Direction |
|---|---|---|---|
| Customers V3 | CustomersV3 | account | Bidirectional |
| Vendors V2 | VendorsV2 | msdyn_vendor | Bidirectional |
| Products V2 | ReleasedProductsV2 | msdyn_sharedproductdetails | F&O → Dataverse |
| Currency | Currencies | transactioncurrency | F&O → Dataverse |
| Sales orders | SalesOrderHeadersV2 | salesorder | Bidirectional |
| Purchase orders | PurchaseOrderHeadersV2 | msdyn_purchaseorder | F&O → Dataverse |

### Impact on Our Config Agents

Our Config Agents must be aware of dual-write when configuring:
- **Customer/Vendor setup:** Changes in F&O propagate to Dataverse in near-real-time. Agent must validate both sides.
- **Product setup:** Released products sync to Dataverse shared product details
- **Currency/tax setup:** Foundation data syncs across both platforms
- If dual-write is enabled, the agent must check dual-write map health before and after configuration changes

---

---

## Electronic Reporting (ER) Framework — Verified

### What ER Is

ER is D365's configurable tool for creating and maintaining regulatory electronic reporting and payment formats. **Configuration instead of coding** — business users can create/modify formats without developers.

**Key concepts:**
- **Data Model** — business-domain abstraction over database tables
- **Model Mapping** — maps data model to F&O data sources
- **Format** — defines the output file structure (XML, JSON, Excel, PDF, Word, TEXT, OPENXML)
- **Configuration** — versioned wrapper containing model + mapping + format

**Supported output formats:** TEXT, XML, JSON, PDF, Word, Excel, OPENXML

### ER for Payment Processing

| Region | Format | Use Case |
|---|---|---|
| US | NACHA/ACH | Electronic vendor payments (domestic) |
| US | Positive Pay | Bank check fraud prevention |
| Europe | SEPA Credit Transfer (ISO 20022) | Euro vendor payments |
| Europe | SEPA Direct Debit | Customer collections |
| Global | SWIFT MT940 | Bank statement import |
| Global | BAI2 | Bank statement import (US) |
| Global | camt.053 | Bank statement import (ISO 20022) |

### ER Configuration Sources

- **Dataverse repository** — primary source (replacing LCS)
- **Microsoft configuration provider** — standard formats published by Microsoft
- **Custom provider** — organization-specific formats
- Localizations can be child versions of Microsoft's standard configurations

### Impact on Our Platform

Our Config Agent (Cash & Bank) needs an ER skill:
- **Menu path:** Organization administration > Electronic reporting > Configurations
- Import ER configurations from Microsoft's repository
- Assign configurations to payment methods
- Agent must validate ER configuration version matches F&O version
- Custom formats require form tools to modify (ER designer is visual)

---

## Financial Dimension Configuration for Integrating Applications — CRITICAL

**This is the #1 overlooked setup that causes data import failures.**

**Menu path:** General ledger > Chart of accounts > Dimensions > Financial dimension configuration for integrating applications

### What It Does

Defines how dimension strings (like `001-02-ABC`) are parsed during DMF imports and OData operations. Without this, any entity import involving ledger dimensions or default dimensions WILL FAIL.

### Four Format Types

| Format Type | Required For | Includes Main Account? |
|---|---|---|
| **Default dimension format** | Importing default dimensions on customers, vendors, products | No |
| **Ledger dimension format** | Importing journal lines, transactions, GL data | Yes (first segment) |
| **Budget register dimension format** | Budget register entries | Context-dependent |
| **Budget planning dimension format** | Budget planning imports | Context-dependent |

### Critical Rules (verified from Microsoft docs + community)

1. **Only ONE format of each type can be active at a time**
2. **The format is GLOBAL** — applies to all legal entities (not company-specific)
3. **If no format is set up → import ERRORS** with cryptic messages
4. The format defines the ORDER of dimension segments in the import string
5. Extra segments in import files are DROPPED (not errored) if the format has fewer segments
6. Missing segments are treated as blank
7. **Must be configured BEFORE any data import that involves dimensions**

### Agent Pre-Configuration Checklist

Our Config Agent must verify this BEFORE any data migration:
```
1. Navigate to: GL > Chart of accounts > Dimensions > 
   Financial dimension configuration for integrating applications
2. Verify "Ledger dimension format" is active with correct segment order:
   MainAccount - Dimension1 - Dimension2 - ... 
3. Verify "Default dimension format" is active:
   Dimension1 - Dimension2 - ...
4. Match format to actual import file structures
5. If multi-company with different dimensions → format must include
   ALL dimensions across ALL legal entities (unused ones left blank)
```

---

## Azure DevOps ALM Pipeline — Verified Architecture

### Build Pipeline (X++ Compilation)

**Key change:** Microsoft now supports building X++ on Microsoft-hosted agents (no build VM required).

**Prerequisites:**
1. X++ project files (.rnrproj) in Azure DevOps repo
2. NuGet packages published to Azure Artifacts feed:
   - `Microsoft.Dynamics.AX.Platform.CompilerPackage` (X++ compiler)
   - `Microsoft.Dynamics.AX.Platform.DevALM.BuildXpp` (Platform reference)
   - `Microsoft.Dynamics.AX.Application1.DevALM.BuildXpp` (App reference 1)
   - `Microsoft.Dynamics.AX.Application2.DevALM.BuildXpp` (App reference 2)
   - `Microsoft.Dynamics.AX.ApplicationSuite.DevALM.BuildXpp` (AppSuite reference)
3. nuget.config in source control pointing to your feed

**Limitation:** Microsoft-hosted build supports compilation and packaging ONLY — no X++ unit testing (SysTest), no database sync, no AOS runtime.

### Pipeline Stages

```
┌────────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────────┐
│   Build    │───▶│   Package    │───▶│   Upload    │───▶│   Deploy     │
│            │    │              │    │   to LCS/   │    │   to Env     │
│ Compile    │    │ Create       │    │   PPAC      │    │              │
│ X++ code   │    │ deployable   │    │   Asset     │    │ Trigger via  │
│ via MSBuild│    │ package      │    │   Library   │    │ LCS/PPAC API │
└────────────┘    └──────────────┘    └─────────────┘    └──────────────┘
```

**Microsoft's official sample pipelines:**
- GitHub: [microsoft/Dynamics365-Xpp-Samples-Tools/CI-CD/Pipeline-Samples](https://github.com/microsoft/Dynamics365-Xpp-Samples-Tools/blob/master/CI-CD/Pipeline-Samples/README.md)
- Includes YML and Classic Pipeline JSON samples

### Impact on Our Solution Agent

Our Solution Agent that generates X++ code must:
1. Create properly structured .rnrproj project files
2. Generate code that compiles against the reference NuGet packages
3. Submit PRs to Azure DevOps repos (not direct commit)
4. Trigger build pipeline after PR merge
5. Monitor build status and report compilation errors
6. Never deploy directly — deployment goes through release pipeline with approval gates

---

## Power Automate + Business Events Integration

### How Business Events Work

F&O emits **Business Events** when specific business actions occur. Power Automate can subscribe to these events via the Dataverse connector.

**Two trigger types:**
1. **"When a Business Event occurs"** — F&O-specific business events (invoice posted, PO confirmed, workflow approved)
2. **"When a record is created, updated, or deleted"** — Dataverse CUD events on F&O virtual entities

### Common Business Event Scenarios for Our Platform

| Event | Trigger | Agent Response |
|---|---|---|
| Invoice posted | AP Business Event | Notify consultant, update DevOps task |
| PO approved | Workflow Business Event | Track procurement milestone |
| Payment processed | Cash Management Event | Update cash flow forecast |
| Config change detected | Custom event | Validation Agent checks for drift |
| Data import completed | DMF Event | Migration Agent runs validation |
| Period closed | GL Business Event | Financial Reporting Agent generates reports |

### Integration with Our Approval Workflow

Power Automate can handle our config change approval flow:
```
Agent proposes config change
    → Power Automate flow triggered
    → Approval sent to Teams (Adaptive Card)
    → Consultant approves/rejects in Teams
    → Flow calls back to our platform
    → Agent applies or cancels change
    → Audit logged
```

This means we might NOT need a fully custom approval system — Power Automate + Approvals connector handles this natively with Teams integration.

---

## ISV Solution Ecosystem

### Common ISV Categories for F&O Implementations

| Category | Major ISVs | Skills Impact |
|---|---|---|
| **Tax Automation** | Avalara (sales/use tax), Vertex, Thomson Reuters | Tax Agent needs awareness of ISV tax engines vs native tax |
| **AP Automation** | Kofax, ABBYY, Tungsten Network | AP Agent needs invoice scanning/OCR integration awareness |
| **EDI** | TrueCommerce, SPS Commerce, DiCentral | Procurement/Sales agents need EDI document flow knowledge |
| **Document Management** | Lasernet, Docentric, SSRS extensions | Reporting agents need document generation awareness |
| **Warehouse** | Prodware, Insight Works | WMS agent needs ISV extension awareness |
| **Manufacturing** | PROS, Siemens (MES integration) | Production agent needs MES integration patterns |
| **eCommerce** | Sana Commerce, Virto Commerce | Sales agent needs eCommerce order flow knowledge |
| **Banking** | Bottomline Technologies, Finastra | Cash/Bank agent needs bank connectivity ISV awareness |

**Key ISV reference:** "Best Dynamics 365 Add-Ons 2025" eBook (29 ISV solutions compared across 8 categories) — ERP Software Blog + MSDynamicsWorld collaboration

### How ISVs Affect Our Platform

1. **MCP server automatically exposes ISV extensions** — the dynamic MCP server makes ISV-added forms, entities, and actions available without additional configuration
2. **ISV skills should be optional add-ons** — not every customer uses the same ISVs
3. **ISV configuration is often the most complex part** — our skills engine should support ISV-specific configuration guides
4. **ISV data entities** follow the same DMF patterns — our Migration Agent handles them with the same tools

---

## Next Research Areas Remaining

- [ ] Download and analyze actual BPC Excel catalog (800+ processes, 3000+ scenarios with menu paths)
- [ ] Deep dive into per-entity documentation (Microsoft has individual pages for each built-in entity)
- [ ] Study the BPA MCP Analytics server API/tool definitions and DAX query patterns
- [ ] Research Regulatory Configuration Service (RCS) for tax and e-invoicing
- [ ] Research Copilot Studio as potential complement to our custom Teams bot approach
- [ ] Study the D365FO client MCP Python package (d365fo-client) for development tooling
- [ ] Research cutover planning templates and mock go-live checklists
- [ ] Study X++ extension patterns and code review guidelines for Solution Agent

---

*This is a living research document. Each section should be verified against the latest Microsoft Learn content before being incorporated into agent skills.*
