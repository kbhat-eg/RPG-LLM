# Business Rules: Count List (LE Module)

**System:** ASLAGR
**Module Prefix:** LE
**Programs Analyzed:** LE100R, LE600R, LE602R, LE603R, LE605R, LE610R, LE611R, LE612R, LE620R, LE622R, LE623R, LE692R, LE700R, LE702R, LE703R, LE710R, LE711R, LE712R
**NXKORR Overrides:** None found
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- Every count-list session is anchored to a batch record in `LBCHPF` (batch register). If `LBCHPF` does not contain the entered batch number (`w_batc`), LE100R exits immediately (`goto end_pgm`) — no count-list lines can be viewed or edited.
- Count-list lines are stored in `LTELPF` keyed by firm + batch + list number + suffix + line number. Lines with `LTELPF.LESUFF <> 0` are treated as secondary-suffix lines and are excluded from the main edit subfile (`lesuff <> 0` → `leave`).
- Item descriptions displayed alongside count lines are looked up in `VVARPF` (item register). An unknown item number (`*in90 = *on` on `VVARL1` chain) returns blank description fields — the line is still shown.
- Unit-of-measure conversion for entered counts is resolved through `VVENPF`. If the conversion factor is not found, the unit validation subroutine (`sjekk_enh`) sets `b_feil = *on` and `*in31`, blocking the save.
- The batch must have been created via `LC500R` (batch selection program). If `LC500R` returns a non-blank return code (`w_retu <> *blank`), LE100R exits without opening any count list.

---

## 2. Validation Rules

### Batch Selection (LE100R, LE600R, LE610R, LE620R)
- Calling `LC500R` with return-type 'R' (LE100R), 'T' (LE600R), or 'L' (LE610R/LE620R): if `w_retu <> *blank` after the call → program exits (`goto end_pgm` / `goto slutt`). The batch identifier is mandatory for all count-list operations.
- From version 8.01, the LDA position 1006–1015 (`l_batc`) is checked first. If `l_batc <> ' '`, the batch is taken from LDA and `LC500R` is bypassed.

### Line Edit Validation (LE100R — xc2win / xc7win subroutines)
- Choice code 2 (change) and choice code 7 (sequential update): if `LBCHPF.LCSTEL = 1` (cyclic count flag) and CO402R switch `u_702 = *off`, editing is blocked. Message displayed via `AA007R`: "Ikke tillatt å endre linjer fra syklisk telling - må slettes". The line cannot be saved; program returns to subfile display.
- Unit validation (`sjekk_enh`): if the entered unit (`c2enh1`) is not found in `VVENPF` for the item, `b_feil = *on` sets `*in31`, and the change window loops (`goto xc2_ut` / `goto xc7_ut`) — the update is blocked until a valid unit is entered or the user cancels.
- The unit field (`LTELPF.LEENH1`) cannot be changed from version 6.11 onward (`eval leenh1 = c2enh1` is commented out). Only quantity (`leantt`), special quantity (`lespny`), and customer quantity (`lekpny`) are updatable.

### Subfile Population Filters (LE100R — crt_subfile)
- Lines where `LTELPF.LEFIRM <> w_firm` are skipped (cross-firm contamination guard).
- Lines where `LTELPF.LEBATC <> w_batc` are skipped.
- Lines where `LTELPF.LENUMM <> LBCHPF.LCNUMM` are skipped (list number mismatch).
- Lines where `LTELPF.LESUFF <> 0` are skipped (secondary suffix lines not shown in main list).

### Print Parameter Validation (LE600R — count-list print)
- Batch entry: same LC500R/LDA check as LE100R.
- After batch lookup, `LBCHPF` fields are used to set `d_batc`, `d_flag`/`d_tlag` (warehouse range from `LBC1PF` via SQL in version 8.02). If the SQL returns error or no-data (`sqlcod < 0 or sqlcod = 100`), warehouse fields are left at defaults (0 and 99).
- Warehouse range defaults: if `d_togr = 0` → set to 99; if `d_thgr = 0` → set to 99; if `d_tugr = 0` → set to 999; if `b1tvar = *blank` → set to '99999999'. These are auto-corrections, not blocks.

### Variance Report Parameters (LE610R)
- Batch entry: `LC500R` with type 'L'. Exit if `w_retu <> *blank`.
- Group ranges, item ranges, and warehouse ranges are passed directly to `LE610C` (batch CL) without additional interactive validation — range checks are the responsibility of the batch program.

### Miscellaneous Items Report (LE620R)
- Batch entry: `LC500R` with type 'L'. Exit if `w_retu <> *blank`.
- Same pass-through of parameters to `LE620C`/print program without interactive range validation.

---

## 3. Configuration and Authorization Rules

- CO402R switch (key = inferred from `l_filg` + firm): if the switch value `u_702 = *on`, the cyclic-count edit block is bypassed, allowing lines from cyclic counts to be edited. Default is `*off` (cyclic lines are protected).
- LDA position 944–946 (`l_firm`): the active firm is mandatory. All file access in LE programs is scoped to `l_firm`. If LDA position 944–946 is 0, all CHAIN/READ operations return not-found.
- LDA position 1006–1015 (`l_batc`, version 8.01+): if set, bypasses the interactive `LC500R` batch-selection prompt. Used when programs are launched from menu or scheduled batch with a pre-known batch ID.
- Audit fields (`LTELPF.LEEDAT`, `LTELPF.LEETIM`, `LTELPF.LEEUSR`) are written on every update using the current system time and LDA position 911–920 (`l_user`). If `l_user` is blank, audit fields contain blanks.
- From version 6.20, `LTELPF.LETIMS` (timestamp) is also written on every update.

---

## 4. Financial / Transactional Rules

- Counted quantity (`LTELPF.LEANTT`) and special/customer quantities (`LESPNY`, `LEKPNY`) are the only fields that can be modified interactively. Unit (`LEENH1`) is frozen after initial creation (version 6.11+).
- Count totals and variance calculations are performed downstream in the batch print programs (LE600C, LE610C) — interactive programs do not compute variances.
- The count-list number (`LBCHPF.LCNUMM`) controls which lines are shown. Multiple lists can exist within a single batch; only lines matching the current `LCNUMM` are displayed in LE100R.

---

## 5. Status and Lifecycle Rules

- Count-list lines created with `LTELPF.LESUFF = 0` are primary count lines. Secondary lines (`LESUFF <> 0`) exist for multi-count scenarios and are excluded from the main edit subfile.
- `LBCHPF.LCSTEL = 1` marks the batch as a cyclic (continuous) count. Cyclic count lines cannot be edited interactively (only deleted and re-counted), unless the CO402R switch overrides this.
- Deletion (choice code 4, xb4win): any line can be deleted via the delete window regardless of cyclic flag. The delete subroutine does not check `LCSTEL`. Only choice codes 2 and 7 are blocked for cyclic lines.

---

## 6. Special Conditions

- If the subfile is populated with lines from `LTELL2` (sorted by item number, `w_seqe = 'V'`), the sort key changes from line number to item number. Users can switch between sort sequences via the positioning fields (`b2line` / `b2vare`).
- Choice code 7 (sequential update) iterates forward through all remaining lines in the current sort sequence. The cyclic-count block applies to choice 7 identically to choice 2.
- When `b2line` is entered for positioning, `w_seqe` is forced to blank (line-number sequence). When `b2vare` is entered, `w_seqe` is forced to 'V' (item-number sequence).

---

## 7. Subprogram Calls Affecting Logic

| Called Program | Called From | Purpose / Effect on Logic |
|---|---|---|
| `LC500R` | LE100R, LE600R, LE610R, LE620R | Batch selection; non-blank return code exits the calling program entirely |
| `AA007R` | LE100R (xc2win, xc7win) | Displays blocking message for cyclic-count edit attempt; user cannot proceed |
| `CO402R` | LE100R (init) | Reads switch `u_702`; if `*on`, lifts the cyclic-count edit block |
| `LÅ900R` | LE610R, LE620R | Fetches next file-member sequence number for report output |
| `VG510R` | LE610R (sp_input) | F4 lookup for product group (overgroup) from `VOGRPF` |
| `LE610C` | LE610R | CL batch driver for variance report printing |
| `LE620C` | LE620R | CL batch driver for miscellaneous items count report |
| `LE600C` | LE600R | CL batch driver for count-list print |

---

## 8. Reference Tables

| Table | Role in Module |
|---|---|
| `LBCHPF` | Batch register — mandatory anchor for all count-list sessions |
| `LBC1PF` | Batch extension table — holds per-batch warehouse filter (`L1LAGE`) used by LE610R and LE600R |
| `LTELPF` | Count-list lines — primary data file for all read/update/delete operations |
| `VVARPF` | Item register — item descriptions displayed in count list |
| `VVENPF` | Unit-of-measure conversion — unit validation in `sjekk_enh` |
| `VOGRPF` | Product overgroup register — F4 inquiry in parameter screens |
