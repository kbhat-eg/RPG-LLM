# Business Logic for Warehouse and Inventory Management

This module covers warehouse item master maintenance and inventory inquiry in the ASLAGR system. The programs covered are LL100R (warehouse item maintenance — VLAGPF), LL101R (warehouse item overview — informational), LL500R (item number lookup for inventory inquiry), LL502R (stock balance display), LL600R (inventory list print — parameter screen), LL602R (inventory list data extraction — batch), LL610R (ABC analysis parameters), and LL700R (purchase planning inquiry).

---

## Prerequisites / Master Data Requirements

1. **Item Must Exist in VVARPF (LL100R, LL500R)**
   - Logic: When creating a new warehouse item record (VLAGPF) or performing an inventory inquiry, the item number is validated against VVARPF (item master). If the item does not exist in VVARPF, the create or inquiry is blocked.
   - File: VVARPF
   - Condition: `not %found(VVARPF)` → `*in31 = *on` (LL500R), `b_feil = *on` (LL100R).

2. **Warehouse Must Exist in RA10PF on Copy (LL100R — switch 6.2h)**
   - Logic: When copying a warehouse item record to a new warehouse, the target warehouse code is validated against RA10PF (warehouse register). An invalid target warehouse blocks the copy.
   - File: RA10PF
   - Field: RAJKOD
   - Condition: `not %found(RA10PF)` → copy blocked.

3. **Shelf Location (Plass) Must Be Valid (LL100R)**
   - Logic: If a shelf location (c2plas) is entered, it is validated via MD710R (shelf/location validation program). If MD710R returns a non-blank status, the shelf location is invalid and the save is blocked.
   - Program: MD710R
   - Fields: VLPLAS, VLPLA1, VLPLA2
   - Condition: `MD710R returns w_stat <> *blank` → `*in32/33/34 = *on`, return to screen.

4. **Prioritized Supplier Must Not Be Blocked (LL100R — switch 6.20)**
   - Logic: If a prioritized supplier (c2sldo / VLBLDO) is entered on the warehouse item record, RS760R is called to check whether that supplier is blocked or passworded. A blocked supplier cannot be set as the prioritized supplier.
   - Program: RS760R
   - Field: VLBLDO
   - Condition: `RS760R returns w_stat <> *blank` → `*in35 = *on`, return to screen.

5. **LL600R: Supplier Must Exist in RLEVPF**
   - Logic: If a supplier number (a1lvnr) is entered as a filter for the inventory list, it is validated against RLEVPF (supplier register). An unknown supplier number blocks the report launch.
   - File: RLEVPF
   - Field: RLLEVR
   - Condition: `a1lvnr > 0 and not %found(RLEVPF)` → `*in35 = *on`, return to screen.

6. **LL600R: Case Worker / Buyer Must Exist in RA26PF**
   - Logic: If a saksbehandler / innkjøper code (a1sakb) is entered as a filter, it is validated against RA26PF (case worker register). An unknown code blocks the report launch.
   - File: RA26PF
   - Field: RAZSAK
   - Condition: `a1sakb <> *blank and not %found(RA26PF)` → `*in38 = *on`, return to screen.

7. **LL700R: Item Must Exist in VVARPF**
   - Logic: LL700R (purchase planning) validates the entered item number against VVARPF before proceeding. An invalid item number blocks the display.
   - File: VVARPF
   - Condition: `not %found(VVARPF)` → `*in31 = *on`.

8. **LL700R: Warehouse Must Be Valid via FÅ720R**
   - Logic: The warehouse number is validated via FÅ720R. If validation returns b_feil = *on, the warehouse number is invalid and `*in32 = *on` blocks the display.
   - Program: FÅ720R
   - Condition: `b_feil = *on` → `*in32 = *on`.

---

## Validation Rules

9. **Start-of-Validity Date Must Be a Valid Date (LL100R)**
   - Logic: If a start-of-validity date (c2stda / VLSTDA) is entered on the warehouse item record, it is validated with TEST(d). An invalid date format blocks the save.
   - Field: VLSTDA
   - Condition: `c2stda <> 0 and TEST(d) fails` → `*in31 = *on`, return to screen.

10. **Expiry Date Must Be a Valid Date (LL100R — switch 6.21)**
    - Logic: If an expiry date (c2suda / VLUDAT) is entered, it is validated with TEST(d). An invalid date format blocks the save.
    - Field: VLUDAT
    - Condition: `c2suda <> 0 and TEST(d) fails` → `*in36 = *on`, return to screen.

11. **LL600R: From-Range Must Not Exceed To-Range**
    - Logic: For all five paired range fields (item number, warehouse, over-group, main-group, sub-group), the from value must be less than or equal to the to value. Any inversion blocks the report launch.
    - Conditions:
      - `a1fvar > a1tvar` → `*in30 = *on`.
      - `a1flag > a1tlag` → `*in31 = *on`.
      - `a1fogr > a1togr` → `*in32 = *on`.
      - `a1fhgr > a1thgr` → `*in33 = *on`.
      - `a1fugr > a1tugr` → `*in34 = *on`.

12. **LL600R: Sort Key Must Be 1–5**
    - Logic: The sort key (a1sort) for the inventory list must be between 1 and 5 inclusive.
    - Condition: `a1sort < 1 or a1sort > 5` → `*in36 = *on`, blocked.

13. **LL600R: Break Concept Must Be 0 or 1**
    - Logic: The bruddbegrep (break/grouping concept) parameter (a1brud) must be either 0 (no break) or 1 (break enabled).
    - Condition: `a1brud > 1` → `*in37 = *on`, blocked.

14. **LL610R: ABC Basis (glag) Must Be 0–3**
    - Logic: LL610R (ABC analysis parameters) requires the glag (basis for calculation) field to be in the range 0 to 3.
    - Condition: `glag < 0 or glag > 3` → `*in37 = *on`, blocked.

15. **LL610R: Sort Key Must Be 1–4**
    - Logic: The sort key for the ABC analysis report must be between 1 and 4 inclusive.
    - Condition: `a1sort < 1 or a1sort > 4` → `*in36 = *on`, blocked.

---

## Configuration and Authorization Rules

16. **LL100R: Item Type Change L→S Requires Confirmation If Active Stock Exists (switch 6.13)**
    - Logic: If the warehouse item type (VLVTYP) is being changed from 'L' (lagervare — stock item) to any other type, and the item has active stock holdings in the warehouse (determined by calling LL766R), the user must explicitly confirm the type change via a confirmation window (xc3msg). If the user declines (b_endr_vtyp = *off), the type is reset to 'L' and the save is blocked.
    - Program: LL766R
    - File: VLAGPF
    - Fields: VLVTYP
    - Condition: `s_vtyp = 'L' and c2vtyp <> 'L' and b_lagr = *on and b_endr_vtyp = *off` → `c2vtyp` reset to 'L', return to screen.

17. **LL100R: Deletion Is Disabled (switch 6.2a)**
    - Logic: From switch 6.2a, the delete option (valg=4) on warehouse item records is disabled. The corresponding code block is commented out. Warehouse items cannot be deleted through this program.
    - Condition: `b1valg = 4` → no action (code is disabled/commented).

18. **LL100R: Function Key Access Control via AD005R (switch 8.00)**
    - Logic: Certain function keys (F1=New, F6=Create, F8) are subject to authorization checking via AD005R for the user l_user against program 'LL100R'. If AD005R returns w_status = 'Sperret' (blocked), the function key action is suppressed.
    - Program: AD005R
    - Condition: `w_status = 'Sperret'` → function key action skipped.

19. **LL602R: Non-Stock Items Are Skipped (switch 5.70)**
    - Logic: LL602R (inventory list batch extraction) reads VLAGPF sequentially and checks VLVTYP for each record. Items with VLVTYP ≠ 'L' are skipped and not written to the work file LWA1PF.
    - File: VLAGPF
    - Field: VLVTYP
    - Condition: `VLVTYP <> 'L'` → goto les_lager (item excluded from inventory list).

20. **LL602R: Item Must Exist in VVARPF**
    - Logic: For each VLAGPF record processed, VVARPF is chained to confirm the item exists in the master. If `*in91 = *on` (not found), the record is skipped.
    - File: VVARPF
    - Condition: `not %found(VVARPF)` → item skipped.

21. **LL602R: Sub-Group Warehouse Management Flag Controls Inclusion**
    - Logic: If the sub-group for the item has VUGRPF.VGULAG = 1 (sub-group managed by warehouse), items in that sub-group that are not managed at the warehouse level are excluded. Similarly, the main-group (VHGRPF.VGHLAG = 1) and over-group (VOGRPF.VGOLAG = 1) flags are checked hierarchically to determine inclusion.
    - Files: VUGRPF, VHGRPF, VOGRPF
    - Fields: VGULAG, VGHLAG, VGOLAG
    - Condition: Group warehouse-management flag set but item not in managed mode → item excluded.

---

## Financial / Transactional Rules

22. **LL700R: Price Fetched via VP750R**
    - Logic: LL700R calls VP750R to retrieve the current purchase price and any campaign price for the item. The prices displayed (purchase price, campaign price) are read-only in the display screen. No blocking occurs here, but incorrect price group configuration may result in zero prices being shown.
    - Program: VP750R
    - Files: VVPRPF, FKPRPF

23. **LL700R: 12-Month Rolling Sales Statistics from SSTAPF**
    - Logic: LL700R reads 12 months of sales statistics from SSTAPF (sales statistics file) keyed by year + item number. The array a_psal is populated with monthly sales quantities. This is informational only.
    - File: SSTAPF
    - Fields: SFPAAR (year), SFVARE (item)

24. **LL700R: 12-Month Rolling Purchase Statistics from SISTPF**
    - Logic: LL700R reads 12 months of purchase statistics from SISTPF (purchase statistics file). The array a_pkjo is populated with monthly purchase quantities.
    - File: SISTPF
    - Fields: SIPAAR (year), SIVARE (item)

25. **LL700R: Replacement Item Display (switch 7.02)**
    - Logic: From switch 7.02, when displaying purchase planning data, replacement item information is fetched from VVERPF/VVE2PF (item replacement files). The replacement chain is displayed alongside the main item but has no blocking effect.
    - Files: VVERPF, VVE2PF
    - Fields: VVENVR (replacement item number), V2EGVR (substitute item)

26. **LL700R: Purchase Planning Labels Configurable via CO402R (switch 7.03)**
    - Logic: CO402R is called with key 'LEDETEKSTER_BESTILLINGSPKT' to determine whether to use standard labels for minimum/maximum stock and order point, or alternative (bestillingspunkt-style) labels. This affects only the display text, not any blocking logic.
    - Program: CO402R
    - Condition: `u_bpkt = *on` → alternative label array a_txt(3)/a_txt(4) used for c2bptx/c2bmtx.

---

## Status and Lifecycle Rules

27. **LL100R: Item Type Change L→S Also Resets Year-to-Date Figures (switch 6.15)**
    - Logic: When an item type is changed from 'L' (stock) to 'S' (special order), year-to-date stock figures in VLAGPF are reset to zero (nullstilt hittil i år tall). This is performed automatically on confirmed type change.
    - Field: VLVTYP

28. **LL100R: Warehouse Item Type Change S→L Generates Warning Log (switch 6.26)**
    - Logic: When an item type is changed from 'S' (special order) back to 'L' (stock item), a warning is generated and a log entry is written. This change requires the user to confirm the conversion.
    - Condition: `s_vtyp = 'S' and c2vtyp = 'L'` → warning/log triggered.

29. **LL100R: Warehouse Record Expiry Date Tracking (switch 6.21)**
    - Logic: VLAGPF records have an expiry date (VLUDAT). This date is displayed on the maintenance screen. There is no automatic expiry enforcement in LL100R — the date is informational, but can be used by other programs to filter out expired warehouse records.
    - Field: VLUDAT

30. **LL500R: Item Not Found Triggers Error — Inquiry Blocked**
    - Logic: In LL500R (item number lookup for inventory), after the user enters an item number, VVARPF is chained. If the item does not exist, `*in31` is set and the inquiry to LL502R is not launched.
    - File: VVARPF
    - Condition: `*in90 = *on` (not found) → `*in31 = *on`, return to entry screen.

---

## Special Conditions (Program-Specific)

31. **LL100R: Warehouse Filter from Logged-On User (switch 7.08/8.03)**
    - Logic: From switch 7.08, the default warehouse in the list filter can be taken from the logged-on user's profile via AUSRPF (user register). Switch 8.03 allows the installation to control via CO402R whether this automatic warehouse filter applies.
    - File: AUSRPF
    - Program: CO402R (switch 8.03 key controls behaviour)

32. **LL100R: Bestillingspunkt Label Override via CO402R (switch 7.04)**
    - Logic: CO402R is called with key 'LEDETEKSTER_BESTILLINGSPKT'. If the switch is active (u_bpkt = *on), the labels for order point and order quantity on the C2 screen use alternative text from array a_txt(3)/a_txt(4) instead of the standard a_txt(1)/a_txt(2).
    - Program: CO402R
    - Condition: `u_bpkt = *on` → alternative label text.

33. **LL100R: Priority Supplier Validation via VVPRPF and LLDIPF (switch 6.32 — disabled in 7.02)**
    - Logic: From switch 6.32, when a priority supplier (VLBLDO) is set, it was validated against VVPRPF (item price file) and LLDIPF (supplier distribution file) to confirm the supplier has a price for this item. Switch 7.02 (Gipling special version) disabled this validation: `c*` comments out the 6.32 validation block.
    - Current status: Validation disabled by 7.02.

34. **LL100R: Stock Balance Inquiry via VL750C (switch 6.25)**
    - Logic: Selecting valg=9 on a warehouse item record calls VL750C to display the current stock balance (beholdning) for that item. This is informational only — no blocking logic.
    - Program: VL750C
    - Condition: `b1valg = 9` → call VL750C.

35. **LL602R: Item Group Range Applied as Composite Key**
    - Logic: LL602R applies the group range filter (f_vgrp to t_vgrp) as a composite key built from over-group + main-group + sub-group. Items outside the specified group range are excluded from the output work file LWA1PF.
    - File: LWA1PF (output work file)

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| MD710R | LL100R — shelf location save | Validate shelf/location code | Blocks save if location is invalid |
| RS760R | LL100R — priority supplier save | Check supplier blocked/password status | Blocks save if supplier is blocked |
| LL766R | LL100R — item type change L→anything | Check if active stock exists | Forces confirmation window; blocks type change if user declines |
| AD005R | LL100R — F1/F6/F8 keys (switch 8.00) | Authorization check for user + function | Blocks function key action if 'Sperret' |
| LL601R | LL600R — confirmed parameters | Batch print of inventory list | Reads LWA1PF work file populated by LL602R |
| LL602R | LL600R — internal batch call | Extract VLAGPF records to LWA1PF | Filters by item type, group, supplier, saksbehandler |
| VP750R | LL700R — item display | Fetch purchase price and campaign price | Populates price display fields |
| FÅ720R | LL700R — warehouse validation | Validate warehouse number | Blocks display if warehouse is invalid |
| VL750C | LL100R — valg=9 | Display stock balance for item | Informational only |
| LL502R | LL500R — after item validation | Display stock balance | Called only if item is valid in VVARPF |
| VV500R | LL600R — F4 on item field | Item number lookup | Fills a1fvar/a1tvar from VVARPF |

---

## Reference Tables

| Table | Key | Purpose in Warehouse/Inventory Processing |
|-------|-----|-------------------------------------------|
| VLAGPF | VLFIRM + VLLAGE + VLVARE | Warehouse item master — all per-warehouse parameters (min, max, order point, type, shelf, supplier) |
| VVARPF | VVFIRM + VVVARE | Item master — validated on create; VVENHL (unit) read for unit display |
| RLEVPF | RLFIRM + RLLEVR | Supplier register — validated in LL600R filter; used for priority supplier name lookup |
| RA10PF | RAFIRM + RAJKOD | Warehouse register — validated on copy of warehouse item record |
| RA26PF | RAFIRM + RAZSAK | Case worker / buyer register — validated in LL600R filter |
| VUGRPF | VGFIRM + VGUOGR + VGUHGR + VGUUGR | Sub-group register — VGULAG flag controls inclusion in LL602R |
| VHGRPF | VGFIRM + VGHOGR + VGHHGR | Main-group register — VGHLAG flag controls inclusion |
| VOGRPF | VGFIRM + VGOOGR | Over-group register — VGOLAG flag controls inclusion |
| SSTAPF | SFPAAR + SFVARE | Sales statistics — 12-month rolling, displayed in LL700R |
| SISTPF | SIPAAR + SIVARE | Purchase statistics — 12-month rolling, displayed in LL700R |
| VVERPF | VVENVR | Item replacement register — replacement chain displayed in LL700R (switch 7.02) |
| VVE2PF | V2ENVR + V2EGVR | Extended item replacement — substitute items displayed in LL700R |
| VVPRPF | VPVARE + VPLDOR | Item price file — used (when 6.32 active) for priority supplier validation |
| LLDIPF | LPLDOR | Supplier distribution file — used (when 6.32 active) for priority supplier validation |
| LWA1PF | Work key | Inventory list work file — output from LL602R, input to LL601R |
| AUSRPF | ABUSER | User profile register — warehouse default lookup (switch 7.08/8.03) |
