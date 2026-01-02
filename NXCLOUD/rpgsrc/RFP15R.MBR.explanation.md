# Program Overview

This RPG program (`RFP15R`) is designed to process accounting transaction records scanned from another system (Nexstep Archive) into the company's accounting system, ASOKON. It reads scanning input, transforms and validates data, manages accounting periods and company-specific logic, and posts transactions into the accounting files. The code is heavily commented, often in Norwegian, and maintains transaction logic compliant with accounting year, periods, and special cases (blocked periods, company-specific rules, etc.).

---
## High-Level Flow

1. **Initialization:** Sets up default variables, keys, and reads company-specific configurations.
2. **Main Loop:** Reads each transaction (remspfr/remsl1) and processes if the firm matches/active.
3. **Per-transaction logic:** 
    - Determines period and year, managing blocked (sperret) periods, and posts messages when posting to a closed period is attempted.
    - Posts transactions into the main file.
    - Handles breaks in firm or period, then updates summary/batch records accordingly.
4. **Subroutines:** Manage new firm logic, new period logic (generating new batch numbers), update transaction and batch files, and initialization.

---
## Key Sections and Explanations

### 1. File Declarations

```rpg
fremsl1    uf   e   k disk    rename(remspfr:remsl1r) prefix(x:1)
fafirl1    if   e   k disk    rename(afirpfr:afirl1r)
fraa3pf    if   e   k disk
fraa1pf    if   e   k disk
frbunlu    uf a e   k disk    rename(rbunpfr:rbunlur)
frtrapf    o    e   k disk
```
- Files represent transactions, firm registry, period status, batch file, and output.
- `prefix(x:1)` prefixes all fields from `remspfr` with `x`.

---

### 2. Variable Definitions

Includes working variables (e.g., current firm, working period/year, batch/line sums), arrays for periods, and variables for conversion rules.

---

### 3. Main Processing Loop

#### a. Read and Select Transactions

```rpg
c     read      remsl1                                 90
c     dow       *in90 = *off
```
- Reads transactions until end-of-file.

#### b. Company Filtering

```rpg
c     if xtfirm <> l_firm
c       goto nyles
c     endif
```
- Only process transactions for the current company.

#### c. Period and Year Calculation

Handles different situations:
- If period is zero, deduce period from date.
- If period is blocked, find first unblocked period.
- Sets flags (`w_p_f_comp`) and period/year variables.

#### d. Blocked (Sperret) Period Handling

When posting to a blocked period:
- Produces a warning message via external call (`call 'AA007R'`) and requests user action.
- If posting isn't accepted to blocked period, advances to next available period/year.

#### e. Batch and Firm Break Logic

On period, year, or firm change:
- Updates batch subtotals and writes batch summary (via `opdat_rbun` subroutine).
- On new firm, loads register/config (via `nytt_firma`).

#### f. Posting a Transaction

- Calls `opdat_rtra` subroutine, which:
    - Adjusts signage for negative amounts.
    - Performs per-line calculations.
    - Maps all relevant transaction fields (accounts, project, text, amounts, etc.).
    - Applies document code conversion rules if needed.

#### g. Deletes Input Record

```rpg
c     delete    remsl1r
```
- Deletes the transaction from input after processing.

---

### 4. Subroutines

#### a. `nytt_firma`

- Loads firm register (`afirl1`) and period configurations.
- Loads period exceptions (`raa3pf`) for the current firm.

#### b. `ny_periode`

- Determines the next available batch number (`buntnr`) for the current firm and period.

#### c. `opdat_rtra`

- Posts an individual transaction to the main accounting output file (`rtrapf`).
- Applies document code conversions as per the `l_blk` rules if needed.

#### d. `opdat_rbun`

- Writes batch summaries to the batch file (`rbunlu`), including totals and period info.
- Zeros out batch summary accumulators for new batch.

#### e. `*inzsr`

- Program initialization routine, sets up keys and loads configuration.
- Reads blocked period information from `raa1pf` if present, used to determine what periods are open for posting.
- Loads any document code conversion table into `l_blk`.

---

### 5. Document Code (Bilagskode) Conversion

Conversion of document codes is handled by extracting rules from a configuration field (`l_blk`), processed in `opdat_rtra`:
- For up to 4 possible conversion rules, checks if the current document code matches a 'from' code. If so, replaces with the corresponding 'to' code.

---

## Special Logics

- **Sperret (Blocked) Period:** Handles cases where a period is closed for posting. If user approves, can post; otherwise, finds the next valid period.
- **Negative Amounts:** All amounts should be positive for posting; negatives are inverted.
- **Batch Summaries:** When switching periods or firms, totals are written into a batch/summary file.

---

## Business Rules

- **Firm-Specific Logic:** Only processes transactions for the active company.
- **Accounting Periods:** Ensures transactions are posted to valid periods and handles exceptional/atypical period mappings.
- **Document Code Mapping:** Converts document codes as per firm-specific logic loaded into user data area.

---

## Extensibility

- Much of the configuration (blocked periods, document code mappings) comes from user data areas or database fields, so company-specific rules are handled dynamically without code changes.

---

## Comments

- The program uses classic ("fixed-form") RPG and not modern free-format.
- Field and variable names are largely Norwegian (e.g., "bilag" = voucher/document, "periode" = period, "firma" = company, "beløp" = amount).

---

## Summary

This is a robust and flexible transaction "preparation" program for scanned input, handling company-specific rules, period validation, and batch/transaction writing, with a strong focus on extensibility and correctness for financial compliance. Its logic ensures that only valid transactions (both in terms of company and period), with properly adjusted amounts and codes, are posted to the main accounting register.