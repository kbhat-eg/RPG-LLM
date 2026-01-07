# RPG Program RFA15R - Explanation

## Overview

This RPG program, named **RFA15R**, processes transactional records from the order/invoicing system for transfer into the finance/accounting system. It groups transactions into batches ("bunt" or "bundles") per company and period, populates posting records, updates batch summaries, and ensures that postings are balanced and tracked by period/fiscal year according to company settings. The program also contains logic to match prepayments to open items in customer accounts.

The code is written in fixed-format RPG IV ('traditional RPG') and operates on IBM i (AS/400, iSeries) databases/files.

---

## Head Section

**Header (H-spec):**
- `decedit(',')` and `dateedit(*dmy.)` set decimal and date formatting.
- `option(*nodebugio)` disables debug I/O for performance.

**Title/Description:**
- The program adapts transaction records from the order/invoice system.
- Handles special cases for fiscal periods based on code tables.
- Noted changes for versions (annotated in margins, e.g., `6.10`, `6.20`).
- Identifies file/field renaming (FOVF to BOVF, etc.).
- Tracks version control and change history.

---

## File Definitions

- **Input Files:**
  - `bovfl1` (transaction file; renamed from `bovfpfr`)
  - `afirl1` (company master; renamed from `afirpfr`)
  - `raa3pf` (status/codes for fiscal periods)
  - `rktrld` (customer transaction lookup for reconciling prepayments, only v7.00+)

- **Output Files:**
  - `rbunlu` (batch header, renamed from `rbunpfr`)
  - `rtrapf` (posting/transaction output)

---

## Arrays and Variables

- **Period Array:**
  - `per` array holds period ending months, per company.

- **Key Variables:**
  - Used to compose file keys for SETLL/CHAIN/READ operations, e.g., `rbunlu_nivo`, `rbunlu_memb`, etc.

- **Work Variables:**
  - Track current firm, period, batch, line number, sums, etc.

- **Save Variables:**
  - `s_firm`, `s_peri`: hold last processed firm/period to detect breaks.

---

## Input Specification (I-spec)

- **raa3pf**: Maps fields `ra3p01`..`ra3p13` into the `per()` array to load period end info for the company.

---

## Main Routine Logic

### 1. **Initialization**
- Sets `w_firm` to `l_firm` (input firm).
- Sets start period to zero.
- Resets pointer and reads first transaction.

### 2. **Main Loop**
- For each transaction:
  - If end of file, exit.
  - If firm number exceeds the current ("innestående firma"), exit.
  - On firm or period break: run appropriate subroutine (`nytt_firma` or `ny_periode`).
  - **Delete** previous record after processing (effectively moves records).
  - For each transaction:
    - Edit fiscal year/period data for output.
    - Call `opdat_rtra` to write posting record and update sums.
    - On batch (firm/period) break, call `opdat_rbun` to write batch summary.
    - Delete processed transaction record.

### 3. **On End**
- After finishing, if any records remain, update the last batch summary.
- Set LR indicator to end program.

---

## Subroutines

### 1. **nytt_firma**
- Loads company master record for new firm.
- Loads fiscal period status/codes.

### 2. **ny_periode**
- Finds next available batch number for new period:
  - Uses `readp` to get previous record for batch control.
  - If no previous batch, start at 1.
  - If the member has changed or no record found, also start at 1.
  - Otherwise, increment previous batch.

### 3. **opdat_rtra**
- Calculates posting sums (debit, credit, total, per line).
- Populates all transaction/posting fields (firm, batch, amounts, period, references).
- Handles foreign currency, references, and other attributes.
- **Prepayment matching logic (v7.00+):**
  - Tries to pair prepayments (bilagskode 62 or 6) with open items or with the same amount.
  - If found, sets references and status for reconciliation.
- Writes posting record to output file.

### 4. **opdat_rbun**
- Summarizes batch (bunt) totals.
- Handles special period logic (for companies using non-standard fiscal periods).
- Writes batch summary to output file.
- Resets batch totals.

### 5. **\*inzsr**
- Program initialization on entry (set up keys, initial values, etc.).

---

## Key Lists

Key lists (KLIST/KFLD) are defined for all main files to streamline CHAIN, SETLL, and READP/READE operations.

---

## Comments & Special Handling

- Extensive Norwegian-language comments; "bunt" = batch, "firma" = company, "periode" = period.
- Handles period breaks, year transitions.
- Supports member/environment processing (`w_memb`).
- Line numbers and version-specific annotations (e.g., "6.10", "7.00").

---

## Typical Workflow

1. **Read each transaction record** for the selected company/period.
2. **On company or period change:**
   - Fetch company config and fiscal period info.
   - Find the next available batch number.
3. **Process and transfer each transaction:**
   - Write to posting file (`rtrapf`).
   - Delete the original after processing.
   - Tally batch totals.
4. **On batch end (company/period break):**
   - Write batch summary (`rbunlu`).
   - Reset batch counters.
5. **On program end:**
   - Check for any unwritten batch summaries.
   - End cleanly.

---

## Version Extensions

- **6.10**: Only processes the current firm.
- **6.20**: Project field (`proj`) is now alphanumeric.
- **6.21**: Corrects bug with batch number assignment using member in key.
- **7.00+**: Adds prepayment matching for customer transactions (reskontro reconciliation).

---

## Key Takeaways

- **Batch-Oriented:** Batches transactions per company and period.
- **Flexible Fiscal Period Handling:** Uses code tables for "off-standard" fiscal years.
- **Robust Reference/Matching:** Tries to automatically link prepayments and open items.
- **Clean-up:** Deletes source records upon successful transfer.
- **Extensible:** Versioned and annotated for easy tracking of changes and extensions.

---

## Onboarding Notes

- **Be familiar with the company's batch and fiscal period routines.**
- **Understand Norwegian accounting terminology.**
- **Know the file structure and key layouts.**
- **If updating/maintaining, use the version notes to avoid regression.**
- **Test with companies using both standard and non-standard fiscal years.**