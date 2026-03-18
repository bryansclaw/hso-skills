# Organization Administration Foundation Setup

Source: Microsoft Learn + verified research — 2026-03-18

## Legal Entity Setup

Navigation: **Organization administration > Organizations > Legal entities**

A legal entity is an organization identified through registration with a legal authority. Legal entities can enter into contracts and must prepare financial statements.

### Required Fields
| Field | Description |
|---|---|
| Name | Full legal name of the entity |
| Short name | Abbreviated name |
| Country/Region | Country of registration (determines localization features, tax requirements) |
| Primary address | Registered business address |
| Tax registration | Tax ID / VAT number / EIN |
| DUNS number | Dun & Bradstreet number (optional) |

**Note:** Country/Region selection determines which localization features are available. For example, India enables TDS/TCS withholding tax; US enables 1099 reporting.

## Organization Hierarchies

Navigation: **Organization administration > Organizations > Organization hierarchies**

### Hierarchy Purposes
Each hierarchy has a purpose that determines which D365 features use it:

| Purpose | Used By |
|---|---|
| Centralized payments | AP/AR — pay from one entity on behalf of others |
| Budget planning | Budgeting — define budget ownership structure |
| Budget control | Budgeting — enforce budget limits per org level |
| Default security | Security — restrict data access by org unit |
| Retail | Commerce — store organization |
| Reporting | Financial reporting — rollup structure |

**Rule:** Each purpose can be assigned to only ONE hierarchy at a time.

## Number Sequences

Navigation: **Organization administration > Number sequences > Number sequences**

### Scope Levels
| Scope | Description |
|---|---|
| **Shared** | Same sequence across all legal entities |
| **Company** | Different sequence per legal entity |
| **Legal entity** | Alias for Company scope |
| **Operating unit** | Per department/cost center |

### Common Sequences Needed
Every module requires number sequences:
- GL: Journal vouchers, batch numbers
- AP: Vendor account numbers, invoice numbers
- AR: Customer account numbers, invoice numbers, collection letter numbers
- FA: Fixed asset numbers
- Inventory: Item numbers (if auto-generated)
- SO/PO: Order numbers
- Production: Production order numbers

**Critical error if missing:** "Number sequence not set" during data import or transaction creation.

### Configuration
1. Create number sequence (format, scope, segments)
2. Assign to module parameters (each module has a Number sequences tab in its parameters page)
3. Verify: status must be **Active** or **Continuous**

## Global Address Book

Navigation: **Organization administration > Global address book > Global address book parameters**

### Party Types
- Person (individual)
- Organization (company, operating unit)
- Legal entity (registered entity)
- Team

### Address Components (dependency chain)
```
Address Format → Country/Region → State/Province → City → Postal Code
```
Each level depends on the previous. Import in this order during data migration.

### Address Purposes
Define what each address is used for:
- Business (default)
- Delivery / Ship-to
- Invoice / Bill-to
- Remit-to (vendor payment address)
- Home (employee)
- Other (custom)

## PPAC Environment Awareness (2026)

**Starting February 2026:** New implementations must use Power Platform Admin Center (PPAC), not LCS.

Environment types:
- **UDE** (Unified Developer Environment) — replaces Tier-1 dev VMs
- **USE** (Unified Sandbox Environment) — replaces Tier-2 sandbox
- **Production** — via PPAC

All F&O environments are now Dataverse-native, enabling:
- Power Platform capabilities by default
- Dual-write integration
- Power Automate flows triggered by F&O events
- Model-driven apps accessing F&O data via virtual entities
