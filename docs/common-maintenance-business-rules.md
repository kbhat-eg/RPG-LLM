# Business Rules: Common Maintenance (RM Module)

**System:** ASOKON / ASOFAK
**Module Prefix:** RM / RMP
**Programs Analyzed:** RM001R, RM002R, RM003R, RM004R, RM009R, RM100R, RM103R, RM105R, RM110R, RM120R, RM503R, RM510R, RM720R, RMP001R, RMP010R, RMP020R, RMP030R, RMP97R
**NXKORR Overrides:** NXKORR/rpgsrc/RM001R.MBR, NXKORR/rpgsrc/RM002R.MBR, NXKORR/rpgsrc/RM009R.MBR, NXKORR/rpgsrc/RMP010R.MBR
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- All RM-module export programs (RM001R, RM002R, RM009R) are driven by a posting file scoped to a single firm. The firm is read from LDA position 944–946 (`l_firm`). Any record where the firm field of the posting file differs from `l_firm` causes an immediate exit (`goto slutt` / `*inlr = *on`).
- RM001R (accounts-receivable export to Mamut/Tripletex) requires sales order header data in `SOHEPF`. The `control_init` subroutine performs a SETLL+READE on `SOHEL1` (keyed by firm + year + invoice number). If no matching sales-order header is found (`*in90 = *on`), the KID generation subroutine exits without computing a KID value — the field is left blank in the export record.
- RM001R also requires `RA04PF` (fixed accounts table) to resolve the accounts-receivable account number (`RADK05`). If `RA04L1` is not found for the firm, the fallback account used depends on whether `RADK05 = *zero`; if so, account 1510 is used as default (RM002R NXKORR override — `eval radk05 = 1510`).
- RM002R (accounts-payable export) reads purchase-posting file `BOVFUT`. Records where `BFFIRM <> l_firm` exit the loop immediately. The program requires `RA04PF` for the AR/AP account (`RADK05`); default 1510 applies if missing.
- RM009R (posting-to-Mamut line aggregation) reads `FOVIL1` (or `FOVUPF` in base version). The firm filter `FFFIRM <> l_firm` terminates processing. The program requires `FOVIL1` to be positioned by firm before reading begins (version 6.10: `setll fovil1` keyed by `fovfl1_firm`).
- RMP010R (MalProff inventory movement import) reads `MPINPUT` (256-byte input file). It requires `MJLOPF` (movement types) and `VVEPPF` (item-EAN register) to resolve item numbers from GTIN/EAN codes. A GTIN of 0 is treated as an error (version 8.02: `w_feil` set) — the line is rejected.

---

## 2. Validation Rules

### RM001R — Accounts-Receivable Mamut Export (NXKORR override)
- Firm filter: if `FOVUL1.FFFIRM <> l_firm` → `goto slutt`, terminate.
- Account number > 9999 (sub-ledger account): replaced with `RA04PF.RADK05` (the configured AR clearing account). If `ffdkto > 9999` → `ffdkto = radk05`; if `ffckto > 9999` → `ffckto = radk05`.
- VAT code conversion: if the debit or credit VAT code exists as a key in `RB00PF` (code conversion table, type='MVAKODE'), the code is replaced with `RB00PF.R00NKO` (new code). If `RA04PF.RADK05` appears as debit or credit account, VAT code is forced to 0.
- Voucher type 17 (rounding line): treated as a rounding correction for the adjacent invoice (type 1) or credit note (type 3). Rounding lines do not generate a separate export record; they adjust the preceding record's amount.
- Voucher type 40: remapped to type 5 for Mamut compatibility.
- Output format switch: CO402R key 'MAMUT' — if switch value character 1 = '1', indicator `*in40` is set `*off`, meaning the export format does NOT wrap text fields in double-quotes (format variant for Aspect4). Default (`*in40 = *on`) wraps fields in quotes (Tripletex format). LDA position 931–933 (`l_filg`) is passed as the file group key.

### RM002R — Accounts-Payable Mamut Export (NXKORR override)
- Firm filter: if `BOVFUT.BFFIRM <> l_firm` → `goto slutt`, terminate.
- Account number > 9999: same substitution with `RA04PF.RADK05` as RM001R.
- VAT code conversion via `RB00PF` (type='MVAKODE'): same pattern as RM001R.
- Version 8.04 exception: if `bfckto = radk14` or `bfdkto = radk14` (rounding account), skip VAT code conversion (`goto eikon`). This prevents the rounding account from acquiring an incorrect remapped VAT code.
- Voucher type conversion: `RB10PF` (voucher-type mapping table) is checked for `bfbilk`. If found (`%found`), the export voucher type is replaced with `RB10PF.RANKOD`. This allows voucher codes to be remapped for the Mamut format without changing source data.
- Project field: cleared (`clear w_proj`) if `bfdkto > 9999` or `bfckto > 9999` (sub-ledger lines carry no project in the AP export — version 7.05).

### RM009R — FOVF Posting Aggregation (Base version)
- Firm filter: if `FOVIL1.FFFIRM <> l_firm` → `*inlr = *on`, `goto ut`.
- Account type detection: `w_tdko` = first character of debit account number; `w_tcko` = first character of credit account number. If either starts with '3' (receivables range) and the two-character prefix `w_tdko2` / `w_tcko2 = *zeros`, the record is treated as an aggregation break point.
- Version 6.22: if debit account > 9999 and `w_tdko = 3` → set `x = 1` (trigger reset of first-record flag on next iteration). Same logic for credit account. This prevents incorrect aggregation across sub-ledger account breaks.
- Aggregation rule: debit records with the same account are accumulated (`w_belp += ffbelp`, `w_mvag += ffmvag`). When the account changes, the previous aggregate is written to `FOVUPF` via `READP` to position back and write the total.

### RMP010R — MalProff Import (NXKORR override)
- GTIN = 0: treated as error (`w_feil` set, version 8.02). The line is not processed.
- Item number resolved from GTIN/EAN via `VVEPPF`. If not found, the line is rejected.
- Movement type resolved via `MJLOPF`. If not found or not mapped, the line is skipped.

---

## 3. Configuration and Authorization Rules

- CO402R switch 'MAMUT' (RM001R NXKORR): controls output quoting format. Value '1' at position 1 → no quoting (`*in40 = *off`, Aspect4 format). Blank or not found → quoting enabled (`*in40 = *on`, Tripletex format).
- LDA position 931–933 (`l_filg`) is passed as the file group identifier to CO402R in RM001R. This allows per-installation customisation of the Mamut export format without code changes.
- LDA position 944–946 (`l_firm`): all processing is strictly firm-scoped. Cross-firm records terminate the export loop.
- `RA04PF` must contain a record for the active firm with a non-zero `RADK05` value. Otherwise, the fallback account 1510 is used (RM002R). RM001R relies on `RA04PF` being successfully CHAINed in `*inzsr`; if `%found = *off` or `radk05 = *zero`, the default 1510 is set.

---

## 4. Financial / Transactional Rules

- RM001R: when debit account (`ffdkto`) is non-zero, the export balance is `FFBELP` as-is (positive). When debit account is zero (credit-only line), the balance is negated (`ffbelp * -1`) before export, so Mamut receives a positive credit value.
- RM002R: same sign-reversal logic — if `bfdkto = 0`, the balance is negated.
- RM009R: multiple lines with the same debit account are accumulated into a single output record. The accumulated balance (`w_belp`) and VAT amount (`w_mvag`) are written to the output via READP+update of the first matching record in `FOVUPF`.
- VAT code 0 is forced when the debit or credit account equals `RADK05` (the clearing account). This prevents the clearing account line from carrying a VAT code in the export.

---

## 5. Status and Lifecycle Rules

- RM001R reads `FOVUL1` (sorted posting work file, version 8.01+). Prior to 8.01, it read `FOVUPF` (unsorted). The sorted file ensures voucher-type ordering for correct rounding-line association.
- RM009R: after aggregation, `w_belp` is reset to 0 and `w_fØrs` (first-record indicator) is reset to 0 at each account break. Unprocessed trailing balance at EOF is written in the `ut` section if `*inlr = *on and w_belp > 0`.
- RMP010R: uses `RMPIPF` (MalProff import staging file) and `RMPFPF` (MalProff posting file) as output. Both are keyed update-add files. Failed imports are tracked via the `w_feil` field within the input record structure.

---

## 6. Special Conditions

- RM001R KID calculation: order type (`SOHEPF.SOOTYP`) → `VOTYPF.VAORUT` → `AFORPF.AFKIDK`. `AFKIDK = 1` uses `SOHEPF.SOKIDD` directly. `AFKIDK = 2` uses `SOHEPF.SOKIDR`. `AFKIDK = 3` calls `AK710R` to compute a checksum-based KID from `SOKIDR`. If no matching order-type or routine is found, KID is left blank.
- RM002R: the due-date field uses a separate data structure (`b_fdati`) distinct from the journal date (`b_jdati`). Both are reformatted from ISO to YYYYMMDD for the export. In version 7.04, a bug was fixed where the wrong date structure was used for the formatted due date.
- RM009R version 6.22 `x` flag: when `x = 1` is set (cross-boundary sub-ledger account detected), the next iteration resets `w_fØrs = 0` before accumulation, causing a fresh aggregation start. Without this, aggregation would incorrectly span across sub-ledger boundary accounts.

---

## 7. Subprogram Calls Affecting Logic

| Called Program | Called From | Purpose / Effect on Logic |
|---|---|---|
| `CO402R` | RM001R (NXKORR, init) | Reads 'MAMUT' switch; controls double-quote wrapping in export |
| `AK710R` | RM001R (control_init) | Computes checksum KID from order reference number |
| `RA04L1` (file CHAIN) | RM001R, RM002R | Resolves AR/AP clearing account; missing record triggers default 1510 |
| `RB00L1` (file CHAIN) | RM001R, RM002R | VAT code conversion lookup for Mamut-compatible codes |
| `RB10L1` (file CHAIN) | RM002R | Voucher type conversion for AP export |

---

## 8. Reference Tables

| Table | Role in Module |
|---|---|
| `FOVUPF` / `FOVIL1` | AR posting file (FOVUPF base, FOVIL1 sorted logical) — source for RM001R and RM009R |
| `BOVFUT` | AP posting file — source for RM002R |
| `RA04PF` | Fixed accounts register — AR/AP clearing account lookup (`RADK05`, `RADK14`) |
| `RB00PF` | Code conversion table (type='MVAKODE') — VAT code remapping for Mamut |
| `RB10PF` | Voucher type conversion table — AP voucher code remapping for Mamut |
| `SOHEPF` | Sales order header — KID and invoice reference lookup in RM001R |
| `VOTYPPF` | Order type register — maps order type to KID-generation routine |
| `AFORPF` | Payment routine register — defines KID field source (`AFKIDK`) |
| `RHOVPF` | General ledger register — account description lookup |
| `RKUNPF` | Customer register — address fields for export |
| `MPINPUT` | MalProff input staging file — 256-byte fixed records for RMP010R |
| `RMPIPF` | MalProff import register — output of RMP010R |
| `RMPFPF` | MalProff posting file — second output of RMP010R |
| `MJLOPF` | Movement type table — maps MalProff movement codes to system types |
| `VVEPPF` | Item-EAN register — resolves item number from GTIN/EAN in RMP010R |
