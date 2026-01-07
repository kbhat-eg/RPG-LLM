# Comprehensive Documentation: Customer Discount Matrix Import/Update

## Overview

This program processes customer discount matrix records, primarily importing data from a flat file (`xlfrab`) and updating the main discount matrix file (`frablu`). It supports both selective and complete replacement of discount categories, and handles a wide range of business rules regarding customer, product, and discount attributes.

The code is used in a business context where customer-specific, product-specific, and category-specific discounts must be loaded, updated, or deleted in bulk, typically as part of a batch update process.

---

## File Descriptions

- **xlfrab**:  
  Flat file, 512 bytes per record, contains discount matrix data to be imported. Opened as user-opened (`usropn`), read sequentially.

- **frablu** (renamed as `frablur`):  
  Main discount matrix file, keyed, used for updating/writing discount records.

- **frabld** (renamed as `frabldr`):  
  Used for deleting discount records by category.

- **skonlk** (commented out):  
  Not active in this version, but previously used for supplier number lookups.

---

## Data Flow

1. **Initialization**:  
   The program opens and reads the input file (`xlfrab`).

2. **Conditional Deletion**:  
   - If `l_rabkat = 'J'`, all relevant discount categories from the matrix are deleted before import.
   - If `l_alle = 'J'`, all discount categories are deleted.

3. **Data Import**:  
   For each record in `xlfrab`:
   - Parse and normalize each field (customer, product, category, discounts, dates, etc.).
   - Handle special characters and encoding issues (notably Norwegian letters).
   - Convert and format fields as required (e.g., padding, zero-filling, trimming).
   - Update or insert the corresponding record in `frablu`.

---

## Key Business/Domain Logic

### 1. **Field Parsing and Normalization**

Each record in `xlfrab` is a semicolon-separated string of fields. The code reads and parses each field in order, using `%scan` and `%subst` to extract and normalize values. This includes:

- Left-padding with zeros for numeric fields to ensure correct keying.
- Trimming spaces from values.
- Handling empty fields by setting them to a default value (often `0`).
- Special handling for Norwegian letters (e.g., `�, �, �`) via `%xlate` to ensure consistent character encoding.

### 2. **Discount Category Deletion**

- **Selective Deletion** (`l_rabkat = 'J'`):  
  For each unique discount category in the input file, all corresponding entries in `frablu` are deleted before new records are written.  
  This ensures that partial updates do not cause duplicate or conflicting discount rules.

- **Complete Deletion** (`l_alle = 'J'`):  
  All entries in `frablu` are deleted, typically used for a full matrix replacement.

### 3. **Data Validation and Formatting**

- **Numeric Field Padding**:  
  For fields like customer number, project number, group codes, etc., the code ensures the value is left-padded with zeros to the required length.

- **Date Handling**:  
  Dates may be input in `dd.mm.yyyy` format and are converted to `yyyy-mm-dd` for storage. If the date is invalid or not in the expected format, it is set to blank or default.

- **Negative Values**:  
  For fields like coverage degree (`dekningsgrad`), negative values are detected and handled by inverting the value as required.

### 4. **Record Update/Insert**

- The program attempts to `chain` (lookup) an existing record in `frablu` using the composite key.
- If found, the record is updated; otherwise, a new record is written.
- User and timestamp fields are set for auditing.

---

## Notable Design Decisions and Conventions

- **Manual Field Extraction**:  
  Instead of using a data structure or parsing utility, the code manually extracts each field using `%scan` and `%subst`. This is likely due to the variable/legacy nature of the input file or for maximum control over field normalization.

- **Batch-Oriented Processing**:  
  The program is designed for batch execution, processing one file at a time, with options for full or partial replacement.

- **Field Normalization Loops**:  
  For each key field, a loop is used to left-pad with zeros until the field reaches the required length. This normalization is essential for correct key formation and matching.

- **Character Encoding Handling**:  
  The code explicitly replaces certain hex values with correct Norwegian letters. This is a workaround for inconsistencies in upstream data encoding.

- **Date Conversion Subroutine**:  
  The `reddato` subroutine standardizes date input to the required format, ensuring robustness for various input scenarios.

- **Key Lists for File Operations**:  
  Key lists (`klist`) are defined for both main and delete operations, ensuring consistent and maintainable key usage throughout the code.

---

## Relationships with Other Modules

- **skonlk**:  
  The code contains commented-out logic for interacting with the `skonlk` file, which appears to be used for supplier number lookups or updates. This functionality is currently not active, but may be relevant in other contexts or future updates.

---

## Maintenance Notes

- **Field Order and Lengths**:  
  The input file format and field order must be strictly adhered to. Any changes in the upstream data source must be reflected in the parsing logic.

- **Error Handling**:  
  The code is robust in field normalization and date conversion, but does not have explicit error logging. Consider adding error handling/logging for production use.

- **Performance**:  
  Deleting and rewriting large portions of the discount matrix can be resource-intensive. Ensure this program is scheduled during off-peak hours if possible.

---

## Summary Table: Main Fields Processed

| Field Name      | Description                  | Normalization/Handling                  |
|-----------------|-----------------------------|-----------------------------------------|
| d_firm          | Company number              | Left-padded to 3 digits                 |
| d_prgr          | Price group                 | 2 chars                                 |
| d_rkat          | Discount category           | Left-padded to 3 digits                 |
| d_kkat          | Customer category           | Left-padded to 4 digits                 |
| d_kund          | Customer number             | Left-padded to 6 digits                 |
| d_kpro          | Customer project            | Left-padded to 6 digits                 |
| d_ogrp/hgrp/ugrp| Product groups              | Left-padded to 2/2/3 digits             |
| d_ldor          | Supplier number             | Left-padded to 6 digits                 |
| d_modn          | Module number               | Left-padded to 8 digits                 |
| d_vare          | Item number                 | As is                                   |
| d_enhe          | Unit                        | As is                                   |
| d_lety          | Supplier type               | Default to '0' if blank                 |
| d_nota          | Note                        | As is                                   |
| d_pslm          | Markup                      | Left-padded to 10 digits                |
| d_rab1/2/n/b/t  | Discount rates (various)    | Left-padded to 10 digits, decimals      |
| d_ra1d/2d/nd/bd/td | Discount dates           | Left-padded to 10 digits, decimals      |
| d_ra1f/2f/nf/bf/tf | Discount from dates      | Date conversion                         |
| d_ra1t/2t/nt/bt/tt | Discount to dates        | Date conversion                         |
| d_dekn          | Coverage degree             | Handles negative, left-padded           |
| d_dekd          | Coverage degree date        | Handles negative, left-padded           |
| d_dekf/dekt     | Coverage from/to dates      | Date conversion                         |

---

## Subroutines

### `reddato`
- Converts `dd.mm.yyyy` to `yyyy-mm-dd` if needed.
- If already in correct format, does nothing.
- If unrecognized format, blanks the date.

### `*inzsr`
- Initializes key lists for file operations.

---

## Conclusion

This program is a core part of the customer discount matrix management process. It is highly field-order dependent and includes significant normalization logic for legacy data. The code is robust for batch updates and supports both selective and complete matrix replacement.

**When onboarding new developers:**
- Emphasize the importance of field order and normalization.
- Explain the business need for both selective and full deletion before import.
- Highlight the encoding and date handling workarounds.
- Point out that any changes to the input format or business rules must be reflected in both the field extraction and normalization logic.

For further questions, consult the business analysts responsible for pricing and discount policies, as well as the documentation for the `frablu` and related files.