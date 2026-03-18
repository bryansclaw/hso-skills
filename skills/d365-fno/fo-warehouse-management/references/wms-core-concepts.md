# Warehouse Management Core Concepts

Source: Microsoft Learn + verified research — 2026-03-18

## Enabling WMS

**Per-warehouse toggle:** Warehouse management > Setup > Warehouse > [warehouse] > "Use warehouse management processes" = Yes

**⚠️ This is a ONE-WAY toggle.** Once enabled, it CANNOT be reversed for that warehouse. The warehouse permanently operates under WMS rules.

## How WMS Processing Works

### The Three Core Components

1. **Location Directives** — WHERE (rules for where to put or pick inventory)
2. **Work Templates** — WHAT (defines the pick+put work sequence)
3. **Wave Templates** — HOW (defines how work is released to the warehouse floor)

### Processing Flow

**Inbound:**
```
PO received → Location directive finds PUT location → 
Work template creates PUT work → Worker executes on mobile device
```

**Outbound:**
```
SO released to warehouse → Wave processes the demand → 
Work template creates PICK+PUT work → Location directive finds PICK location → 
Worker picks on mobile device → Stages → Ships
```

## Location Directives

Navigation: **Warehouse management > Setup > Location directives**

**The most complex WMS configuration.** Rules evaluated TOP-TO-BOTTOM; first match wins.

### Structure
- **Header:** Work order type (Purchase put, Sales pick, Transfer pick, etc.) + Site + Warehouse
- **Lines:** Sequence of rules, each with: From quantity, To quantity, Unit, Strategy
- **Actions:** Per line — define location selection strategy (Empty location, Consolidate, Fixed pick location, etc.)

### Common Directive Types
| Type | Purpose |
|---|---|
| Purchase orders - Put | Where to put received goods |
| Sales orders - Pick | Where to pick goods for sales |
| Transfer issue - Pick | Where to pick for transfers out |
| Transfer receipt - Put | Where to put goods transferred in |
| Production - Pick | Where to pick raw materials |
| Production - Put (report as finished) | Where to put finished goods |
| Counting | Where cycle counts happen |

## Work Templates

Navigation: **Warehouse management > Setup > Work > Work templates**

Define the SEQUENCE of operations:

### Example: Sales Pick Work Template
| Line | Work type | Mandatory |
|---|---|---|
| 1 | Pick | Yes |
| 2 | Put (to stage location) | Yes |

### Example: Purchase Put Work Template
| Line | Work type | Mandatory |
|---|---|---|
| 1 | Pick (from receiving dock) | Yes |
| 2 | Put (to storage location) | Yes |

Work templates and location directives work TOGETHER:
- Template says "do a pick" → directive says "from where"
- Template says "do a put" → directive says "to where"

## Wave Templates

Navigation: **Warehouse management > Setup > Waves > Wave templates**

### Wave Step Methods (configured per template)
Key methods in typical order:
1. **createWork** — generates work based on demand lines
2. **allocateWave** — allocates inventory to work
3. **print** — generates labels/documents
4. **containerize** — groups items into containers (if applicable)

### Processing Options
- **Manual waves:** User manually creates and processes waves
- **Auto-release:** Wave auto-creates and processes when SO is released to warehouse
- Configure on wave template: "Automate wave creation" and "Process wave at release to warehouse"

## Mobile Device Configuration

### Menu Items
Navigation: **Warehouse management > Setup > Mobile device > Mobile device menu items**

| Activity Type | Examples |
|---|---|
| Work (system-directed) | Sales picking, purchase put-away, transfer picking |
| Work (user-directed) | User selects which work to process |
| Indirect | Clock in/out, change warehouse, inventory inquiry |
| Inquiry | License plate inquiry, item inquiry |

### Menu Structure
Navigation: **Warehouse management > Setup > Mobile device > Mobile device menu**
- Organize menu items into the hierarchy workers see on their devices
- Can nest menus within menus
- Assign menus to specific workers or worker groups

## Key Decisions Before WMS Configuration

1. **Which warehouses need WMS?** (Remember: one-way toggle)
2. **Location structure:** How many aisles, racks, shelves, bins?
3. **License plate tracking:** Required for WMS warehouses
4. **Zone design:** Receiving zone, picking zone, staging zone, shipping zone
5. **Cycle count strategy:** How often? Which items?
6. **Wave processing:** Manual or automatic?
7. **Containerization:** Pre-pack or pack at ship?
