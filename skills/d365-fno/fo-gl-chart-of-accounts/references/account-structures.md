# Account Structures Configuration

Source: Microsoft Learn — verified 2026-03-18
Navigation: **General ledger > Chart of accounts > Structures > Configure account structures**

## What Account Structures Do

Account structures define which combinations of main account + financial dimensions are valid when posting transactions. They control what segments users see during data entry and what the system validates during posting.

## Rules and Limits

- **Maximum 11 segments** per structure. If you think you need more, evaluate whether a dimension could be derived via a hierarchy or reporting tree instead of data entry.
- **Advanced rules** can add up to 16 segments total (11 base + 5 via advanced rules).
- **Main account is REQUIRED** in every structure. It doesn't have to be the first segment, but it identifies which structure applies during entry.
- Each main account value can exist in **ONLY ONE** structure assigned to the same ledger. No overlapping main account ranges between structures.
- Budgeting dimensions **MUST** be in the base account structure — budgeting does NOT use advanced rules.

## Creating an Account Structure

1. Go to **General ledger > Chart of accounts > Structures > Configure account structures**
2. Click **New**
3. Name the structure
4. Add **segments** (main account + dimensions)
5. For each segment, define **allowed values**:
   - `*` = all values allowed
   - `" "` = blank allowed
   - Specific values or ranges (e.g., `100000..399999`)
6. Use the **Allowed value details** section for complex criteria (begins with, is between, includes, etc.)

## Best Practice Example (from Microsoft)

**Scenario:** Company tracks BS at account+business unit level, P&L at account+BU+department+cost center level. When posting to sales accounts, also require Customer.

**Structure 1: Balance Sheet**
| Main Account | Business Unit |
|---|---|
| 100000..399999 | *; " " |

**Structure 2: Profit and Loss**
| Main Account | Business Unit | Department | Cost Center |
|---|---|---|---|
| 400000..999999 | *; " " | *; " " | *; " " |

**Advanced Rule on Structure 2:**
- Criteria: Where Main Account is between 400000 and 499999
- Then add: Customer dimension (cannot be left blank)

**Result:**
- Posting to account 150000 (BS): system asks for Business Unit only
- Posting to account 600000 (P&L): system asks for BU + Dept + CC
- Posting to account 410000 (Sales revenue): system asks for BU + Dept + CC + Customer

## Activation

Account structures must be **ACTIVATED** before they can be used:
- New structures start in **Draft** status
- After configuration, click **Activate**
- Activation can take several minutes
- During activation, the system synchronizes all unposted transactions against the new structure
- **Wait for completion** before saving or navigating away
- Status changes from Draft → Active

## Modifying Existing Structures

- Can modify an active structure — it goes into a "modified" state
- Must re-activate after changes
- Re-activation triggers re-synchronization of unposted transactions
- **Risk:** Changing a structure that already has transactions can cause posting failures if existing combinations become invalid

## Assigning to Ledger

1. Go to **General ledger > Ledger setup > Ledger**
2. In the **Account structures** section, click **Add**
3. Select the activated structure
4. Click **Select**
5. Save — triggers synchronization (wait for completion)

## Troubleshooting

- **"Account structure validation failed"** during posting → the account+dimension combination doesn't match any active structure assigned to this ledger. Check: Is the structure activated? Does it include the main account range? Are the dimension values in the allowed values?
- **Overlapping structures error** → two structures assigned to the same ledger cover the same main account range. Remove one or adjust the ranges.
- **"Cannot add structure"** → wait for any pending synchronization to complete first.
- **Budget won't accept dimension** → dimension is in an advanced rule, not the base structure. Budgeting requires base structure dimensions.
