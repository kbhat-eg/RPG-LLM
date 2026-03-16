# Business Logic for ABC Analysis Parameter Setup and Classification

The ABC analysis module (ASLOGI system, MB prefix) maintains the MABCPF parameter register, which defines the scope and boundary percentages used to classify inventory items into a 3×3 ABC matrix (AX/AY/AZ through CX/CY/CZ) based on sales value and sales frequency. MB100R is the interactive parameter setup screen; MB101R shows the resulting item classifications; MB401R clears old ABC codes from VVARPF and VLAGPF before reclassification; MB600R/MB601R print ABC analysis reports; MB700R/MB701R execute the automated classification run. Every blocking rule is derived from boundary percentage ordering constraints, valid ABC code range checks, master data existence requirements (supplier, sortiment), and dimension hierarchy validity enforced at parameter entry and print time.

---

## Prerequisites / Master Data Requirements

- **MABCPF** (ABC parameter register): A parameter record keyed by `firm + kode` must exist before MB101R can display item classifications, before MB600R/MB601R can print, and before MB700R/MB701R can run a classification. MB100R creates and updates these records; without one, no ABC run is possible.
- **MABDPF** (ABC detail register): Must have records populated by a prior MB701R classification run before MB101R can display results. If MABDPF is empty for the parameter code, MB101R shows an empty subfile.
- **VVARPF** (item register): MB401R clears the ABC code field on VVARPF records matching the parameter scope (item range, sortiment, group hierarchy). MB101R updates `vvabc` (item-level ABC code) in VVARPF when the user changes a classification. If an item key is not found in VVARPF (chain fails), the update is silently skipped.
- **VLAGPF** (warehouse-item register): MB401R clears the warehouse-level ABC code on VLAGPF records. MB101R updates the warehouse ABC code in VLAGPF. If the warehouse-item record does not exist, the update is silently skipped.
- **RLEVPF** (supplier register): MB600R validates the supplier code entered in the print parameters (`a1levr`) by chaining RLEVL1. If `*in91 = *on` (supplier not found), an error indicator is set and the print is blocked until a valid supplier or blank is entered.
- **VSHEPF** (varesortiment/assortment register): MB600R validates the sortiment code (`a1vsor`) by chaining VSHEL1. If `*in92 = *on` (sortiment not found) and the sortiment field is non-blank, an error indicator is set and the print is blocked.
- **VSDTPF** (sortiment detail, dimension): MB601R chains VSDTL1 to verify that an item belongs to the requested sortiment scope when filtering by `a1vsor`. If not found, the item is excluded from the report.
- **MM747R** (forecast ABC update service): Called by MB101R at v5.63+ when a user manually updates an ABC code, to propagate the updated classification to the forecast system. Must be present in the library list. If absent, the program abends.
- **VP750R** (campaign link update): Called by MB101R at v6.20+ when ABC codes are changed. Must be present if the campaign module is active. If absent and campaigns are in use, the program abends.

---

## Validation Rules

- **ABC boundary ordering** (MB600R): The print parameter screen validates that the A/B/C percentage boundaries are in ascending order: `a1aint < a1bint` (A-boundary must be less than B-boundary), `a1bint < a1cint` (B-boundary must be less than C-boundary), and similarly for the X/Y/Z frequency boundaries. If any boundary is out of order, the screen is redisplayed with an error and the print is blocked.
- **ABC code range validity** (MB600R): The from-code (`a1fabc`) must be less than or equal to the to-code (`a1tabc`). Valid codes are restricted to the 3×3 matrix: AX, AY, AZ, BX, BY, BZ, CX, CY, CZ. If an entered code is not in this set, an error indicator is set and the screen is redisplayed; the print is blocked.
- **Group range ordering** (MB600R): The from-overgroup (`a1fogr`) must be less than or equal to the to-overgroup (`a1togr`). Similarly, the from-item range must be less than or equal to the to-item range. Violations set an error indicator and block the print.
- **Supplier existence** (MB600R): If `a1levr <> 0` (supplier filter specified), chains RLEVL1 by firm + supplier number. If `*in91 = *on` (not found), the save/print is blocked with an error.
- **Sortiment existence** (MB600R): If `a1vsor <> *blank`, chains VSHEL1. If `*in92 = *on` (not found), the save/print is blocked with an error.
- **ABC parameter code for clearing** (MB401R v6.20+): MB401R accepts `d_pabc` to specify which ABC code to clear (e.g., clear only AX items). If `d_pabc = *blank`, all ABC codes within the scope are cleared. If a specific code is provided but the item's current ABC code does not match, that item is skipped during clearing.
- **Dimension hierarchy traversal** (MB401R): When `d_vsor` (sortiment) is non-blank, MB401R traverses VSDTL6 to find items in that sortiment. If VSDTL6 has no entries for the sortiment, zero items are cleared — no error, but the parameter scope is effectively empty. When clearing by ogrp/hgrp/ugrp hierarchy, MB401R traverses each level in sequence; missing levels cause that branch to yield zero records.
- **Update flag** (MB100R parameter `d_upda`): MB701R checks the `d_upda` flag in the assembled parameter block. If `d_upda = *off`, the classification results are computed but VVARPF and VLAGPF are not updated; the run is read-only. No error is raised; incorrect flag setting produces a silent no-op on master data.

---

## Configuration and Authorization Rules

- **Firm number from LDA** (positions 944–946): MB100R, MB101R, MB400R, MB600R, MB601R, MB700R, and MB701R all read `l_firm` from the LDA. All MABCPF, MABDPF, VVARPF, and VLAGPF lookups use this firm as the primary key component.
- **ABC boundary percentages** (`d_aint`/`d_bint`/`d_cint` and `d_xint`/`d_yint`/`d_zint`): Stored in the MABCPF parameter block. The A/B/C boundaries define cumulative value percentage thresholds; the X/Y/Z boundaries define cumulative frequency percentage thresholds. Items are classified into the 3×3 matrix by combining their position on both axes. Incorrect boundary ordering is blocked at MB600R parameter entry; incorrect boundary ordering passed directly to MB701R is not re-validated and may produce meaningless results.
- **Warehouse type filter** (`d_lagr`/`d_ltyp` arrays): MB700R assembles arrays of warehouse codes and types from MABCPF into the parameter block before calling MB701R. MB701R filters VLAGPF records by these arrays. If the arrays are empty (no warehouses configured in MABCPF), all warehouse records for the firm may be included or the warehouse dimension may be skipped entirely depending on array initialization.
- **Sort flags**: The parameter block includes flags controlling whether classification is based on value, frequency, or both axes. MB701R uses these to select the comparison algorithm. An invalid sort flag combination produces undefined classification results; there is no runtime error.
- **Parameter loop** (MB700R): MB700R loops all MABCPF records for the firm using MABCL1, assembling parameters for each code and calling MB701R. If MABCPF has no records for the firm, no MB701R calls are made and no items are reclassified.

---

## Financial / Transactional Rules

- **ABC classification basis**: Items are ranked by cumulative sales value (amount) and cumulative sales frequency (transaction count) over the date range `d_fdat`–`d_tdat` in the parameter block. The percentages are computed against firm-level totals. An item's A/B/C tier depends on its cumulative value rank; X/Y/Z tier depends on its cumulative frequency rank.
- **Clear before reclassify** (MB401R → MB701R): MB401R clears all ABC codes in scope from VVARPF and VLAGPF before MB701R writes new ones. This is destructive: if MB701R fails partway through, the items cleared but not yet reclassified are left with blank ABC codes. There is no rollback.
- **Manual override** (MB101R): Users can manually change the ABC code of an individual item in the MABDPF subfile. The change updates VVARPF (`vvabc`) and VLAGPF (warehouse ABC), and triggers MM747R (forecast) and VP750R (campaign) propagation. A manual override is overwritten the next time MB401R + MB701R run for the same parameter scope.
- **History toggle** (MB101R F13): Pressing F13 switches the detail subfile between current and historical classification data. This is display-only; no financial records are modified.

---

## Status and Lifecycle Rules

- **MABCPF parameter lifecycle**: Created via MB100R option F6. Updated via option 2 (change). Deleted via option 4. No soft-delete; deletion is immediate and physical. If MABCPF is deleted, subsequent MB700R runs skip that code.
- **MABDPF classification lifecycle**: Populated exclusively by MB701R classification runs. MB101R reads MABDPF for display and writes individual overrides. A full reclassification (MB401R + MB701R) replaces all MABDPF records within scope. No retention of prior classification versions.
- **VVARPF ABC code lifecycle**: `vvabc` in VVARPF is cleared by MB401R and written by MB701R. It is a derived field; its value is always the result of the last completed classification run or manual override in MB101R.
- **VLAGPF ABC code lifecycle**: Warehouse-level ABC code in VLAGPF is cleared by MB401R and written by MB701R. If a warehouse-item combination exists in VLAGPF but has no matching MABDPF entry (e.g., the item was not in scope), the warehouse ABC code is cleared by MB401R but never repopulated by MB701R — it remains blank until the next run that includes this item.

---

## Special Conditions (Program-Specific)

- **MB100R — ABC parameter maintenance**: Positioning in the subfile is by overgroup (`d_hogr`) or sortiment code (`d_vsor`). Options 2 (change), 3 (copy), 4 (delete), 5 (view) are available. Option 4 confirmed deletes the MABCPF record but does not cascade-delete MABDPF. Orphaned MABDPF records for the deleted parameter remain until the next classification run overwrites them (if the parameter is recreated) or indefinitely.
- **MB101R — ABC detail item view**: F11 toggles between two sorting modes (by ABC code, by item number). F13 shows historical data. Direct update of the ABC code field triggers writes to VVARPF, VLAGPF, MM747R, and VP750R. If any of these called programs is absent from the library list, the program abends with no partial rollback.
- **MB401R — clear old ABC codes**: Traverses items by sortiment (VSDTL6) or by ogrp/hgrp/ugrp hierarchy. The `d_pabc` parameter (v6.20+) restricts clearing to items with a specific current ABC code. If `d_pabc = *blank`, all ABC codes in scope are cleared regardless of current value. The clear loop does not report counts; there is no confirmation of how many records were cleared.
- **MB600R — print parameter screen**: All validation (boundary ordering, code range, group range, supplier existence, sortiment existence) runs at MB600R. Invalid parameters block the print without modifying any data. MB600R does not call MB401R; clearing is separate from printing.
- **MB601R — ABC analysis report**: Reads MABDL5 (MABDPF logical) for items matching the parameter filters. For each item, chains VSDTL1 to verify sortiment membership if a sortiment filter is active. Items failing the sortiment check are excluded from the report without error. Applies additional filters: abc code range (`a1fabc`–`a1tabc`), group ranges, supplier, warehouse.
- **MB700R / MB701R — automated classification run**: MB700R is the driver that loops MABCPF and calls MB701R once per parameter code. MB701R performs the actual classification algorithm, writes MABDPF, and updates VVARPF/VLAGPF if `d_upda = *on`. If MB701R abends for one parameter code, subsequent codes in MB700R's loop are not processed (the loop uses `*inlr` for termination; an abend bypasses normal termination).

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| MB401R | Called before MB701R classification run | Clears ABC codes from VVARPF and VLAGPF within the parameter scope; v6.20+ `d_pabc` restricts to specific code | Destructive clear; items not subsequently reclassified are left with blank ABC codes |
| MB701R | Called by MB700R for each MABCPF parameter code | Executes ABC classification algorithm; writes MABDPF; updates VVARPF/VLAGPF if d_upda=*on | Core classification engine; must complete successfully for codes to be reclassified |
| MB700R | Scheduled or manual batch driver | Loops all MABCPF records for firm; assembles warehouse arrays; calls MB701R per code | If MABCPF empty, no classification runs; abend in MB701R stops remaining codes |
| MM747R | MB101R on manual ABC code change (v5.63+) | Propagates updated ABC code to forecast system | Must be in library list; absent program causes abend |
| VP750R | MB101R on manual ABC code change (v6.20+) | Updates campaign link for item with new ABC classification | Must be in library list if campaign module active; absent program causes abend |
| MB601R | MB600R print request after validation passes | Reads MABDPF filtered by parameter range; chains VSDTL1 for sortiment membership | Print only; no master data updates |

---

## Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| MABCPF (MABCL*) | firm, kode | ABC parameter register — defines analysis scope, boundaries, date range, update flag |
| MABDPF (MABDL*) | firm, kode, vare, lage | ABC detail register — classification result per item and warehouse per parameter code |
| VVARPF | firm, vare | Item register — `vvabc` field updated by MB701R and MB101R manual override |
| VLAGPF | firm, vare, lage | Warehouse-item register — warehouse ABC code updated by MB701R and MB101R |
| VSHEPF (VSHEL*) | firm, vsor | Varesortiment/assortment register — validated by MB600R; traversed by MB401R for item scope |
| VSDTPF (VSDTL*) | firm, vsor, vare | Sortiment detail — links items to sortiment codes; used by MB401R traversal and MB601R filter |
| RLEVPF (RLEVL*) | firm, levr | Supplier register — validated by MB600R when supplier filter is specified |
