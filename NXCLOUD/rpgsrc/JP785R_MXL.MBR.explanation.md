# Documentation: Program JP785R (MGB System)

## Overview

**Program:** JP785R  
**System:** MGB  
**Purpose:**  
This program is responsible for copying business condition data from various "import" physical files (e.g., JBEUPF, JRAUPF, JPFUPF, JLEUPF, etc.) to their corresponding "export/active" files (e.g., JBETPF, JRAEPF, JPFAPF, JLEVPF, etc.). The process involves clearing out the target tables, copying data, and reporting progress and errors.

**Domain Context:**  
The program is part of a pricing/conditions subsystem, handling batch import and activation of business rules, discounts, surcharges, supplier data, logistics, campaign headers/lines, and freight information.

---

## Structure & Flow

### 1. Initialization

- **INZSR Subroutine:**  
  - Initializes variables.
  - Determines if freight data should be processed by calling external program `CO402R`. If so, sets the `u_frak` switch.

### 2. Logging and Progress Reporting

- Uses external programs (`AB700R`, `AB705R`, `AB710R`) to log and report the progress and status of the import/copy jobs.  
  - `AB700R`: Start of import
  - `AB705R`: Progress updates
  - `AB710R`: Job completion

### 3. Main Processing Steps

#### a. Cleanup (Rydd)

- **rydd_beurau**: Deletes all records from condition, discount, surcharge, supplier, and (optionally) freight target tables.
- **rydd_logi**: Deletes all records from logistics header and line tables.
- **rydd_kahkav**: Deletes campaign header/line records for a specific responsible person ('SOLVE').

#### b. Copy/Import (Skriv)

- **skriv_beurau**:  
  - Inserts data from import files (`JBEUPF`, `JRAUPF`, `JPFUPF`, `JLEUPF`, `JFBUPF`) into their respective target files.
  - Handles freight data if enabled.
  - Contains (commented-out) logic for updating calculation codes in supplier table based on surcharge data.

- **skriv_logi**:  
  - Copies logistics header and line data from `JVK1PF` and `JVK2PF` to `JVKHPF` and `JVKLPF`.

- **skriv_kahkav**:  
  - Copies campaign headers (`MGK1ST` → `JKAHPF`) and lines (`MGK2ST` → `JKAVPF`) using SQL cursors for row-by-row processing.

#### c. Error Handling

- **feil**:  
  - Logs SQL errors via `AB705R` and sets the error state.

---

## Key Data Flows

| Source File | Target File | Description                    |
|-------------|-------------|--------------------------------|
| JBEUPF      | JBETPF      | Conditions (Betingelse)        |
| JRAUPF      | JRAEPF      | Discount elements (Rabatt)     |
| JPFUPF      | JPFAPF      | Surcharges (Påslag)            |
| JLEUPF      | JLEVPF      | Supplier data (Leverandør)     |
| JFBUPF      | JFBEPF      | Freight data (Frakt)           |
| JVK1PF      | JVKHPF      | Logistics header               |
| JVK2PF      | JVKLPF      | Logistics lines                |
| MGK1ST      | JKAHPF      | Campaign header                |
| MGK2ST      | JKAVPF      | Campaign lines                 |

---

## Notable Design Decisions & Conventions

- **Batch Processing:**  
  The program is designed for batch runs, with clear separation between cleanup and copy phases to ensure no stale data remains in target tables.

- **SQL Integration:**  
  Uses embedded SQL for all data manipulation, which is crucial for performance and transactional integrity in bulk operations.

- **Progress Logging:**  
  Progress and errors are consistently reported using external programs, ensuring traceability and monitoring in operational environments.

- **Switchable Freight Processing:**  
  The inclusion of freight data processing is controlled by a dynamic switch (`u_frak`), determined by a configuration call to `CO402R`. This allows flexible operation depending on business requirements.

- **Campaign Data:**  
  Campaign header and line copy operations use explicit SQL cursors and fetch/insert loops, as opposed to bulk `INSERT ... SELECT`, likely due to the need for row-level control or data transformation (although in this code, it's a direct copy).

- **Error Handling:**  
  After each SQL operation, the code checks for errors and logs them immediately, aborting the current subroutine if issues are found.

- **Versioning and Change Log:**  
  The program header includes a detailed change log, documenting enhancements and extensions (e.g., addition of logistics, campaign, and freight processing).

---

## Interactions with Other Modules

- **External Programs:**
  - `AB700R`, `AB705R`, `AB710R`: Used for job logging and status reporting.
  - `CO402R`: Used to determine business configuration (whether freight data should be processed).

- **Data Sources:**
  - Various physical files, both for import (staging) and target (active/production) data.

---

## Business/Domain-Specific Logic

- **Data Staging:**  
  The system uses a two-stage approach: first, data is imported into staging files (prefixed with UPF), then activated by copying into production files (PF).

- **Selective Deletion:**  
  For campaign data, only records associated with the responsible person 'SOLVE' are deleted before new data is loaded, indicating user- or process-specific data isolation.

- **Freight Data:**  
  Freight processing is only performed if enabled in configuration, reflecting business rules that may change over time.

---

## Error Handling & Robustness

- All SQL operations are checked for errors using `sqlcod` and/or `sqlstt`.
- On error, the `feil` subroutine is invoked, which logs the error and aborts the current process.
- For campaign data, SQL cursors are used to ensure that all rows are processed, and proper cleanup is performed on cursor exhaustion.

---

## Summary Table: Subroutines

| Subroutine      | Purpose                                                     |
|-----------------|------------------------------------------------------------|
| rydd_beurau     | Delete all records from target condition, discount, etc.   |
| rydd_logi       | Delete all records from logistics header/line tables        |
| rydd_kahkav     | Delete campaign data for responsible 'SOLVE'                |
| skriv_beurau    | Copy data from import to target files (main business data)  |
| skriv_logi      | Copy logistics data from import to target files             |
| skriv_kahkav    | Copy campaign data from staging to production tables        |
| feil            | Log and handle SQL errors                                   |
| *inzsr          | Initialization, including freight processing switch         |

---

## Patterns & Conventions

- **File Naming:**  
  - UPF suffix: Import/staging files  
  - PF suffix: Production/active files  
  - Prefixes (JB, JR, JP, JL, JF, JV, JK, MG): Indicate business area (conditions, discount, surcharges, supplier, freight, logistics, campaign, etc.)

- **Release Management:**  
  - In-line change log with version numbers and comments, facilitating traceability.

- **SQL Error Codes:**  
  - Immediate error logging and status update on any non-zero SQL code.

---

## Special Notes

- **Commented Out Code:**  
  Some SQL update and commit statements are commented out, possibly reflecting deprecated logic or future enhancement placeholders.

- **Data Structure Definitions:**  
  Data structures are defined for campaign header/line processing, matching the physical file layouts.

- **No Transaction Control:**  
  There is no explicit transaction control (COMMIT/ROLLBACK) in the active code; this may rely on job-level settings or be managed externally.

---

## In Summary

JP785R is a batch utility that manages the lifecycle of business condition data, ensuring clean and consistent activation of new pricing, discount, surcharge, supplier, logistics, campaign, and freight data. It is robust, well-logged, and designed for operational reliability in a business-critical pricing subsystem, with modularity for future extension and clear separation of concerns.