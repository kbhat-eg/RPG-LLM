# Business Rules: Item Price Management (JX Module)

**System:** ASVADM
**Module Prefix:** JX
**Programs Analyzed:** JX100R, JX420R, JX500R, JX600R, JX720R
**NXKORR Overrides:** NXKORR/rpgsrc/JX500R.MBR (price inquiry override)
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- Item prices are stored in `JVPRPF` keyed by item number + vendor number + from-date + status. JX100R and JX500R both use logical views on `JVPRPF`.
- The item-vendor link (`JVLEPF`) must exist for a price to be displayed or used. JX500R (NXKORR): if `JVLEPF` is not found for the item + vendor combination → the price row is skipped entirely (`iter`). No price is shown for an unlinked vendor.
- Vendor master data is in `JLEVPF` (vendor register). JX420R validates the price-giving vendor against `JLEVPF` before allowing deletion. JX600R validates both the price-giving vendor and the specific vendor against `JLEVPF`.
- Product group hierarchy is required for JX600R print filtering: overgroup in `VOGRPF`, main group in `VHGRPF`, undergroup in `VUGRPF`. Each is validated by CHAIN if entered.
- Priority vendor data is fetched via `JV164R`, which returns up to three priority vendor numbers. This is used by JX500R (NXKORR) when the CO402R switch `BRUK_PRIORITERT_LEVERANDØR_F8` is active.
- Logistical item (Trelast) NOBB numbers are retrieved from `JVKLPF` (NXKORR JX500R, version 6.33). If the item is in `JVKHPF` (logistical item register), a SQL query to `JVKLPF` fetches the NOBB number for display.

---

## 2. Validation Rules

### JX100R — Item Price Maintenance

**Write-Protection via CO402R (version 6.20):**
- CO402R switch for `AVALPF` controls which fields are write-protected on the price record. If the switch is active for a given field, that field cannot be edited. The specific switch key varies by installation.

**Item-Vendor Link Creation Check (version 6.32):**
- When creating a new price record: if `JVLEPF` (item-vendor link) does not already exist for the item + vendor combination → the system checks whether to create it or block. The exact blocking condition depends on the DDS/screen indicators not fully visible in the first 300 lines.

### JX420R — Delete Prices for Price-Giving Vendor

- `b1pgiv = 0` (no price-giving vendor entered): `*in31` blocks the deletion. A price-giving vendor is mandatory.
- Price-giving vendor not found in `JLEVPF`: `*in32` is set with an error indicator. However, the user can confirm override via `AA007R` (confirmation dialog) and proceed with deletion even if the vendor is not in the vendor master.
- Specific vendor (when entered separately) not found in `JLEVPF`: `*in33` blocks unconditionally — no override is available for an unknown specific vendor.
- On confirmation, calls `JX421R` to perform the actual record deletions.

### JX500R — Price Inquiry / Selection (NXKORR override)

The following conditions cause a price row to be **skipped** (`iter`) during subfile population:

**Expired item-vendor link:**
- `JVLEPF.JVLUDA <> *loval and JVLEPF.JVLUDA < w_udat` (expiry date is set and is in the past) → skip.

**Missing item-vendor link:**
- `not %found(jvlelr)` (no `JVLEPF` record for item + vendor) → skip.

**Expired price date (version 6.33):**
- `JVPRPF.JXTDAT <> *loval and JXTDAT < w_udat` (to-date is set and is in the past) → skip (changed from version 6.32 which showed expired prices in red).

**Old price without to-date (version 8.03/8.04):**
- SQL query: `select max(jxgdat) from jvprpf where jxvare = :p_vare and jxldor = :jxldor and jxtdat = '0001-01-01'`.
- If `JVPRPF.JXTDAT = *loval` (no to-date) AND `JXGDAT < w_udat` (from-date is in the past) AND `JXGDAT < w_gdat_max` (not the most recent price without a to-date) → skip.
- This rule ensures that only the most recent open-ended price for a vendor is shown, preventing display of all historical prices that have no explicit end date.

**Priority vendor filter (version 8.02 NXKORR):**
- CO402R switch `BRUK_PRIORITERT_LEVERANDØR_F8`: if active (`u_pril = *on`) and `*in40 = *on` (filter mode enabled), prices for vendors not in the priority vendor list (up to 3 vendors returned by `JV164R`) are skipped.
- The priority filter is toggled by F8 (`vis_prio` subroutine). When `u_pril = *on` and user presses F8, `*in40` toggles between on (filtered) and off (unfiltered). When filter is off, all vendors are shown regardless of priority.

### JX600R — Price List Print Parameters

- `b1rtyp` (report type): must be 1, 2, or 3. Any other value → `*in31` blocks.
- `b1pldo = 0` (no price-giving vendor entered): `*in32` blocks.
- Price-giving vendor not in `JLEVPF`: `*in33` blocks.
- Invalid date format → `*in34` blocks.
- Invalid comparison operator (must be EQ, GT, or LT): `*in35` blocks.
- Specific vendor not in `JLEVPF`: `*in36` blocks.
- Overgroup not in `VOGRPF` → `*in37` blocks.
- Main group (hoved group) not in `VHGRPF` → `*in38` blocks.
- Undergroup not in `VUGRPF` → `*in39` blocks.

### JX720R — Vendor Price Validation

- Validates that a vendor number exists in `JLEVPF` (accounts-payable vendor register).
- Checks whether a price exists in `JVPRPF` for the combination of item + vendor.
- Returns `p_stat = '1'` only if both conditions are met (vendor found in `JLEVPF` AND price record found in `JVPRPF`). Used by calling programs to gate price-dependent operations.

---

## 3. Configuration and Authorization Rules

- CO402R switch `BRUK_PRIORITERT_LEVERANDØR_F8` (JX500R NXKORR, version 8.02): if active for the firm, the F8 key becomes available and the priority-vendor filter is enabled by default (`*in40 = *on` on entry). When the switch is not active, `u_pril = *off` and the F8 key is hidden — all vendors are always shown.
- CO402R `AVALPF` write-protection (JX100R, version 6.20): specific price fields can be protected against editing on a per-installation basis. The AVALPF register defines which fields are read-only.
- LDA position 944–946 (`l_firm`): all JX programs scope to the active firm from LDA. Cross-firm price access is not possible.
- LDA position 1001–1002 (`l_lage`, JX500R NXKORR): warehouse context is passed to `JV164R` when fetching priority vendors. Priority vendors may differ per warehouse.
- LDA position 931–933 (`l_filg`): file group identifier, passed to CO402R for switch lookups.

---

## 4. Financial / Transactional Rules

- `JVPRPF.JXPRIS` is the item price field returned to callers via JX500R (`p_pris`). The selected price is the one on the row the user chooses (choice code 1 in the subfile).
- `JVPRPF.JXGDAT` (from-date) is returned as `p_gdat` (version 7.02). Callers use this date to determine the applicable price validity period.
- `JVPRPF.JXRSTS = 1` marks a price as locally created (not imported from a central price catalogue). JX500R marks such rows with `b1rkod = 'L'` for visual distinction.
- Deletion via JX420R/JX421R removes all price records for the specified price-giving vendor (and optionally a specific vendor). This is an irreversible bulk deletion — no soft-delete or archive.

---

## 5. Status and Lifecycle Rules

- `JVLEPF.JVLUDA` (vendor link expiry date): when set to a past date, the item-vendor relationship is considered expired. JX500R skips all prices for expired vendor links. JX500R version 6.31 used to show expired links in red; version 6.32 changed this to complete suppression.
- `JVPRPF.JXTDAT` (price to-date): when set to a past date, the price is expired. Version 6.33 changed behaviour from showing expired prices (with indicator) to skipping them entirely.
- Version 8.03/8.04 rule: a price without a to-date (`JXTDAT = *loval`) that is not the most recent price for the vendor is also suppressed. This prevents the display of stale open-ended prices that have been superseded by a newer price from the same vendor.

---

## 6. Special Conditions

- Logistical / Trelast items (JX500R NXKORR, version 6.33): if the item exists in `JVKHPF` (logistical item register), the program queries `JVKLPF` using SQL to retrieve the NOBB number (`JVQUNO`) linked to the item + vendor. For non-logistical items, the NOBB number is retrieved from `JVARPF.JVNOBB` directly. This NOBB number is displayed in the subfile for reference.
- The `b_logi` flag (`*on` = logistical item) controls which SQL query is used for NOBB retrieval. If neither query returns a result, NOBB is displayed as 0.
- JX420R override confirmation: when the price-giving vendor is not found in `JLEVPF` (`*in32`), the user is prompted via `AA007R` (yes/no dialog). Confirming 'yes' allows deletion to proceed despite the missing vendor master record. This override is only available for the price-giving vendor — not for the specific vendor (`*in33` is unconditional).

---

## 7. Subprogram Calls Affecting Logic

| Called Program | Called From | Purpose / Effect on Logic |
|---|---|---|
| `CO402R` | JX100R (v6.20), JX500R NXKORR (v8.02) | Reads AVALPF write-protection and `BRUK_PRIORITERT_LEVERANDØR_F8` switches |
| `JV164R` | JX500R NXKORR (v8.02) | Returns up to 3 priority vendor numbers for the item + warehouse; used by filter |
| `AA007R` | JX420R | Confirmation dialog for deletion when price-giving vendor not in `JLEVPF` |
| `JX421R` | JX420R | Performs the actual price record deletions after parameter validation |

---

## 8. Reference Tables

| Table | Role in Module |
|---|---|
| `JVPRPF` | Item price register — primary data file keyed by item + vendor + from-date + status |
| `JVLEPF` | Item-vendor link register — must exist for a vendor-price combination to be shown; expiry date `JVLUDA` |
| `JLEVPF` | Accounts-payable vendor register — vendor master; required for JX420R and JX600R validation |
| `JVARPF` | Item attribute register — NOBB number for non-logistical items (JX500R) |
| `JVKHPF` | Logistical item register — presence indicates a Trelast item; triggers `JVKLPF` lookup |
| `JVKLPF` | Logistical item-vendor link — NOBB number (`JVQUNO`) for logistical items |
| `VOGRPF` | Product overgroup register — overgroup validation in JX600R |
| `VHGRPF` | Product main-group register — main-group validation in JX600R |
| `VUGRPF` | Product undergroup register — undergroup validation in JX600R |
