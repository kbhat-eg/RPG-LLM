# RPG Program Explanation: SBFILR – Generate New Budget from Register

This RPG/400 (ILE RPG) program is designed for IBM i (AS/400) environments. Its title ("Danner nytt budsjett utfra register") means **"Create new budget from register"** in Norwegian. The code is not free-form and uses the classic fixed-form syntax.

## General Purpose

The program reads input records (presumably budget base data), processes them, aggregates monthly values, and writes or updates records in a budget file. It builds new keys using data from various master files (like customer and item master). It can also delete old budget lines before generating new ones.

---

## Structural Overview

- **Files Declared**: The program works with multiple database files, both for reading reference data and for writing/updating budgets.
    - *skbul1*, *skbulu*: Budget files, for old and new budgets.
    - *rkunl1*, *vvarl1*, *vhgrl1*, *vugrl1*: Customer, item, item group master files.

- **Data Structures (DS and SDS)**: Used for input parameters, internal processing, and key building.

- **Indicators**: Old-style RPG indicators (`*INxx`) are widely used for flow control.

- **Input and Calculations**: Reads input data, multiplies/aggregates monthly values, builds keys, and updates/creates new budget records.

- **Subroutines**: Logical sections of the program, e.g., key building (`nykey`), initialization (`*INZSR`), budget indicator setup (`binit`), and record deletion (`slett`, `xslett`).

---

## Main Sections Explained

### 1. Parameter Handling and Initialization
- **Parameter Passing**: Receives a 64-byte parameter with multiple fields overlaid for meaning (company, code, etc).
    ```rpg
    d wsparm                        64
    d  wsfirm                        3  0 overlay(wsparm:01)  // Company number
    d  wshhaa                        4  0 overlay(wsparm:04)  // Some code/year
    d  wsvers                        2    overlay(wsparm:08)  // Version
    d  wsakod                       12    overlay(wsparm:10)  // Action code
    d  wsaabb                        1    overlay(wsparm:22)  // Category code
    d  wskode                       12    overlay(wsparm:23)  // Budget concept code
    ```
- **Parameter unpacking**: In `*entry` subroutine, reads values, sets working variables, sets up indicators for the budget concept.

- **Initialization Subroutine (`*INZSR`)**:
    - Builds key lists for each file.
    - Prepares the environment for reading and updating the correct records.

---

### 2. Removing Old Budget Lines

**Subroutines:**
- `slett`: Sets key fields to zero/blank.
- `xslett`: Uses key to read all matching budget lines, validates that they match the parameters, and deletes them if present (`delete skbulur`). It loops to process all matching lines. This ensures a clean slate before new records are generated.

---

### 3. Processing Input Data

The main body (after some initial GOTO and tag logic) processes the input.

- **Monthly Budget Values:**
    - Reads monthly base values (`inbu01` ... `inbu12`) from the input and multiplies by 1000 to scale up.
    - Reads adjustment values (`inbd01` ... `inbd12`) and adds them to working totals.

    ```rpg
    c     inbu01        mult      1000          wwbu01
    ...
    c                   add       inbd01        wwbu01
    ```

- **Key Building (`nykey` subroutine)**:
    - Fills working variables from item and customer master files, if found.
    - Resets all key fields to blank/zero.
    - Depending on indicator arrays (set by `binit`, which interprets the budget concept code), moves appropriate groupings (like department, warehouse, order type, etc.) into the key structure for the new budget records.

- **Budget Record Update/Write:**
    - Tries to `chain` (find) a matching record.
    - If found, adds current values to existing ones and updates.
    - If not found, sets up all fields and writes a new record.

---

### 4. Indicator Arrays and Budget Structure Mapping (`binit`)

- Interprets the 12-character budget code.
- Sets indicator variables (`*IN41` to `*IN92`), which subsequently inform which dimension(s) the budget aggregation is being done for (e.g., department, warehouse, customer, item group, etc.).
- This serves to make the program flexible for many different budget perspectives.

---

### 5. Supporting Key Lists

- **Key lists (`klist`)** prepare composite keys for chain/read/update operations in various files. These are declared in the `*INZSR` subroutine and used throughout the program.

---

## Key Variables

- **wwbu01-wwbu12**: Working variables for each month's budget value.
- **sbbu01-sbbu12**: Fields in the target budget record.
- **skbulu_*:**: Key fields for new budget (department, warehouse, order type, etc.).
- **skbul1_*:**: Key fields for old budget.

---

## High-Level Flow

1. **Startup**: Receives parameters, parses them, sets up the environment, and deletes any old budget lines that match the scenario.
2. **Main Processing Loop**:
    - For each input record:
        - Aggregates base and adjustment values.
        - Builds a target key using item/customer master files as context.
        - Updates existing or writes new budget lines using the built key.
3. **Finish**: Ends program after all input is processed.

---

## Typical Use Case

- **Re-budgeting**: Before a new budget is created, old data for the same company/concept/year is deleted. Then input lines are processed, budget lines built per defined dimensions (set by code), and written to the file.
- **Flexibility**: The use of dynamic mapping via code and indicators allows a single program to handle many types of budget breakdowns (by department, by item group, etc.).

---

## Noteworthy RPG/400 Practices

- **Indicators for Control**: Old-school RPG uses indicator arrays to manage conditional logic.
- **Multi-dimensional Budgeting**: "Budget concept code" enables flexible aggregation.
- **Structured Key Building**: Key fields are dynamically filled according to the desired roll-up level.

---

## Summary Table

| Section                | Purpose / Action                                                                             |
|------------------------|----------------------------------------------------------------------------------------------|
| Parameter handling     | Receives and breaks out the fields from the calling program                                  |
| Initialization         | Sets up file keys, indicator mappings, etc.                                                  |
| Delete old budgets     | Removes all old budget records for the current scenario                                      |
| Input reading          | Reads base data from input file, scales and aggregates monthly values                        |
| Key creation           | Gathers item/customer groupings as needed for budget structure                               |
| Record upsertion       | Finds/writes budget records: adds to existing or creates new                                 |
| Indicator mapping      | Decodes budget concept code to set dimension indicators                                      |

---

## In Practice

- *Before using*: Make sure the master data files are up-to-date and the correct budget concept code is supplied.
- *Customizing*: To add new budget breakdowns, update the `binit` routine to interpret new code positions.
- *Debugging*: If budgets aren't being written as expected, check the input code, indicator mapping, and key-building subroutine.

---

## Conclusion

This is an extensible budget-generation program that cleans old data and creates new, aggregated records based on flexible keying logic. It is representative of classical AS/400 RPG batch data processing, designed for reusability and adaptability via parameterization.