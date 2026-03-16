# Business Rules: Offer Price Management (VT Module)

**System:** ASOFAK
**Module Prefix:** VT
**Programs Analyzed:** VT100R, VT110R, VT120R, VT600R
**NXKORR Overrides:** None found
**Date:** 2026-03-16

---

## 1. Prerequisites / Master Data Requirements

- Offer prices are stored in `VTILPF` (offer price register). VT100R and VT110R both maintain this file via multiple logical views keyed by item + vendor + delivery type, by delivery type, by from-date, by to-date, by campaign, and by price group.
- Customer club prices are stored in `VKLUPF` (customer club price register). VT120R maintains `VKLUPF`. This is a separate table from `VTILPF` — offer prices and customer club prices are distinct record types with distinct logical views.
- Purchase periods are stored in `VTI2PF`. VT100R and VT110R also maintain purchase period records alongside offer prices.
- Item records must exist in `VVARPF` (item register) for item validation in VT100R and VT110R. An item not found in `VVARPF` will cause the item description fields to be blank, though the exact blocking behaviour depends on DDS indicators not fully visible in the declaration sections.
- Delivery types must be defined in `VLTPPF` for validation on the delivery-type field. Unit-of-measure values are validated against `VVENPF`.
- Price group codes are defined in `RA30PF`. VT100R's price group field is validated against `RA30PF` when populated.
- Product group hierarchy (`VOGRPF`, `VHGRPF`, `VUGRPF`) is referenced for F4 inquiry from VT600R's parameter screen.
- Vendor (price-giving vendor) is in `RLEVPF` (vendor register) for VT100R.
- For warehouse validation in VT600R, `RA510R` (warehouse inquiry program) is called; the warehouse code must be known to `RA510R`.

---

## 2. Validation Rules

### VT100R — Offer Price Maintenance

VT100R is a large interactive maintenance program with multiple logical views and supports creating, changing, and viewing offer price records in `VTILPF`. Key blocking rules (inferred from declarations and change log):
- Item lookup in `VVARPF`: if item not found, description is blank. Whether creation is blocked depends on DDS indicators.
- Unit-of-measure (`VVENPF`): if an entered unit is not found in `VVENPF`, the unit field generates an error indicator.
- Delivery type (`VLTPPF`): if entered delivery type is not in `VLTPPF`, the delivery-type field generates an error indicator.
- Price group (`RA30PF`): if entered price group code is not in `RA30PF`, the field is flagged invalid.
- From-date / to-date: standard date validation applies. A to-date earlier than from-date would be illogical but is validated at the DDS/screen level.
- Version 8.01 (`PRG2`): price group field extended from 1 to 2 positions. Records created before 8.01 with single-character price groups remain valid; new records use 2-character codes.

### VT110R — Offer Price per Item (called from item maintenance)

- Same core validation as VT100R — identical file set, same logical views on `VTILPF`.
- Entry point: called from item maintenance with the item number pre-filled. The item is fixed and cannot be changed within this program.
- Unit validation via `VENHPF` (rather than `VVENPF` in VT100R). If the unit is not found in `VENHPF` → the unit field is flagged.

### VT120R — Customer Club Price Maintenance

- Same structure and validation pattern as VT100R, but operates on `VKLUPF` (customer club price register) rather than `VTILPF`.
- Customer club prices are keyed differently (by customer/club rather than by vendor/price group), but delivery type, unit, and date validations apply identically.

### VT600R — Offer Price Print Parameters

**At least one search criterion must be supplied — all three empty blocks the run:**
- If `b1dato = 0` AND `b1kamp = *blank` AND `b1fper = 0` AND `b1tper = 0` → `*in31` is set, blocking the print run.

**Only one search criterion type at a time:**
- `b1dato <> 0 and b1kamp <> *blank` → `*in32` blocks (date and campaign combined).
- `b1dato <> 0 and b1fper <> 0` → `*in32` blocks (date and period combined).
- `b1kamp <> *blank and b1fper <> 0` → `*in32` blocks (campaign and period combined).

**Date format validation:**
- If `b1dato <> 0`: `*dmy test(d) b1dato` → if `*in33` set, the date is invalid and the run is blocked.
- If `b1fper <> 0`: `*dmy test(d) b1fper` → if `*in34` set, the from-period date is invalid.
- If `b1tper <> 0`: `*dmy test(d) b1tper` → if `*in35` set, the to-period date is invalid.

**Period range consistency:**
- If `b1fper <> 0` and `b1tper = 0` → `b1tper` is automatically set to `b1fper` (from = to). This is an auto-correction, not a block.
- If `b1tper <> 0` and `b1fper = 0` → `*in36` blocks (to-period without from-period is not allowed).
- If from-period > to-period (after date conversion): `*in37` blocks.

**Product group range consistency:**
- Composite group key: `w_fvgr = b1fogr * 100000 + b1fhgr * 1000 + b1fugr`; `w_tvgr = b1togr * 100000 + b1thgr * 1000 + b1tugr`.
- If `w_fvgr > w_tvgr` → `*in38` blocks (from-group greater than to-group across the composite hierarchy).

**Warehouse validation (version 5.70):**
- If `b1lage <> 0` (warehouse entered): calls `RA510R` with `w_firm` and `b1lage`. If `RA510R` does not find the warehouse, `*in39` blocks.

**F4 Inquiry (spØrring subroutine):**
- F4 on `B1FOGR` (from overgroup): calls `VG510R`; resets `b1fhgr = 0` and `b1fugr = 0` before calling (clears subordinate group fields).
- F4 on `B1TOGR` (to overgroup): calls `VG510R`; resets `b1thgr = 0` and `b1tugr = 0`.
- F4 on `B1FHGR` (from main group): calls `VG511R` with overgroup context; resets `b1fugr = 0`.
- F4 on `B1THGR` (to main group): calls `VG511R`; resets `b1tugr = 0`.
- F4 on `B1FUGR` (from undergroup): calls `VG512R` with overgroup + main group context.
- F4 on `B1TUGR` (to undergroup): calls `VG512R`.
- F4 on `B1LAGE` (warehouse): calls `RA510R` — same as the warehouse validation call.

---

## 3. Configuration and Authorization Rules

- LDA position 944–946 (`l_firm`): all VT programs scope to the active firm. Cross-firm access is not possible.
- VT100R uses multiple logical views keyed to allow browsing by item, vendor, delivery type, campaign, and price group simultaneously. The active sort sequence is controlled by the display file (DDS F-keys) rather than by RPG configuration switches.
- VT600R version 5.70 adds warehouse as a filterable parameter. This is backward-compatible — if `b1lage = 0`, no warehouse filter is applied.
- Version 8.01 (`PRG2`): price group field expanded from 1 to 2 positions in `VTILPF`. The data structure overlay positions are adjusted (`d_fogr` at position 39, `d_thgr` at position 46, etc.). Any integration writing to `VTILPF` must use the updated field widths.

---

## 4. Financial / Transactional Rules

- Offer prices in `VTILPF` define the special (promotional) price for an item+vendor combination during a specific period or campaign. These prices override standard prices when active.
- Customer club prices in `VKLUPF` define special prices for specific customer clubs/groups. These are structurally separate from vendor offer prices.
- `VTI2PF` (purchase period) records define the purchasing window associated with an offer price. When an offer price is created in VT100R, an associated purchase period record is also written to `VTI2PF`.
- The offer price print (VT600R → VT600C) generates a report of current offer prices filtered by date, campaign, period, product groups, and warehouse. This report is read-only and does not alter prices.

---

## 5. Status and Lifecycle Rules

- Offer prices are date-bounded by from-date and to-date fields in `VTILPF`. Prices outside the active date range are not automatically removed — they remain in the table and must be archived or deleted manually.
- Campaign codes link offer prices to specific promotions. Multiple price records can share the same campaign code; VT600R can filter and print all prices for a given campaign.
- The `VTI2PF` purchase period is the buying window (when orders must be placed). The offer price itself may have a different validity window (when the price applies to deliveries). Both are stored separately.

---

## 6. Special Conditions

- Composite group key validation in VT600R: the three-level group hierarchy (overgroup × 100000 + main group × 1000 + undergroup) is collapsed into a single numeric key for range comparison. This means a from-group of (1, 2, 0) = 100,2000 and a to-group of (1, 1, 999) = 101,999 would incorrectly show the from-group as greater. Ranges should be entered with from ≤ to at all three levels independently to avoid misleading `*in38` blocks.
- F4 on overgroup fields resets subordinate group fields to 0 (VT600R). This prevents inconsistent range combinations where an overgroup is entered after a main group or undergroup has already been populated.
- The period auto-correction rule (from-period set, to-period blank → to-period = from-period) is a silent auto-fill. Users who enter only a from-period will get a single-date range without any warning or confirmation.
- Warehouse (`b1lage`) is stored as a 2-digit numeric code (positions 54–55 in the `d_list` parameter structure, version 5.70). The warehouse parameter was added to allow offer price printing scoped to a specific warehouse rather than all warehouses.

---

## 7. Subprogram Calls Affecting Logic

| Called Program | Called From | Purpose / Effect on Logic |
|---|---|---|
| `VG510R` | VT600R (spØrring) | F4 lookup for overgroup from `VOGRPF`; clears subordinate group fields |
| `VG511R` | VT600R (spØrring) | F4 lookup for main group from `VHGRPF` with overgroup context |
| `VG512R` | VT600R (spØrring) | F4 lookup for undergroup from `VUGRPF` with overgroup + main group context |
| `RA510R` | VT600R (spØrring, sjekk_input) | Warehouse inquiry — validates `b1lage`; blocks with `*in39` if not found |
| `VT600C` | VT600R | CL batch driver for offer price report generation |

---

## 8. Reference Tables

| Table | Role in Module |
|---|---|
| `VTILPF` | Offer price register — primary data file for VT100R and VT110R |
| `VTI2PF` | Purchase period register — associated buying window per offer price record |
| `VKLUPF` | Customer club price register — separate offer prices for customer clubs (VT120R) |
| `VVARPF` | Item register — item validation and description lookup |
| `VLTPPF` | Delivery type register — delivery type validation in VT100R/VT110R |
| `VVENPF` | Unit-of-measure conversion — unit validation in VT100R |
| `VENHPF` | Unit register — unit validation in VT110R |
| `RA30PF` | Price group register — price group code validation |
| `RLEVPF` | Vendor register — price-giving vendor validation in VT100R |
| `VOGRPF` | Product overgroup register — F4 inquiry and range validation in VT600R |
| `VHGRPF` | Product main-group register — F4 inquiry and range validation in VT600R |
| `VUGRPF` | Product undergroup register — F4 inquiry and range validation in VT600R |
