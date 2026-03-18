# Financial Reporting Setup

Source: Microsoft Learn (financial-reporting-getting-started) — verified 2026-03-18

## Prerequisites for Financial Reporting

Before generating reports for a legal entity, must have:
1. **Fiscal calendar** configured
2. **Ledger** configured (CoA, currencies, calendar assigned)
3. **Chart of accounts** with main accounts
4. **Currency** set
5. At least **one transaction posted** to at least one account
6. MainAccount listed in the **Selected** column on Financial reporting setup page

## Installation

Financial reporting is now an **add-in** (not built into the base product):

### For LCS-managed environments:
1. In LCS, confirm Power Platform integration is configured
2. Select "Install a new add-in"
3. Search for "Financial reporting"
4. Agree to terms and install

### For PPAC-managed environments (UDE):
- Financial reporting add-in installation through PPAC is not yet self-service
- Contact Microsoft support for PPAC-based environment setup

**⚠️ Warning:** Uninstalling the add-in permanently deletes all reports, designs, and configurations. Recovery is NOT supported.

## Accessing Financial Reporting

Available in multiple locations:
- **General Ledger > Inquiries and reports**
- **Budgeting > Inquiries and reports > Basic budgeting**
- **Budgeting > Inquiries and reports > Budget planning**
- **Budgeting > Inquiries and reports > Budget control**
- **Consolidations**

## Financial Reporting Setup Page

Navigation: **General ledger > Ledger setup > Financial reporting setup**

### Dimensions Tab
- Set the ORDER of financial dimensions for reporting
- This order determines how dimensions appear in report rows/columns
- **Changes require a Data Mart reset to take effect**

### Attributes Tab
- Optionally enable **Vendor** and/or **Customer** as reporting attributes
- Useful for filtering reports by vendor/customer
- Only valuable if you don't enter multiple vendors/customers in a single voucher
- **Warning:** Enabling adds processing time to the data integration

## Security Roles

| Role | Can Do |
|---|---|
| **Security administrator** | Maintain financial reporting security |
| **Accounting Manager / Supervisor / Controller / Budget Manager** | Maintain (design) financial reports |
| **CEO / CFO / Accountant** | Generate financial reports |
| **sysadmin** | Added to all financial reporting roles automatically |

After adding/changing a user's role, access is available within a few minutes.

## Default Financial Reports

D365 provides pre-built report definitions:
- **Balance Sheet** — current period and YTD
- **Income Statement** — current period, YTD, prior year
- **Trial Balance** — detail and summary versions
- **Cash Flow Statement** — direct and indirect method
- **Budget vs Actual** — with variance calculations

These use the default row/column definitions and can be customized.

## Report Building Blocks

### Row Definitions
Define WHAT rows appear:
- **FD (Financial Dimensions)** — links to GL accounts/dimensions
- **CAL (Calculation)** — formulas between rows
- **DESC (Description)** — text rows (headers, subtitles)
- **TOT (Total)** — sum of row ranges

### Column Definitions
Define WHAT columns appear:
- **FD (Financial Dimensions)** — actual GL data for a period
- **CALC (Calculation)** — variance, percentage between columns
- **DESC (Description)** — column headers
- **WKS (Worksheet)** — external Excel data

### Reporting Tree Definitions (Optional)
- Define organizational rollup hierarchy
- Each node = reporting unit (legal entity + dimension filter)
- Enables drill-down from consolidated to departmental

### Report Definitions
- Combine: Row definition + Column definition + (optional) Reporting tree
- Settings: Base period, fiscal year, company scope, currency
- Output: Screen, Excel, PDF
- Can be scheduled as batch jobs

## Report Expiration
- **Financial report retention policies** feature
- New reports auto-expire after 90 days
- Existing reports get 90-day expiration from feature install date
- Can be changed between 1-365 days
- Expired reports are deleted automatically
