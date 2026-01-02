```markdown
## Program Overview

This ILE RPG program, named **RMP010R**, processes inventory transactions from an external system called **MalProff**. The main purpose is to read and validate inventory movement records from a flat input file, enrich the data by looking up related information in database files, update inventory balances, and handle errors—including sending notification emails if problems are found.

### File Declarations

- **mpinput**: Main input file with records (256 bytes, disk file).
- **mjlol2**: Lookup file for warehouse numbers, renamed as `mjlol2r`.
- **vvepl2**: Lookup file for product data (item number, unit), renamed as `vvepl2r`.
- **rmpilu**: Physical file for temporary MalProff info, renamed as `rmpilur`.
- **rmpflu**: Error record file for MalProff input, renamed as `rmpflur`.
- **ra09l1**: Lookup file for sales reps, renamed as `ra09l1r`.
- **rax9l1**: Side registry of sales reps, renamed as `rax9l1r`.

### Global Variables and Data Structures

- **digit**: String constant of valid numeric characters (used for checking).
- **Field extraction variables**: Most variables starting with `l_` or `d_` are used for breaking out data from the input or loaded files.
- **hlrec & overlays**: Buffer for data passed to the inventory update routine (`VL001R`), with overlays mapping positions in the record to specific fields (item, warehouse, date, time, type, amount, etc.).
- **Various working variables**: For storing intermediate values like GLN, EAN, quantities, keys, status, text for error messages, email addresses, etc.

### Exception Error Structure

Defined via *SDS (Program Status Data Structure)* to hold information for error handling routines (e.g., last file name, error code, job info).

---

## Main Processing Logic

### Pre-Processing

- Trims trailing blanks from the input record and sets up initial variables, including a copy of the record for writing to error logs if needed.

### Input Record Parsing

- Sequentially extracts fields from the input record (GLN, date, time, line number, EAN/GTIN, quantity, order number, department, customer number, customer name, etc.) by scanning for `;` delimiters.
- Handles possible negative quantities (by checking for a `-` in the quantity and adjusting the sign).
- Some fields (reference, invoice number, invoice date) are commented out—likely not used at present.

### Field Validation

- Validates **date** and **time** fields using RPG's `TEST` operation; if invalid, logs error and branches to clean-up.
- Checks that GLN and EAN/GTIN are strictly numeric.
    - If not, logs a specific error mentioning which field failed.

### Data Enrichment

- **Warehouse lookup**: Uses GLN to look up the warehouse number in the `mjlol2` file.
- **Product lookup**: Uses EAN/GTIN to find the item number and unit of measure in `vvepl2`.

### MalProff Temporary Data Update

- Prepares and updates (or writes new) records in the `rmpilu` file for temporary storage of MalProff-related transaction info.

### Inventory Update

- Populates the inventory update data structure (`hlrec` and overlays).
- Calls the external program `VL001R` to perform the actual inventory update.
- Checks for errors in the update (by `v_stat`, status returned from `VL001R`); if error, writes a message and triggers error handling.

---

## Error Handling Routines

### FileErr Subroutine

Handles unexpected file (database) errors:
- Fills out `p_stat` and `p_errt` with information about the error for return to the caller.
- Branches to finally clean up (`slutt1`).

### Feil (Error) Subroutine

Handles *business logic* or data errors:
- Logs the issue by calling `RMP97C`, passing job, program, sequence, and descriptive text.
- Avoids duplicate error log entries for the same order/line/GLN number.
- Writes an error record to the `rmpflu` file, incrementing a sequence number if needed.
- Looks up potential recipients in the sales rep files (`ra09l1`, `rax9l1`) for notification emails.
- Prepares email subject and text, then calls `AF728R` to send the email, passing recipient addresses and message details.

---

## Initialization Subroutine

Performs program initialization:
- Receives and stores parameters: `w_stat` (status), `w_fil` (file name), `COMFLAG` (commit flag), `p_stat`, `p_errt`.
- Sets up key lists for various database lookups.
- Prepares firm variables and initializes working status to 'Ok'.

---

## Key Technical Points

- **Overlay Data Structures**: The use of overlayed DS fields on `hlrec` allows for easy mapping of flat layout fields for interfacing with other programs.
- **Error Handling**: The program is robust in its data validation and error handling, ensuring that errors are logged/properly communicated and not duplicated.
- **Notifications**: On errors, the program sends emails with details to the relevant sales reps, based on registry files.
- **Extensive Comments**: The code is heavily commented (some in Norwegian) for maintainability.

---

## Summary Flow

1. **Read raw transaction record (delimited by ;) from MalProff.**
2. **Parse and extract all fields.**
3. **Validate all required fields (date, time, GLN, EAN/GTIN, etc.).**
4. **Look up warehouse and product data using keys derived from GLN and EAN.**
5. **Update (or create) temporary transaction record for MalProff.**
6. **Call inventory update routine.**
    - If any error occurs, log the error, update error logs, and email notifications.
7. **Finish processing and clean up.**

---

## For New Developers

- **Business Context**: This program transforms and validates external inventory transactions before applying them to the ERP, ensuring correctness and full auditing/tracing of exceptions.
- **Maintenance Tips**:
    - When adding new fields, update the parsing/validation sections.
    - Modify email logic in the `feil` subroutine if notification requirements change.
    - Most lookups and business keys are built using company, warehouse, product, and sequence fields.
- **Error tracing** is centralized and robust—new types of errors can be integrated by following the existing patterns.
- **Database dependencies**: Ensure all referenced physical/logical files are present and correctly mapped to the current data model before running/testing.
```
