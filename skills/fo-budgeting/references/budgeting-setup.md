# Budgeting Setup and Budget Control

Source: Microsoft Learn (budgeting-overview, budget-control-overview-configuration) — verified 2026-03-18

## Budgeting Components

The resource planning cycle: **Planning → Budgeting → Forecasting**

### Budget Plans
- Tightly integrated with Excel
- Configure unlimited monetary and quantitative scenarios
- Define budgeting organizational hierarchy (top-down AND bottom-up)
- After approved, convert budget plan → budget register entry
- Support rolling forecasts (regular comparison of budget vs actuals)

### Budget Register Entries
- Record the expenditure budget
- Tools for maintaining budget and keeping amounts traceable via budget codes
- Budget types: Original budget, Revision, Transfer, Carry-forward, Pre-encumbrance, Encumbrance
- Can import from external systems or create manually

### Budget Control
- Configurable framework for controlling spending against budget
- **Hard control:** Prevents postings that exceed budget
- **Soft control:** Warns users but allows them to proceed
- Fully integrated — evaluates available budget for planned AND actual purchases

## Budget Control Configuration

Navigation: **Budgeting > Setup > Budget control > Budget control configuration**

### Step-by-Step Configuration Tabs

#### 1. Budget Cycle Time Span
- Define starting and ending periods for budgeting and budget control
- Often corresponds to fiscal calendar but can span fiscal years

#### 2. Define Parameters
- Select which financial dimensions to use for budget control (can be subset of all dimensions)
- Set default time interval: **Fiscal year**, **Fiscal year to date**, **Fiscal period**, or **Quarterly**
- Specify default budget manager
- Set threshold for user notifications (warning when approaching limit)

#### 3. Over Budget Permissions
- Specify user groups with permission to exceed budget
- Options: Prevent exceeding by ANY amount, or prevent exceeding past the threshold
- Different organizations need different strictness based on maturity

#### 4. Budget Funds Available Calculation
- Define the formula: what counts as "available funds"
- Typical formula: **Budget - Actuals - Encumbrances - Pre-encumbrances = Available**
- Can include/exclude: Draft documents, unposted documents
- **Warning:** Changing calculation mid-cycle doesn't retroactively affect previously checked documents

#### 5. Documents and Journals
- Select which source documents are subject to budget control checks:
  - Purchase requisitions (pre-encumbrances)
  - Purchase orders (encumbrances)
  - Vendor invoices
  - Travel requisitions
  - Journals
- Can evaluate per line or per document

#### 6. Budget Groups
- Group dimension combinations for budget checking
- Example: All marketing departments share one budget pool

#### 7. Activate Budget Control
- After all configuration → Activate
- Budget control is now enforced on selected documents

## Integrated Budget Sources

| Source Module | What It Provides |
|---|---|
| **HR / Position forecasting** | Headcount budget + compensation costs |
| **Fixed assets** | Planned depreciation expense |
| **Project Operations** | Project cost budgets |
| **Supply Chain** | Demand and supply forecasts → convertible to GL budgets |
| **Excel** | Budget plans can be maintained via Excel templates |

## Budget Register Entry Import

Data entity: `BudgetAccountEntry`
- Requires: Budget model, Date/period, Account + Dimensions, Amount, Budget type
- **Critical prerequisite:** Budget register dimension format must be active in Financial dimension configuration for integrating applications
- Entity handles: Original budgets, transfers, revisions

## Key Decisions
1. **Budget model structure:** Single model or multiple (Original + Revised + Forecast)?
2. **Control level:** Hard stop vs soft warning vs no control?
3. **Dimension scope:** Which dimensions are budget-controlled? (Must be in account structures, not advanced rules)
4. **Time granularity:** Annual budget allocated across periods, or per-period budgets?
5. **Allocation method:** Equal across periods? Seasonal? Custom?
