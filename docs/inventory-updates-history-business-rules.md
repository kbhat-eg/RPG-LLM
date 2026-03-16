# Business Rules: Inventory Updates and History (VL Module)

**System:** ASOFAK
**Module Prefix:** VL
**Programs Analyzed:** VL001R, VL002R, VL003R, VL700R, VL710R, VL711R, VL712R, VL713R, VL714R, VL715R, VL740R
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- Every item processed for inventory updates must exist in `VVARPF` (item register). An item not found in `VVARPF` causes immediate termination of the update (`w_feil = 'Ukj.Vare'`).
- Every warehouse code (`hllage`) referenced in a transaction must exist in `RA10PF` (warehouse codes). An unknown warehouse is logged to `VLWWPF` with `w_feil = 'Ukj.Lager'` and the update is aborted.
- The line type of a transaction must be defined in `VALTPF` (line type table). If not found, or if `VALTPF.VALLAG = 0`, the update is skipped entirely — the line type does not affect inventory.
- The order type of a transaction must be defined in `VAOTYPF` (order type table). If not found, or if `VAOTYPF.VAOLAG = 0`, the update is skipped — the order type does not affect inventory.
- The accumulator type (`VAOTYPF.VAOAKK`) must not be 0 or 5; both values bypass inventory posting.
- The system letter governing the transaction must be one of `F` (finance/order), `B` (purchase order), or `L` (logistics). Any other system letter causes immediate termination.
- Unit-of-measure conversion data must be present in `VVENPF` (unit conversion) for unit conversions to function. If the conversion factor is 0 or the unit is absent, the converted balance is returned as 0.
- For cost-price calculation (VL002R), the item must exist and must not be of type `'D'` (disposal/write-off). Type `'D'` terminates cost-price processing.
- For warehouse-entry-based price-group retrieval (VL711R, VL714R), the LDA (Local Data Area) positions 944–946 (firm), 947–950 (department), and 1001–1002 (warehouse) must be populated at runtime.
- For packing-slip-to-warehouse-entry (LI301R calling VL logic), `VAOTYPF.VAOAKK` must equal 2; all other accumulator types exit without inventory posting.

---

## 2. Validation Rules

### Item Validation (VL001R)
- `VVARPF` lookup by firm + item number: if `*in90 = *on` (not found) → `w_feil = 'Ukj.Vare'`, terminate.
- Item type `VVARPF.VVVTYP` must equal `'L'` (regular stocked item), unless the transaction's alternative code `hlaltk = 'SL'` (special logistics). Any other item type causes termination.
- Delivery type `LODTPF.HLLETY = '1'` (direct delivery) is excluded from inventory; terminate if found.
- Item sub-group `VGULAG` in `VGUOPF` must not equal 1 (sub-group 1 = no inventory control); terminate if `VGULAG = 1`.

### Warehouse Validation (VL001R)
- `RA10PF` lookup by firm + warehouse code: if not found → write error record to `VLWWPF`, set `w_feil = 'Ukj.Lager'`, terminate.

### Transaction Type Validation (VL001R)
- `VALTPF` lookup by line type: if not found → terminate; if `VALLAG = 0` → terminate (line type has no inventory effect).
- `VAOTYPF` lookup by order type: if not found → terminate; if `VAOLAG = 0` → terminate (order type has no inventory effect).
- `VAOTYPF.VAOAKK = 0` → terminate (no accumulator).
- `VAOTYPF.VAOAKK = 5` → terminate (accumulator type 5 is excluded from inventory update).

### Cost Price Validation (VL002R)
- If `p_type = 0` on entry → immediately goto avslutt, no processing performed.
- If item not found in `VVARPF` (`*in90`) → abort cost-price calculation.
- If `VVARPF.VVVTYP = 'D'` → abort cost-price calculation.

### Balance Reporting Validation (VL715R)
- If item type is `'D'` (disposal), `'T'` (transit), or `'S'` (special/procurement-only) → balance is not reduced by on-order quantity; returned balance = 0 for those types.
- If the requested sales unit is not found in any of the four conversion slots (w_enh1..w_enh4), balance is returned as 0.

### Price Group Validation (VL713R)
- The requested price group must exist in `VVPRPF` for the combination of firm + item + supplier + price group + unit + delivery type. If no matching record exists, `p_oprg` is returned as blank.
- Price records are filtered by `VVPRPF.VPPDAT` (valid-from date). If the current date is earlier than all valid-from dates, the most recent prior record is selected (date-ordered read with early exit on first match ≥ current date).

### Item-Number Rename Validation (VL740R)
- No validation prevents the rename; the calling program is responsible for ensuring both old and new item numbers are valid before calling VL740R.
- On merge (when calling program passes `p_dele = *on`), existence of the target item number is checked with `key02` (firm + new item number) in each forecast/ABC table. If found, old records are deleted rather than updated.

---

## 3. Configuration and Authorization Rules

- The feature flag `FSTSPF.FAALAG` controls whether warehouse-level data (`VLAGPF`) is consulted before the central item register (`VVARPF`). If `FAALAG = 1`, warehouse data takes precedence; VL715R reads `VLAGPF` first and falls back to `VVARPF` only for missing fields.
- Price groups are warehouse- and department-scoped: VL711R reads the LDA for firm (l_sfir), warehouse (l_lagr), and last-used department (l_savd) to determine which price group applies. If department-specific price group is not found, it falls back to department = 0.
- VL712R takes explicit warehouse and department parameters (rather than reading LDA), so callers that manage warehouse context directly must pass correct values.
- Supplier-linked pricing in VL713R uses a three-level fallback: (1) supplier for the specific warehouse, (2) primary supplier from `VVARPF.VVLDOR`, (3) supplier = 0 (generic price). The first level returning a match is used.
- Tracking information (edit date, edit time, edit user from LDA position 911–920) is written on every inventory update in version 6.10+ programs. The LDA user field at position 911–920 must be populated for audit fields to be meaningful.

---

## 4. Financial / Transactional Rules

### Inventory Balance Calculation (VL700R, VL715R)
- Available balance = `VLAGPF.VLBEHL` (physical stock) minus `VLAGPF.VLOORD` (quantity on open orders).
- If the sales unit of the transaction differs from the warehouse's base unit (`VVARPF.VVENHL`), the balance is multiplied by the applicable conversion factor from `VVENPF`. Conversion factors w_omr1..w_omr4 correspond to units w_enh1..w_enh4 respectively.
- If the conversion factor for the requested unit is 0, the returned balance is 0 (not an error — callers must handle zero balance).

### Cost Price Update (VL002R)
- VL002R calls VL710R to retrieve the current item type before computing cost price. The cost price calculation logic depends on the item type and price group (retrieved via VL711R/VL712R).
- Processing type `p_type` controls which cost-price formula is applied; type 0 is reserved to mean "no update required" and exits immediately.

### History Writing (VL001R)
- Approved transactions write to `VHISPF` (inventory history) and update running balances in `VLAGPF`.
- VL003R recalculates the running balance in `VHISPF` sequentially — it reads all history records for a given item/warehouse in chronological order and recomputes cumulative balance. No blocking; it is a repair/recalculation utility.

### Item Number Rename Across Tables (VL740R)
- The rename propagates to all of the following tables: `LTELPF` (stock-taking lines), `LODTPF` (purchase order lines), `LFDTPF` (purchase suggestion lines), `LLDTPF` (inventory transaction lines), `LBDTPF` (batch stock-taking lines), `HODTPF` (handheld terminal order lines), `MABDPF` (ABC/XYZ data), `MPROPF` (forecast data), `MMATPF` (material planning data), `MPRUPF` (forecast basis data), `MLTIPF` (logistics item data), `SIDTPF` (incoming invoice lines), `SISTPF` (purchase statistics).
- On a merge operation (`p_dele = *on`), if the target item already has records in `MABDPF`, `MPROPF`, `MMATPF`, `MPRUPF`, or `MLTIPF`, the old records for the source item are deleted (not renamed). For tables `LTELPF`, `LODTPF`, `LFDTPF`, `LLDTPF`, `LBDTPF`, `HODTPF`, `SIDTPF`, and `SISTPF`, the rename is always performed regardless of whether a merge target exists.
- All renamed/deleted records receive tracking fields (edit date, edit time, edit user) updated to current values (version 6.10+).

---

## 5. Status and Lifecycle Rules

- Item type `'L'` (regular stocked item) is the only type eligible for standard inventory updates via VL001R. All other item types (`'D'`, `'T'`, `'S'`, etc.) are excluded unless the special `hlaltk = 'SL'` override is present.
- Item type `'D'` (disposal) is excluded from cost-price calculation in VL002R.
- Item types `'D'`, `'T'`, and `'S'` suppress the "stock minus on-order" available balance calculation in VL715R. Balance for these types is returned as 0.
- Delivery type `'1'` (direct delivery) always bypasses inventory: no warehouse stock is created or updated for direct deliveries.
- Sub-group `VGULAG = 1` means the item group is configured as "no inventory control" — inventory updates are entirely suppressed for items in this sub-group.
- Accumulator type `VAOAKK = 0` means the order type has no defined accumulation behavior; inventory is not touched.
- Accumulator type `VAOAKK = 5` is a special exclusion type; inventory is not touched despite the order type otherwise being inventory-relevant.

---

## 6. Special Conditions

- **Unknown warehouse logging:** When a transaction references a warehouse code not in `RA10PF`, VL001R writes a record to `VLWWPF` before aborting. This creates a log of orphaned transactions that can be reviewed and reprocessed after the warehouse code is created.
- **SL override for item type:** The alternative code `hlaltk = 'SL'` bypasses the item type `'L'` restriction. This is used for special logistics items that are not type `'L'` but still require inventory posting.
- **Direct delivery bypass:** Delivery type `'1'` always exits; it is treated as a logistical arrangement that does not affect physical warehouse stock.
- **Merge vs rename in VL740R:** The distinction between a merge (`p_dele = *on`) and a simple rename is controlled entirely by the calling program. VL740R checks only whether the target item number already exists in each forecast/planning table; other tables (transaction lines, statistics) always receive the rename.
- **Handheld terminal orders use a different key:** `HODTPF` uses a composite key of firm + item number (not including warehouse), while other order line tables use firm + item. The rename for handheld records uses `hodtl2_key` (firm + item) and reassigns `hdvare` to the new item number.
- **Balance recalculation (VL003R):** This is a batch repair program. It does not block or validate; it simply resequences all history records for a warehouse/item and recomputes cumulative balances. It is intended for use after manual data corrections.
- **Commitment control:** Comments in VL001R (version notes) reference commitment control (`COMFLAG`). Commitment control must be active during inventory update batch jobs to ensure transactional integrity across `VHISPF` and `VLAGPF`.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Subprogram | Purpose | Blocking Effect |
|--------|-----------|---------|-----------------|
| VL001R | VL710R | Get item type, expiry date, supplier | Provides item type used for type validation |
| VL002R | VL710R | Get item type | Item type 'D' aborts cost price |
| VL002R | VL711R | Get price group from LDA | Price group drives cost formula |
| VL713R | VL721R | Get supplier for warehouse/item | Supplier used as key for price lookup |
| LI301R | VL logic | Warehouse entry from EDI packing slip | Only if VAOAKK = 2; otherwise exits |
| VL710R | (internal) | Reads VLAGPF then VVARPF | FAALAG=1 activates warehouse-first logic |
| VL715R | (internal) | Reads VLAGPF, VVARPF, VVENPF | Returns balance; type D/T/S returns 0 |

---

## 8. Reference Tables

| Table | Description | Key Fields Used |
|-------|------------|-----------------|
| `VVARPF` | Item register (master) | `VVFIRM`, `VVVARE` |
| `VVASPF` | Archived/deleted items | `VVFIRM`, `VVVARE` |
| `VLAGPF` | Warehouse stock records | `VLFIRM`, `VLVARE`, `VLLAGE` |
| `VHISPF` | Inventory transaction history | `VHFIRM`, `VHVARE`, `VHLAGE` |
| `VVENPF` | Unit-of-measure conversions | `VEFIRM`, `VEVARE`, `VEENHE` |
| `VALTPF` | Line type definitions | `VALFIRM`, `VALLITY`, `VALLAG` |
| `VAOTYPF` | Order type definitions | `VAOFIRM`, `VAOOTYP`, `VAOLAG`, `VAOAKK` |
| `VGUOPF` | Item sub-group definitions | `VGUFIRM`, `VGUGRP`, `VGULAG` |
| `RA10PF` | Warehouse code master | `RAFIRM`, `RALAGE` |
| `VLWWPF` | Unknown warehouse error log | `VWFIRM`, `VWLAGE` |
| `FSTSPF` | System configuration flags | `FAAFIR`, `FAALAG` |
| `VVPRPF` | Item price register | `VPFIRM`, `VPVARE`, `VPLDOR`, `VPPRGR`, `VPENHE`, `VPLETY` |
| `LTELPF` | Stock-taking register lines | `LEFIRM`, `LEVARE` |
| `LODTPF` | Purchase order lines | `LDFIRM`, `LDVARE` |
| `LFDTPF` | Purchase suggestion lines | `LFFIRM`, `LFVARE` |
| `LLDTPF` | Inventory transaction lines | `LSFIRM`, `LSVARE` |
| `LBDTPF` | Batch stock-taking lines | `LVFIRM`, `LVVARE` |
| `HODTPF` | Handheld terminal order lines | `HDFIRM`, `HDVARE` |
| `MABDPF` | ABC/XYZ classification data | `MBFIRM`, `MBVARE` |
| `MPROPF` | Forecast data | `MOFIRM`, `MOVARE` |
| `MMATPF` | Material planning data | `MMFIRM`, `MMVARE` |
| `MPRUPF` | Forecast basis data | `MRFIRM`, `MRVARE` |
| `MLTIPF` | Logistics item data | `MCFIRM`, `MCVARE` |
| `SIDTPF` | Incoming invoice lines | `SLFIRM`, `SLVARE` |
| `SISTPF` | Purchase statistics | `SIFIRM`, `SIVARE` |
