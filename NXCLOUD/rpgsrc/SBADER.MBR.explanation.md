# SBADER – Accumulation Register Update (Deletion)

## Overview

This program (SBADER) is part of the ASSTAT system and is responsible for updating the accumulation register by deleting relevant records. It operates either for a single code or for all budget codes, depending on the input parameters. The program interacts with three files:

- **SKNIL1**: Customer-level register (KUNDE-NIVÅ-REGISTER)
- **SKAKL1**: Customer accumulation register (KUNDE-AKKUMULERINGS-REGISTER)
- **SKAKLU**: Customer accumulation register (update/delete target)

## Business Context

This program is invoked to remove accumulation records, typically as part of a larger update or cleanup process. The deletion is selective, based on company, account, and code, and can be executed for a single code or for all codes associated with a company/account.

## Key Data Structures and Files

### Files

- **SKNIL1 (sknil1r)**: Used to validate if a given code should update the accumulation register.
- **SKAKL1 (skakl1r)**: Used to read all relevant accumulation records for a given key.
- **SKAKLU (skaklur)**: Used to delete matching accumulation records.

### Parameters

- The program receives a 128-byte parameter string (`dsparm`), which is parsed into:
  - **wkfirm**: Company number (3,0)
  - **wkhhaa**: Account number (4,0)
  - **wkkode**: Code (12)

### Key Variables

- **w_firm, w_hhaa, w_kode**: Working variables for company, account, and code.
- **skaklu_* variables**: Used to build the full key for SKAKLU/SKAKL1 and to control which records are affected.

## Program Flow

### Initialization (`*inzsr`)

- Parses the input parameter into local fields.
- Builds key lists for file operations (see "Key List Definitions" below).

### Main Logic

1. **Single vs. Multi-Code Operation**
   - If the input code (`wkkode`) is blank, the program processes all codes for the company/account (multi-update).
   - Otherwise, it processes only the specified code.

2. **Validation**
   - For a single code, the program checks if the code exists in SKNIL1 and if it is allowed to update the accumulation register (i.e., `snakkk` is blank).
   - For multi-code, it loops through all SKNIL1 records for the company/account, processing only those where `snakkk` is blank.

3. **Accumulation Register Update (Subroutine `oppakk`)**
   - For each relevant code, the program loops through all SKAKL1 records matching the company, account, and code.
   - For each such record, it builds the full key (including all dimension fields) and attempts to delete the corresponding record in SKAKLU.

4. **Program Termination**
   - Sets LR indicator and returns.

## Key List Definitions

- **sknil1_key**: Used for SKNIL1 access, keyed by company and code.
- **skakl1_key**: Used for SKAKL1 access, keyed by company, account, code, and all dimension fields.
- **skaklu_key**: Used for SKAKLU access, same as SKAKL1.

## Notable Design Choices

- **Key Construction**: All file operations use constructed key lists for precise record access and deletion.
- **Overlaying Parameters**: The input parameter is parsed via overlays, a common pattern in legacy RPG for fixed-format parameter blocks.
- **Selective Deletion**: Only records matching both the primary key (company/account/code) and all dimension fields are deleted.
- **Blank/Zero Initialization**: Dimension fields are initialized to blank/zero before each read loop to ensure correct key construction.

## Interactions and Dependencies

- **External Invocation**: This program is designed to be called from another program, passing the parameter block.
- **File Relationships**: SKNIL1 acts as a control file for which codes are processed; SKAKL1 provides the list of accumulation records; SKAKLU is the actual target for deletions.

## Error Handling and Control

- **Indicators**: RPG indicators (e.g., *in10, *in12, *in14) are used to control flow after file operations.
- **Conditional Processing**: The program skips records or exits loops based on company, code, and validation flags in the master file.

## Business/Domain-Specific Logic

- **snakkk field**: Only codes where this field is blank are eligible for accumulation register updates (deletions).
- **Budget Codes**: The program can operate on all budget codes for a company/account or a specific code, depending on input.

## Summary of Main Sections

### Entry and Parameter Handling

- Receives a parameter block, parses company, account, and code.

### Single Code Processing

- Validates code in SKNIL1.
- If eligible, deletes all matching SKAKLU records as per SKAKL1.

### Multi-Code Processing

- Loops through all SKNIL1 codes for the company/account.
- For each eligible code, deletes all matching SKAKLU records.

### Deletion Logic (`oppakk`)

- For each SKAKL1 record matching company/account/code:
  - Builds the full key with all dimension fields.
  - Deletes the corresponding SKAKLU record if it exists.

### Program Exit

- Sets LR indicator and returns.

## Conventions and Patterns

- **File Renaming**: All files are renamed on the F-spec for clarity and to avoid naming conflicts.
- **Subroutine Structure**: Uses classic RPG subroutines for initialization and core logic.
- **Tag/GOTO Flow Control**: Uses tags and GOTO for loop and section control, which is typical in legacy RPG but should be noted for maintainability.

## Maintenance Notes

- **Adding New Dimensions**: If new dimensions are added to the accumulation register, all key lists and variable assignments must be updated accordingly.
- **Changing Validation Logic**: Any changes to the rules for when a code can update the accumulation register will require updates in the SKNIL1 validation logic.
- **File Structure Changes**: Changes to the structure of SKNIL1, SKAKL1, or SKAKLU must be reflected in the program’s F-specs and data structures.

---

**In summary:**  
SBADER is a controlled deletion utility for the accumulation register, driven by company, account, and code, with the flexibility to process single or multiple codes as dictated by business rules in SKNIL1. It ensures only eligible codes (snakkk blank) are processed and deletes all detailed accumulation records for those codes, maintaining data integrity in the ASSTAT system.