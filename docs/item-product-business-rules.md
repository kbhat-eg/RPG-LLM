# Business Logic for Item / Product Master

This module covers maintenance of the product (item) master, item classification, price maintenance, price group management, and mass price adjustment in the ASOFAK/ASLAGR systems. The programs covered are VV100R (item list/overview), VV101R (item detail maintenance), VV102R (item detail — screen 2: VAT, structure, alternate), VV105R (item price adjustment — per item), VP110R (item price maintenance — per item), VP130R (price group warehouse/department mapping), VP150R (mass price adjustment parameters), VP410R (delete prices from CSV file), and VP411R (batch price deletion processor).

---

## Prerequisites / Master Data Requirements

1. **Item Must Exist in VVARPF Before Warehouse Record Allowed**
   - Logic: VV100R and LL100R (warehouse module) require that the item exists in VVARPF before a warehouse record can be created for it in VLAGPF.
   - File: VVARPF
   - Field: VVVARE
   - Condition: `not %found(VVARPF)` → warehouse creation blocked.

2. **Item Group Hierarchy Must Be Valid (VV100R search)**
   - Logic: When filtering by item group, if a sub-group (VVOGRP) is specified without a main group (VVHGRP), indicator `*in31` is set (invalid search). If a sub-sub-group (VVUGRP) is specified without the sub-group, `*in32` is set.
   - File: VVARPF
   - Fields: VVOGRP (over/main group), VVHGRP (main group), VVUGRP (sub group)
   - Condition: `b2ogrp=0 and b2hgrp≠0` → `*in31`; `b2hgrp=0 and b2ugrp≠0` → `*in32`.

3. **Alternativ Undergruppe (Alternative Sub-group) Must Exist (VV102R)**
   - Logic: If alternative sub-group (c2ahgr/c2augr) is entered, it must exist in VAUGPF. If not found, `*in39 = *on` blocks save.
   - File: VAUGPF
   - Fields: c2ahgr (main), c2augr (sub)
   - Condition: `not %found(VAUGPF)` → `*in39 = *on`, save blocked.

4. **Alternativt Varenummer (Alternate Item Number) Must Exist (VV102R)**
   - Logic: If c2avar (alternate item number) is entered, it must exist in VVARPF. If not found, `*in40 = *on` blocks save.
   - File: VVARPF
   - Field: c2avar
   - Condition: `not %found(VVARPF)` → `*in40 = *on`.

5. **Kontohenvisning (Account Reference) Must Exist (VV102R)**
   - Logic: If c2khev (account reference code) is entered, it must exist in VKHVPF. If not found, `*in47 = *on` blocks save.
   - File: VKHVPF
   - Field: c2khev
   - Condition: `not %found(VKHVPF)` → `*in47 = *on`.

6. **Avgiftskode (VAT Code) Must Exist (VV102R)**
   - Logic: The VAT code (c2kmva) is mandatory and must exist in VMVAPF. If not found, `*in34 = *on` blocks save.
   - File: VMVAPF
   - Field: c2kmva
   - Condition: `not %found(VMVAPF)` → `*in34 = *on`.

---

## Validation Rules

7. **Item Type Change L→S Blocked if Warehouse Records Exist (VV101R)**
   - Logic: If the item type (VVVTYP) is being changed from 'L' (stock/lager) to 'S' (special order/skaffevare) and warehouse records (VLAGPF) already exist for this item, the change is blocked. The `b_endr_vtyp` flag tracks that a type change was attempted.
   - File: VLAGPF
   - Fields: VVVTYP, VLVTYP
   - Condition: `VVVTYP = 'L' → 'S' and VLAGPF records exist` → `*in62` (or similar) set, change blocked (switch 6.14).

8. **Varetype Change Rules (VV101R — switch 6.23)**
   - Logic: Item type changes are restricted to valid combinations. Not all type transitions are permitted. Valid types are L (stock), S (special order), D (direct delivery), T (service/tjeneste). Arbitrary changes between incompatible types are blocked.
   - Condition: Invalid type-to-type transition → error indicator set.

9. **Prisgruppe (Price Group) Access Control (VV101R — switch 5.70)**
   - Logic: Access to the price group field is controlled via RA30PF (user code/permission register). The user must have a valid permission code to modify the price group.
   - File: RA30PF
   - Condition: No permission for price group modification → field protected.

10. **Desimaler (Decimal Places) Must Be 0, 1, 2, or 3 (VV102R)**
    - Logic: The quantity decimal places field (c2desi) must be one of the four valid values: 0, 1, 2, or 3. Any other value blocks save with `*in37`.
    - Field: VVDESI
    - Condition: `c2desi not in (0,1,2,3)` → `*in37 = *on`.

11. **Struktur-kode (Structure Code) Must Be 0 or 20 (VV102R)**
    - Logic: The structure code (c2kstr) must be either 0 (no structure) or 20 (BOM/bill-of-materials). Any other value blocks save with `*in38`.
    - Field: VVKSTR
    - Condition: `c2kstr ≠ 0 and c2kstr ≠ 20` → `*in38 = *on`.

12. **Only One Warehouse View Open at Once (VV101R — switch 6.22)**
    - Logic: If only one warehouse is defined in the system, the warehouse item maintenance screen is opened simultaneously with the item detail to streamline data entry.
    - Condition: Single-warehouse configuration → warehouse screen opens automatically.

13. **Blank Price Group Rows Skipped When Switch Active (VV101R — switch 7.02)**
    - Logic: If switch `u_702` is active, item price rows with a blank price group (prgr = ' ') are skipped when displaying prices, to reduce clutter.

14. **VP150R: Selection Criteria Required**
    - Logic: At least one selection criterion must be specified for mass price adjustment: leverandør (supplier), overgruppe, hovedgruppe, undergruppe, or varenummer range. All blank → `*in31 = *on`, run blocked.
    - Condition: `b1ldor=0 and b1ogrp=0 and b1hgrp=0 and b1ugrp=0 and b1varf=0 and b1vart=0` → `*in31 = *on`.

15. **VP150R: Supplier Must Exist in RLEVPF**
    - Logic: If a supplier number is entered, it must exist in RLEVPF. If not, `*in32 = *on` blocks the run.
    - File: RLEVPF (supplier register)
    - Condition: `*in90 = *on` after chain → `*in32 = *on`.

16. **VP150R: Item Group Hierarchy Validation**
    - Logic: Main group, sub-group, and under-group must each exist in their respective registers (VOGRPF, VHGRPF, VUGRPF). Invalid entries set indicators `*in33`, `*in34`, `*in35` respectively.
    - Files: VOGRPF, VHGRPF, VUGRPF
    - Condition: Each level validated; failure blocks run.

17. **VP150R: Item Range — "From" Must Not Exceed "To"**
    - Logic: If `b1varf > b1vart`, `*in36 = *on` blocks the run.
    - Condition: `b1varf > b1vart` → `*in36 = *on`.

18. **VP150R: Date Must Be Valid and Not in the Past**
    - Logic: The effective date must be a valid date format (TEST(d)); if invalid, `*in37 = *on`. If the date is before today (`w_dato < w_udat`), `*in38 = *on` blocks the run.
    - Condition: Invalid date → `*in37`; past date → `*in38`.

19. **VP150R: Adjustment Factor Required**
    - Logic: At least one adjustment percentage must be non-zero: sale price % (b1spro), cost price % (b1kpro), inbound price % (b1ipro), or base price % (b1bpro), or a fixed deduction (b1dekn). All zero → `*in39 = *on`.
    - Condition: All adjustment fields zero → `*in39 = *on`.

20. **VP150R: Cannot Specify Both Percentage and Fixed Deduction**
    - Logic: If a sale price percentage (b1spro) and a fixed deduction (b1dekn) are both non-zero, this is a conflicting specification: `*in40 = *on`.
    - Condition: `b1spro ≠ 0 and b1dekn ≠ 0` → `*in40 = *on`.

21. **VP150R: Cannot Exclude All Price Codes**
    - Logic: Three flags control which price codes to exclude from adjustment: orpv (original price), ntpv (new price), bupv (supplier price). All three cannot be excluded simultaneously: `*in41 = *on`.
    - Condition: `b1orpv=1 and b1ntpv=1 and b1bupv=1` → `*in41 = *on`.

22. **VP150R: Delivery Type Must Be Blank or Numeric**
    - Logic: The delivery type filter (b1lety) must be blank or contain only numeric characters. Non-numeric input sets `*in42 = *on`.
    - Condition: `%check(digits:b1lety) > 0` → `*in42 = *on`.

23. **VP150R: Price Group Must Exist in RA30PF**
    - Logic: If a price group filter (b1prgr) is specified, it must exist in RA30PF. If not found, `*in43 = *on` blocks the run.
    - File: RA30PF
    - Condition: `*in90 = *on` after chain to RA30PF → `*in43 = *on`.

24. **VV105R: Supplier Must Exist in Item's Price List**
    - Logic: Before running a price adjustment for a specific item, VV105R checks that the entered supplier (b1ldor) has at least one price entry in VVPRPF for this item. If not, `*in35 = *on` blocks the run.
    - File: VVPRPF (item price register)
    - Field: VPLDOR (supplier on price record)
    - Condition: No VVPRPF record found for this item+supplier → `*in35 = *on`.

25. **VV105R: Date and Adjustment Rules (Same as VP150R)**
    - Logic: Same date validity check (TEST(d) → `*in31`), past date check (`w_dato < w_udat` → `*in32`), at least one adjustment non-zero (`*in33`), and cannot combine percentage+deduction (`*in34`).

---

## Configuration and Authorization Rules

26. **VP130R: Price Group Mapping Validation**
    - Logic: When creating a new price group warehouse/department mapping in VPGRPF, both the warehouse (c1glag) and department (c1gavd) codes are validated against RA10PF (warehouses) and RA07PF (departments) respectively. Invalid codes block creation with `*in31` or `*in32`.
    - Files: RA10PF, RA07PF
    - Condition: `not %found(RA10PF)` → `*in31`; `not %found(RA07PF)` → `*in32`.

27. **VP130R: Price Group Code Must Exist in RA30PF**
    - Logic: The actual price group code (c2gprg) entered for the mapping must exist in RA30PF. If blank or not found, `*in31 = *on` blocks the save.
    - File: RA30PF
    - Condition: `c2gprg = *blank or not %found(RA30PF)` → `*in31 = *on`.

28. **VP130R: Duplicate Mapping Blocked During Copy**
    - Logic: When copying a price group mapping to a new warehouse/department combination, the target combination must not already exist in VPGRPF. If it does, `*in33 = *on` blocks the copy.
    - File: VPGRPF
    - Condition: Target combination already exists → `*in33 = *on`.

29. **VP110R: Cannot Delete Unit if Purchase Orders Exist (switch 6.25)**
    - Logic: When deleting a price record with a specific unit of measure, VP110R checks whether there are any open purchase orders for this item+unit combination. If purchase orders exist, deletion is blocked.
    - Condition: Open purchase orders found for item+unit → deletion blocked.

30. **VP110R: Cannot Delete Last Price of Main Supplier Without Special Handling (switch 6.24)**
    - Logic: Stricter test prevents deleting the last price entry for the main supplier. Sub-supplier prices have relaxed rules (switch 6.22/6.23).

31. **VP410R: CSV File Must Be Specified**
    - Logic: If the CSV file name (b1file) is blank when the user confirms the VP410R screen, `*in31 = *on` re-displays the screen.
    - Condition: `b1file = *blank` → `*in31 = *on`.

32. **VP410R/VP411R: File Read via ASPFRSTMF**
    - Logic: VP410R calls ASPFRSTMF to copy the IFS CSV file into the work file (LWEXPF). If the copy fails (`w_stat ≠ *blank`), an error message is shown via AA005R and the program terminates without processing.
    - Condition: `w_stat ≠ *blank` → abort with error message.

33. **VP411R: Invalid Supplier Number in CSV Is Skipped**
    - Logic: For each CSV row, the supplier number column is validated using `%check(digits)`. If non-numeric characters are found, the row is skipped (`leavesr`).
    - Condition: `%check(digits:d_ldor) ≠ 0` → row skipped.

---

## Financial / Transactional Rules

34. **Mass Price Adjustment Applied by VP151R (VP150R calls VP151R)**
    - Logic: VP150R is a parameter screen only. VP151R performs the actual price updates in VVPRPF. Records are filtered by all criteria entered in VP150R and updated with the new prices.
    - File: VVPRPF
    - Condition: All validation in VP150R must pass before VP151R is called.

35. **Item Price Adjustment Applied by VP800R (VV105R calls VP800R)**
    - Logic: VV105R is a parameter screen for per-item price adjustment. VP800R performs the actual update. The supplier must exist on the item's price list before VP800R is called.
    - File: VVPRPF
    - Condition: Supplier must have prices in VVPRPF for this item.

36. **Price Deletion from CSV (VP411R)**
    - Logic: VP411R reads each row from LWEXPF, extracts item number (kol A), unit (kol E), supplier (kol F), price group (kol I), and price date (kol M). For each valid row, it chains VVPRPF and deletes the matching price record if found. No error is raised if the record does not exist.
    - File: VVPRPF
    - Key: VPFIRM + VPVARE + VPLDOR + VPPRGR + VPENHE + VPLETY + VPPDAT
    - Condition: `%found(VVPRPF)` → delete; not found → skip.

---

## Status and Lifecycle Rules

37. **Expired Items Toggle (VV100R)**
    - Logic: Indicator `*in84` toggles the display between showing only active items and all items including expired (utgåtte) ones.
    - File: VVARPF
    - Condition: Toggle controlled by user at runtime.

38. **Item Restored Without Prices Initially Allowed (VV101R — switch 8.03)**
    - Logic: When restoring a previously deactivated item (`not b_ny_vare`), the system allows saving even if price records do not yet exist. This permits restoration without immediate price entry.
    - Condition: `not b_ny_vare` → relaxed price requirement on restore.

---

## Special Conditions (Program-Specific)

39. **VV102R Is Called by VV101R — No Independent Database Updates**
    - Logic: VV102R is a second-screen sub-program called from VV101R. All updates are passed back via parameters. VV102R itself does not write to VVARPF directly. The actual database write occurs in VV101R after VV102R returns.

40. **VP110R: Version 7.01 — Price Group Blank Row Filtering**
    - Logic: Switch 7.01 in VP110R prevents blank-price-group rows from being shown if price records with a non-blank price group exist for the same item. This avoids confusion between global and price-group-specific prices.

41. **VP110R: Version 7.03 — Master Price Copy Indicator**
    - Logic: If a price is copied from a master price file (rather than directly maintained), it is displayed in blue to visually distinguish it from directly maintained prices.

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| VP151R | VP150R after validation | Apply mass price adjustment | Updates VVPRPF for all matching items |
| VP800R | VV105R after validation | Apply per-item price adjustment | Updates VVPRPF for the specific item |
| VP900R | FO730R per line | Fetch current sale price | Applied to FODTPF lines during recalculation |
| ASPFRSTMF | VP410R on submit | Copy IFS CSV file to LWEXPF | Error if file not found or cannot be copied |
| AA005R | VP410R on file error | Display error message | User notified of import failure |
| AB700R / AB705R | VP411R | Job log/audit trail | Logs start and status of batch deletion |
| VG510R/511R/512R | VP150R F4 | Item group lookup | Interactive selection of over/main/sub groups |
| RL500R | VP150R F4 on supplier | Supplier lookup | Interactive supplier selection |
| RA530R | VP130R, VP150R | Price group lookup | Interactive price group selection |

---

## Reference Tables

| Table | Key | Purpose in Item/Product Processing |
|-------|-----|-------------------------------------|
| VVARPF | VVFIRM + VVVARE | Item master — all item attributes, type, group, supplier, unit |
| VLAGPF | VLFIRM + VLVARE + VLLAGE | Warehouse items — stock level, type per warehouse, shelf location |
| VVPRPF | VPFIRM + VPVARE + VPLDOR + VPPRGR + VPENHE + VPLETY + VPPDAT | Item prices — sales, cost, inbound prices by supplier/unit/date |
| VPGRPF | VPGFIR + VPGLAG + VPGAVD | Price group mapping — warehouse+department → price group |
| VAUGPF | Firm + main + sub | Alternative sub-group register |
| VKHVPF | Firm + code | Account reference codes |
| VMVAPF | Firm + code | VAT code register |
| VOGRPF | Firm + code | Over/main group register |
| VHGRPF | Firm + hogr + hhgr | Main group hierarchy |
| VUGRPF | Firm + uogr + uhgr + uugr | Sub-group hierarchy |
| RLEVPF | Firm + supplier | Supplier register |
| RA30PF | Firm + code | Price group code register |
| RA10PF | Firm + code | Warehouse code register |
| RA07PF | Firm + code | Department code register |
| LWEXPF | Sequential | Work file for CSV import (price deletion) |
