# Customer Posting Profiles — Configuration Details

Source: Microsoft Learn (customer-posting-profiles) — verified 2026-03-18
Navigation: **Accounts receivable > Setup > Customer posting profiles**

## What Posting Profiles Do

Customer posting profiles assign GL accounts and document settings to all customers, a group of customers, or a single customer. Used when creating sales order invoices, free text invoices, project invoices, payment journals, collection letters, and interest notes.

## Default Posting Profile

Set on the **Ledger and Sales Tax** tab of **Accounts receivable parameters**. Automatically included on new document headers. Can be changed per document.

## Prepayment Posting Profile

Organizations accepting prepayments should configure a SECOND posting profile specifically for prepayments and link it in AR parameters as the default prepayment profile.

## Account Code Priority

| Account Code | Account/Group Number | Search Priority |
|---|---|---|
| **Table** | Specific customer account | 1 (most specific) |
| **Group** | Customer group assigned to the customer | 2 |
| **All** | Blank | 3 (fallback) |

## Key Fields

| Field | Description | GL Account Needed |
|---|---|---|
| **Summary account** | AR trade account — the main GL account for customer balances. This is the AR control account on the balance sheet. Account for the "Customer balance" posting type. | Yes — AR control |
| **Liquidity account for payments** | Ledger account for cash flow forecasts. Only appears when cash flow forecasts are enabled. | Yes (if forecasting) |
| **Sales tax prepayments** | Account for sales tax on advance payments. Specify in AR parameters which profile to use for prepayments. | Yes (if prepayments) |
| **Liabilities for discount account** | GL account for discount liabilities. | Optional |

## Posting Definitions Alternative

If **Use posting definitions** is enabled on GL parameters, posting definitions are used INSTEAD of posting profiles to control customer transaction posting. Posting definitions offer more complex GL account determination logic.

## Settlement and Table Restrictions

| Field | Purpose |
|---|---|
| **Settlement** | Automatic settlement of transactions. If off, must manually settle. |
| **Interest** | Whether interest is calculated for customers using this profile. |
| **Collection letter** | Whether collection letters are generated for customers using this profile. |

## Best Practices

- At minimum, create one "All" profile as the fallback
- Summary account must be a Balance Sheet > Asset account (AR control)
- If using prepayments: create a separate posting profile for prepayment transactions
- Validate all GL accounts exist before creating profile
- After migration: AR aging report total must equal the summary (AR control) account in GL
- Consider separate profiles for domestic vs export customers (different GL accounts for revenue, different tolerance for settlements)
