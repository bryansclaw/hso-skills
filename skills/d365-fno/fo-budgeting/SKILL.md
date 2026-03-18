---
name: fo-budgeting
description: >
  Configure D365 F&O Budgeting including budget models, budget dimensions,
  allocation terms, budget control configuration, budget register entries,
  budget plans, and budget workflows. Also covers budget vs actuals reporting
  via Financial Reporting and BPA.
  Use when: setting up budgeting, configuring budget control to prevent
  overspending, creating budget plans, or importing budget data.
  DEPENDS ON: fo-gl-chart-of-accounts (dimensions and accounts).
  Level 160 in Microsoft's config template sequence.
metadata:
  platform:
    category: "fo-budgeting"
    riskLevel: "medium"
    configSequence: 8
    configLevel: "160"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "13.30 - Record to Report > Manage budgets"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-use-budgeting-dyn365-finance/"
    mbExam: "MB-310 (10-15% weight: Manage budgeting)"
---

# Budgeting Configuration

## Configuration Sequence

### Step 1: Budget Models
- **Menu:** Budgeting > Setup > Basic budgeting > Budget models
- **MCP:** Form tool
- **Purpose:** Container for budget data. Can create multiple models (Original, Revised, Forecast)
- **Sub-models:** Models can have sub-models for rollup structures

### Step 2: Budget Dimensions
- **Menu:** Budgeting > Setup > Basic budgeting > Budget dimensions
- **MCP:** Form tool
- **Purpose:** Define which financial dimensions are used for budgeting
- **Typically:** Main account + Department + Cost center (subset of GL dimensions)
- **Note:** Budget dimension format must be configured in Financial dimension configuration for integrating applications before budget data import

### Step 3: Budget Allocation Terms
- **Menu:** Budgeting > Setup > Basic budgeting > Budget allocation terms
- **MCP:** Form tool
- **Purpose:** Define how annual budgets distribute across periods (equal, seasonal, custom)

### Step 4: Budget Transfer Rules
- **Menu:** Budgeting > Setup > Basic budgeting > Budget transfer rules
- **MCP:** Form tool
- **Purpose:** Control whether budget can be transferred between accounts/dimensions

### Step 5: Budget Register Entries (Data Import)
- **Menu:** Budgeting > Budget register entries
- **Data entity:** `BudgetAccountEntry`
- **MCP:** Data tool for import, Form tool for manual entry
- **Import notes:**
  - Requires budget register dimension format configured in Financial dimension config
  - Entity: BudgetAccountEntry — includes model, period, account+dimensions, amount
  - Budget type: Original budget, Transfer, Revision, Encumbrance, Pre-encumbrance

### Step 6: Budget Control Configuration
- **Menu:** Budgeting > Setup > Budget control > Budget control configuration
- **MCP:** Form tool (multi-step wizard)
- **Steps within wizard:**
  1. Define budget funds available (budget - actuals - encumbrances - pre-encumbrances)
  2. Select budget dimensions for control
  3. Define budget cycle time spans
  4. Set budget manager and over-budget permissions
  5. Assign to budget groups
  6. Define warning/hard-stop thresholds
  7. Activate budget control
- **Effect:** When active, transactions that exceed budget are warned or blocked

### Step 7: Budget Plans (Advanced)
- **Menu:** Budgeting > Budget plans
- **MCP:** Form tool
- **Features:**
  - Multi-stage approval workflows
  - Excel-based templates for department managers
  - AI-driven planning tools (forecast-based)
  - Integration with HR position forecasting
  - Integration with FA budget depreciation
  - Integration with SCM demand/supply forecasting → GL budget conversion

### Step 8: Budget Workflows
- **Menu:** Budgeting > Setup > Budgeting workflows
- **MCP:** Form tool
- **Purpose:** Approval workflows for budget register entries and budget plans

## Budget Data Migration

### Import via DMF
- Entity: `BudgetAccountEntry`
- File must include: Budget model, Date, Account+Dimensions (uses budget register dimension format), Amount, Budget type
- **Critical:** Budget register dimension format MUST be active in Financial dimension configuration for integrating applications

### Budget vs Actuals Reporting
- **Financial Reporting:** Create column definitions with Budget and Actual columns
- **BPA:** Budget comparison measures available through MCP Analytics server
- **Excel:** Budget register can be maintained via Excel add-in

## Validation Checks
- [ ] Budget models created (at least Original)
- [ ] Budget dimensions defined (subset of GL dimensions)
- [ ] Budget allocation terms configured
- [ ] Budget register dimension format active (Financial dimension config)
- [ ] Budget control configured and activated (if using budget control)
- [ ] Budget workflows configured (if approval required)
- [ ] Test: Enter budget → post transaction → verify budget check (warn or block if over)
- [ ] Test: Budget vs actuals report shows correct comparison

## Common Errors
- "Budget dimension format not configured" → Missing in Financial dimension config for integrating applications
- "Budget funds not available" → Budget control blocking transaction (check over-budget permissions)
- "Budget model not found" → Model name mismatch in import file
- Budget amounts not showing in reports → Budget register entry not posted (status = Draft)

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`
- **Related:** `fo-period-close` (budget vs actuals analysis), `fo-financial-reporting`
