# Explanation of RPG Program: RFP14R

This RPG (Report Program Generator) program processes transaction records, assigning missing accounting periods based on the voucher (bilags) date, updating status fields, and maintaining linkage to vendor/customer numbers for integration with Navision (NAV). Below, the code is explained in detail, section by section, with notes on its structure, logic, and important routines.

---

## 1. Header and Comments

- **/TITLE**  
  Title line, giving a short description: "Prepares transactions from scanning".
- **h-specs**  
  - `decedit(',')` and `datedit(*dmy.)`: Set decimal and date edit codes for numeric and date fields.
  - `option(*nodebugio)`: Suppresses display of input/output debugging information.
- **Comments**  
  - Describe the system, program name, and a summary of the program's function:  
    > "Assigns period from voucher date if period = 0"
  - Change history ("Ref.") and brief version notes.

---

## 2. File Declarations (f-specs)

- **rcmpl1**  
  - Input file for transactions (from scanned vouchers?), renamed and prefixed.
- **rcmrlu** (from 6.30)  
  - Input/output file for vendor/customer lookups, with its own prefix, used for NAV integration.
- **raa1l1** (from 6.30)  
  - Input file for period status, used to check period limits.

---

## 3. Data Definitions (d-specs)

- **User-Related Fields**  
  - `l_user`, `l_fgrp`, `l_firm`, `l_fnav`: Fields read from a user area (likely from externally passed data).
- **Array Definition**  
  - `per`: Array to hold periods, not used directly in the main logic shown.
- **Work Variables**
  - `w_peri`, `w_amd`, `w_20`, `w_40`: For period/year calculations.
- **NAV/Integration Variables**  
  - `w_memb`, `w_jkey`, `w_jsta`, `w_jmld`, `w_stat`: For messaging/error handling.
- **Fields for Navision Lookups** (from 6.30):  
  - `rcmrlu_firm`, `rcmrlu_biln`, `raa1l1_firm`: Used in keyed lookup.

---

## 4. Main Routine

### a. Initialization

- `w_stat` is set to 1 (likely meaning "no transactions found yet").
- Reads the first record from **rcmpl1**.

#### Handling "No records":

If EOF occurs immediately after the read, an error message is set up and program `AB705R` is called to log/report:  
- `w_jsta = 10`
- `w_jmld = 'Ingen poster fra Scanning'` ("No records from Scanning")
- The subroutine is called with required parameters.

---

### b. Main Loop: Process All Transaction Records

A `dow not %eof(rcmpl1)` loop processes each transaction.

#### i. NAV Integration (6.30):

- If certain conditions are met (e.g., vendor/customer number exists, or period is set), the program attempts to update/create a record in **rcmrlu** to track the linkage between document and vendor/customer for Navision posting.
- If found, it's updated; otherwise, a new record is written.

#### ii. Main Assignment Logic

- If the firm and group code from the transaction match the ones passed/loaded (`xtfirm = l_firm and xtfgrp = l_fgrp`):
  - Set `w_stat = 0` to indicate that transactions were found and processed.
  - If the period (`xtrper`) from Compello (scanning system) is zero:
    - Extract the period and year from the voucher date (`xtbdat`):
      - `w_amd` stores the date in *YYMMDD* form.
      - `w_40 = w_amd/100` yields *YYMM*.
      - `w_20 = w_amd/10000` yields *YY*.
      - The period is calculated as the MM part, year as the full year (YY + 2000).
  - Mark the record as processed by setting `xtautx = 'X'`.
  - Update the record in **rcmpl1**.

---

### c. Read Next Transaction

- At the end of the loop, the next record is read.

---

### d. Program End

- Sets *INLR to *ON to end the program and close files.

---

## 5. Subroutines

### *inzsr (Initialization Subroutine)

- Defines the program's entry parameter list.
- Sets up keys for file lookups (`rcmrlu_key` and `raa1l1_key`).
- For period status, if not found, sets maximum value (`ra1hrs = 599999`), which is used as a threshold elsewhere.

---

## 6. Summary Table

| Section                    | Purpose                                              |
|----------------------------|------------------------------------------------------|
| File Declarations          | Input/output files for transactions, lookups, etc.   |
| Data Definitions           | Work variables, arrays, and fields for processing    |
| Main Routine               | Loops through transactions, updates period info      |
| NAV Link Logic (6.30)      | Links voucher to vendor/customer for Navision        |
| Period Assignment          | Assigns period/year from date if period missing      |
| Record Update              | Marks transactions as processed                      |
| Initialization Subroutine  | Sets up keys and initial values                      |

---

## 7. Key Business Logic Points

- **Assigning Periods:** If a scanned transaction is missing an accounting period, it’s automatically derived from the voucher date.
- **Marking for Backup:** All processed records are flagged (`RTAUTX = 'X'`) to denote backup taken.
- **NAV Support:** For transactions relevant to Navision, the vendor/customer number and period are linked or updated.
- **Messaging:** If no records are present, a message is sent to a logging/reporting routine.

---

## 8. Versioning

Enhancements and special logic are tagged with version markers (e.g., 6.10, 6.30).

---

## 9. Error/Edge Handling

- Handles no-records situation gracefully.
- Sets "Sperret periode" (blocked period) threshold so that missing period data doesn't break processing.

---

## 10. Quick Onboarding Notes

- **Input:** Scanned transaction records (Compello).
- **Output:** Updated records with assigned periods and NAV links, status flagged for backup.
- **Error Handling:** Messaging and status updates ensure process traceability.
- **Integration:** Built-in logic for seamless posting into Navision by linking voucher and vendor/customer details.
- **Maintainability:** Modular structure with clear subroutines and versioning.

---

The program is a robust transaction pre-processor designed to ensure all accounting records are complete and ready for downstream ERP integration, providing automated data corrections and bookkeeping integrity.