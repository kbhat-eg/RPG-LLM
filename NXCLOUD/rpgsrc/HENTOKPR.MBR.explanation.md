# RPG Program Explanation

---

## Program Purpose

This RPG (Report Program Generator) program is designed to read, validate, and process accounting journal entries (postings) from a disk file that is probably exported from a spreadsheet. The entries contain details such as account numbers, debit/credit postings, amounts, and additional referencing information. The program checks the validity of data, provides error messaging, and writes validated or error-tagged transaction records to output files.

---

## Overview of Main Components

### File Declarations

- **okpost** : Input file (unformatted, disk-based), presumably containing the journal import data.
- **fra07l1**, **fra28l1**, **frhovl1**, **frkunl1**, **frlevl1**, **frtralu**, **frbunlu** : Various master/reference and transaction files, used for validation and output, each with file-specific record formats.

### Variable Declarations

- **Local Data Area**: 
  - l_feil, l_user, l_firm, l_fnav: Used to track errors, user, firm, etc.
- **Key Variables**: 
  - For each master file, variables like `ra07l1_firm`, `ra07l1_avde`, etc. are used as key fields for record lookups.
- **Constants**: 
  - c_digi, c_dig2: Hold strings for numeric validation.
- **General Variables**: 
  - Various fields for looping (i, j), dates, amounts, debit/credit, timestamps, status and error messages, and record/line counters.
- **Field Variables**: 
  - Variables (starting with d_) for each field in the journal record, for parsing and validation.

---

## Main Program Logic

### File Opening and Initial Setup

- The file `okpost` is opened, and processing status is set to 'Kontroll' (Control/Validation).
- Error flag `l_feil` is cleared.

### Main Processing Loop (`start` tag)

- **Reading Records**: Reads a line from `okpost` into variable `oktran`.
- **End-of-File (EOF) Handling**:
  - If EOF, and if status is 'Oppdat' (Update), update the batch (`getbunt` subroutine), close the file, and end.
  - If status is not 'Oppdat' and errors have been detected (w_feilMf), prompt the user if they want to continue despite errors, invoking a dialogue ('AA007R'). If user agrees, clear error flag and proceed to update mode; otherwise, set error state and end.
- **Row Processing**:
  - If no errors, close and reopen `okpost`, call `getbunt`, set status to 'Oppdat', and restart processing in update mode (to actually update/write records).

### Parsing & Validating a Journal Line

- **Special Handling for 'Periode' Row**:
  - If the row starts with "Periode.", "Periode:", "PERIODE.", or "PERIODE:", extract year and period.
  - Validate numeric format; if invalid, report error.
- **Skip Empty/Irrelevant Rows**:
  - If the row isn't a data row (based on some character checks), skip to next line.

#### Field Extraction Process (Repeatedly):

1. **BILAGSKODE (Voucher Code)**: 
   - Extract up to next semicolon as voucher code, validate numeric.

2. **BILAGSDATO (Voucher Date)**:
   - Extract, transform from `DD.MM.YY` to YYMMDD, validate date format, and check date is within ±1 year from today.

3. **BILAGSNUMMER (Voucher Number)**: 
   - Extract, right-justify and zero-extend as needed, validate numeric.

4. **AVDELING Debet (Dept Debit)**:
   - Extract and validate. If present, check that the department exists in the references.

5. **KONTONUMMER Debet (Account Debit)**:
   - Extract and validate. If present, lookup in account, customer, or supplier files.

6. **BÆRER/SPES Debet (Bearer/Special Debit)**:
   - Extract and lookup in special code reference table.

7. **MOMSKODE Debet (VAT Code Debit)**:
   - Extract VAT code if present.

8. **BELØP (Amount)**:
   - Extract, validate, and convert to numeric value.

9. **AVDELING Kredit (Dept Credit)**:
   - Extract and validate, similar to debit.

10. **KONTONUMMER Kredit (Account Credit)**:
    - Extract and validate. Lookup similar to debit.

11. **BÆRER/SPES Kredit (Bearer/Special Credit)**:
    - Extract and lookup in special code reference.

12. **MOMSKODE Kredit (VAT Code Credit)**:
    - Extract VAT code if present.

13. **TEKST (Text)**:
    - Extract remaining text field.

#### Error Handling

- If any field fails validation, build an error string, flag error, and write the error record to the output (EXCEPT OkUppd).

---

## Updating Transactions

- If status is 'Oppdat', build and write transaction (`rtralur`) records:
  - Populate record fields with parsed values.
  - Update running debet/kredit totals.
  - Set date/time/user for auditing.
  - Write the transaction record.

---

## Batch Processing Subroutine (`getbunt`)

- If year (s_aar) is not blank, update previous batch record with accumulated totals.
- If more records remain, assign a new batch number (by reading last batch and incrementing).
- Write new batch header with current year, period, date/time, and user, and reset subtotals.

---

## Initialization Subroutine (`*inzsr`)

- Defines key lists for file lookups (for department, special codes, account, customer, batch, and transaction files).

---

## Output Specification

- Defines output record format (`OkUppd`) for writing error lines or processed transactions.

---

## Business Rules & Error Messaging

- Norwegian comments and field names suggest the context is Norwegian financial processing.
- Error messages are in Norwegian (e.g., "Feil Bilagskode" = "Invalid Voucher Code").
- The program expects a delimited import file (using semi-colons).
- Handles special Norwegian letters (Æ, Ø, Å) in data import and output.

---

## Additional Notes

- The program is structured in the RPG III/IV "cycle" style, with heavy use of classic C-specs and D-specs.
- The code is robust in validation and provides guidance to the user in case of errors found during the 'Kontroll' phase, before proceeding to update.
- The code assumes dual-phase operation: validation (Kontroll) and update (Oppdat), requiring a clean pass before writing to output files.

---

## Summary

**This program reads journal (voucher) records, validates each field against company master data, and, if valid, writes entries to transaction files in batches. It provides detailed error feedback, ensures referred data exists, and enforces critical accounting controls regarding periods, accounts, and batch integrity.**

---

### Key Takeaways for Developers

- **Two-Phase Processing**: All entries are validated before any updates are done.
- **Data-Driven**: Many field validations depend on lookups in reference/master files.
- **Strong Error Handling**: Errors are captured, flagged, and reported, with the end user in control of whether to proceed.
- **Batch Logic**: Transactions are grouped into "batches"; batch headers/totals are updated as batches change.

---

**Understanding the specific data structures (DFDs and referenced files) and the Norwegian terminology is crucial for onboarding or extending this program.**