# SBDELR – Full Budget Deletion Program Documentation

## Overview

**Program:** SBDELR  
**System:** ASSTAT  
**Description:** Full deletion of budget records (“Sletting av budsjett (hele)”)  
**Files Used:**  
- `SKBUL1` (Budsjett-register, main budget file)
- `SKBULU` (Budsjett-register, update file)

This program is responsible for deleting all budget records matching a given set of key values. It is typically called by another program and receives its filter criteria as parameters. The main business function is to ensure that all budget entries for a specific budget (or set of criteria) are fully removed from the system.

---

## File Definitions

```rpg
fskbul1    if   e           k disk    rename(skbupfr:skbul1r)
fskbulu    uf   e           k disk    rename(skbupfr:skbulur)
```

- **SKBUL1**: Input file, keyed access, renamed record format `skbul1r`. Used to read existing budget records.
- **SKBULU**: Update file, keyed access, renamed record format `skbulur`. Used to delete budget records.

*Note: Both files refer to the same physical file but may have different usage (input vs. update).*

---

## Data Structures and Variables

- **Key Fields:**  
  Variables prefixed with `ww` are working variables used to construct the budget record key (e.g., `wwfirm`, `wwhhaa`, `wwvers`, etc.).
- **Parameter Fields:**  
  Variables prefixed with `wp` are incoming parameters from the calling program.
- **Other:**  
  `budkey` is a KLIST used for file access.

---

## Program Flow

### 1. Initialization

- The program begins by initializing working key fields (`ww*`) from the parameters (`wp*`).
- Non-key selection fields (such as `wwavde`, `wwlage`, etc.) are set to their default values (zero or blank) to ensure the key is properly constructed for the full budget scope.

### 2. Main Deletion Loop

- **Setll/Read:**  
  The program positions (`SETLL`) to the first record matching the constructed key (`budkey`) in `skbul1r`. It then enters a loop to read each matching record.
- **Key Matching:**  
  For each read record, it checks if all key fields (`sbfirm`, `sbhhaa`, `sbvers`, `sbkode`, `sbaabb`) match the working key values. This ensures only the relevant records are processed.
- **Key Extension:**  
  For matched records, the rest of the key fields are copied from the current record into the working variables. This is necessary because the budget file may have additional sub-levels (department, warehouse, type, etc.).
- **Delete:**  
  Using the fully constructed key, the program attempts to `CHAIN` into the update file (`skbulur`). If found, it deletes the record.
- **Loop:**  
  The process repeats until no more records match the primary key.

### 3. Program Termination

- After all relevant records have been processed and deleted, the program sets on LR (`*INLR`) and returns.

---

## Subroutines

### *INZSR (Initialization Subroutine)

- **Purpose:**  
  - Defines the `budkey` KLIST used for all keyed operations on the budget files.
  - Initializes working keys from parameters.
- **Parameter Handling:**  
  - The program receives parameters for firm, code, year, period, and version. These are mapped into the working variables for use in key construction.

---

## Business/Domain Logic

- **Budget Structure:**  
  - The budget is organized by multiple dimensions: firm, main code, version, sub-code, year, department, warehouse, object type, country, district, seller, customer category, customer, object group, main group, sub group, and item.
  - The program is designed to delete all records matching a high-level key (firm, code, version, etc.), but may span multiple sub-levels (e.g., all departments, warehouses, etc.).
- **Full Deletion:**  
  - By setting sub-levels to zero/blank initially, the program ensures it targets the entire budget for the given key, not just a specific sub-division.
- **Dual File Access:**  
  - The use of both input and update access to the same file (under different record formats) is a convention in this system, allowing for efficient read and delete operations.

---

## Notable Design Decisions & Conventions

- **Record Format Renaming:**  
  - The file is opened twice (once as input, once as update) with different record format names. This is a pattern in this codebase to separate read and update logic.
- **Key Handling:**  
  - Key fields are always constructed in working storage before being used in file operations. This ensures consistency and clarity in multi-level key handling.
- **Error Handling:**  
  - Minimal error handling is present; if a record is not found on `CHAIN`, the delete is simply skipped.
- **Parameter Passing:**  
  - The program expects to be called with all necessary key fields as parameters, reflecting a modular, callable design.

---

## External Relationships

- **Calling Program:**  
  - This program is not standalone; it is invoked by another process/program that provides the necessary key parameters.
- **Budget File (`SKBUL1/SKBULU`):**  
  - All deletions are performed directly on the budget register file, which is presumably central to the budgeting module in the ASSTAT system.

---

## Summary

`SBDELR` is a utility program for deleting all budget records matching a specific set of high-level keys. It is designed for batch operation, called from other programs, and operates directly on the budget register file. The program’s structure reflects standard conventions in this codebase for key management, file access, and modular operation.

**If you are onboarding to this module, pay attention to:**
- The multi-level key structure and how working variables are used to build file access keys.
- The dual file definition pattern for read and update operations.
- The expectation that all context (firm, code, version, etc.) is provided by the caller.

For any changes or extensions, ensure key management and file handling conventions are maintained for consistency with the rest of the system.