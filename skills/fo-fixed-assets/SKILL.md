---
name: fo-fixed-assets
description: >
  Configure D365 F&O Fixed Assets module including FA parameters, FA groups,
  depreciation profiles, value models, depreciation books, FA posting profiles,
  and fixed asset lifecycle (acquisition, depreciation, disposal). Also covers
  FA master data migration and opening balance migration.
  Use when: setting up fixed asset management, depreciation methods, asset
  acquisition workflows, or migrating existing asset registers.
  DEPENDS ON: fo-gl-chart-of-accounts (GL accounts for FA posting profiles).
  Level 150 in Microsoft's config template sequence.
metadata:
  platform:
    category: "fo-fixed-assets"
    riskLevel: "medium"
    configSequence: 7
    configLevel: "150"
    requires:
      products: ["F&O"]
      skills: ["fo-gl-chart-of-accounts"]
    bpcProcess: "08 - Acquire to Dispose"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-manage-fixed-assets-dyn365-finance/"
    mbExam: "MB-310 (10-15% weight: Manage fixed assets)"
---

# Fixed Assets Configuration

## Configuration Sequence

### Step 1: FA Parameters
- **Menu:** Fixed assets > Setup > Fixed assets parameters
- **MCP:** Form tool
- **Key settings:** Capitalization threshold, allow acquisition from purchasing, number sequences, posting layer

### Step 2: Fixed Asset Groups
- **Menu:** Fixed assets > Setup > Fixed asset groups
- **Data entity:** `FixedAssetGroups`
- **MCP:** Data tool
- **Purpose:** Classify assets (buildings, vehicles, equipment, furniture, IT, leasehold improvements)
- **Each group defines:** Default depreciation method, service life, convention

### Step 3: Depreciation Profiles
- **Menu:** Fixed assets > Setup > Depreciation profiles
- **Data entity:** `DepreciationProfiles`
- **MCP:** Data tool
- **Common methods:**
  - **Straight line service life** — equal depreciation over useful life
  - **Reducing balance** — accelerated, higher in early years
  - **Manual** — user-defined schedule
  - **Straight line remaining life** — recalculates based on remaining life
  - **200%/150% reducing balance** — US MACRS-like acceleration
- **Note:** Tax depreciation may use different profiles than book depreciation

### Step 4: Value Models
- **Menu:** Fixed assets > Setup > Value models
- **MCP:** Form tool
- **Purpose:** Define how an asset is valued (one asset can have multiple value models)
- **Common:** Book value model (financial reporting), Tax value model (tax depreciation)
- **Assigns:** Depreciation profile, service life defaults, posting layer

### Step 5: Depreciation Books (if using separate tax depreciation)
- **Menu:** Fixed assets > Setup > Depreciation books
- **MCP:** Form tool
- **Purpose:** Separate depreciation tracking that doesn't post to GL (tax-only books)

### Step 6: FA Posting Profiles
- **Menu:** Fixed assets > Setup > Fixed asset posting profiles
- **MCP:** Form tool
- **Key accounts per transaction type:**
  - Acquisition → FA asset account (debit) / offset account
  - Depreciation → Depreciation expense (debit) / Accumulated depreciation (credit)
  - Disposal sale → Cash/AR (debit) / Disposal (credit)
  - Disposal scrap → Loss (debit) / Accumulated depreciation reversal
  - Write-up / Write-down adjustments
- **Levels:** All / Fixed asset group / Specific asset

### Step 7: FA Calendar (Optional)
- **Menu:** Fixed assets > Setup > Fixed asset calendar
- **Purpose:** Define depreciation run frequency if different from fiscal calendar

## Fixed Asset Master Data Migration

| Seq | Entity | Data Entity | Dependencies |
|---|---|---|---|
| 1 | FA groups | `FixedAssetGroups` | None |
| 2 | Depreciation profiles | `DepreciationProfiles` | None |
| 3 | Value models | (Form tool setup) | Depreciation profiles |
| 4 | FA posting profiles | (Form tool setup) | FA groups, GL accounts |
| 5 | Fixed assets | `FixedAssets` | FA groups |
| 6 | Asset books | `FixedAssetBooks` | Fixed assets, value models |

## Opening FA Balances

**Critical rule:** FA opening balances must be posted through FA journals, NOT directly to GL.

### Acquisition Journal Method
1. Create FA acquisition journal (journal name type = Fixed assets)
2. Import acquisition lines: Asset ID, acquisition cost, acquisition date
3. Post → creates FA book entries AND posts to GL (FA asset account debit, offset credit)
4. This creates the asset NBV in the subledger AND the GL simultaneously

### Depreciation Catch-Up
- For assets acquired before go-live with existing depreciation:
- Option A: Import net book value only (acquisition cost − accumulated depreciation as the acquisition amount)
- Option B: Import at original cost, then post accumulated depreciation via depreciation journal
- Option B is more accurate but more complex

### Post-Migration Validation
- FA detail report (net book value per asset) must equal FA GL control accounts
- Accumulated depreciation total must match GL accumulated depreciation account
- Use Account Reconciliation feature (v10.0.44+)

## Validation Checks
- [ ] FA parameters configured (threshold, number sequences)
- [ ] FA groups created for each asset category
- [ ] Depreciation profiles configured (book + tax methods)
- [ ] Value models created and linked to depreciation profiles
- [ ] FA posting profiles have valid GL accounts for all transaction types
- [ ] FA number sequence active
- [ ] Test: Create asset → acquire → run depreciation → dispose → verify GL entries at each step

## Common Errors
- "Posting profile account not specified" → FA posting profile missing account for transaction type
- "Value model not found for asset" → Asset book not created/attached to the fixed asset
- "Depreciation already run for period" → Running depreciation twice (check depreciation calendar)
- NBV mismatch after migration → Acquisition amount + accumulated depreciation don't reconcile

## Related Skills
- **Prerequisite:** `fo-gl-chart-of-accounts`
- **Used by:** `fo-period-close` (depreciation runs during close), `fo-budgeting` (FA budget forecasting)
