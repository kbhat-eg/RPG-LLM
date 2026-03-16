# Business Rules: EDI Processing Inbound (LI Module)

**System:** ASOFAK
**Module Prefix:** LI
**Programs Analyzed:** LI100R, LI102R, LI106R, LI120R, LI200R, LI201R, LI300R, LI301R, LI500R, LI700R (NXKORR override where present)
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- Incoming EDI files must be split into message-type-specific staging tables before processing. LI700R reads raw EDI data from the `ED01` record format and routes messages to: `LWEOPF` (order confirmations – ORDRSP), `LWEPPF` (packing slips – DESADV), `LWEFPF` (invoices – INVOIC), or `LWEBPF` (purchase orders – ORDERS). Downstream programs (LI100R, LI102R, LI106R, LI200R, LI300R) operate on these staging tables, not on the raw EDI data.
- The EAN/GLN number in the EDI message must match a supplier record in `RLEVPF.RLEANL`. LI700R uses the GLN from the EDI header to identify the supplier. If no matching supplier is found in `RLEVPF` by EAN/GLN, the message cannot be associated with a supplier and processing may be incomplete or logged as an error.
- Purchase order headers must exist in `LOHEPF` for packing slip processing. LI500R checks `LOHEPF` for each order referenced in a packing slip. Orders not found in `LOHEPF` are skipped.
- The accumulator type `VAOTYPF.VAOAKK` must equal `2` for automatic warehouse entry (LI301R). All other accumulator types cause LI301R to exit without creating any inventory movement.
- Packing slip colli (package) records are validated in LI201R via access control program AX700R. If `w_tilg = '1'` (access denied) → LI201R exits immediately without displaying any colli data.
- The error log table `ELOGPF` must be available for write operations. LI700R's `*PSSR` error handler writes all runtime errors to `ELOGPF` before continuing to the next EDI record.

---

## 2. Validation Rules

### LI120R – EDI Inquiry Parameters
- Accounting year must not be 0. Year = 0 → `*in31` blocks the inquiry parameter save.
- Supplier must exist in `RLEVPF`. If the supplier number is entered but not found in `RLEVPF` → `*in32` blocks the save.
- Message type must be blank (all types), `'OBKR'` (order confirmations), or `'FAKT'` (invoices). Any other message type → `*in34` blocks the save.

### LI201R – EDI Packing Slip Colli View
- Access is controlled by AX700R at program entry. If AX700R returns `w_tilg = '1'` (no access) → LI201R exits immediately. No colli data is displayed.
- Colli records with `LIHPF.PLNUMM = 0` (zero packing slip number) → skipped; not displayed.
- Colli records with `LIHPF.PLBLIN = 0` (zero packing slip line) → skipped; not displayed.
- For each displayed colli, LI202R is called to retrieve and display the item lines for that colli.

### LI301R – Automatic Warehouse Entry of EDI Packing Slip
- If `VAOTYPF.VAOAKK <> 2` → program exits immediately. Only accumulator type 2 triggers automatic warehouse entry; all other configurations are excluded.
- `LOHEPF.LOKODE` is set to `2` (processing) at the start. If `LOKODE` is already `3` (picked/processed) → the packing slip is not processed again. This prevents double-entry of already-processed packing slips.
- LI302R is called for the actual warehouse entry; if LI302R encounters errors, they are propagated back to LI301R.

### LI500R – Purchase Orders Linked to Packing Slip
- Order lines with `PLNUMM = 0` → skipped.
- Order lines with `PLBLIN = 0` → skipped.
- For each non-zero order line, `LOHEPF` is checked for the order header. If the order header is not found → that order line is skipped (no error raised).
- LO505R is called for detailed order line information. LO505R results drive the display of order line details.

### LI700R – EDI Message Type Splitter
- Message type is identified from the EDI `ED01` record:
  - Positions 19–24: `'ORDRSP'` → write to `LWEOPF`
  - `'DESADV'` → write to `LWEPPF`
  - `'INVOIC'` → write to `LWEFPF`
  - Positions 15–20: `'ORDERS'` → write to `LWEBPF`
- If the message type does not match any of these four patterns → the record is not written to any staging table. Unknown message types are effectively discarded (with error logging via `*PSSR` if the discard causes a downstream error).
- EAN/GLN number is extracted from the EDI header and used to look up the supplier in `RLEVPF.RLEANL`. If no match → the message is processed without a supplier association, which may cause downstream validation failures.
- `*PSSR` error subroutine: any runtime error during LI700R processing writes an error record to `ELOGPF` and continues to the next EDI record. The final status parameter `p_stat` is set to `'E'` if any error occurred during the entire run.
- After splitting, the following subprograms are called conditionally:
  - Always: LI701R (process OBKR/order confirmations), LI702R (process PAKK/packing slips), LI703R (process FAKT/invoices), LI704R (process BEST/purchase orders)
  - Switch `LI700_702` active → calls LI721R (automatic order confirmation updates)
  - Switch `LI700_703` active → calls LI723R (automatic invoice updates)
  - Switch `POG_TRANS` active → calls LI753R (payment transaction processing)

### LI200R – Processing Incoming EDI Packing Slips (partial analysis)
- Validates packing slip status before allowing processing. A packing slip in a terminal status cannot be re-processed.
- Supports manual goods receipt selection — the user can select which packing slips to receive into the warehouse.

### LI300R – Picks Packing Slips for Approval (partial analysis)
- Handles quantity discrepancy flagging: when the received quantity differs from the ordered quantity, the discrepancy is flagged for approval.
- Price discrepancies are similarly flagged; the flagged lines require user action before approval.

---

## 3. Configuration and Authorization Rules

- System switch `LI700_702` (read from `CO402R`) controls whether LI721R is called after EDI splitting. LI721R automatically updates purchase order confirmations without user interaction. When the switch is off, order confirmations require manual review and approval.
- System switch `LI700_703` controls whether LI723R is called for automatic invoice processing. When off, EDI invoices require manual matching and approval.
- System switch `POG_TRANS` controls whether payment transaction processing (LI753R) is triggered as part of the EDI split. This handles EDI-based payment notifications.
- AX700R provides access control for LI201R (colli view). The access check `w_tilg = '1'` is a hard block with no user-facing message — the program simply exits. Users who need access to packing slip colli data must be granted the appropriate authority in the AX700R access table.
- The `VAOTYPF.VAOAKK = 2` requirement in LI301R is a configuration-level setting on the order type. Changing the accumulator type for an order type to 2 enables automatic warehouse entry for all orders of that type. This is a system-level configuration decision.

---

## 4. Financial / Transactional Rules

### EDI Message Routing (LI700R)
- ORDRSP (order response/confirmation) messages are stored in `LWEOPF` and processed by LI701R / optionally LI721R.
- DESADV (despatch advice/packing slip) messages are stored in `LWEPPF` and processed by LI702R.
- INVOIC (invoice) messages are stored in `LWEFPF` and processed by LI703R / optionally LI723R.
- ORDERS (purchase order) messages are stored in `LWEBPF` and processed by LI704R.
- Each message type flows through its own processing chain. Errors in one message type do not prevent other message types in the same EDI file from being processed (error recovery via `*PSSR`).

### Automatic Warehouse Entry (LI301R)
- Automatic warehouse entry creates inventory receipts without user confirmation. This is only activated when `VAOAKK = 2` (the order type is configured for automatic receipt).
- The `LOKODE = 3` (already picked) check prevents duplicate inventory postings if LI301R is called more than once for the same packing slip.
- `LOKODE = 2` (processing) is set at the start of LI301R. If the program fails mid-processing, `LOKODE` remains at 2, preventing automatic re-processing. Manual intervention is required to reset the status.

### Quantity and Price Discrepancies (LI300R)
- Quantity discrepancies (received ≠ ordered) are flagged on packing slip lines. The discrepancy remains visible until the user explicitly approves or rejects the difference.
- Price discrepancies (invoiced price ≠ purchase order price) are similarly flagged. AP matching cannot complete for invoice lines with unresolved price discrepancies.

---

## 5. Status and Lifecycle Rules

- Packing slips in `LOHEPF` transition through status codes managed by `LOHEPF.LOKODE`:
  - `LOKODE = 2`: In processing by LI301R
  - `LOKODE = 3`: Fully picked/processed — LI301R will not reprocess
  - Other values: Available for processing
- EDI messages move through the staging tables (`LWEOPF`, `LWEPPF`, `LWEFPF`, `LWEBPF`) as they are processed. Processed records remain in the staging table for inquiry and audit purposes.
- The final status `p_stat = 'E'` from LI700R signals to the calling job that at least one EDI record encountered an error. The calling job can use this status to trigger alerts or reprocessing workflows.
- Errors logged to `ELOGPF` by LI700R's `*PSSR` handler remain until explicitly purged. The error log provides an audit trail of all EDI processing failures.

---

## 6. Special Conditions

- **`*PSSR` error recovery in LI700R:** The `*PSSR` subroutine is the RPG program-exception handler. In LI700R, it is used to catch unexpected runtime errors (e.g., conversion errors on malformed EDI data, locked records) and continue processing the next EDI record rather than aborting the entire run. This "fail-forward" design ensures that one bad record does not block processing of subsequent valid records. All caught errors are logged to `ELOGPF`.
- **EAN/GLN lookup criticality:** The GLN number is the only mechanism for associating an incoming EDI message with a supplier in `RLEVPF`. If a supplier's GLN has changed (supplier mergers, carrier changes) and `RLEVPF.RLEANL` has not been updated, all EDI messages from that supplier will be misidentified or unidentified. There is no fallback supplier lookup method in LI700R.
- **Colli skip rules in LI201R and LI500R:** Records with `PLNUMM = 0` or `PLBLIN = 0` are silently skipped. These represent header-level records or placeholder lines that have no physical package or line content. The skip is intentional and not an error condition.
- **SSCC codes:** DESADV (packing slip) messages in the EDI standard use SSCC (Serial Shipping Container Code) to identify individual packages. LI201R displays colli-level detail using SSCC-linked records. SSCC uniqueness per packing slip is enforced by the EDI standard, not by the RPG programs.
- **Automatic processing switches:** The three switches (`LI700_702`, `LI700_703`, `POG_TRANS`) allow the system to be configured for fully automated EDI processing or manual review. Disabling all three switches puts the system in manual-approval mode for all EDI message types. Enabling all three allows straight-through processing without human intervention, which increases throughput but reduces oversight.
- **LI100R, LI102R, LI106R size:** These three programs are among the largest in the system (each exceeds 40,000 tokens). They handle the core OBKR (order confirmation), PAKK (packing slip), and FAKT (invoice) processing respectively. Detailed blocking rules for these programs require direct code analysis beyond what was available; the rules documented here are based on the patterns established by the programs that were fully analyzed.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Subprogram | Purpose | Blocking Effect |
|--------|-----------|---------|-----------------|
| LI700R | LI701R | Process ORDRSP (order confirmations) | Errors logged; processing continues |
| LI700R | LI702R | Process DESADV (packing slips) | Errors logged; processing continues |
| LI700R | LI703R | Process INVOIC (invoices) | Errors logged; processing continues |
| LI700R | LI704R | Process ORDERS (purchase orders) | Errors logged; processing continues |
| LI700R | LI721R | Auto-update order confirmations (switch LI700_702) | Only called if switch active |
| LI700R | LI723R | Auto-update invoices (switch LI700_703) | Only called if switch active |
| LI700R | LI753R | Payment transaction processing (switch POG_TRANS) | Only called if switch active |
| LI201R | AX700R | Access control for colli view | w_tilg='1' → immediate exit |
| LI201R | LI202R | Retrieve item lines for colli | Drives colli detail display |
| LI301R | LI302R | Actual warehouse entry | Errors propagated to LI301R |
| LI500R | LO505R | Order line detail retrieval | Missing order → line skipped |

---

## 8. Reference Tables

| Table | Description | Key Fields Used |
|-------|------------|-----------------|
| `RLEVPF` | Supplier master (incl. EAN/GLN) | `RLFIRM`, `RLLEV`, `RLEANL` |
| `LOHEPF` | Purchase order / packing slip headers | `LOFIRM`, `LOORDRE`, `LOKODE` |
| `LWEOPF` | EDI staging – order confirmations (ORDRSP) | `LWFIRM`, `LWMSG` |
| `LWEPPF` | EDI staging – packing slips (DESADV) | `LWFIRM`, `LWMSG` |
| `LWEFPF` | EDI staging – invoices (INVOIC) | `LWFIRM`, `LWMSG` |
| `LWEBPF` | EDI staging – purchase orders (ORDERS) | `LWFIRM`, `LWMSG` |
| `LIHPF` | Packing slip colli (package) records | `PLFIRM`, `PLNUMM`, `PLBLIN` |
| `VAOTYPF` | Order type definitions | `VAOFIRM`, `VAOOTYP`, `VAOAKK` |
| `ELOGPF` | EDI error log | `ELFIRM`, `ELDAT`, `ELTIM` |
| `CO402R` | System switches | `LI700_702`, `LI700_703`, `POG_TRANS` |
