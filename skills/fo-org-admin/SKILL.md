---
name: fo-org-admin
description: >
  Configure D365 F&O organization structure: legal entities, operating units,
  organizational hierarchies, number sequences, global address book, and system
  parameters. Use when: setting up a new implementation, creating legal entities,
  configuring organization hierarchy, number sequences, or system-wide parameters.
  This is ALWAYS the first module configured (Level 10-15 in Microsoft's config
  template sequence). MUST be completed before any other module configuration.
  NOT for: GL chart of accounts (use fo-gl-chart-of-accounts), module-specific
  parameters (use the relevant module skill).
metadata:
  platform:
    category: "fo-foundation"
    riskLevel: "high"
    configSequence: 1
    configLevel: "10-15"
    requires:
      products: ["F&O"]
    bpcProcess: "99 - Administer to Operate"
    msLearnPath: "https://learn.microsoft.com/en-us/training/paths/configure-use-general-ledger-dyn365-finance/"
    sbdPhase: "Initiate"
---

# Organization Administration Configuration

## Prerequisites
- D365 F&O environment provisioned via PPAC (not LCS — LCS project creation blocked for new customers since Feb 2026)
- Environment must be Tier 2+ or UDE (Unified Developer Environment) for MCP server access
- System Administrator security role

## Configuration Sequence (MUST follow this order)

### Step 1: System Parameters (Level 10)
- **Menu:** System administration > Setup > System parameters
- **MCP approach:** Form tool (`form_open_menu_item` → `form_set_control_values`)
- **Key settings:** Help system, batch processing defaults, session configuration
- **Data entity:** N/A (parameter table — use form tools)

### Step 2: Global Address Book Setup (Level 15)
- **Menu:** Organization administration > Global address book > Global address book parameters
- **MCP approach:** Form tool for parameters, Data tool for reference data
- **Key entities:**
  - `AddressFormatHeaders` / `AddressFormatLines` — address formats
  - `AddressCountryRegions` — countries/regions
  - `AddressStates` — states/provinces
  - `AddressCities` — cities
  - `AddressZipCodes` — postal codes
  - `ContactTypes` — contact information types
  - `AddressPurposes` — address purposes (business, delivery, invoice, etc.)

**Dependency order for GAB:**
1. Address formats
2. Countries/regions
3. States/provinces (depends on countries)
4. Cities (depends on states)
5. Postal codes (depends on cities)
6. Contact types
7. Address purposes

### Step 3: Legal Entities
- **Menu:** Organization administration > Organizations > Legal entities
- **MCP approach:** Data tool preferred (`data_create_entities` using `LegalEntities`)
- **Data entity:** `LegalEntities`
- **Critical fields:** Name, short name, country/region, address, tax registration numbers
- **Note:** Each legal entity needs a primary address — address data from Step 2 must exist first

### Step 4: Operating Units
- **Menu:** Organization administration > Organizations > Operating units
- **MCP approach:** Data tool (`OperatingUnits`)
- **Types:** Department, cost center, value stream, business unit

### Step 5: Organization Hierarchies
- **Menu:** Organization administration > Organizations > Organization hierarchies
- **MCP approach:** Form tool (tree structure requires UI navigation)
- **Note:** Hierarchy purposes determine which D365 features use the hierarchy (centralized purchasing, budget control, etc.)

### Step 6: Number Sequences (Shared)
- **Menu:** Organization administration > Number sequences > Number sequences
- **MCP approach:** Data tool for bulk creation (`NumberSequences`), Form tool for scope assignment
- **Data entity:** `NumberSequences`
- **Critical:** Number sequences are referenced by every module. Must be set up before module-specific parameters.
- **Common gotcha:** If "Number sequence not set" error occurs during data import, this step was missed

### Step 7: Date Effectivity & Fiscal Calendar Foundation
- **Menu:** General ledger > Calendars > Fiscal calendars
- **Note:** Although this is technically a GL entity, fiscal calendars must exist before ledger setup. Some implementations configure this as part of Org Admin.

## Validation Checks (Agent must verify after configuration)
- [ ] At least one legal entity exists and is active
- [ ] Legal entity has a primary address with valid country/region
- [ ] Address hierarchy is complete (format → country → state → city)
- [ ] Number sequence scopes are defined for all required modules
- [ ] Organization hierarchy has at least one purpose assigned
- [ ] Operating units exist for each required type (if applicable)
- [ ] System parameters are set (not using defaults blindly)

## MCP Tool Priority
- **Data tools first** for: legal entities, operating units, address data, number sequences
- **Form tools** for: system parameters, organization hierarchy tree, number sequence scope assignment

## Common Errors
- "Number sequence not set" → Step 6 incomplete
- "Country/region not found" → Step 2 address data missing
- "Legal entity not found" → Step 3 must complete before any other module
- Hierarchy purpose conflicts → two hierarchies assigned same purpose

## Related Skills
- After this skill: `fo-gl-chart-of-accounts` (Level 20-25)
- This skill is a prerequisite for ALL other F&O configuration skills
