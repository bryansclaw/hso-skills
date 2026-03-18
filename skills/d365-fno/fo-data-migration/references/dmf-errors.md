# DMF Error Catalog

Source: Microsoft Learn (dm-error-descriptions) — verified 2026-03-18

## Named Errors

### 1. Update Conflict — Multiple Processes
**Error:** "Import to target failed due to an update conflict as more than one process is trying to update the same record"
**Cause:** Recurring import files sent at high frequency + parallel processing + same record appears in multiple files
**Fix:** Deduplicate source files (each record in only one file) OR enable sequential processing

### 2. Fields Not Mapped
**Error:** "There are fields which aren't mapped to Entity"
**Cause:** Template exported with "First row header = No" in fixed-width format — exported file lacks column names
**Fix:** Re-export template with First row header = Yes. Ensure all columns have proper headers.

### 3. Package Download Error
**Error:** "Error downloading data package for job - Record for ID {GUID} not found"
**Cause:** Dev environment shares database with another environment (e.g., UAT). Export job appears in both but file only exists in source environment's blob storage.
**Fix:** Download from the environment where the export was executed.

### 4. XML Format Error
**Error:** "XML isn't in correct format — Exception from HRESULT: 0xC0010009"
**Cause:** Data project has mappings for columns that don't exist in the file (columns removed from file but mapping retained)
**Fix:** Fix the mapping in the data project OR add missing columns back to file.

### 6. Zero Input Columns
**Error:** "The number of input columns for OLE DB Destination can't be zero"
**Cause:** Input file is empty — no column information
**Fix:** Verify file contains records AND column headers.

### 7. Invalid Source Map
**Error:** "Source map is invalid for data project {name}"
**Cause:** Package generates invalid mapping
**Fix:** Upload package directly to AOS via Data Management to debug mapping.

### 8. Entity Mapping After Refresh
**Error:** "The mapping is incorrect for entity" / "Data entities missing after entity refresh"
**Cause:** Duplicate label usage — two entities share the same label. First entity keeps the label name, second gets renamed.
**Fix:** Ensure each entity has a unique label. Check role-based privileges if entity is missing.

### 9. Query Validation Failed
**Error:** "File upload fails with Query validation failed — External table isn't in the expected format"
**Cause:** Excel formatting issues, or Excel sheet labeled "Confidential" (permissions exception)
**Fix:** Fix Excel formatting. Remove "Confidential" label. Or use CSV instead of Excel.

### 10. Number Sequence Not Set
**Error:** "Import fails with Number sequence isn't set"
**Cause:** Autogenerate number feature enabled during import but no number sequence reference configured
**Fix:** Configure the number sequence reference in module parameters BEFORE import.

### 11. Excel 0xC02020E8
**Error:** "Excel Import failed with error code 0xC02020E8"
**Cause:** Excel sheet contains unmapped fields
**Fix:** Remove unmapped fields from the Excel sheet.

### 12. Incorrect Data in Column
**Error:** "The column in entity has incorrect data"
**Cause:** Null, empty, or duplicate data in mandatory columns
**Fix:** Check all rows for mandatory field completeness. Fix specific rows.

### 13. Schema Mismatch (BYOD)
**Error:** "Schema mismatch in AxDb table and BYOD table"
**Cause:** Entity schema updated but AxDb or BYOD not refreshed
**Fix:** Refresh entity list (Data Management > Framework Parameters > Entity Settings > Refresh Entity List). Then republish entity to BYOD.

### 14. Duplicate Imports via REST API
**Error:** Duplicate data imported via package REST API
**Cause:** GetAzureWriteUrl() called without unique file name — same blob ID reused
**Fix:** Include a GUID in the file name when calling GetAzureWriteUrl().

### 15. Withholding Tax Registration Error
**Error:** "The mapping is incorrect for entity Withholding tax registration"
**Cause:** TaxWithholdingTaxRegistrationNumberEntity is set for India (IN) country codes only
**Fix:** Create an India-based legal entity, OR temporarily add India address to the legal entity for import/export.

## Error Codes

| Code | Cause | Resolution |
|---|---|---|
| **DMF021** | Enum value mappings missing — can't determine possible values | Remove affected enum column from mapping |
| **DMF023** | Database connection failed (BYOD unreachable or patching) | Verify BYOD connection string, retry later |
| **DMF030** | OLEDB source properties error — incorrect field mapping | Remove all fields except keys, re-add incrementally to find problem field. Check for duplicate entity labels. |
| **DMF040** | Excel query failed — can't retrieve column names or worksheet | Change file type from Excel to CSV or another format |
| **DMF042** | Data management service unreachable (patching in progress) | Transient — retry. If persistent, contact Support. |
| **DMF043** | Package execution failed — unknown pattern | Review execution details for specific error message |
| **DMF045** | SSIS package execution error | Review execution details. Contact Support if insufficient info. |
| **DMF1830** | XML not in correct format | Verify XML file is not empty. For composite entities, include ALL child entity elements. |
| **DMF1855** | File type doesn't match project | Use the file type set in the project, or change the project's file type |
| **DMF1957** | Unicode data can't export as ASCII | Change export file type to one supporting Unicode (XML or *-Unicode) |
| **DMF1958** | Invalid enum value in import file | Check enum data in file against valid enum values in D365 |
