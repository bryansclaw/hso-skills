---
name: fo-financial-reporting
description: >
  Configure D365 F&O Financial Reporting including report definitions, row
  definitions, column definitions, reporting tree definitions, and report
  generation. Also covers integration with Business Performance Analytics (BPA)
  and the MCP Analytics server for AI-driven financial analysis.
  Use when: creating financial statements (Balance sheet, Income statement,
  Cash flow), designing custom reports, or configuring financial reporting
  dimensions.
  DEPENDS ON: fo-gl-chart-of-accounts (chart of accounts must be configured),
  fo-period-close (for period-end reporting).
metadata:
  platform:
    category: "fo-financial-reporting"
    riskLevel: "low"
    configSequence: 12
    configLevel: "25"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "13.60 - Record to Report > Analyze financial performance"
    msLearnPath: "https://learn.microsoft.com/en-us/dynamics365/finance/general-ledger/financial-reporting-getting-started"
    mbExam: "MB-310 (within Implement financial management 40-45%)"
---

# Financial Reporting Configuration

## Two Reporting Systems

### 1. Financial Reporting (Report Designer)
- Native D365 financial report builder
- **Menu:** General ledger > Inquiries and reports > Financial reports
- Uses three building blocks: Row definitions, Column definitions, Report definitions
- Optional: Reporting tree definitions (for organizational rollups)
- Supports: Multi-company, multi-currency, budget vs actual, year-over-year

### 2. Business Performance Analytics (BPA)
- Pre-built Power BI reports on standardized data models
- Included in D365 Finance license (no extra cost)
- Refreshes 2x daily, rolling 4-year history
- Extensible via Microsoft Fabric
- AI-queryable via MCP Analytics server (natural language → DAX queries)

## Financial Reporting Setup

### Step 1: Financial Reporting Setup
- **Menu:** General ledger > Ledger setup > Financial reporting setup
- **MCP:** Form tool
- **Dimensions tab:** Set the ORDER of financial dimensions for reporting
- **Attributes tab:** Optionally enable Vendor/Customer as reporting attributes
- **Note:** Changes require a Data Mart reset to take effect

### Step 2: Row Definitions
- **Menu:** Financial reports > Row definitions (in Report Designer)
- **MCP:** Form tool (opens Report Designer)
- **Purpose:** Define WHAT rows appear on the report
- **Row types:**
  - **FD (Financial Dimensions)** — links to GL accounts/dimensions
  - **CAL (Calculation)** — formulas (sum, percentage, ratio)
  - **DESC (Description)** — text-only rows (headers, subtitles)
  - **TOT (Total)** — sum of a range of rows
- **Common reports:**
  - Balance Sheet: Assets section + Liabilities section + Equity section
  - Income Statement: Revenue rows - Expense rows = Net Income
  - Cash Flow: Operating + Investing + Financing activities

### Step 3: Column Definitions
- **Purpose:** Define WHAT columns appear (time periods, budget vs actual, currencies)
- **Column types:**
  - **FD (Financial Dimensions)** — actual GL data for a period
  - **CALC (Calculation)** — variance, percentage, ratio between columns
  - **DESC (Description)** — text column headers
  - **WKS (Worksheet)** — links to external Excel data
- **Common patterns:**
  - Current month + YTD + Prior year YTD
  - Actual + Budget + Variance + Variance %
  - Q1 + Q2 + Q3 + Q4 + Full Year

### Step 4: Reporting Tree Definitions (Optional)
- **Purpose:** Define organizational rollup hierarchy
- **Use cases:** Department rollup, business unit rollup, geographic rollup
- **Each node:** Maps to a reporting unit (legal entity + dimension filter)
- **Enables:** Drill-down from consolidated totals to individual departments

### Step 5: Report Definitions
- **Purpose:** Combine Row + Column + (optional) Tree into a complete report
- **Settings:** Base period, fiscal year, company scope, currency
- **Output:** On-screen, Excel, PDF
- **Scheduling:** Can be scheduled as batch jobs

### Step 6: Generate Reports
- **Menu:** General ledger > Inquiries and reports > Financial reports
- **MCP:** Form tool (select report → set parameters → generate)
- **Note:** First generation after changes may take longer (Data Mart refresh)

## BPA Integration

### BPA Reports (Pre-built)
| Report Category | Key Reports |
|---|---|
| Financial overview | Trial balance, balance sheet, P&L |
| Revenue analysis | Revenue by customer, product, region |
| Expense analysis | Expense by category, department, vendor |
| Cash flow | Cash position, forecast vs actual |
| Budget | Budget vs actuals, variance analysis |

### MCP Analytics Server
- Agents can query BPA data using natural language
- Server generates DAX queries against BPA dimensional models
- Row-level security enforced based on user role
- **Use case:** "What was our gross margin by product category last quarter?" → agent queries BPA → returns structured answer

## Validation Checks
- [ ] Financial reporting setup configured (dimension order)
- [ ] Data Mart refreshed after setup changes
- [ ] At least Balance Sheet and Income Statement reports defined
- [ ] Row definitions map to correct GL account ranges
- [ ] Column definitions include required periods
- [ ] Reporting tree defined (if multi-entity or departmental reporting needed)
- [ ] Reports generate correctly with test data
- [ ] BPA installed and configured (if using AI analytics)
- [ ] Test: Generate Balance Sheet → verify total assets = total liabilities + equity

## Common Errors
- Report shows blank/zero → Data Mart needs refresh after setup changes
- "Dimension not found" → Dimension order in reporting setup doesn't match GL dimensions
- Report doesn't match trial balance → Row definition account ranges have gaps or overlaps
- BPA data stale → BPA refreshes 2x/day; check refresh schedule
- Report shows wrong company → Report definition company filter incorrect

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`
- **Related:** `fo-period-close` (reports generated after close), `fo-budgeting` (budget vs actual columns)
- **Analytics:** BPA MCP Analytics server for AI-driven analysis
