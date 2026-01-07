# RFI15R – Transaction Preparation from Orders/Inventory/Purchasing

## Overview

**RFI15R** is a core batch program in the ASOKON system that prepares and transfers financial transaction records from the inventory/order/purchasing subledgers into the general ledger staging files. The program handles grouping, summarization, and enrichment of transaction data, ensuring proper bundling (bunt) and periodization, and updates both detail (transaction) and summary (bundle) files. It also manages period deviations as defined in the code register.

This program is a modified copy of `RFA15R` with the following key changes:
- All references from `FOVF..` to `LOVF..` (file and field names).
- Added transfer of VAT base amount and supplier invoice number.
- Enhanced batch total calculations.
- Only transactions for the current company are processed.
- Integration with work order/project/activity tables (from version 6.20).
- Adaptations for expanded number ranges (from version 6.30).

## Key Files and Their Roles

- **LOVFL1**: Input transaction file (inventory/order/purchase transactions).
- **AFIRL1**: Company master file (company information).
- **RAA3PF**: Code register (used for period deviations).
- **RBUNLU**: Bundle summary file (stores batch totals).
- **RTRAPF**: Transaction staging file (detail records for GL transfer).
- **RP4OL1**: Work order/project/activity cross-reference (enriches transactions with project data).

## Main Processing Flow

1. **Initialization (`*INZSR`)**
   - Receives member (partition) identifier as parameter.
   - Initializes key lists for file access.
   - Sets up variables for period and batch processing.

2. **Main Loop**
   - Reads transactions from `LOVFL1` for the specified company and period.
   - For each record:
     - Detects breaks on company or period (using `nytt_firma` and `ny_periode` subroutines).
     - Redacts and enriches transaction details.
     - Writes detail records to `RTRAPF`.
     - Deletes processed input records from `LOVFL1`.
   - On company/period break or end of file, writes batch summary to `RBUNLU`.
   - At end of processing, ensures the last batch summary is written.

## Key Business Logic and Domain-Specific Details

### Period Handling and Deviations

- The program supports non-standard accounting periods (e.g., 13-period years, irregular periods) by consulting the `RAA3PF` code register.
- The period array (`per(13)`) is loaded from the code register and used to translate or adjust periods when writing summaries.

### Bundle Number Assignment

- For each new company/period, the program assigns the next available bundle number by reading `RBUNLU` in descending order and incrementing the highest found.
- Bundle numbers reset per company and period.

### Project/Work Order Enrichment

- For each transaction, if a work order is present, the program looks up related project and activity codes in `RP4OL1` and enriches the transaction record.
- This supports later project-based reporting and cost tracking.

### VAT and Supplier Invoice Handling

- Transfers VAT base amounts and supplier invoice numbers from the source transaction to the GL staging file, supporting accurate VAT reporting and reconciliation.

### Only Current Company Processing

- The program processes only transactions for the specified (current) company, as enforced by comparing file/company fields.

### Batch and Line Summarization

- Accumulates batch totals (debits, credits, total amounts) as transactions are processed.
- Writes these totals to `RBUNLU` at each company/period break and at end of processing.

## Notable Patterns, Conventions, and Design Decisions

- **Subroutine Structure**: The code uses classic RPG subroutines (`begsr`/`endsr`) for modularization: company break, period break, transaction writing, and batch summary writing.
- **File Renaming**: The use of `RENAME` in F-specs allows the program to work with files that have different field names but similar structures (e.g., `lovfpfr:lovfl1r`).
- **Key Lists**: Key lists (`klist`) are defined for all indexed file access, supporting dynamic lookups and maintainability.
- **Error Handling**: Status indicators (e.g., `*IN90`, `*IN91`) are checked after file operations to handle end-of-file and record-not-found conditions gracefully.
- **Conditional Logic by Version**: Comments and conditional code (e.g., `6.10`, `6.20`, `6.21`, `6.30`) indicate enhancements and bug fixes across versions, providing traceability for changes.
- **Data Deletion**: Processed records are deleted from the input file to prevent re-processing.
- **Array Usage**: The period array is loaded from the code register file, supporting flexible handling of non-standard periods.

## Relationships with Other Modules

- **Upstream**: Receives data from subledger processes (orders, inventory, purchasing).
- **Downstream**: Feeds the GL transfer process via `RTRAPF` (detail) and `RBUNLU` (batch summary).
- **Master Data**: Consults company master (`AFIRL1`) and code register (`RAA3PF`) for validation and enrichment.
- **Project/Work Order Integration**: Enriches transactions with project and activity codes for project accounting.

## Special Considerations

- **Number Expansion**: From version 6.30, the program was reviewed and updated to support expanded number ranges (e.g., longer keys, larger bundle numbers).
- **Localization**: Some field and comment text is in Norwegian, reflecting the business context and user base.
- **Historical Notes**: Extensive change history is maintained in the header comments, documenting both business and technical changes.

## Summary Table: Subroutines

| Subroutine    | Purpose                                                        |
|---------------|---------------------------------------------------------------|
| `nytt_firma`  | Loads company master and code register for new company.        |
| `ny_periode`  | Assigns next bundle number for company/period, resets totals.  |
| `opdat_rtra`  | Writes a transaction detail record to `RTRAPF`.                |
| `opdat_rbun`  | Writes a batch summary record to `RBUNLU`, resets batch totals.|
| `*INZSR`      | Program initialization, parameter handling, key setup.         |

## Key Data Flows

1. **Input**: `LOVFL1` (inventory/order/purchasing transactions)
2. **Enrichment**: `AFIRL1` (company), `RAA3PF` (periods), `RP4OL1` (projects)
3. **Output**: `RTRAPF` (transaction details), `RBUNLU` (batch summaries)
4. **Cleanup**: Deletes processed records from `LOVFL1`

## Conclusion

RFI15R is a robust transaction preparation module that bridges subledger activity into the general ledger, supporting complex requirements such as period deviations, bundle tracking, project accounting, and VAT reconciliation. It is a critical part of the financial data flow, and understanding its structure and logic is essential for maintaining and extending the ASOKON financial system.