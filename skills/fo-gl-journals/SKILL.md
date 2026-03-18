---
name: fo-gl-journals
description: >
  Configure and manage D365 F&O general ledger journals including journal names,
  voucher series, journal controls, posting restriction rules, periodic journals,
  intercompany accounting, default descriptions, and journal approval workflows.
  Also covers journal import via DMF using the General Journal entity.
  Use when: setting up journal names, configuring approval workflows, creating
  journal entries, importing opening balances, or configuring intercompany.
  DEPENDS ON: fo-gl-chart-of-accounts (accounts, dimensions, ledger setup must exist).
metadata:
  platform:
    category: "fo-general-ledger"
    riskLevel: "medium"
    configSequence: 3
    configLevel: "25"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "13.40 - Record to Report > Record financial transactions"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-use-general-ledger-dyn365-finance/"
    mbExam: "MB-310 (within Implement financial management 40-45%)"
---

# General Ledger — Journal Configuration

## Configuration Sequence

### Step 1: Journal Names
- **Menu:** General ledger > Journal setup > Journal names
- **Data entity:** `LedgerJournalNames`
- **MCP:** Data tool
- **Key fields:** Name, description, journal type, voucher series, approval workflow
- **Common journal types:**
  - **Daily** — standard GL journal entries
  - **Periodic** — recurring accruals/deferrals
  - **Allocation** — cost allocation journals
  - **Intercompany** — cross-legal-entity entries
  - **Vendor invoice register** — AP quick-entry
  - **Vendor payment** — AP payment journals
  - **Customer payment** — AR payment journals

### Step 2: Voucher Series (Number Sequences)
- **Menu:** General ledger > Journal setup > Journal names > [journal] > Voucher series
- **MCP:** Form tool (linked from journal name)
- **Critical:** Each journal name should have a dedicated number sequence for vouchers
- **Set-based processing note:** When using set-based processing on General Journal entity import, voucher numbers MUST be provided in the file. If not using set-based processing, the system auto-generates from the voucher series.

### Step 3: Journal Controls
- **Menu:** General ledger > Journal setup > Journal control
- **MCP:** Form tool
- **Purpose:** Restrict which account types and segments can be used in specific journals
- **Example:** Limit "Payroll Journal" to only post to payroll-related main accounts

### Step 4: Posting Restriction Rules
- **Menu:** General ledger > Journal setup > Posting restriction rules
- **MCP:** Form tool
- **Purpose:** Restrict which users/groups can post specific journal types

### Step 5: Default Descriptions
- **Menu:** General ledger > Journal setup > Default descriptions
- **Data entity:** `DefaultDescriptions`
- **MCP:** Data tool
- **Purpose:** Auto-populate voucher descriptions based on transaction type
- **Tip:** Reduces manual data entry errors and improves audit trail

### Step 6: Periodic Journals
- **Menu:** General ledger > Periodic tasks > Periodic journals
- **MCP:** Form tool
- **Use cases:** Monthly rent accruals, recurring deferrals, depreciation entries
- **Note:** Can be scheduled as batch jobs via cron

### Step 7: Intercompany Accounting
- **Menu:** General ledger > Intercompany accounting > Intercompany accounting
- **MCP:** Form tool
- **Setup:** Define due-to/due-from account pairs between legal entities
- **Dependency:** Both legal entities must have ledger configured
- **Automatic:** When posting in Company A debits an account in Company B, D365 auto-creates the offsetting intercompany journal

### Step 8: Journal Approval Workflows
- **Menu:** General ledger > Journal setup > General ledger workflows
- **MCP:** Form tool (visual workflow designer)
- **Common patterns:**
  - Journal amount > threshold → requires manager approval
  - Specific account ranges → requires controller approval
  - All manual journals → require at least one approval

## General Journal Entity Import (LedgerJournalEntity)

### Entity Details
| Field | Notes |
|---|---|
| `JOURNALBATCHNUMBER` | Header key. Use ~10,000 lines per batch. Can be placeholder replaced by number sequence. |
| `LINENUMBER` | Required. Second part of primary key. |
| `JOURNALNAME` | Must match a configured journal name. Same value for all lines with same batch number. |
| `ACCOUNTTYPE` | Bank, Customer, Ledger, Vendor. Default = Ledger. |
| `ACCOUNTDISPLAYVALUE` | For Ledger: requires financial dimension config for integrating applications. For others: primary key value. |
| `ACCOUNTDEFAULTDIMENSIONDISPLAYVALUE` | For non-Ledger account types only. Requires default dimension format configured. |
| `OFFSETACCOUNTTYPE` | Same options as ACCOUNTTYPE. |
| `OFFSETACCOUNTDISPLAYVALUE` | Same rules as ACCOUNTDISPLAYVALUE. |
| `TRANSDATE` | Required. Same VOUCHER + different TRANSDATE = separate physical vouchers. |
| `VOUCHER` | Required. Provide values when using set-based processing. |
| `CURRENCYCODE` | Required. |
| `DEBIT` / `CREDIT` | Transaction amounts. |

### Performance Tips
- Set-based processing = enabled by default (fast but requires voucher numbers in file)
- ~10,000 lines per JOURNALBATCHNUMBER for optimal performance
- Keep physical vouchers small (~10 lines)
- High cardinality dimensions (many new values) slow imports
- Disable set-based processing if you want auto-generated vouchers (slower)
- Clean staging tables before large imports

## Validation Checks
- [ ] Journal names created for each required type
- [ ] Voucher series assigned and number sequences active
- [ ] Journal controls configured (if restricting account usage)
- [ ] Default descriptions set up for common transaction types
- [ ] Intercompany pairs configured (if multi-entity)
- [ ] Workflow configured for manual journal approval (if required)
- [ ] Financial dimension config for integrating applications is active (required for journal import)
- [ ] Test: Create journal → add lines → validate → post → verify GL entries

## Common Errors
- "Voucher already used" → Duplicate voucher number in import file
- "Journal name not found" → JOURNALNAME value doesn't match configured names
- "Account structure validation failed" → Account+dimension combination not valid in active structure
- "Financial dimension format not configured" → Missing ledger dimension format in integration config
- Lines posting to wrong period → TRANSDATE outside expected fiscal period

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`
- **Used by:** `fo-data-migration` (opening balance imports), `fo-period-close` (closing entries)
