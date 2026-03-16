# Tele-Payments / Electronic Banking — Business Rules

## Introduction

The GL (electronic banking) module manages the complete lifecycle of electronic payment processing on the IBM i ERP system. It covers the creation and maintenance of payment proposals drawn from the AP transaction register, the transfer of those proposals to bank communication files in various formats (DIRREM, ISO 20022 pain.001 XML, proprietary AU/remittance formats), the reception and matching of bank return files, and the posting of settled transactions to the general ledger. Configuration of bank connection parameters (bank code, communication type, IFS path, payroll type) is maintained in a fixed-data register and is a hard prerequisite for all payment runs. Multiple programs enforce concurrency controls and duplicate-transfer guards to prevent double payment.

---

## Prerequisites / Master Data Requirements

- A record must exist in GFASPF (fixed data for electronic payment mediation) for the current company before any payment transfer or proposal print can proceed. If GFASPF is not found, GL200R returns sentinel value `'XX'` in the bank-connection parameter and GL103R returns a blank payroll-type; both conditions abort downstream processing.
- GABFPF (payment proposal) must contain at least one proposal line for the selected proposal number before GL104R (maintenance) or GL105R (transfer) can operate.
- Suppliers in RLEVPF must not have RLEVPF.RLREMI = `'J'` (remittance exclusion flag). Suppliers with this flag are silently excluded from proposal extraction in GL100R.
- AP transactions in RLTRPF must carry attestation status RLTRPF.ROATTS = `'1'` or `'J'` (approved/attested) to be eligible for inclusion in a payment proposal.
- GAVTPF (agreement register) must hold a valid bank agreement record for the supplier/bank combination to support electronic payment mediation.
- The company record in AFIRPF must exist; AFIRPF is read by GL105R to populate sender information in the bank file header.

---

## Validation Rules

### GL120R — Fixed Data Maintenance (GFASPF)

All of the following conditions block saving of the GFASPF record (subroutine `xa1tst`, indicators *in30–*in37):

| Condition | Indicator | Blocking Message / Effect |
|---|---|---|
| GFASPF.GEFIN = 0 (organization number blank/zero) | *in35 = ON | Block — organization number required |
| GFASPF.GEFEFS blank (sequence number type) | *in30 = ON | Block — sequence number type required |
| GFASPF.GEFBAN blank (bank code) | *in31 = ON | Block — bank code required |
| GFASPF.GEFKOM blank (communication type) | *in32 = ON | Block — communication type required |
| GFASPF.GEFKOM = `'IFS'` AND GFASPF.GEFIFS blank | *in37 = ON | Block — IFS path required when IFS communication |
| GFASPF.GEFSBF blank (proposal sort field) | *in33 = ON | Block — proposal sort required |
| GFASPF.GEFSAM blank (merge-to-collection-file flag) | *in34 = ON | Block — collection-file merge setting required |
| GFASPF.GEFLON ≠ `'3'` (payroll type) | *in36 = ON | Block — payroll type must be `'3'` |

### GL100R — Payment Proposal Extraction

- Supplier with RLEVPF.RLREMI = `'J'`: excluded from proposal (no error, silently skipped).
- AP transaction with RLTRPF.ROATTS not in (`'1'`, `'J'`): excluded from proposal.
- Due-date and discount calculations are performed per payment terms; transactions that do not fall within the selected date window are excluded.

### GL101R — Proposal Renumbering

No blocking validation. Pure maintenance utility.

### GL102R — Payment Proposal Print

- Supplier address validation: missing address fields produce warning lines on the report but do not abort printing.
- SWIFT code versus account-number requirement: if the bank country requires SWIFT, a missing SWIFT code is flagged.
- Norwegian suppliers (country code NO) without a valid postal code are flagged.

### GL104R — Payment Proposal Maintenance

- If **any** line in GABFPF for the current proposal has GABFPF.GASTAT = `'G'` (transferred to bank): indicator *in38 is set ON, locking the entire proposal against editing. All input fields are protected; no changes are permitted.
- Lines with GABFPF.GAREKO = `'F'` (error return from bank) are highlighted for user action.

### GL105R — Transfer to Remittance Files

- If GABFPF.GASTAT = `'G'` on any line: transfer is blocked (proposal already sent).
- If another bank connection's data is already present in the collection file (wpfore = `'1'`): transfer is blocked to prevent merging incompatible bank formats.
- GFASPF must exist for the current company (read at program start); missing record aborts the transfer.

### GL110R — Receipt Processing (Return from Bank)

- Return file is matched against GABFPF by proposal/line; unmatched records are logged as errors.
- Duplicate return file detection is handled by GL210R (see below).

### GL115R — Print Receipt List and Post

- Calls GL125R to delete the payroll proposal when GABFPF.GAREKO = `'1'` (1st-pass return).

### GL210R — Duplicate Return File Check

- Reads GATRPF (bank transfer register) for BETFOR23 and BETFOR04 records.
- For each record, chains to GUBFPF (payment proposal) by firm/proposal/line.
- If the GUBFPF record is **not found**: sets wxfeil = `'F'` (error flag returned to caller), blocking further receipt processing for that file.

---

## Configuration and Authorization Rules

- The GFASPF record is company-scoped (keyed by firm number from LDA position 944–946). One configuration record per company is the supported model.
- GFASPF.GEFBAN (bank code): returned by GL200R to calling programs; value `'XX'` is a sentinel indicating missing configuration and must be treated as an error by callers.
- GFASPF.GEFKOM = `'IFS'` requires GFASPF.GEFIFS to be populated with a valid IFS path. GL300R uses this path to write the ISO 20022 XML payment file.
- GFASPF.GEFLON must equal `'3'` for payroll processing; any other value blocks the fixed-data save in GL120R.
- GFASPF.GEFEFS (sequence number type) controls how the sequence number is assigned to the bank file; must be populated.
- Company change (GL400R) performs a mass update of the firm number across GLONPF, GTRTPF, GABFPF, GULFPF, GUL1PF, GUL2PF, GURFPF, GUR1PF, GUR2PF, GURMPF. This is a destructive bulk operation with no blocking logic; it should only be executed under controlled conditions.

---

## Financial / Transactional Rules

- Payment proposal amounts are derived from RLTRPF (AP transactions). Discount amounts and due dates are calculated from RLTRPF fields and the associated payment-terms record in RA03PF.
- When a proposal is transferred (GL105R), GABFPF.GASTAT is set to `'G'` for each transferred line. This is the single flag that permanently blocks re-transfer and re-editing.
- On first-pass return (GABFPF.GAREKO = `'1'`): GL125R deletes the payroll proposal entry. This is a non-reversible operation.
- On error return (GABFPF.GAREKO = `'F'`): the line is flagged for manual review; the proposal remains in an editable state only if no other lines have GASTAT = `'G'`.
- GL400R (company change) rewrites the firm number on all payment-related files. This operation has no undo; it must be executed only when the data library has been duplicated for the new company.

---

## Status and Lifecycle Rules

| Status / Flag | File | Field | Meaning | Effect |
|---|---|---|---|---|
| Transferred to bank | GABFPF | GASTAT = `'G'` | Proposal line sent to bank | Blocks editing (*in38=ON) and re-transfer |
| Error return | GABFPF | GAREKO = `'F'` | Bank returned error | Highlighted; manual action required |
| 1st-pass return | GABFPF | GAREKO = `'1'` | Bank acknowledged 1st pass | GL125R triggers GL125R deletion of payroll proposal |
| Remittance exclusion | RLEVPF | RLREMI = `'J'` | Supplier excluded from remittance | Excluded from all payment proposals |
| Attestation required | RLTRPF | ROATTS = `'1'` or `'J'` | AP transaction attested | Required for inclusion in payment proposal |

---

## Special Conditions (Program-Specific)

### GL105R — Collection-File Conflict

When GFASPF.GEFSAM (merge to collection file) is active, GL105R checks whether another bank connection's data is already present in the sammenfil (collection file). If present (wpfore = `'1'`), the transfer is blocked. This prevents two incompatible bank-format records from being merged into the same outbound file.

### GL210R — Duplicate Transfer Guard

GL210R is called before any receipt file is processed. It reads all BETFOR23 and BETFOR04 records from GATRPF and verifies that each references a valid GUBFPF entry. A missing GUBFPF entry means the return file references a proposal/line that no longer exists or was never sent, indicating a duplicate or erroneous file. The caller receives wxfeil = `'F'` and must abort.

### GL300R — ISO 20022 XML Generation

GL300R (introduced 2023) generates a pain.001 XML payment file to the IFS path defined in GFASPF.GEFIFS. This program is only invoked when GFASPF.GEFKOM = `'IFS'`; no XML file is created for other communication types.

### GL103R — Payroll Type Lookup

GL103R fetches GFASPF.GEFLON and returns it to the caller. If GFASPF is not found (*in60 = ON), the returned wplonn value remains blank. Callers that require a non-blank payroll type must treat blank wplonn as a configuration error.

---

## Subprogram Calls Affecting Logic

| Calling Program | Called Program | Purpose | Blocking Effect |
|---|---|---|---|
| GL100R | GL103R | Fetch payroll type from GFASPF | Blank return = missing config |
| GL101R | (none) | Renumber proposal lines in GABFPF | None |
| GL104R | (none) | Maintain payment proposal | *in38=ON if GASTAT=`'G'` |
| GL105R | GL200R | Fetch bank code from GFASPF | `'XX'` return = aborts transfer |
| GL105R | (CL driver) | Submit transfer job | Blocked if wpfore=`'1'` |
| GL110R | GL210R | Duplicate return file check | wxfeil=`'F'` blocks processing |
| GL115R | GL125R | Delete payroll proposal on 1st-pass return | Irreversible deletion |
| GL120R | (none) | Maintain GFASPF fixed data | *in30–*in37 block save |
| GL300R | (none) | Write ISO 20022 XML to IFS | Requires GEFIFS populated |

---

## Reference Tables

| Table (Physical File) | Key Fields | Relevant Fields | Role in Module |
|---|---|---|---|
| GABFPF (GUBFPF) | Firm, Proposal nr, Line nr | GASTAT, GAREKO | Payment proposal; GASTAT=`'G'` blocks edit/retransfer |
| RLTRPF | Firm, Voucher, Line | ROATTS | AP transactions; must be attested to enter proposal |
| RLEVPF | Firm, Supplier nr | RLREMI, RLNAVN | Supplier register; RLREMI=`'J'` excludes from remittance |
| GFASPF | Firm | GEFBAN, GEFKOM, GEFIFS, GEFLON, GEFEFS, GEFSBF, GEFSAM, GEFIN | Fixed data for electronic payment; must exist and be fully populated |
| GAVTPF | Firm, Supplier, Bank | Agreement fields | Bank agreement register; required for payment mediation |
| GATRPF | Firm, Transfer nr | Record type (BETFOR23/04) | Bank transfer register; used for duplicate detection |
| AFIRPF | Firm | Company details | Company register; provides sender info for bank file header |
| GLONPF / GTRTPF | Firm | Various | Payroll/transfer registers; updated by GL400R company change |
