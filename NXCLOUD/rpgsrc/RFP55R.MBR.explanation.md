# Program Overview

This ILE RPG program is designed to process accounting transaction records scanned from invoices and prepare them for posting. It reads from a "scanning" source file and creates/updates transaction and batch (bunt) records. The program appears to be tailored for a Norwegian installation (as evidenced by comments and field names). The business logic includes handling of special periods (such as locked or blocked periods), batch sums, and error/message handling.

---

## Key Components

### Control and File Specification

- **`h decedit(',') datedit(*dmy.)`**  
  Sets decimal point and date format.
- **`h option(*nodebugio)`**  
  Enhances performance by disabling debug I/O.

- **Files (`F-Specs`)**:
    - `fremsl1`: Transaction records from scanning, input file (with record prefix for field names).
    - `fafirl1`: Company register, input file.
    - `fraa3pf`, `fraa1pf`: Status codes/periods (input).
    - `frbunlu`: Batch records, update file.
    - `frtrapf`: Transaction posting file, output.

---

### Data and Work Fields

- **Local Data and Arrays**:
    - `per(13)`: Array for period information.
    - Various fields for batch, company, period, sums, error signals, keys, and flags.
- **Special Variables**:
    - `w_kv_bilnr`: Last processed document number for duplicate checking.
    - `w_sp_per`: Last blocked period for posting (“sperret periode”).

---

### Input Record Format

- **`/TITLE`** and extensive header comments:  
    - Origins, maintenance history, and field change tracking.

---

### Main Routine Logic

#### a) Main Loop

```rpg
c     read      remsl1                                 90
c     dow       *in90 = *off
   // ...
   c    read      remsl1                                 90
c     enddo
```

- **Reads records from the scanning input file** (`remsl1`) until end-of-file (`*in90`).

#### b) Company & Period Control

- **Only processes the current company**
    - Skips records if `xtfirm <> l_firm`.
- **Period Calculation for Blocked Periods**
    - If record has an unset period, deduces period/year from document date.
    - Adjusts period if blocked (sperret).
    - If period is blocked and not approved, proposes moving to next open period or prompts via message.

#### c) Batch and Period Breaks

- **Checks for "breaks"** (value changes) in company or period.
    - When batch or period changes, posts (writes) batch summary records and resets accumulators.
    - Subprocedures handle initializations for new company or period.

#### d) Writing Transactions

- For each record, writes a transaction posting record (`rtrapf`).
- Accumulates batch and period totals for later batch record write/update.

#### e) Deleting Source Record

- After processing, deletes input record (`delete remsl1r`).

#### f) Final Writes

- After loop, if any remaining batch, writes final batch record.

---

### Subroutines

#### *nytt_firma*: New Company

- Loads company register (`afirl1`) and exceptional periods (`raa3pf`).

#### *ny_periode*: New Period

- Locates the next available batch number (`w_bunt`) for the period, using `rbunlu`.

#### *opdat_rtra*: Write Transaction

- Adjusts negative amounts to positive (for both base and currency).
- If amount is zero and currency code is blank, transfers value from currency amount.
- Writes transaction fields to output file.

#### *opdat_rbun*: Write Batch Record

- Writes summary info for current batch (firm/period, sums, etc.).
- Handles period mapping for exceptional periods.
- Resets batch accumulators.

#### *pssr*: Error Routine

- Called on unexpected errors.
- Logs error and aborts program.

#### *inzsr*: Initialization

- Sets up work variables, batch, period, and company info.
- Reads special blocking period (`raa1pf`), used for later posting checks.

---

### Status and Message Handling

- **Blocked periods**:  
    - If posting is attempted in a blocked period, checks if it's a duplicate.  
    - Calls external program (`AA007R`) to prompt/confirm new period before continuing.
- **No records**:  
    - If the input is empty, sets message variables and calls `AB705R` to log a message.

---

## Key Points for Onboarding

- **Special periods and business rules** are handled in subroutines and with extensive flag checking.
- **All period or batch changes** trigger "breaks" where summary data is written and accumulators are reset.
- **Amounts are normalized** to positive numbers.
- **Input is deleted after processing** to avoid duplicate postings.
- **External calls** are made for handling blocked periods and logging errors/messages.

---

### Example Workflow

1. **Starts** and reads the first scanning record.
2. Checks for company/period change (batch break).
3. Adjusts period/year if blocked.
4. Writes transaction detail.
5. Deletes the processed scanning record.
6. Repeats until end-of-file.
7. After the last record, writes final batch summary and ends.

---

## File Naming Legend

- **remsl1**: Scanned transactions (input)
- **afirl1**: Company master data (input)
- **raa3pf / raa1pf**: Period/status info (input)
- **rbunlu**: Batch summary (output/update)
- **rtrapf**: Transaction postings (output)

---

## Special Topics

- **Sperret periode (blocked period)**:
    - Posting to blocked periods is not allowed unless approved.
    - System can prompt user (via program calls) to transfer to the next open period.
- **Buntnr (batch number)**:  
    - System automatically finds and assigns the next available batch number per period/company.
- **Field mapping and prefixes**:  
    - `prefix(x:1)` is used on input files to avoid naming conflicts.

---

## Summary

This program is a batch-processing utility for scanned accounting transactions. It collates and posts transactions in the correct period, ensuring business rules (like period locks) are followed. The code is organized into routines handling company/period breaks, transaction writing, and batch summaries. Error and status messaging are robust, supporting both automated and user-interactive correction of period issues.

For onboarding, understanding the flow of data—from scanning input, through batch/period control, to output posting, and the handling of exceptional (blocked) periods—is crucial. The modular routine structure supports maintainability and business rule evolution.