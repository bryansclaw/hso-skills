# DMF Performance Optimization

Source: Microsoft Learn (optimize-data-migration) — verified 2026-03-18

## Overview

Data migration performance depends on entity configuration, batch processing settings, data quality, and environment tier. Not all entities are optimized for migration — some are optimized for OData integration instead.

**Critical:** Do NOT compare Tier-1 performance to Tier-2+ environments. Results are not representative.

## Optimization Steps (in order)

### 1. Disable Change Tracking
- **Where:** Data management > Data entities > [select entity] > Action Pane > Change tracking tab > Disable Change Tracking
- **Why:** Change tracking adds overhead to every write operation. Not needed during migration.

### 2. Enable Set-Based Processing
- **Where:** Data management > Data entities > [entity] > check "Set-based processing" column
- **What it does:** Uses SQL bulk operations instead of row-by-row processing. 5-10x faster.
- **Limitation:** Not all entities support it. If you enable it and save, you may get: "Set operations not supported for [entity name]"
- **Example:** General Journal entity supports set-based. Customer definitions (CustCustomerBaseEntity) does NOT.
- **Requirements for set-based support:**
  - Data sources can't be read-only
  - ValidTimeStateEnabled property = No
  - All tables must have TableType = Regular
  - QueryType can't be Union
  - Main data source can't prevent cross-company saves (embedded sources can)

### 3. Create a Data Migration Batch Group
- **Where:** System administration > Setup > Batch group
- **Why:** During cutover, assign most/all compute nodes to a dedicated batch group so migration gets maximum resources.

### 4. Enable Priority-Based Batch Scheduling
- Available since Platform update 31
- Optimizes how batch jobs are distributed across nodes
- Enable if the batch platform shows contention

### 5. Configure Maximum Batch Threads
- **Where:** System administration > Setup > Server configuration > Maximum batch threads
- **Default:** 8 per server
- **Can increase to:** 12 or 16
- **Do NOT exceed 16** without significant performance testing
- **Total capacity:** threads per server × number of servers in batch group

### 6. Run Import in Batch Mode
- If not run as batch, a single thread handles everything — no parallelism
- Always use batch mode for migration-volume imports

### 7. Configure Entity Execution Parameters
- **Where:** Data management > Framework parameters > Entity settings > Configure entity execution parameters
- **Two key settings:**

#### Import Threshold Record Count
- Controls how many records per thread
- Lower = more parallel chunks (but more overhead)
- Higher = fewer chunks (less overhead per chunk)
- Test to find the sweet spot for each entity

#### Import Task Count
- Number of parallel threads for this entity
- Maximum = (batch threads per server) × (servers in batch group)
- Example: 8 threads × 4 servers = max task count of 32
- **Some entities don't support multi-threading.** Error: "Custom sequence is defined for [entity], more than one task is not supported." → set task count = 1

### 8. Clean Staging Tables
- **Where:** Data management > Job history cleanup tile
- **Prerequisite:** Enable "Execution history cleanup" feature in Feature Management
- Staging table bloat slows imports — clean regularly
- Schedule as a recurring batch job

### 9. Update SQL Statistics
- **For sandbox environments only** (production auto-maintains)
- Run `sp_updatestats` via direct SQL or LCS/PPAC
- Critical before large imports — gives SQL optimizer current data distribution info

### 10. Manage Validations
Per entity, you can disable:
- **Where:** Data management > Data entities > [entity] > Action Pane > Entity structure

| Setting | What it controls | Disable when... |
|---|---|---|
| **Run business validations** | Executes validateWrite() method + event handlers on target table | Data is mature, already validated externally |
| **Run business logic in insert/update** | Executes insert()/update() methods + event handlers | Data is clean, skip triggers for speed |
| **Call validate Field method** | Per-field validation | Specific fields known to be valid |

**Warning:** Disabling validations is DANGEROUS. Only for mature, pre-validated data in later migration iterations. Always run WITH validations on the first import to catch issues.

### 11. Clean Source Data
- Invalid or inconsistent data increases time spent on validation and error handling
- Fix data quality issues BEFORE import, not during
- Reduces unnecessary validation cycles and error log processing

## Iterative Testing Approach

Microsoft recommends an iterative process:
1. Start with 1,000 records → establish baseline
2. Increase to 10,000 → test optimization settings
3. Increase to 100,000 → validate at scale
4. Full volume → final performance test

### Metrics to Track Per Entity Per Test

| Metric | Track |
|---|---|
| Entity name | Which entity |
| Record count | How many records |
| Source format | Excel, CSV, XML |
| Change tracking | Disabled? |
| Set-based processing | Enabled? |
| Import threshold | Record count per thread |
| Import task count | Thread count |
| Business validations | On/off |
| Business logic in insert/update | On/off |
| Required performance | Target time for cutover window |
| Actual performance | Real measured time |

## If Entity Can't Be Optimized

If a standard entity can't meet performance requirements even after all optimizations:
- **Create a new optimized entity** (developer task)
- A developer can accelerate this by duplicating an existing entity and removing unnecessary fields/validations
- The new entity can be purpose-built for migration with set-based support
