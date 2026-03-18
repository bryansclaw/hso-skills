# Fixed Assets Setup and Lifecycle

Source: Microsoft Learn (fixed-assets) — verified 2026-03-18

## What Fixed Assets Does

Fixed assets are items of value owned by the organization: buildings, vehicles, land, equipment. D365 manages:
- Acquisition information entry
- Depreciation management
- Capitalization threshold determination
- Adjustments (write-ups and write-downs)
- Disposal

**Integration:** When using GL with FA, you can view current value of all assets. FA handling must comply with international accounting standards and country-specific legislation.

## FA Lifecycle (Business Process)

```
Acquisition → Depreciation → Adjustments → Disposal
     ↓              ↓              ↓            ↓
  GL: Asset     GL: Expense    GL: Adjust   GL: Gain/Loss
  account       + Accum Dep    entries      + Clear asset
```

## Setup Components

### FA Parameters
- **Menu:** Fixed assets > Setup > Fixed assets parameters
- Capitalization threshold (minimum value to be treated as FA)
- Whether acquisition can come from purchasing module
- Number sequences for asset IDs
- Posting layer settings

### FA Groups
- Classify assets by type (buildings, vehicles, equipment, furniture, IT)
- Each group carries default: depreciation method, service life, convention
- FA groups link to FA posting profiles for GL account determination

### Depreciation Profiles
Common methods available:
- **Straight line service life** — equal expense each period over useful life
- **Reducing balance** (200%, 150%, etc.) — accelerated, higher early-year expense
- **Straight line remaining life** — recalculates based on remaining life (used after adjustments)
- **Manual** — user-defined depreciation schedule
- **Factor** — applies a factor to the asset's base

### Value Models
- Define how an asset is valued — one asset can have MULTIPLE value models
- **Common pattern:** Book value model (for financial reporting) + Tax value model (for tax depreciation)
- Each value model links to a depreciation profile
- Determines which posting layer (Current, Operations, Tax) the depreciation posts to

### Depreciation Books (Optional)
- Separate depreciation tracking that does NOT post to GL
- Used for tax-only depreciation tracking when tax rules differ from book rules

### FA Posting Profiles
Key GL accounts per transaction type:
| Transaction | Debit Account | Credit Account |
|---|---|---|
| Acquisition | FA Asset account | Offset (clearing, AP, bank) |
| Depreciation | Depreciation expense | Accumulated depreciation |
| Disposal (sale) | Cash/AR + Accum. Dep reversal | FA Asset + Gain/Loss |
| Disposal (scrap) | Loss + Accum. Dep reversal | FA Asset |
| Write-up | FA Asset | Revaluation gain |
| Write-down | Revaluation loss | FA Asset |

Levels: All assets / FA group / Specific asset

## Key Processes

### Run Depreciation
- **Menu:** Fixed assets > Journal entries > Depreciation proposal
- Creates depreciation journal lines for all assets due for depreciation
- Can be run monthly, quarterly, or annually
- Must be reviewed and posted (not auto-posted)
- Can be batch-scheduled for month-end automation

### Dispose of an Asset
- **Menu:** Fixed assets > Journal entries > Fixed assets journal
- Transaction type: Disposal - sale or Disposal - scrap
- System calculates gain/loss based on NBV vs proceeds
- Clears asset from books and GL

### Year-End
- Depreciation must be run through year-end before GL year-end close
- Verify FA net book value = GL FA accounts before closing

## Data Entities
| Entity | Purpose |
|---|---|
| `FixedAssetGroups` | Import FA group definitions |
| `DepreciationProfiles` | Import depreciation methods |
| `FixedAssets` | Import asset master records |
| `FixedAssetBooks` | Import asset book assignments (value model + asset) |
