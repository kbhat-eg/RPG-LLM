# RK661BR – Documentation

## Overview

**Program:** RK661BR  
**System:** ASOKON  
**Description:** This program processes detail records (poster) from the `rkwbl3` file and consolidates (trekker sammen) those that belong to the same "generalnota" (general invoice/note) into a single record per year and collective (samf). All other records (not on a generalnota) are copied as-is.  
**Special Version:** Tailored for Mestergruppen.

**Business Context:**  
In the context of invoice and accounting processing, a "generalnota" represents a consolidated invoice or note that groups several postings. This program ensures that for each year and collective (samfunn/samf), only one summary posting is created in the output file. Records not associated with a generalnota are transferred unchanged.

## File Definitions

- **rkwbl3**: Input detail records (disk file, keyed, renamed to `rkwbl3r`, all fields prefixed with `Z`).
- **rkwdpf**: Output detail records (disk file, update, keyed).

## Key Data Structures and Variables

- **Local Working Variables:**  
  These are used to accumulate sums and track the current group for consolidation.
    - `w_firm`, `s_kund`, `s_bdat`, `s_raar`, `s_samf`, `s_rfda`, `s_rtda`: Used for grouping and key management.
    - `w_belø`, `w_rest`, `w_rent`, `w_rgrl`: Accumulators for amounts to be consolidated.

- **User Data Structure (UDS):**  
  Contains user/session-specific information, e.g., `l_user`, `l_firm`, `l_fnav`.

## Main Processing Logic

### Initialization

- The program initializes the key list for reading the input file (`rkwbl3`) using the company/firm (`w_firm`).

### Main Loop

1. **Read Input Records:**  
   The program reads through all records in `rkwbl3` for the current firm.

2. **Processing Each Record:**
    - **Records Not on Generalnota (`zrnsamf = *zero`):**
      - Call subroutine `felt` to copy all fields from the input record to the output record structure.
      - Write the record as-is to the output file (`rkwdpf`).
      - Proceed to the next input record.

    - **Records on Generalnota (`zrnsamf <> *zero`):**
      - If this record is for a new year (`zrnraar`) or new collective (`zrnsamf`) compared to the current group (`s_raar`, `s_samf`):
        - If there is an accumulated group (i.e., `s_samf <> *zero`), call subroutine `trekk` to finalize and write the accumulated record.
      - Call subroutine `felt` to copy the current input record's fields into the output record structure.
      - Accumulate amounts (`w_belø`, `w_rest`, `w_rent`, `w_rgrl`) and update group tracking variables (`s_kund`, `s_bdat`, `s_raar`, `s_samf`, `s_rfda`, `s_rtda`).

3. **After Loop Completion:**  
   If there is an uncommitted accumulated group (i.e., `w_rest <> *zero`), finalize and write it out.

4. **Set LR:**  
   Indicate end-of-program for RPG cycle.

## Subroutines

### `trekk`
- **Purpose:**  
  Finalizes and writes out the accumulated (consolidated) "generalnota" record for the current group.
- **Actions:**
  - Assigns accumulated values to the output record fields.
  - Sets `rntext` to 'GENERALNOTA' for identification.
  - Clears the accumulators for the next group.

### `felt`
- **Purpose:**  
  Copies all relevant fields from the input (`Z`-prefixed fields from `rkwbl3`) to the output record structure.
- **Actions:**  
  Field-by-field assignment to ensure all data (including audit and reference fields) are copied.

### `*inzsr`
- **Purpose:**  
  Program initialization.  
- **Actions:**  
  Sets up the key list for input file access, using the firm value from the user data structure.

## Design Considerations and Conventions

- **Grouping Logic:**  
  Generalnota consolidation is based on both year (`raar`) and collective (`samf`). When either changes, the current group is finalized and a new group is started.
- **Field Prefixing:**  
  Input fields are all prefixed with `Z` due to the `prefix('Z')` directive, which helps distinguish input from output fields.
- **Explicit Field Mapping:**  
  The `felt` subroutine ensures a one-to-one mapping between input and output fields, which is important for both data integrity and maintainability.
- **No Use of SQL or APIs:**  
  The code relies on native RPG file access and record-level operations, consistent with legacy batch processing patterns.
- **Business Rule Enforcement:**  
  Only records on a generalnota are grouped; others pass through unchanged.

## Relationships to Other Modules or Data

- **rkwbl3**: Source of detail records, likely populated by earlier processing or data entry.
- **rkwdpf**: Output file for processed/consolidated records, likely consumed by downstream accounting or reporting processes.

## Potential Points of Interest

- **Extensibility:**  
  The grouping logic could be adjusted if business rules change (e.g., grouping by additional fields).
- **Auditability:**  
  The use of `rntext = 'GENERALNOTA'` provides traceability for consolidated records.
- **Performance:**  
  As this is a sequential process over potentially large files, performance depends on input file organization and key design.

## Summary

RK661BR is a batch RPG program that processes postings from the detail file, consolidating those on a generalnota per year and collective, and copying others as-is. The design is straightforward but robust, reflecting a need for auditability and clear separation between grouped and ungrouped records. The program is tightly coupled to the data structure of `rkwbl3` and `rkwdpf` and assumes sequential processing for each firm.