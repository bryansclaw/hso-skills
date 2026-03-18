# Journal Configuration

Source: Microsoft Learn + verified research — 2026-03-18

## Journal Names Setup

Navigation: **General ledger > Journal setup > Journal names**

### Key Fields
| Field | Description |
|---|---|
| Name | Short code (e.g., "GenJnl", "VendPay", "CustPay") |
| Description | Display name |
| Journal type | Determines behavior: Daily, Allocation, Periodic, Vendor invoice register, Vendor payment, Customer payment, Vendor disbursement, Fixed assets, Budget |
| Voucher series | Number sequence for voucher numbers in this journal |
| Detail level | New voucher (separate voucher per line) or One voucher only (all lines share one voucher) |
| Approval workflow | If set, journal requires workflow approval before posting |
| Private for user group | Restricts who can use this journal name |

### Common Journal Types Needed
| Type | Purpose | Typical Name |
|---|---|---|
| Daily | Standard GL entries | GenJnl |
| Vendor payment | AP payment runs | VendPay |
| Customer payment | AR payment receipt | CustPay |
| Vendor invoice register | Quick AP invoice entry | VendReg |
| Fixed assets | FA transactions | FAJnl |
| Allocation | Cost allocations | AllocJnl |
| Periodic | Recurring entries | PerJnl |

## Voucher Series

Each journal name should have its own number sequence for vouchers:
- Navigate to the journal name record > Voucher series field
- Select or create a number sequence
- **Set-based processing note:** When importing via General Journal entity with set-based ON, voucher numbers MUST be in the import file. When OFF, system auto-generates from this series (slower).

## Journal Controls

Navigation: **General ledger > Journal setup > Journal control**

Restricts which account types and main account segments can be used in specific journal names:
- Example: "PayrollJnl" can only post to main accounts 60000-69999 (payroll expenses)
- Prevents users from posting to wrong accounts

## Posting Restriction Rules

Navigation: **General ledger > Journal setup > Posting restriction rules**

Restricts which users or user groups can POST (not just create) specific journal types:
- Example: Only Finance Managers can post journals over $100,000
- Combines with workflow for dual-control

## Intercompany Accounting

Navigation: **General ledger > Intercompany accounting > Intercompany accounting**

Setup for automatic due-to/due-from entries when posting across legal entities:
1. Define pairs: Company A ↔ Company B
2. Assign the due-to account (Company A's perspective)
3. Assign the due-from account (Company B's perspective)
4. When a journal in Company A debits an account in Company B, D365 automatically creates the offsetting intercompany journal
5. Both legal entities must have their ledger configured first
