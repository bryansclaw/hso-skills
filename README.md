# HSO Skills

Agent skill library for HSO's Microsoft Dynamics practice.

## Structure

```
├── skills/
│   ├── d365-fno/              # Dynamics 365 Finance & Operations
│   │   ├── fo-org-admin/
│   │   ├── fo-gl-chart-of-accounts/
│   │   ├── fo-accounts-payable/
│   │   ├── fo-dm-validation/
│   │   └── ... (23 skills)
│   │
│   ├── d365-ce/               # (future) Dynamics 365 Customer Engagement
│   ├── power-platform/        # (future) Power Platform
│   └── azure-devops/          # (future) DevOps & ALM
│
└── docs/
    └── d365-fno/              # Architecture & research docs
        ├── ARCHITECTURE.md
        ├── FNO-CONFIG-MIGRATION-DEEP-DIVE.md
        └── FNO-RESEARCH-ADDENDUM.md
```

## D365 F&O Skills (`skills/d365-fno/`)

23 skills covering all core F&O modules:

| Category | Skills |
|---|---|
| **Foundation** | fo-org-admin |
| **General Ledger** | fo-gl-chart-of-accounts, fo-gl-journals, fo-gl-posting-profiles |
| **Financial** | fo-accounts-payable, fo-accounts-receivable, fo-tax-configuration, fo-cash-bank-management |
| **Assets** | fo-fixed-assets, fo-budgeting |
| **Supply Chain** | fo-inventory-management, fo-procurement, fo-sales-marketing, fo-production-control, fo-warehouse-management |
| **Reporting** | fo-financial-reporting, fo-period-close |
| **Integration** | fo-integration-patterns |
| **Data Migration** | fo-data-migration, fo-dm-configuration, fo-dm-master-data, fo-dm-opening-balances, fo-dm-validation |

Each skill follows the [AgentSkills](https://agentskills.io) spec:
- **SKILL.md** — YAML frontmatter + instructions (decision logic, sequence, validation)
- **references/** — extracted procedural content from Microsoft Learn

## Skill Format

```yaml
---
name: fo-gl-chart-of-accounts
description: >
  Design and configure D365 F&O chart of accounts...
  Use when: ...
  NOT for: ...
metadata:
  platform:
    category: "fo-general-ledger"
    riskLevel: "high"
    configSequence: 2
    configLevel: "20-25"
---
```
