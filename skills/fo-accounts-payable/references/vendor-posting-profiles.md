# Vendor Posting Profiles — Configuration Details

Source: Microsoft Learn (vendor-posting-profiles) — verified 2026-03-18
Navigation: **Accounts payable > Setup > Vendor posting profiles**

## What Posting Profiles Do

Vendor posting profiles control the posting of vendor transactions to the general ledger. They assign GL accounts and document settings to all vendors, a group of vendors, or a single vendor.

## Default Posting Profile

The default posting profile is set on the **Ledger and Sales Tax** FastTab of the **Accounts payable parameters** page. This default is automatically included on the header of new documents. You can change it per document if needed.

## Account Code Priority (How D365 Finds the Right Profile)

| Account Code | Account/Group Number | Search Priority |
|---|---|---|
| **Table** | Specific vendor account | 1 (most specific — checked first) |
| **Group** | Vendor group assigned to the vendor | 2 |
| **All** | Blank | 3 (fallback — used if no Table or Group match) |

**Simplest setup:** One posting profile with Account code = "All". All vendors use the same GL accounts.

**Multi-profile setup:** Create separate profiles for domestic vs foreign vendors, or trade vs service vendors.

## Key Fields

| Field | Description | GL Account Needed |
|---|---|---|
| **Summary account** | The AP control account — main GL account for vendor balances. When set, the "Do not allow manual entry" flag is automatically enabled on this main account. | Yes — this is the AP control account on Balance Sheet |
| **Settle account** | Liquidity ledger account for cash flow forecasts. Only available when cash flow forecasting is enabled. | Yes (if using cash flow) |
| **Sales tax prepayments** | Account for sales tax on payments received in advance. The specific profile for prepayments is set in AP parameters > Ledger and sales tax > Prepayment journal voucher field. | Yes (if using prepayments) |
| **Arrival** | Account for unapproved vendor invoices posted via the Invoice register journal. Basic info entered at receipt; when register is posted, transactions go here. When approved, debt transfers from arrival to summary (AP control) account. | Yes (if using invoice register) |
| **Offset account** | Offset for unapproved vendor invoices. Pairs with the Arrival account — represents vendor purchases not yet approved. | Yes (if using invoice register) |

## Table Restrictions Section

| Field | Description |
|---|---|
| **Settlement** | Enable automatic settlement of transactions with this profile. If cleared, must manually settle via "Settle open transactions" page. |
| **Cancel** | Allow cancellation of transactions with this profile. |
| **Close** | Select a different posting profile to change to when transactions are fully settled. |

## Posting Definitions Alternative

If the **Use posting definitions** option is selected on **General ledger parameters**, transaction posting definitions for vendor invoices are used INSTEAD of the summary account in posting profiles. Posting definitions provide more flexible GL account determination.

## Best Practices

- Always define at least an "All" level profile as a fallback
- The summary account should be a Balance Sheet > Liability account
- Validate GL accounts exist before creating profiles
- If using multiple profiles (Table/Group), test the priority resolution to ensure the right profile is selected
- After migration: verify AP subledger total = summary account balance in GL
