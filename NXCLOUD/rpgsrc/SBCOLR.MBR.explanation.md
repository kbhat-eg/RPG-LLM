# Documentation: SBCOLR – Budget Copy (Full)

## Overview

**Program:** SBCOLR  
**System:** ASSTAT  
**Description:**  
This program performs a full copy of budget records from one version to another within the budget register. It reads source budget records from the `SKBUL1` physical file and writes new records to the `SKBULU` update file, using specified selection and target keys.

---

## Business Context

This program is part of the budgeting subsystem. It automates the process of copying all budget entries (potentially for a given company, year, version, etc.) from a source to a target, possibly as part of a budget versioning or annual rollover process.

The naming and field conventions are Norwegian:
- "budsjett" = budget
- "kopiering" = copying
- "hele" = whole/all

---

## File Definitions

- **SKBUL1**  
  - Input file (DISK, keyed), renamed record format `SKBUPFR` to `SKBUL1R`.
  - Source for budget records to be copied.

- **SKBULU**  
  - Update file (DISK, keyed), renamed record format `SKBUPFR` to `SKBULUR`.
  - Target for new budget records.

---

## Data Structures and Variables

- **Key Fields:**  
  Variables prefixed with `ww` (working with source) and `wp` (entry parameters) represent key or selection fields for budget records (e.g., company, budget code, version, etc.).

- **Keylists:**  
  - `FRAKEY` (source key): Used for reading from `SKBUL1`.
  - `TILKEY` (target key): Used for writing to `SKBULU`.

- **Parameter Passing:**  
  The program is designed to be called by another program, receiving parameters that define the source and target for the budget copy operation.

---

## Main Logic Flow

### 1. Initialization

- All selection variables (`ww*`) are initialized to zeros or blanks as appropriate.
- Entry parameters (`wpfirm`, etc.) are moved into the working selection variables.

### 2. Main Copy Loop

- The program uses a keyed SETLL and READ loop to process all records in `SKBUL1` that match the source key (`FRAKEY`).
- For each record:
  - It checks that the current record matches all relevant selection fields (company, main account, version, code, etc.).
  - For each matching record, it attempts to CHAIN (retrieve) the corresponding record in the target file (`SKBULU`) using the target key (`TILKEY`).
  - If the target record does not exist (`*IN92` is *ON after CHAIN), it prepares the record for writing:
    - Updates the target version/main account fields (`sbhhaa`, `sbvers`) to the target values.
    - Writes the new record to `SKBULU`.

- The loop continues until all relevant records are processed.

### 3. Program Termination

- The program sets on LR (Last Record) and returns.

---

## Key Design Decisions and Conventions

- **Keylist Usage:**  
  Both source and target file operations use keylists (`FRAKEY`, `TILKEY`) to ensure consistent and maintainable key management. This reduces duplication and centralizes key field definitions.

- **Record Matching:**  
  The program explicitly checks each key field to ensure only the intended records are copied. This is critical for business integrity, as budget records are highly sensitive to correct versioning and account selection.

- **Idempotency:**  
  Before writing a new record to the target, the program checks if the record already exists to avoid duplicates.

- **Parameterization:**  
  The program is fully parameter-driven, making it reusable for different company codes, budget versions, etc.

- **Record Format Renaming:**  
  Both files use the same base record format (`SKBUPFR`) but are renamed locally for clarity and to avoid conflicts.

---

## Interactions and Dependencies

- **Upstream:**  
  - Receives parameters from a calling program, which likely handles user interface or job control.

- **Downstream:**  
  - Writes to the `SKBULU` file, which is the update file for budget records. Other processes may pick up these records for further processing, reporting, or approval.

- **Data Integrity:**  
  - The program does not delete or update existing records—only inserts new ones if they do not already exist.

---

## Special Notes

- **Error Handling:**  
  - The program uses RPG indicators to control flow, but there is no explicit error reporting or logging. Failures in file I/O (other than record-not-found) are not handled.

- **Field Naming:**  
  - Most fields are named with Norwegian abbreviations. New developers should familiarize themselves with these conventions for easier code navigation.

- **No User Interaction:**  
  - This is a batch/utility program, not intended for interactive use.

---

## Summary Table: Key Fields

| Field    | Description                  | Usage             |
|----------|------------------------------|-------------------|
| firm     | Company number               | Key, selection    |
| hhaa     | Main account                 | Key, selection    |
| vers     | Version                      | Key, selection    |
| kode     | Budget code                  | Key, selection    |
| aabb     | Account group                | Key, selection    |
| ...      | (other fields)               | Key, selection    |

---

## Conclusion

This program is a utility for copying all budget records from a specified source key to a specified target key, ensuring no duplicates are created. It is designed for batch processing and integrates tightly with the budget register subsystem. The code structure emphasizes parameterization, keylist usage, and explicit record matching to maintain data integrity and flexibility.