# Flat-File Visma Interface — Business Rules

## Introduction

The Flat-File Visma Interface (module prefix **RB**) exports accounting data from the IBM i ERP system to external accounting packages (Visma Business and Uni Micro). Programs produce CSV/fixed-format text files containing customer invoices, supplier invoices, payment vouchers, and customer/supplier address data. All export files are written to intermediate IBM i physical files (`RV01PF`, `RV02PF`, `RV03PF`, `BOBFUT`) that are subsequently transferred to the accounting system.

Export is blocked or skipped whenever the source data does not pass firm-match, code-mapping, or address-lookup validations described below.

---

## Prerequisites and Master Data Requirements

| Requirement | Table | Key Fields | Used By |
|---|---|---|---|
| Firm record must exist in accounts-receivable voucher file | `FOVUPF` | `FFFIRM` | RB001R, RB002R, RB031R |
| Firm record must exist in accounts-payable voucher file | `BOVUPF` / `BQBWPF` | firm field | RB003R, RB006R, RB007R |
| Firm record must exist in supplier ledger file | `LOVUPF` | firm field | RB004R, RB022R |
| Voucher code must be mapped in the voucher-code conversion table | `RB10PF` | firm + voucher code | RB001R, RB003R, RB022R |
| VAT code must be mapped in the VAT conversion table | `RB00PF` | type `'MVAKODE'` + VAT code | RB001R, RB003R |
| Customer must exist in customer register for address export | `RKUNPF` | firm + customer number | RB002R, RB011R |
| Supplier must exist in supplier register for address export | `RLEVPF` | firm + supplier number | RB004R |
| Department code must be resolvable from order header | `SOHEPF` | order header key | RB003R |
| `@FIRM_BEGIN` / `@FIRM_END` delimiters require a valid firm header record | `RB00PF` (firm banner field `RAFIRB`) | — | RB001R |

---

## Validation Rules

### VR-01 — Firm Mismatch Blocks Export (All Programs)

Each export program reads records sequentially and compares the record's firm code against the session firm loaded from the Local Data Area (`l_firm`, positions 944–946).

- **RB001R / RB002R**: `if l_firm <> fffirm → goto slutt` (RB001R) / `goto nyles` (RB002R). Processing stops entirely for mismatched records; no further rows are exported for that firm.
- **RB003R / RB006R / RB007R**: same pattern on the accounts-payable firm field.
- **RB004R / RB022R**: same pattern on the supplier-ledger firm field.

*Effect*: The entire export run is abandoned for any record where the session firm does not match the data firm. Records from a different firm are never written to the output file.

### VR-02 — Voucher Code Not Mapped Blocks Line Export

**Program**: RB001R, RB003R, RB022R

The voucher code (`ffbiko` / `bobiko`) is used as a key into `RB10PF` via composite key `rb10l1_key chain rb10l1`. If the chain fails (`not %found`), the output line for that voucher entry is skipped.

*Effect*: Any accounts-receivable or accounts-payable voucher line whose voucher code has no entry in `RB10PF` is silently omitted from the export file.

### VR-03 — VAT Code Not Mapped Blocks VAT Column

**Program**: RB001R, RB003R

The VAT code is looked up in `RB00PF` with type `'MVAKODE'`. If not found, the VAT amount column in the CSV output row is written as zero/blank.

*Effect*: Lines with unmapped VAT codes are still exported, but the VAT-amount field will be zero, which may cause reconciliation failures in Visma Business.

### VR-04 — Customer Not Found Skips Address Row

**Programs**: RB002R, RB011R

Customer lookup: `rkunl1_key chain rkunl1`; indicator `90` = not found. If `%in(90) = *on`, the address row is skipped entirely.

*Effect*: Invoices for customers that have no record in `RKUNPF` will not have a corresponding address row in the export, which will prevent Visma from resolving the debtor.

### VR-05 — Invoice Number and Factoring Number — Zero Written as Blank

**Program**: RB002R (v8.02)

If the invoice number (`fffakn`) or factoring number is zero, they are written as blanks in the CSV output rather than `000000`. This is a formatting rule; it does not block the row but prevents Visma from treating 0 as a valid reference number.

### VR-06 — Only First Line-Zero Record Per Invoice is Exported

**Program**: RB011R

The program only writes address records when `ffbill = 0` (the header billing line for an invoice). Subsequent lines for the same invoice are not written.

*Effect*: Each invoice maps to at most one address row. Subsequent detail lines are ignored for address export purposes.

### VR-07 — Debit/Credit Classification Required Before Summarisation

**Program**: RB006R

Before summarising voucher lines, `RB006R` must set `BFDBKR` to `'D'` (debit) or `'K'` (credit) based on whether the debit account is non-zero. Rows where neither condition applies (both debit and credit account fields are zero) will not receive a classification. `RB007R` uses `BFDBKR` as a break key; unclassified rows may fall into an incorrect break group.

### VR-08 — Customer Number Threshold for Uni Micro Export

**Program**: RB044R

The program only writes a customer row when `bfkund > 9999`. Customers with a number of 9999 or lower are excluded from the Uni Micro customer-info export.

### VR-09 — Year Calculation Must Produce Valid 4-Digit Year

**Program**: RB031R

Year is computed as: `w_aarr = 2000 + %dec(%subst(w_aaxx:3:2):2:0)`. The two-digit year substring extracted from the date field is added to 2000. Dates with a century indicator other than the current century will produce an incorrect year, silently.

---

## Configuration and Authorization Rules

### CA-01 — Post-Code Stripping Controlled by System Switch

**Programs**: RB002R, RB004R

The switch `ØKONOMI_EKSTERNT_POSTNR_POSTSTED` is read via the `CO402R` sub-program at initialisation. If the switch is active (`= '1'`), the post code is stripped from the post-place field before writing to the CSV, because Visma Business stores post code and post place in separate columns. If the switch is inactive, the full post-place string (including any embedded post code) is written as-is.

*Blocking effect*: If `CO402R` cannot be called (program not found, authority failure), the switch value defaults to inactive and post codes are not stripped.

### CA-02 — Visma Firm Banner (`@FIRM_BEGIN` / `@FIRM_END`)

**Program**: RB001R

The first and last lines of each export segment are the firm delimiter strings `@FIRM_BEGIN(rafirb)` and `@FIRM_END`. The firm banner value `RAFIRB` is read from `RB00PF`. If this lookup fails, the delimiter records are written with a blank firm banner, which will cause Visma Business to reject the import file.

### CA-03 — KID Reference Number Calculation

**Program**: RB001R

KID (Norwegian payment reference) numbers are calculated by calling the sub-program `AK710R`. If `AK710R` is not available or returns an error, the KID field in the export row will contain an unchecked reference number. Visma Business may reject lines with invalid KID check digits.

---

## Financial and Transactional Rules

### FT-01 — Voucher Summarisation Break Keys

**Program**: RB007R

Output rows to `BOBFUT` are accumulated and summarised on the following break keys in order:
1. Accounting date
2. Debit/credit flag (`BFDBKR`)
3. Debit account
4. Credit account
5. Debit department
6. Credit department
7. Voucher code

A new summarised output row is written whenever any of these keys changes. Voucher text is looked up from `RA01PF` using the voucher code. If no text is found, the text column is written as blank.

### FT-02 — Supplier Voucher Export Format (RV03PF)

**Programs**: RB004R, RB022R

Lines written to `RV03PF` use the supplier address from `RLEVPF`. If the supplier is not found in `RLEVPF`, the address columns are written as blank. The supplier voucher rows are otherwise always written; missing address data does not block the row.

### FT-03 — Uni Micro CSV Format Record Types

**Program**: RB031R

The Uni Micro format requires two types of records per invoice:
- Record type `60`: invoice detail line
- Record type `80`: invoice header

If the customer cannot be found in `RKUNPF`, the customer name falls back to `SONAVN` (order header customer name). No blocking occurs, but the name may be less accurate.

---

## Status and Lifecycle Rules

No explicit status-field transitions are managed by this module. The export programs are stateless batch jobs; they read source tables and write output files. The source records (`FOVUPF`, `BOVUPF`, `LOVUPF`) are not updated or flagged by these programs.

---

## Special Conditions

### SC-01 — Accounts-Payable Summarisation Two-Step Process

The summarisation for accounts-payable vouchers is a two-step process:
1. **RB006R** reads `BQBWPF` and writes the debit/credit classification flag (`BFDBKR`).
2. **RB007R** reads a sorted view of `BQBWPF` and writes summarised rows to `BOBFUT`.

Both programs must complete successfully and in order. If `RB006R` is not run before `RB007R`, the break keys used by `RB007R` will be uninitialized and summarisation will produce incorrect output.

### SC-02 — Source File Rename Aliases

All programs open physical files with `rename(physfile:alias)` to avoid conflicts when multiple logical views are used. Compilation will fail if the physical file record format name does not match the expected value in the `rename()` parameter.

---

## Subprogram Calls Affecting Logic

| Program | Called Sub-Program | Purpose | Failure Effect |
|---|---|---|---|
| RB001R | `AK710R` | KID reference number calculation | Blank/invalid KID in output |
| RB001R | `CO402R` | Read system switch for post-code stripping | Switch defaults to inactive |
| RB002R | `CO402R` | Read system switch for post-code stripping | Switch defaults to inactive |
| RB004R | `CO402R` | Read system switch for post-code stripping | Switch defaults to inactive |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| `RB00PF` | VAT code and firm banner conversion | type + code |
| `RB10PF` | Voucher code conversion (ERP → Visma) | firm + voucher code |
| `FOVUPF` | Accounts receivable voucher file | firm + voucher fields |
| `BOVUPF` / `BQBWPF` | Accounts payable voucher file / working file | firm + voucher fields |
| `LOVUPF` | Supplier voucher file | firm + voucher fields |
| `RKUNPF` | Customer register (address data) | firm + customer number |
| `RLEVPF` | Supplier register (address data) | firm + supplier number |
| `SOHEPF` | Order header (department lookup) | order number |
| `RA01PF` | Voucher text table | voucher code |
| `RV01PF` | Output: customer/AR export file | — |
| `RV02PF` | Output: AP export file | — |
| `RV03PF` | Output: supplier/PO export file | — |
| `BOBFUT` | Output: summarised AP voucher file | — |
