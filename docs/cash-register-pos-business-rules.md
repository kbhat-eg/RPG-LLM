# Cash Register / POS — Business Rules

## Introduction

The BO (cash register / POS) module manages all aspects of store point-of-sale operations: daily settlements, receipt handling, store-status configuration, and the transfer of store receipts into the ERP order/invoice system. Settlement processing is tightly controlled by concurrency locks (via FA730R/FA720R job-register programs), by workstation group validation, and by the state of individual receipts. The NXKORR library contains patch versions of the two most critical settlement programs (BO120R and BO122R) that add economy-system routing and audit logging. Transfer of receipts to ERP orders (BO700R) applies hold logic for negative pack slips.

---

## Prerequisites / Master Data Requirements

- BSTSPF (store status register) must contain a record for the store before settlement number-series fields (BSTSPF.BAAAVS, BAADET, BAALST) can be used in automated settlement (BO123R). If BSTSPF is not found (*in60 = ON), the number-series fields are set to blank, effectively making number allocation fail.
- BWSDPF (workstation/cash register register) must contain an entry for each workstation that participates in settlement. BO122R validates workstation/group membership against BWSDPF; unrecognized workstations are excluded from settlement extraction.
- BOHEPF (store receipt header) must exist for the receipt number referenced by BO310R (display receipt) and BO700R (receipt-to-order transfer). If not found, both programs abort via `goto avslutt`.
- RKUNPF (customer register) is read by BO310R to display the customer name on a receipt. A missing customer record means the name field remains blank; this is non-blocking for display but affects printed documents.
- AFPSPF (hold setting register) is read by BO700R (version 7.01) to determine whether negative pack slips should be held. If the relevant AFPSPF record indicates hold, the order is created with a hold flag.
- BDOPPF (job-plan register) is checked by BO120R for job-plan-based settlement (BDOPPF.LDJOBP = `'J'`). If this flag is set, settlement runs automatically from the job plan rather than interactively.
- AVALPF (workstation group validation register) is checked by BO120R version 7.01 for workstation group authorization. Invalid groups are rejected before settlement begins.

---

## Validation Rules

### BO120R — Daily Settlement Parameter Screen

- Calls FA730R (job register check) at startup. If FA730R returns w_onof = `1` (settlement already running for this store/job): program executes `goto xslutt`, aborting the settlement attempt entirely. This is the primary concurrency block.
- NXKORR BO120R (v8.03–v8.07): additionally reads XTILPF by routing code to determine the economy-system type (xtnumm), and separates the settlement-run parameters from accounting-update parameters. Logic changes do not relax the concurrency check.
- Version 7.01: validates the workstation group against AVALPF before proceeding. Invalid group = blocked.
- Version 8.01: on exit from settlement, calls FA720R to release the job lock in the job register. Failure to call FA720R leaves the lock permanently set, preventing future settlements.
- BDOPPF.LDJOBP = `'J'`: triggers automated settlement path; the interactive parameter screen is bypassed.

### BO122R — Settlement List Extraction

- For each BOHEPF record read sequentially: if BOHEPF.BOFIRM ≠ l_sfir (LDA company): record is skipped (wrong company).
- Validates workstation membership: chains to BWSDPF; if the workstation is not in the expected group, the receipt is excluded from the settlement batch.
- BOHEPF.BKASST flag (old vs. new cash register system): controls which processing path is applied to each receipt.
- NXKORR BO122R: adds an AS710R call for job-execution logging (audit trail). This is non-blocking; a logging failure does not stop settlement.

### BO123R — Automated Daily Settlement (Job Plan)

- Reads BDOPPF for settlement parameters.
- Calls AS100R to allocate number-series entries. If BSTSPF is not found (*in60 = ON): number-series fields (BAAAVS, BAADET, BAALST) are set to blank, and the call to BO121C proceeds with empty number series, which will cause number-allocation failure downstream.
- After number allocation, calls BO121C to execute the actual settlement.

### BO124R — Settlement Totals Print

- For each BPSOPF (picking S-line) record: if BPSOPF.BOKAVS = `1` (receipt already processed in a previous settlement): skip this record (`goto astart`). This prevents double-counting.
- Flag b_ukre: if set, credit-only receipts are excluded from the totals.

### BO125R — Settlement Details Print

- Processes BPSOPF records with level breaks on store/type/workstation/receipt number (bsofir/bsooty/bsowsd/bsonum).
- On successful processing of a receipt, sets BOHEPF.BOKDET = `1` (processed in settlement). If credit-only exclusion is active, sets BOKDET = `4` instead.
- BOKAVS = `1` is the key status field that prevents double-settlement. BO125R sets this field after processing; BO124R reads it to skip already-processed receipts.

### BO310R — Display Receipt

- Chains to BOHEPF by key (firm + receipt number). If not found: `goto avslutt` (program terminates without displaying anything).
- Customer name is retrieved from RKUNPF; missing customer is non-blocking (name field blank).
- Calls FA703R (payment terms lookup) to display terms on screen.

### BO700R — Receipt Transfer to ERP Order

- Chains to BOHEPF by receipt key. If not found: `goto avslutt` (no transfer occurs; order is not created).
- Creates FODTPF (order lines) and FOHEPF (order header) records from the receipt.
- Calls FO730R to recalculate the new order's totals.
- Version 7.01: reads AFPSPF to check the hold setting for negative pack slips. If the hold flag is set in AFPSPF, the created FOHEPF order is placed on hold (not released for picking/shipping).

### BO510R — Store Status Register Print

- Simple sequential read of BSTSPF with company filter. No blocking conditions.

### BO400R — Company Change

- Mass update of all BO tables (bulk firm-number rewrite). No blocking logic.

---

## Configuration and Authorization Rules

- **Concurrency control**: FA730R / FA720R form a software lock on the settlement process. FA730R sets a "running" flag (returns w_onof = `1` if already running); FA720R clears it on exit. If BO120R terminates abnormally without calling FA720R, the lock must be manually cleared in the job register.
- **Store status register** (BSTSPF): controls the number-series pools for automated settlement. BSTSPF fields BAAAVS (receipt number series), BAADET (detail number series), and BAALST (last number used) must be populated and within valid ranges before settlement can allocate numbers.
- **Workstation register** (BWSDPF): defines which physical workstations belong to which store/group. BO122R uses BWSDPF to validate that each receipt's originating workstation is part of the current settlement batch.
- **BOKAVS = 1**: receipt processed in settlement. This flag is set by BO125R. Once set, BO124R and any future settlement pass skip this receipt. This flag is the settlement idempotency control.
- **BOKDET**: set to `1` (normal) or `4` (credit-only excluded) by BO125R to track how each receipt was handled in settlement detail printing.
- **XTILPF** (NXKORR BO120R): economy-system routing table. The xtnumm field determines which downstream accounting system receives the settlement data. Missing XTILPF entries for the current routing code cause the economy-system type to default to blank.

---

## Financial / Transactional Rules

- Settlement is the process that aggregates POS receipt totals and posts them to the accounting system. Double-settlement is prevented by BOHEPF.BOKAVS = `1`; once a receipt is settled, it cannot re-enter a settlement batch.
- Transfer of a receipt to an ERP order (BO700R) creates real order records in FOHEPF and FODTPF. The transfer is non-reversible through this program; order cancellation must be handled by the order management module.
- Negative pack slips (credit notes) transferred via BO700R are subject to the AFPSPF hold setting (v7.01). A held order will not proceed to warehouse picking until manually released.
- FO730R is called after FOHEPF/FODTPF creation to recalculate order totals (discounts, VAT, net amount). Failure of FO730R may leave the order with incorrect totals.

---

## Status and Lifecycle Rules

| Status | File | Field | Value | Meaning | Effect |
|---|---|---|---|---|---|
| Settlement running | Job register | w_onof | `1` | Another settlement in progress | BO120R aborts immediately |
| Receipt settled | BOHEPF | BOKAVS | `1` | Already processed in settlement | Skipped by BO124R |
| Settlement detail processed | BOHEPF | BOKDET | `1` | Normal settlement detail | No further processing |
| Credit-only excluded | BOHEPF | BOKDET | `4` | Excluded from settlement | No further processing |
| BSTSPF not found | BSTSPF | — | *in60=ON | Store status missing | Number-series fields blank; BO123R continues but fails on allocation |
| Receipt not found | BOHEPF | — | *in60=ON | Receipt does not exist | BO310R/BO700R abort |
| Hold on negative pack slip | AFPSPF | Hold flag | Set | Credit transfer should be held | FO730R order placed on hold |

---

## Special Conditions (Program-Specific)

### BO120R — Automated vs. Interactive Settlement

When BDOPPF.LDJOBP = `'J'`, the settlement is initiated from the job plan (batch mode). The parameter screen is bypassed and parameters are read directly from BDOPPF. The concurrency check via FA730R still applies in batch mode.

### NXKORR BO120R — Economy System Routing

The NXKORR patch versions (v8.03–v8.07) add a lookup to XTILPF by routing code to retrieve the economy-system type (xtnumm). This type controls which external accounting interface receives the settlement data. The settlement itself is not blocked by XTILPF lookup failure, but missing routing entries cause the accounting interface to receive a blank economy-system code, which may cause it to reject the settlement data.

### BO700R — Receipt Hold for Negative Pack Slips

Added in version 7.01. When BO700R creates an ERP order from a receipt that represents a credit (negative quantities), it reads AFPSPF to determine whether the hold flag is active. If active, the order header (FOHEPF) is written with the hold indicator set. This prevents immediate fulfilment of credit orders, allowing manual review.

---

## Subprogram Calls Affecting Logic

| Calling Program | Called Program | Purpose | Blocking Effect |
|---|---|---|---|
| BO120R | FA730R | Check if settlement already running | w_onof=1 → abort settlement |
| BO120R | FA720R | Release settlement lock on exit | Not called = permanent lock |
| BO120R | AVALPF check | Validate workstation group (v7.01) | Invalid group → blocked |
| BO123R | AS100R | Allocate number series | BSTSPF missing → blank series |
| BO123R | BO121C | Execute settlement | Called after AS100R |
| BO124R | (BPSOPF read) | Read settlement lines | BOKAVS=1 → skip |
| BO125R | (BOHEPF update) | Set BOKDET on processed receipts | Sets settlement processed flag |
| BO310R | FA703R | Lookup payment terms | Non-blocking |
| BO700R | FO730R | Recalculate order totals | Non-blocking for creation; totals may be wrong if fails |
| BO700R | AFPSPF read | Check hold for negative pack slips (v7.01) | Hold flag set → order on hold |
| NXKORR BO122R | AS710R | Job execution logging | Non-blocking |

---

## Reference Tables

| Table (Physical File) | Key Fields | Relevant Fields | Role in Module |
|---|---|---|---|
| BOHEPF | Firm, Receipt nr | BOKAVS, BOKDET, BOFIRM, BKASST, BOWSID | Store receipt header; BOKAVS controls settlement |
| BODTPF | Firm, Receipt nr, Line | Line details | Store receipt lines |
| BSTSPF | Store code | BAAAVS, BAADET, BAALST | Store status / number series |
| BWSDPF | Workstation | Group, Store | Workstation register; validates settlement inclusion |
| BDOPPF | Job plan code | LDJOBP | Job plan; LDJOBP=`'J'` triggers automated settlement |
| BPSOPF | Firm, Type, Workstation, Receipt | BOKAVS, settlement fields | Picking S-lines for settlement |
| AFPSPF | Routine | Hold flag | Hold setting for negative pack slips (BO700R v7.01) |
| AVALPF | Workstation group | Authorization | Workstation group validation (BO120R v7.01) |
| FOHEPF | Firm, Order nr | Order header fields, hold flag | ERP order header created by BO700R |
| FODTPF | Firm, Order nr, Line | Order line fields | ERP order lines created by BO700R |
| XTILPF | Routing code | xtnumm (economy system type) | Economy-system routing (NXKORR BO120R v8.03+) |
| RKUNPF | Firm, Customer nr | RKNAVN | Customer name for receipt display |
