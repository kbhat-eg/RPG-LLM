# Business Rules: Data Fixes and Conversions (FX Module)

**System:** ASOFAK
**Module Prefix:** FX
**Programs Analyzed:** FX700R, FX702R, FX703R, FX751R, FX760R, FX800R, FX801R, FX820R, FX888R, FX901R
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- FX760R (unit conversion across all tables) requires that `FENKPF` (unit conversion master) contains at least one record. If `FENKPF` is empty at the start of the program, FX760R terminates immediately without processing any table. This is the only program in the module with a hard startup prerequisite.
- FX703R (VAT code assignment) reads item VAT flag `VVARPF.VVKMVA` to determine the correct VAT code. The item must exist in `VVARPF` for VAT code assignment to apply; order/invoice lines for items not in the item register are skipped.
- FX751R (item group copy) requires items to exist in either `VVARPF` (active items) or `VVASPF` (archived items) for group codes to be copied. If an item is absent from both, the order/invoice line's group fields are left unchanged.
- FX800R (clean up archived items that exist in active register) reads `VVARPF` first and uses those records as the basis for deletion from `VVASPF`. Items must exist in `VVARPF` for the corresponding `VVASPF` record to be deleted.
- FX801R (clean up orphaned warehouse records) reads `VLAGPF` and checks `VVARPF` for each item. No prerequisite beyond the existence of these two tables.
- FX901R (copy primary supplier to price records) reads `VVARPF` sequentially and updates `VVPRPF`. Both tables must be accessible for the program to run; no other prerequisites.

---

## 2. Validation Rules

### FX700R – Null Expiry Dates
- No validation or blocking. Processes all records in `VVARPF` and `VVASPF` unconditionally. Sets `VVARPF.VVNUDA` (production date) and `VVARPF.VVUDAT` (expiry date) to null for every record. This is a blanket reset with no filtering.

### FX702R – Clear Price Code on D/T Items
- Only processes items where `VVARPF.VVVTYP = 'D'` (disposal) or `VVVTYP = 'T'` (transit/through-put). All other item types are skipped.
- For eligible items, clears `VVARPF.VVKPRI` (price code) unconditionally.

### FX703R – Set VAT Code on Order/Invoice Lines
- Skips any order line (`FODTPF`) or invoice line (`SODTPF`) where the item number is blank.
- Skips any line where the VAT code is already set (non-blank). This prevents overwriting manually assigned VAT codes.
- If the item exists in `VVARPF`:
  - `VVARPF.VVKMVA = 1` → set VAT code to `'4'`
  - `VVARPF.VVKMVA` = any other non-zero value → set VAT code to `'H'`
  - `VVARPF.VVKMVA = 0` → set VAT code to `'3'` (default)
- If the item is not found in `VVARPF`, the default VAT code `'3'` is applied.

### FX751R – Copy Item Group Codes to Order/Invoice Lines
- If the item number on an order/invoice line is blank → skip.
- Item is looked up in `VVARPF` first. If not found, `VVASPF` is tried. If not found in either → group fields on the line are left unchanged (no error, no default value applied).
- For found items, the three group levels are copied: `VVARPF.VVOGRP` → order/invoice line's main group, `VVARPF.VVHGRP` → sub-group, `VVARPF.VVUGRP` → under-group.

### FX760R – Unit Conversion Across All Tables
- **Critical block:** At program startup, FX760R checks whether `FENKPF` contains any records. If the table is empty (`*eof` immediately after setll) → the program terminates without processing any conversion. This is the only startup prerequisite check in the FX module.
- For each record in each target table, the unit value is looked up in `FENKPF`. If the unit is not found in `FENKPF` → that field is left unchanged (no conversion, no error). Only units that exist in `FENKPF` are converted.
- Target tables processed: `VVARPF`, `VVASPF`, `VVPRPF`, `VVENPF`, `VTILPF`, `VHGRPF`, `VUGRPF`, `FRABPF`, `FPRIPF`, `FODTPF`, `SODTPF`.

### FX800R – Remove Archived Items Existing in Active Register
- Reads `VVARPF` sequentially. For each active item, performs a chain to `VVASPF` using firm + item number.
- If found in `VVASPF` → delete the `VVASPF` record. No confirmation, no logging.
- If not found in `VVASPF` → continue to next active item. No action, no error.

### FX801R – Remove Orphaned Warehouse Records
- Reads `VLAGPF` sequentially. For each warehouse record, performs a chain to `VVARPF` using firm + item number.
- If item NOT found in `VVARPF` (`*in91 = *on`) → deletes all `VLAGPF` records for that firm + item combination.
- If item found in `VVARPF` → no action for that warehouse record.

### FX820R – Copy H-Group/U-Group to Archive Fields
- No validation or blocking. Processes all records in both `VVARPF` and `VVASPF` unconditionally. Copies `VVHGRP` to `VVAHGR` and `VVUGRP` to `VVAUGR` for every record.

### FX888R – Test Program (Generate FNUFPF from Invoices)
- Hard-coded filters: firm = 200, year = 2012, specific date range, and `SOHEPF.SOBINR <> 214521` (one specific invoice number is always skipped).
- This is not production logic. The hard-coded values confirm it is a one-time test/migration program. It should not be run in production without reviewing and updating the filters.
- Reads `SOHEPF` (invoice headers) and writes to `FNUFPF` (new-format invoice file).

### FX901R – Copy Primary Supplier to Price Records
- Reads `VVARPF` sequentially. For each item, reads all `VVPRPF` records for that firm + item combination.
- Updates `VVPRPF.VPLDOR` (supplier number on price record) to match `VVARPF.VVLDOR` (primary supplier on item master).
- No validation of supplier existence. If `VVARPF.VVLDOR = 0` or blank, that value is propagated to all price records for the item, potentially clearing the supplier reference on all price lines.

---

## 3. Configuration and Authorization Rules

- All FX programs are batch utilities. No interactive screens, no user access control checks, and no AD005R calls. Access restriction must be enforced at the job/menu level before these programs are executed.
- FX888R is the only program with hard-coded firm/year/filter values. It must be treated as a development artifact and not included in standard operational menus.
- FX760R's behavior (terminate if no unit table entries) is the closest to a runtime configuration guard in this module. If the unit conversion table (`FENKPF`) has not been populated, the conversion run produces no changes, preventing partial conversion.

---

## 4. Financial / Transactional Rules

### VAT Code Assignment (FX703R)
- The VAT code assignment hierarchy is:
  1. If VAT code is already set on the line → do not change (preserve existing code).
  2. If item blank → skip entirely.
  3. If `VVKMVA = 1` → code `'4'` (reduced rate / exempt category 4).
  4. If `VVKMVA` is another non-zero value → code `'H'` (high rate / special category).
  5. Otherwise (not found or `VVKMVA = 0`) → code `'3'` (standard rate).
- This assignment applies to both order lines (`FODTPF`) and invoice lines (`SODTPF`) in the same run.

### Unit Conversion (FX760R)
- The conversion maps old unit codes to new unit codes using `FENKPF` as the conversion table. All unit-bearing fields across 11 tables are updated in a single run.
- The conversion is irreversible once committed. If the conversion was run with incorrect `FENKPF` data, all affected records would contain incorrect unit values. There is no rollback mechanism within the program.

### Supplier on Price Records (FX901R)
- FX901R synchronizes the supplier reference on price records with the item master's primary supplier. After running FX901R, `VVPRPF.VPLDOR` will equal `VVARPF.VVLDOR` for all items.
- If an item has multiple price records with different supplier-specific prices (supplier-differentiated pricing), FX901R overwrites all of those supplier references with the single primary supplier. This destroys supplier-differentiated pricing for any item where the price records had different supplier values.

---

## 5. Status and Lifecycle Rules

- FX programs do not maintain or check status flags. They are one-shot conversion utilities designed to be run once (or occasionally) to fix data consistency issues.
- FX800R and FX801R perform permanent deletions without status transitions. Records deleted by these programs cannot be recovered without a database backup.
- FX700R's blanket expiry-date null operation is designed to be run when the expiry-date feature is being deactivated or reset. Once run, all expiry date data in `VVARPF` and `VVASPF` is lost.

---

## 6. Special Conditions

- **FX760R empty-table termination:** The check for an empty `FENKPF` is a safeguard that prevents partial unit conversion. However, it does not validate that `FENKPF` contains the correct mappings — only that it is non-empty. A `FENKPF` with incorrect mappings would still cause the conversion to run and corrupt unit data across all 11 tables.
- **FX703R preserves existing VAT codes:** The skip condition for already-set VAT codes ensures that manually overridden codes are not destroyed by a re-run. FX703R is safe to run multiple times; only lines without a VAT code are affected on subsequent runs.
- **FX751R fallback to VVASPF:** The fallback to the archived item register (`VVASPF`) means that archived items still contribute their group codes to order/invoice lines. This is intentional — historical transactions may reference archived items, and their group classification should still be available.
- **FX801R item number scope:** FX801R deletes all `VLAGPF` records for an orphaned item (all warehouses). This is a complete removal of warehouse records for items that no longer exist in the active item register. It does not check for pending transactions against those warehouse records.
- **FX901R propagates zero supplier:** If `VVARPF.VVLDOR = 0` (no primary supplier assigned), FX901R sets `VVPRPF.VPLDOR = 0` for all price records of that item. This is a data-risk: running FX901R on a system where primary suppliers have not been fully assigned will clear supplier references from all price records for those items.
- **FX888R production risk:** The hard-coded values in FX888R (firm 200, year 2012, specific invoice exclusion) make this program dangerous to run in a current environment where those parameters are no longer valid. It should be treated as a historical artifact.

---

## 7. Subprogram Calls Affecting Logic

All FX programs operate through direct file I/O (RPG CHAIN, READE, UPDATE, DELETE). No subprogram calls that affect blocking logic were found in the analyzed programs. All blocking is implemented inline within each program.

---

## 8. Reference Tables

| Table | Description | Key Fields Used |
|-------|------------|-----------------|
| `VVARPF` | Active item register | `VVFIRM`, `VVVARE`, `VVVTYP`, `VVKPRI`, `VVKMVA`, `VVHGRP`, `VVUGRP`, `VVOGRP`, `VVLDOR`, `VVNUDA`, `VVUDAT` |
| `VVASPF` | Archived/deleted item register | `VVFIRM`, `VVVARE` (same structure as VVARPF) |
| `VVPRPF` | Item price register | `VPFIRM`, `VPVARE`, `VPLDOR` |
| `VVENPF` | Unit-of-measure conversions | `VEFIRM`, `VEVARE`, `VEENHE` |
| `VLAGPF` | Warehouse stock records | `VLFIRM`, `VLVARE`, `VLLAGE` |
| `VTILPF` | Item supply/delivery parameters | `VTFIRM`, `VTVARE` |
| `VHGRPF` | Item H-group (sub-group) definitions | `VHFIRM`, `VHGRP` |
| `VUGRPF` | Item U-group (under-group) definitions | `VUFIRM`, `VUGRP` |
| `FRABPF` | Price/discount table | `FAFIRM` |
| `FPRIPF` | Price list | `FPFIRM` |
| `FODTPF` | Sales/order lines | `FOFIRM`, `FOBILAG`, `FOVARE` |
| `SODTPF` | Invoice lines | `SOFIRM`, `SOBILAG`, `SOVARE` |
| `FENKPF` | Unit conversion master (source of truth for unit codes) | `FEFIRM`, `FEENHE` |
| `SOHEPF` | Invoice headers | `SOFIRM`, `SOBILAG` |
| `FNUFPF` | New-format invoice file (FX888R target) | `FNFIRM`, `FNBILAG` |
