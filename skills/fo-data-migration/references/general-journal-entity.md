# General Journal Entity — Import Details

Source: Microsoft Learn (entity-general-journal) — verified 2026-03-18
Data Entity: `LedgerJournalEntity`
Related Menu: **General ledger > Journal entries > General journals**

## When to Use

Use for ALL high-volume GL journal import scenarios, including opening balance migration. Using DMF is recommended for high volume. The entity supports set-based processing for best performance.

## Key Fields

| Field | Required | Description |
|---|---|---|
| **JOURNALBATCHNUMBER** | Yes | Primary key for the header. First segment of composite key. Can use a placeholder value replaced by number sequence during import. Exclude if all lines are for the same header. |
| **LINENUMBER** | Yes | Second segment of primary key. |
| **JOURNALNAME** | Yes | Must match a configured journal name. Same value for all rows with the same JOURNALBATCHNUMBER. |
| **DESCRIPTION** | No | Header-level description. Same for all rows in the batch. |
| **POSTINGLAYER** | No | Header value. Same for all rows in the batch. |
| **ACCOUNTTYPE** | No | Bank, Customer, Ledger, or Vendor. Default = Ledger. |
| **ACCOUNTDISPLAYVALUE** | Yes | When ACCOUNTTYPE=Ledger: requires Financial dimension configuration for integrating applications (ledger dimension format). When ACCOUNTTYPE is other: the primary key value (vendor account, customer account, etc.). |
| **ACCOUNTDEFAULTDIMENSIONDISPLAYVALUE** | No | For non-Ledger account types only. Requires default dimension format configured in Financial dimension configuration. Not applicable when ACCOUNTTYPE=Ledger. |
| **FINTAGDISPLAYVALUE** | No | Financial tag values. Requires financial tag separator and tag count configured. |
| **OFFSETACCOUNTTYPE** | No | Same options as ACCOUNTTYPE. Default = Ledger. |
| **OFFSETACCOUNTDISPLAYVALUE** | Conditional | Same rules as ACCOUNTDISPLAYVALUE for the offset side. |
| **TRANSDATE** | Yes | Transaction date. **Critical:** Same VOUCHER + different TRANSDATE = separate physical vouchers (usually unintended). |
| **VOUCHER** | Yes | Voucher number. Required when using set-based processing. Can be a placeholder replaced during posting. |
| **CURRENCYCODE** | Yes | ISO currency code. |
| **DEBIT / CREDIT** | Yes | Transaction amount. |
| **EXCHANGERATE** | No | If omitted, system uses rate based on TRANSDATE. |
| **REPORTINGCURRENCYEXCHRATE** | No | For reporting currency conversion. If omitted, system calculates from TRANSDATE. |

## Performance Configuration

### Set-Based Processing (default: ON)
- **Recommended:** Leave enabled for best performance
- **Requirement:** Voucher numbers MUST be provided in the import file when enabled
- **If disabled:** System auto-generates voucher numbers from the journal name's voucher series (slower but easier)
- **Do NOT use multiple threads** — the entity does not recommend disabling set-based or using multi-threading

### Sizing Guidelines
- **JOURNALBATCHNUMBER:** Target ~10,000 lines per batch number for optimal throughput
- **Physical vouchers:** Keep small — ~10 lines or fewer per voucher
- **High cardinality dimensions:** If many of the imported dimension values are NEW (not previously used), import takes longer due to dimension creation overhead. Pre-import dimension values if possible.

## Financial Dimension Configuration Requirement

**This must be configured BEFORE any journal import involving dimensions.**

Navigation: **General ledger > Chart of accounts > Dimensions > Financial dimension configuration for integrating applications**

### For ACCOUNTTYPE = Ledger (ACCOUNTDISPLAYVALUE)
- Requires active **Ledger dimension format**
- Format defines: MainAccount - Dim1 - Dim2 - Dim3...
- Example: if format = MainAccount-Department-CostCenter, then value = "110110-002-04"
- Extra segments beyond the format are DROPPED (not errored)
- Missing segments are treated as blank

### For ACCOUNTTYPE = Customer/Vendor/Bank (ACCOUNTDEFAULTDIMENSIONDISPLAYVALUE)
- Requires active **Default dimension format**
- Format defines: Dim1 - Dim2 - Dim3... (NO main account)
- Example: if format = Department-CostCenter, then value = "001-02"

### Rules
- Only ONE format per type can be active
- Format is GLOBAL — same across all legal entities
- If not configured → import fails with cryptic error messages
- Format must include ALL dimensions used across ALL legal entities (unused = blank in file)

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| "Voucher already used" | Duplicate voucher in file | Ensure unique vouchers |
| "Journal name not found" | JOURNALNAME doesn't match configured names | Create journal name first |
| "Account structure validation failed" | Account+dimension combo not valid | Check account structures |
| "Financial dimension format not configured" | No ledger dimension format active | Set up Financial dimension config |
| Lines in wrong period | TRANSDATE outside expected fiscal period | Verify dates |
| Unexpected extra vouchers | Same VOUCHER but different TRANSDATE on lines | Make TRANSDATE consistent within a voucher |
