## Program Overview

This program, **SCDELR**, is part of the **ASSTAT** system and is responsible for **deleting all budget entries** ("Sletting av budsjett (hele)") from the budget register for a specific key combination. The program is written in traditional (fixed-format) RPG and operates primarily on two physical files: `slbul1` and `slbulu`, which represent the budget register and its update file, respectively.

---

## Business Context

Within the ASSTAT system, budgets are maintained at a detailed level, with each entry identified by a complex composite key. This program is invoked to **delete all budget records** that match a specific set of key values, likely as part of a budget reset, reinitialization, or data correction process. The program is designed to be called from another program, receiving its key parameters via a parameter list.

---

## File Definitions

- **slbul1**  
  - Input, keyed, disk file.  
  - Renamed record format: `slbul1r`  
  - Represents the main budget register.

- **slbulu**  
  - Update, keyed, disk file.  
  - Renamed record format: `slbulur`  
  - Used for deleting records.

---

## Key Data Structures and Variables

- **Key Fields (`ww*`)**  
  Variables prefixed with `ww` are used to build the key for reading and deleting records in the budget register.  
  These include fields like `wwfirm` (company), `wwhhaa` (main account), `wwvers` (version), `wwkode` (code), `wwaabb` (sub account), and several others representing dimensions such as department, warehouse, type, country, district, salesperson, category, etc.

- **Parameter Fields (`wp*`)**  
  Variables prefixed with `wp` are populated from the parameter list received from the calling program. They provide the initial key values for the deletion operation.

---

## Main Processing Logic

### 1. Parameter Handling and Initialization

- The program begins by defining a *parameter list* (`*entry plist`) to receive the key values from the calling program. These are mapped to the local `wp*` variables.
- The key fields (`ww*`) are initialized using these parameter values (mainly `wwfirm`, `wwkode`, `wwaabb`).
- Selected key fields are set to zero or blank, effectively broadening the key to encompass all possible sub-dimensions under the specified main key (e.g., all departments, warehouses, product types, etc.).

### 2. Building the Key List

- The **KLIST** named `BUDKEY` is constructed from all the `ww*` fields. This composite key is used for SETLL, CHAIN, and DELETE operations against the budget files.

### 3. Deletion Loop

- The program uses `SETLL` to position to the first record in `slbul1r` matching the composite key.
- It enters a loop (`TAG02`) where it:
  - Reads the next record in key order.
  - Checks if the read record matches all the primary key fields (`scfirm`, `schhaa`, `scvers`, `sckode`, `scaabb`).
  - If matched, it loads all sub-dimension fields from the record into the corresponding `ww*` variables.
  - Attempts to `CHAIN` the record in the update file (`slbulur`) using the full composite key.
  - If found, executes a `DELETE` on that record.
  - Loops back to process the next record.

- The loop continues until a record is encountered that does not match the main key fields, at which point the program exits the loop.

### 4. Program Termination

- After processing, the program sets on LR (last record indicator) and returns.

---

## Design Considerations and Conventions

- **Key Generalization:**  
  By setting many key fields to zero or blank, the program ensures that *all* budget records under the specified main keys are targeted for deletion, regardless of their sub-dimensions. This is a deliberate design to facilitate full-budget wipes for a particular account/version combination.

- **Record Matching:**  
  The program is careful to compare only the main key fields to determine the scope of deletion, ensuring that it does not inadvertently delete records outside the intended scope.

- **Separation of Read and Update Files:**  
  The use of separate files for reading (`slbul1`) and updating (`slbulu`) is typical in this codebase, possibly due to record locking or journaling requirements.

- **Error Handling:**  
  The program checks for the existence of records before attempting to delete (`CHAIN` with indicator 92). If the record is not found, delete is not attempted.

- **Parameter Passing:**  
  The program is designed to be called by another program, receiving all necessary key information via the parameter list.

---

## Integration with Other Modules

- **Calling Program:**  
  This program is not standalone; it expects to be called by another module, which supplies the relevant key parameters.

- **Data Files:**  
  The program's actions directly impact the budget register and its update file, so downstream processes or reports relying on these files will reflect the deletions performed here.

---

## Summary

**SCDELR** is a utility program for mass deletion of budget records within the ASSTAT system, targeting all records under a given main key structure. It is designed for use by other programs or batch processes, ensuring efficient and controlled purging of budget data as needed for business operations. The code demonstrates careful handling of composite keys and follows established conventions in the system for file access and record manipulation.