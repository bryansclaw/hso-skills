# D365 AI Implementation Platform

AI-powered implementation accelerator for Microsoft Dynamics 365 Finance & Operations.

## What This Is

A platform architecture and agent skill library for automating D365 F&O implementations — from discovery through configuration, data migration, and go-live. Inspired by [OpenClaw](https://github.com/openclaw/openclaw)'s agent orchestration patterns, adapted for the Microsoft stack.

## Repository Structure

```
├── ARCHITECTURE.md                    # Full platform architecture
├── FNO-CONFIG-MIGRATION-DEEP-DIVE.md  # Module config sequences & data entities
├── FNO-RESEARCH-ADDENDUM.md           # Deep research: MCP, PPAC, BPA, dual-write
│
└── skills/                            # 23 agent skills with extracted references
    ├── fo-org-admin/                  # Foundation (Level 10-15)
    ├── fo-gl-chart-of-accounts/       # General Ledger (Level 20-25)
    ├── fo-gl-journals/                # GL Journals
    ├── fo-gl-posting-profiles/        # Cross-module posting bridge
    ├── fo-accounts-payable/           # AP (Level 120)
    ├── fo-accounts-receivable/        # AR (Level 140)
    ├── fo-tax-configuration/          # Tax (Level 40-130)
    ├── fo-cash-bank-management/       # Bank (Level 100)
    ├── fo-fixed-assets/               # FA (Level 150)
    ├── fo-budgeting/                  # Budgeting (Level 160)
    ├── fo-inventory-management/       # Inventory (Level 300-310)
    ├── fo-procurement/                # Procurement (Level 320)
    ├── fo-sales-marketing/            # Sales (Level 330)
    ├── fo-production-control/         # Production (Level 410-412)
    ├── fo-warehouse-management/       # WMS (Level 400)
    ├── fo-financial-reporting/        # Financial Reporting
    ├── fo-period-close/               # Period & Year-End Close
    ├── fo-integration-patterns/       # MCP, Dual-write, OData, Business Events
    ├── fo-data-migration/             # DMF General Reference
    ├── fo-dm-configuration/           # Stage 1: Config Migration
    ├── fo-dm-master-data/             # Stage 2: Master Data Migration
    ├── fo-dm-opening-balances/        # Stage 3: Opening Balances
    └── fo-dm-validation/              # Pre-validation, OData FK discovery
```

## Skill Format

Each skill follows the [AgentSkills](https://agentskills.io) spec:

```
skill-name/
├── SKILL.md                    # YAML frontmatter + instructions
└── references/
    └── [topic].md              # Extracted procedural content from MS Learn
```

- **SKILL.md** — orchestration: decision logic, configuration sequence, validation checks, common errors
- **references/** — knowledge: field-by-field configuration details extracted from Microsoft Learn documentation

## Key Concepts

- **D365 ERP MCP Server** — Microsoft's Model Context Protocol server for headless F&O interaction (data tools, form tools, action tools)
- **Business Process Catalog (BPC)** — Microsoft's 15 end-to-end process framework, used to organize skills
- **Microsoft Config Template Sequence** — Official dependency-ordered levels (10-650) that determine configuration order
- **OData Pre-Validation** — Fast reference data validation via OData instead of slow DMF import cycles
- **Subledger-First Opening Balances** — Control accounts populated through module journals, not direct GL posting

## Tech Stack (Platform)

- **.NET 8 / ASP.NET Core** — Gateway service
- **Microsoft Semantic Kernel** — Agent runtime
- **Bot Framework SDK** — Teams integration
- **Blazor Server** — Admin UI
- **D365 ERP MCP Server** — F&O environment connector
- **Azure Cosmos DB** — Session/config storage
- **Azure DevOps** — ALM pipelines for X++ deployment

## Stats

- 23 skills covering all core F&O modules
- 27 extracted reference files with procedural content from Microsoft Learn
- 61 total markdown files, ~460 KB
