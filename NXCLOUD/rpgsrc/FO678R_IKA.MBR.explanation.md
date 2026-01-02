# RPG Program Overview: Faktura-journal til Ikas Kreditsystemer

This RPG program (`FO678R`) is designed to produce an invoice journal ("Faktura-journal") for the Ikas Credit Systems. It pulls together invoice, customer, and company information, processes it, and prepares output records.

---

## Header and Metadata

- **H-specs**: Set report title, decimal and date edit words, and compile options.
- **Comments Header**: Describes the program, its purpose, and a change log, including explanations for various historical modifications (e.g., support for longer invoice/customer numbers, KID number calculation, etc.).

---

## File Declarations (F-Specs)

- **Input Files**:
  - `SOHEL1`, `SOHELU`: Invoice header files (renamed from physical file SOHEPFR).
  - `rkunl1`: Customer register (renamed).
  - `AFORL1`: Formular register (renamed).

- **Output Files**:
  - `RE0IPF`: Output file for the report.

---

## Data Structures (D-Specs)

### Arrays

- `TLFI`, `TLFU`: Arrays for temporary storage/processing, each with 11 elements.

### Data Structures

- **Parameter and Working Area**:
  - `DSPARM`: Full incoming 32-byte parameter string.
  - `DSAARR`, `DSOTYP`, `DSFFNR`, etc.: Individual fields within the parameter.
  - Strings for input mapping, customer data, invoice info, etc.

- **Other Data Structures**:
  - For KID number calculation, working keys, and fields extracted from LDA (Local Data Area).

---

## Comments on Output Formats

- `A1HDR`: Report heading
- `A1DET`: Report detail lines
- `T1TOT`: Report totals

---

## Mainline Logic (C-Specs)

### Initialization

- Subroutine `XSTRSR`:
  - Sets up necessary fields, retrieves next available document number if needed, reads the first record, and prepares for processing.

### Entry Point (`*ENTRY`)

- Retrieves parameter(s) from a previous program call into `DSPARM` and its subfields.
- Initializes key fields.

### Processing Loop

- Reads records from `SOHEL1` (main invoice header file) in a loop (`DOWEQ *OFF` on indicator *IN61).
- Checks if data relates to a new company or year (jumps to re-init if needed).
- Handles filtering by "from" and "to" invoice number ranges.
- Calls subroutine `XAJOUR` to prepare data for journal output.
- For non-zero invoice amounts, writes output record (`RE0IPFR`).
- Reads the next invoice header record for continued processing.

### Termination

- After the loop, sets the LR (last record) indicator for normal program ending.

---

## Subroutines

### HNTNUM – Fetch/Update Number Register

- Calls external program `AS100R` to fetch the next available document/journal number.

### XAJOUR – Prepare Data for Journal Output

- Resets some date fields.
- Converts/validates invoice and due dates.
- Looks up customer information in the customer register (`rkunl1r`), populates address and contact fields, and formats telephone numbers.
- Handles organization and customer category fields, providing default values if missing.
- Maps invoice and reference details.
- Calculates the appropriate sign for invoice amounts (debits/credits).
- Legacy (commented-out) code for KID calculation (Norwegian reference payment ID).
- Depending on setup, assigns either a "short" or "long" KID from historical register.

### XSTRSR – Startup Initialization

- Sets up key fields for file processing (company, year, invoice numbers, etc.).
- Optionally fetches the next available invoice/journal number if not working with a range.
- Reads first record from main invoice header file.

### *INZSR – Program Initialization

- Sets up field equivalences, key lists for files (used in CHAIN and SETLL operations), and initializes various working fields and arrays.
- Retrieves Formular register entry for reporting format, if needed.

---

## Key Lists (KLIST/KFLD)

Defined for use in file operations:
- `KEY01`: For heading file SOHEL1.
- `KEY04`: For customer register.
- `KEY10`: For formular register.

---

## Parameter Handling

- Inputs from the prior program invocation are parsed using the PLIST and PARM structure, and their content is unpacked to working fields.

---

## Special Notes (from comments and code):

- **KID Calculation**: The program has support for Norwegian KID numbers, with code adapting to requirements for either long or short forms, based on a routine register.
- **Customer and Invoice Number Expansion**: Updates ensure support for extended-length keys.
- **Data Formatting**: Telephone numbers, date fields, and addresses are formatted specifically for output records, sometimes using array manipulations for compactness.
- **Conditional Execution**: Several sections of code are commented out (with `**`), hinting at optional features or phased-out code.

---

## Summary

- **Purpose**: Produce a structured invoice journal with accurate customer, invoice, and reference data, following Ikas Credit Systems requirements.
- **Data Flow**: Read invoice headers and customer info, process and format data, handle KID and number register interactions, and output journal records.
- **Customization**: The program is adaptable for changes in field lengths, record formats, and KID processing, as documented through the code's change history.

---

This program is a fairly standard example of an IBM i (AS/400) RPG legacy application, using fixed-form syntax for database access, data manipulation, and report generation. It is highly tailored to Norwegian accounting and invoicing standards, evident from its KID routines and date formatting.