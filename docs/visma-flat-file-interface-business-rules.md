# Business Logic for Visma/Global/Duett Flat File Export Interface

The Visma flat file export interface transforms internal ASOFAK/ASOFON transaction data into CSV and batch-format flat files consumed by external accounting systems (Visma, Global, Duett). Programs in the RV prefix family read sales invoice lines (FOVF/FOVIPF), GL voucher lines (BOVFL/LOVFL), and supplier invoice lines (LOVF), then output delimited records to RV01PF, RV02PF, and RV12PF. Every record is filtered by firm number from the Local Data Area. KID (payment reference) computation relies on the order-type-to-routine-register chain, and VAT codes are converted through lookup tables before output.

---

## Prerequisites / Master Data Requirements

- **LDA firm number (positions 944–946)** must be set before any RV program runs. Every line-level loop begins with `if l_firm <> fffirm → goto slutt` (RV001R) or equivalent. A mismatch silently skips the record; no error is raised.
- **SOHEPF** (sales order header) must contain a record matching the invoice line's firm, order number, and suffix. RV001R chains this file to obtain `sootyp` (order type) and `sokidd`/`sokidr` (KID fields).
- **VOTYPPF** (order type register) must contain a record for `sootyp`; if the chain fails (`*in90 = *on`), the KID-determination subroutine proceeds with a blank routine code, meaning no KID is written.
- **AFORPF** (routine register) must contain a record for the routine code returned by VOTYPPF; field `afkidk` (1, 2, or 3) drives which KID source is used.
- **RB10PF** (billing type/billing code conversion) must contain a record for `firm + ffbilk`. If the exact firm key is not found, a fallback to firm `000` is attempted (v7.04+). If neither exists, the billing type is written blank.
- **RB00PF** (code conversion, type `MVAKODE`) must contain a record to map internal VAT codes to the accounting system VAT code. If absent, the original internal code is passed through.
- **RKUNPF** / **RLEVPF** — customer and supplier registers — must contain records for the customer/supplier number on the invoice; if absent, name and address fields are written blank.
- For RV011R (supplier invoices): **SIHEPF** must have a record for `firm + binr + year` when v7.05 embedded SQL logic is active. **LOHEPF** must have a record to supply `lokidi`.
- For RV012R (Global batch): **LOVFL3** records must be present; the output file RV12PF must be cleared before each run, as the program prepends `@FIRM_BEGIN`, `@IMPORT_METHOD`, and batch header records.

---

## Validation Rules

- **Company filter** (blocking): Each detail-line loop tests `if l_firm <> fffirm` (RV001R/RV009R), `if l_firm <> lgfirm` (RV011R/RV012R), or `if bqvwl1_firm <> bffirm` (RV003R). Any mismatch causes `goto slutt` / `*inlr = *on`; the record is skipped or the program terminates without output.
- **VAT code resolution** (blocking output integrity): RV001R first checks `if ffdmom = *blank use ffcmom else ffdmom`. It then chains RB00PF with type `MVAKODE` and the resolved code. If not found, the internal code is used as-is. RV021R/RV022R (Duett variants) follow the same pattern.
- **Billing code resolution**: RV001R chains RB10PF using `firm + ffbilk`. Failure causes a retry with firm=000. If both fail, blank is written for billing type. Processing does not stop; a blank billing type may cause rejection by the receiving accounting system.
- **KID determination** (RV001R, RV011R, RV021R, RV005R):
  - Chain VOTYPPF on `sootyp`; if not found, no KID is written.
  - Chain AFORPF on `vaorut`; field `afkidk` controls source:
    - `afkidk = 1` → `t1kkid = sokidd`
    - `afkidk = 2` → `t1kkid = sokidr`
    - `afkidk = 3` → call AK710R to compute KID from `sokidr`; AK710R must return successfully.
  - RV005R calls AK710R unconditionally for every record.
- **Supplier invoice KID** (NXKORR RV011R v7.05): Uses embedded SQL `select shotyp into :w_otyp from sihepf` to get order type. If the SQL returns no row, `w_otyp` remains blank and KID determination falls through. Then LOHEPF is read via `setll/read` on `lohel1_key`; if not found, `lokidi` is blank.
- **RV003R company filter**: If `bqvwl1_firm <> bffirm`, the program sets `*inlr = *on` and `goto ut` immediately — no records are updated.
- **RV008R break logic**: Amount summarization breaks when `ffdkto` (debet account starting with '3') changes. Records not matching the break criteria are accumulated silently.
- **RV012R summarization**: Breaks on `biln + dkto` and `biln + ckto`. If the same voucher has no debet or no kredit account, no summary record is written for that side.

---

## Configuration and Authorization Rules

- **Accounting system selection** (RV015R): Called by the surrounding process before initiating an export run. Chains FSTSPF by firm number. Returns `p_kode = (faak08 * 10) + faak03` (combined code indicating which accounting system is active) and `p_drop = faak04` (flag to skip NX economics update). If FSTSPF has no record for the firm, both outputs remain at their initialized values.
- **Firm number source**: All RV programs read the firm number exclusively from LDA positions 944–946 (`l_firm`). No interactive prompt overrides this. If the LDA is not set for the correct firm before calling an RV program, all records are skipped.
- **Output file format selection**: The calling process selects which program to invoke (RV001R for Visma sales invoices, RV002R/RV005R for GL vouchers, RV011R for supplier invoices, RV012R for Global batch, RV021R/RV022R for Duett). Each program writes to a dedicated output physical file (RV01PF, RV02PF, RV12PF). These files must exist and be cleared before the run.

---

## Financial / Transactional Rules

- **Amount sign convention**: RV001R writes `ffbelp` directly. RV012R negates amounts for kredit-side records by writing negative `lganta * lgkopr`.
- **VAT amount**: Written from `ffdmva` (debet VAT) or `ffcmva` (kredit VAT) depending on account-side indicator.
- **Summarization** (RV008R → RV009R path): RV008R reads FOVIPF, detects debet accounts starting with '3', accumulates `w_belp + ffisum` and `w_mvag + ffimva` within the same break key (firm/date/biln). Writes one FOVUPF record per break. RV009R then reads FOVUPF and formats as RV01PF.
- **GL voucher counter-posting** (RV002R v8.00+): For each GL line in BOVFL3, the program fetches the counter-posting debet/kredit account from BODTPF. Summarizes per voucher and account combination. V9.01 additionally chains SOHEPF and retries with prior-year SOHEPF if not found.
- **KID clearing after write**: RV001R executes `eval t1kkid = *blank` after each output record write to prevent carry-over of the previous record's KID.

---

## Status and Lifecycle Rules

- The RV programs are batch-oriented export programs. There is no interactive status maintained. Each run overwrites the output file from the beginning.
- **RV003R** updates BQVWPF distribution parameter records: sets `bfdbkr = 'D'` if `bfdkto <> 0`, sets `bfdbkr = 'K'` if `bfckto <> 0`. These flags indicate whether a distribution record has a debet or kredit account assigned and must be set correctly before RV004R reads BQVWPF.
- **RV004R** reads BQVWPF sorted by firm/date/D-K indicator/dkto/ckto/bilk/davd/cavd. It writes a BOVFUT break record whenever any of these sort keys changes. BOVFUT is then consumed by RV005R.
- If FOVF/BOVFL/LOVFL contains no records for the LDA firm, the output file receives zero records and the program terminates normally.

---

## Special Conditions (Program-Specific)

- **RV001R vs. RV011R**: RV001R processes sales invoices from FOVF; RV011R processes supplier invoices from LOVFL1. Both write to RV01PF but use different primary-file field names (`ff*` vs. `lg*`). The NXKORR override of RV011R adds SIHEPF (supplier invoice header) and LOHEPF (purchase order header) to source KID from `lokidi` rather than the routine-register chain.
- **RV012R Global batch format**: Unique among RV programs in writing structured batch headers (`@FIRM_BEGIN(nnnn)`, `@IMPORT_METHOD(3)`, `@WaBnd(...)`, `@WaVo(...)`) rather than plain CSV. The firm number embedded in `@FIRM_BEGIN` must match the external accounting system's firm identifier, which is configured separately from the LDA firm number.
- **RV021R/RV022R Duett format**: Identical logic to RV001R/RV002R but appends `rkemal` (customer email address) as the final CSV field. Added at v7.01. If `rkemal` is blank in RKUNPF, an empty field is written; the Duett import accepts this.
- **RV015R subprogram**: Returns a combined code from FSTSPF fields `faak08` (8) and `faak03` (3) as `(faak08 * 10) + faak03`. The calling program must test this value to determine whether to suppress the NX economics update (`p_drop = faak04`). RV015R does not modify any data.
- **AK710R KID computation**: Called by RV001R (when `afkidk = 3`), always called by RV005R, and by RV022R. Must be present in the library list. If AK710R fails or is absent, the program abends; there is no fallback for KID-3 computation.

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| AK710R | `afkidk = 3` in AFORPF (RV001R, RV011R); every record in RV005R, RV022R | Computes KID from `sokidr` or `lgkidr` using modulus-10/11 algorithm | If absent or returns error, the calling program abends; no KID is written |
| RV015R | Called by surrounding process before export | Chains FSTSPF; returns accounting system code and drop flag | Determines whether Visma/Global/Duett export runs and whether NX update is suppressed |
| RV003R | Called before RV004R on BQVWPF records | Sets `bfdbkr='D'` or `'K'` based on account fields | Must run first; if skipped, RV004R may produce incorrect output |
| RV004R | Called after RV003R | Reads BQVWPF and summarizes to BOVFUT | BOVFUT must be empty before RV004R; otherwise duplicate records accumulate |
| RV008R | Called before RV009R | Reads FOVIPF and summarizes to FOVUPF | FOVUPF must be empty before RV008R |

---

## Reference Tables

| Table | Key Fields | Purpose in RV Programs |
|-------|-----------|------------------------|
| FOVF (FOVIPF/FOVFL) | firm, biln, line | Sales invoice lines — primary input for RV001R, RV021R |
| BOVFL1/BOVFL3 | firm, biln, line | GL voucher lines — primary input for RV002R, RV022R |
| LOVFL1/LOVFL3 | firm, biln, line | Supplier invoice lines — primary input for RV011R, RV012R |
| FOVUPF | firm, biln | Summarized sales invoice lines — primary input for RV009R |
| BOVFUT | firm, biln | Summarized GL lines from RV004R — primary input for RV005R |
| SOHEPF | firm, numm, suff | Sales order headers — provides sootyp, sokidd, sokidr |
| SIHEPF | firm, binr, year | Supplier invoice headers (NXKORR RV011R) — provides shotyp |
| LOHEPF | firm, numm, suff | Purchase order headers (NXKORR RV011R) — provides lokidi |
| VOTYPPF | firm, otyp | Order type register — maps sootyp to routine code vaorut |
| AFORPF | firm, orut | Routine register — provides afkidk (KID source selector) |
| RB10PF | firm, bilk | Billing code conversion — maps internal billing code to accounting system billing type |
| RB00PF | firm, type, kode | General code conversion — type MVAKODE maps VAT codes |
| RKUNPF | firm, kund | Customer register — name and address for output |
| RLEVPF | firm, levr | Supplier register — name and address for RV011R output |
| FSTSPF | firm | System status register — accounting system selection flags (used by RV015R) |
| BQVWPF | firm, date, ... | Distribution parameters — updated by RV003R, read by RV004R |
| RV01PF | firm | Output flat file for sales/supplier invoices |
| RV02PF | firm | Output flat file for GL vouchers |
| RV12PF | firm | Output flat file in Global batch format |
