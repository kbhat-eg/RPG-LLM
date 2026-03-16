# Business Logic for Item/Product Maintenance (NOBB/ASVADM)

This module covers maintenance of NOBB product data (Norsk Byggevare Base — the Norwegian building products database) and its synchronisation with the EVR (internal item register). The programs covered are JV100R (NOBB item overview/list), JV101R (NOBB item detail maintenance), JV113R (alternate item numbers — JVALPF), JV120R (transfer new items to EVR — parameter screen), JV121R (transfer new items to EVR — batch), JV130R (update item info from NOBB to EVR — parameter screen), JV141R (hold/release items by group), JV150R (price code change — parameter screen), JV151R (price code change — batch), and JV160R (item registration from order).

---

## Prerequisites / Master Data Requirements

1. **Supplier Must Exist in JLEVPF (JV120R, JV130R, JV150R)**
   - Logic: If a supplier number (b1ldor) is entered as a filter, it is validated against JLEVPF (NOBB supplier register). If the record is not found, the operation is blocked.
   - File: JLEVPF
   - Field: JLLDOR
   - Condition: `b1ldor <> 0 and not %found(JLEVPF)` → `*in32 = *on`, `b_feil = *on`.

2. **Supplier Must Not Be Blocked or Password-Protected (JV120R — switch 6.31)**
   - Logic: After JLEVPF is found, RS760R is called to check whether the supplier has a blocked ('S') or password-protected ('P') status. If either status applies, the transfer is refused.
   - Program: RS760R
   - Fields: w_retk (return code from RS760R), JLENUM (supplier external number)
   - Condition: `w_retk = 'P' or w_retk = 'S'` → `*in41 = *on`, `b_feil = *on`.

3. **Price-Giving Supplier Must Exist and Must Not Be Blocked (JV120R — switch 6.31)**
   - Logic: If a prisgivende leverandør (price-giving supplier) b1pgiv is entered, it is also validated against JLEVPF and then passed to RS760R. A blocked or passworded price-giver blocks the transfer.
   - File: JLEVPF
   - Condition: `b1pgiv <> 0 and not %found(JLEVPF)` → `*in36 = *on`; `w_retk = 'P' or 'S'` → `*in40 = *on`, `b_feil = *on`.

4. **Delivery Type Must Exist in VLTPPF (JV120R)**
   - Logic: The delivery type code (b1lety) is validated against VLTPPF (delivery type register). An invalid delivery type blocks the transfer.
   - File: VLTPPF
   - Field: VYLETY
   - Condition: `not %found(VLTPPF)` → `*in37 = *on`, `b_feil = *on`.

5. **Price Group Must Exist in RA30PF (JV120R — switch 5.70)**
   - Logic: If a price group code (b1prgr) is entered, it is validated against RA30PF (price group register). An unknown price group blocks the transfer.
   - File: RA30PF
   - Field: REDKOD
   - Condition: `b1prgr <> *blank and not %found(RA30PF)` → `*in38 = *on`, `b_feil = *on`.

6. **Sortiment Code Must Exist in JSORPF (JV120R — switch 6.10)**
   - Logic: If a sortiment code (b1osor) is entered, it is validated against JSORPF (sortiment register). An unknown sortiment code blocks the transfer.
   - File: JSORPF
   - Field: JSOSOR
   - Condition: `b1osor <> *blank and not %found(JSORPF)` → `*in39 = *on`, `b_feil = *on`.

---

## Validation Rules

7. **Item Group Hierarchy Validation — Over-, Hoved-, Undergroup (JV120R, JV130R, JV150R)**
   - Logic: If an over-group (b1ogrp), main group (b1hgrp), or sub-group (b1ugrp) is entered as a filter, each level is validated against its respective register file. The over-group must exist in VOGRPF; the main group must exist in VHGRPF keyed by over+main group; the sub-group must exist in VUGRPF keyed by over+main+sub group.
   - Files: VOGRPF (over-group), VHGRPF (main group), VUGRPF (sub-group)
   - Conditions:
     - `b1ogrp <> 0 and b1hgrp = 0 and b1ugrp = 0 and not %found(VOGRPF)` → `*in33 = *on`, `b_feil = *on` (JV120R) or `*in34` (JV130R).
     - `b1hgrp <> 0 and b1ugrp = 0 and not %found(VHGRPF)` → `*in34 = *on` (JV120R) or `*in35`.
     - `b1ugrp <> 0 and not %found(VUGRPF)` → `*in35 = *on` (JV120R) or `*in36`.

8. **JV130R: Supplier Is Mandatory**
   - Logic: Unlike JV120R where either supplier or price-giver can be supplied, JV130R (update item info) requires b1ldor to be non-zero. A blank supplier blocks the update launch.
   - File: JLEVPF
   - Condition: `b1ldor = 0` → `*in31 = *on`, `b_feil = *on`.

9. **Price Code Must Be Blank, N, or B (JV150R)**
   - Logic: JV150R (price code change) accepts only three valid price code values: blank (default), 'N' (net price), or 'B' (brutto price). Any other value blocks the operation.
   - Condition: `b1pkod <> ' ' and b1pkod <> 'N' and b1pkod <> 'B'` → `*in31 = *on`, `b_feil = *on`.

10. **Either Supplier or Module Number Required (JV150R)**
    - Logic: JV150R must have either a supplier number (b1ldor) or a module number (b1modn) specified as a selection criterion. Neither being zero blocks the price code change.
    - Condition: `b1ldor = 0 and b1modn = 0` → `*in32 = *on`, `b_feil = *on`.

11. **JV150R: Supplier Must Exist in JLEVPF**
    - Logic: If a supplier is entered, it is validated against JLEVPF. An invalid supplier number blocks the price code change.
    - File: JLEVPF
    - Condition: `b1ldor <> 0 and not %found(JLEVPF)` → `*in33 = *on`, `b_feil = *on`.

12. **JV150R: Module Number Must Exist in JMODPF**
    - Logic: If a module number (b1modn) is entered, it is validated against JMODPF (module register). An invalid module number blocks the operation.
    - File: JMODPF
    - Field: JUMODN
    - Condition: `b1modn <> 0 and not %found(JMODPF)` → `*in37 = *on`, `b_feil = *on`.

13. **JV141R: Cannot Simultaneously Select Hold and Release**
    - Logic: JV141R (hold/release NOBB items by group) has two mutually exclusive actions: hold (b1valh=1) and release (b1valf=1). Selecting both at once is blocked.
    - Condition: `b1valh = 1 and b1valf = 1` → `*in31 = *on`, `b_feil = *on`.

14. **JV141R: Items with Expired Supplier Linkage Are Skipped**
    - Logic: During hold/release processing (switch 6.34), the supplier linkage record in JVLEPF is checked for the item. If no JVLEPF record is found for the item and supplier, the item is silently skipped (leavesr — not processed).
    - File: JVLEPF
    - Fields: JVLVNR (item number), JVLLDO (supplier), JVLUDA (expiry date)
    - Condition: `not %found(JVLEPF)` → item skipped.

15. **JV141R: Items with Past JVLUDA Are Skipped Unless p_utgp = 1**
    - Logic: If the supplier-item record in JVLEPF has JVLUDA (utgår dato — expiry date) set to a past date, the item is excluded from hold/release processing unless the p_utgp (include expired) parameter is set to 1.
    - Condition: `p_utgp = 0 and jvluda <> *loval and jvluda < w_udat` → item skipped.

16. **JV141R: New-Items Mode Skips Items Already in EVR / Release Mode Skips Items Not in EVR**
    - Logic: During batch processing, VVARPF is checked for the EVR item number. In new-items mode (p_nye_varer = *on), items already in VVARPF are skipped (they are already registered). In release/hold mode (p_nye_varer = *off), items not yet in VVARPF are skipped.
    - File: VVARPF
    - Condition:
      - `p_nye_varer = *on and %found(VVARPF)` → item skipped.
      - `p_nye_varer = *off and not %found(VVARPF)` → item skipped.

17. **JV120R: Prisendringsdato Must Be a Valid Date (switch 6.30)**
    - Logic: If a price-change date (b1dato) is entered for filtering, it is validated with TEST(d). An invalid date format blocks the transfer.
    - Condition: `b1dato <> 0 and TEST(d) fails` → `*in43 = *on`, `b_feil = *on`.

---

## Configuration and Authorization Rules

18. **JV113R: Alternate Item Number Type Must Be T or F**
    - Logic: JV113R (alternate item number maintenance) requires the type code (c1btyp / k1btyp) to be either 'T' (trade / trade classification number) or 'F' (fabrikant — manufacturer number). Any other type is rejected.
    - File: JVALPF
    - Field: JVBTYP
    - Condition: `c1btyp <> 'T' and c1btyp <> 'F'` → `*in31 = *on`, return to screen.

19. **JV113R: Alternate Item Variant Number Cannot Be Blank on Create**
    - Logic: When creating a new alternate item number entry, the variant identifier (c1bvar) must be non-blank.
    - Field: JVBVAR
    - Condition: `c1bvar = *blank` → `*in32 = *on`, return to screen.

20. **JV113R: Alternate Item Text Cannot Be Blank on Update**
    - Logic: When entering or modifying the text (c2btxt) for an alternate item number record, the text field may not be left blank.
    - Field: JVBTXT
    - Condition: `c2btxt = *blank` → `*in31 = *on`, return to screen.

21. **JV113R: Duplicate Key Blocked on Copy**
    - Logic: When copying an alternate item number record to a new key (k1btyp + k1bvar), the target key is checked against JVALPF. If a record already exists with that key, the copy is blocked.
    - File: JVALPF
    - Condition: `%found(JVALPF)` for target key → `*in32 = *on`, return to screen.

---

## Financial / Transactional Rules

22. **JV121R: Expired NOBB Items Are Excluded from Transfer (switch 6.3c)**
    - Logic: When JV121R scans JVARPF/JVLEPF to build the candidate item list for EVR transfer, items flagged as deleted in NOBB (JVSTAT) or those with a past expiry date on JVLEPF (JVLUDA) are excluded. The condition `jvluda <> *loval and jvluda < today` causes the item to be skipped unless p_utgp (include expired) is set.
    - Files: JVARPF, JVLEPF
    - Fields: JVSTAT, JVLUDA (JVLEPF)
    - Condition: `jvluda <> *loval and jvluda < w_udat and p_utgp = 0` → item excluded.

23. **JV121R: Transfer Creates or Updates JVDTPF Holding Record**
    - Logic: For each eligible item, JV121R chains JVDTPF (item detail/holding register) by firm + item number. If found, it updates the record. If not found, it writes a new record with JVDHKO (hold code) and JVDFIR (firm). This controls whether the item is held or released in the EVR staging process.
    - File: JVDTPF
    - Fields: JVDFIR, JVDVAR, JVDHKO

24. **JV151R: Price Code Change Applies to All Items for Supplier or Module**
    - Logic: JV151R (batch price code change) reads JVLEPF by supplier (behandl_ldor) or JVARPF by module number (behandl_modn). For each item that matches the group filters (over/main/sub-group or module), JVDTPF.JVDPKO is updated with the new price code.
    - File: JVDTPF
    - Fields: JVDPKO (price code)
    - Condition: Group match required; if p_modn <> 0 then only items where jvmodn = p_modn are updated.

25. **JV151R: When Only Supplier Is in Parameters, Skip Group Checks (switch 6.33)**
    - Logic: When p_ldor is non-zero and all group parameters (p_modn, p_ogrp, p_hgrp, p_ugrp) are zero, the subroutine sjekk_lest is bypassed. The item is accepted if its own JVLLDO (supplier) matches p_ldor, without requiring a group match.
    - Condition: `p_modn = 0 and p_ogrp = 0 and p_hgrp = 0 and p_ugrp = 0 and jvlel3.jvlldo = p_ldor` → b_ok = *on, proceed.

---

## Status and Lifecycle Rules

26. **JV100R: Deleted NOBB Items Display as '** SLETTET **'**
    - Logic: JV100R shows NOBB items from JVARPF. Items with a deleted status in NOBB (JVSTAT = 4) are displayed with the text '** SLETTET **' appended to the item description in the list. They remain visible but are clearly marked as deleted.
    - File: JVARPF
    - Field: JVSTAT
    - Condition: `JVSTAT = 4` → display-only flag; deletion in NOBB triggers visual marker.

27. **JV101R: Field Protection via AVALPF**
    - Logic: JV101R uses AVALPF (field validation configuration) to determine which fields on the item detail screen are mandatory, optional, or protected for the current installation. Fields configured as mandatory in AVALPF cannot be left blank when saving.
    - File: AVALPF
    - Condition: AVALPF entries for program JV101R — mandatory fields vary by customer configuration.

28. **JV101R: Duplicate LVAR (Supplier Item Number) Check**
    - Logic: If a supplier item number (JVLVAR / lev.varenr) is entered, it is checked for uniqueness across all items in JVARPF for the same supplier. A duplicate supplier item number within the same supplier is blocked.
    - File: JVARPF
    - Field: JVLVAR
    - Condition: Another item already has the same JVLVAR for the same supplier → save blocked.

29. **JV160R: Expired Items Excluded from Order Registration List (switch 6.33)**
    - Logic: JV160R (item registration from order) builds a list of NOBB items that can be registered on the order. From switch 6.33, items whose JVLEPF record has JVLUDA (expiry date) set to a past date relative to today are excluded from the displayed list. The flag b_utgv is set when expiry checking is applied.
    - Files: JVARPF, JVLEPF
    - Fields: JVLUDA
    - Condition: `jvluda <> *loval and jvluda < w_udat` → item excluded from list.

30. **JV160R: Deposit Requirement for New T-Type Items (switch 7.fb)**
    - Logic: When JV160R is called with parameter p_depo indicating that a deposit has been paid on the order, only items of type 'T' (tjenestevarer — service items) that are new to the EVR (not yet in VVARPF) are subject to deposit checking. The check controls whether such items can be registered without deposit confirmation.
    - Parameter: p_depo
    - File: VVARPF
    - Condition: `p_depo = *blank and new T-type item` → `*in33 = *on`, return to list without registering.

31. **JV160R: Multiple Search Modes — SQL Search for Free-Text (switch os71 / 6.3q)**
    - Logic: From switch 6.3q, the SQL used for free-text search applies ISO date format and Norwegian language sort (`Set Option DatFmt=*ISO, SRTSEQ=*LANGIDUNQ, LANGID=NOR`). Earlier OS/400 SQL fixes (os71) corrected compatibility issues with OS/400 v6.1. The search uses JSOKPF (search text file) for free-text lookups across item descriptions.
    - File: JSOKPF (search text index), JVARPF
    - Condition: When w_seqe contains 'S', 'SO', 'SH', 'SU', 'F', 'FO', 'FH', 'FU' — SQL free-text search mode is activated (b_open_sok = *on).

32. **JV160R: Subfile EOF Guard — Items Filtered by NOBB Number Boundary**
    - Logic: In the subfile build loop (crt_subfile), when reading JVALPF/JVARPF sequentially, the loop exits if JVBNOB (NOBB number) changes from the expected value. This prevents showing items from a different NOBB group in the same subfile page.
    - Condition: `jvbnob <> w_bnob` → `*in15 = *on`, leave loop.

---

## Special Conditions (Program-Specific)

33. **JV100R: Subgroup Hierarchy — Warehouse Management Check**
    - Logic: JV100R reads the item hierarchy (VOGRPF, VHGRPF, VUGRPF) when displaying NOBB items. Items belonging to sub-groups configured for warehouse management (VUGRPF.VGULAG = 1) but not yet in VLAGPF may generate a warning indicator.
    - Files: VUGRPF, VLAGPF

34. **JV121R: Transfer Display Screen — Valg 1 Registers Item**
    - Logic: In JV121R, selecting valg=1 on a subfile row triggers the registration of that NOBB item into the EVR (VVARPF). The call chain goes to JV142R (item detail creation) and JV750R (price data fetch). If the item registration fails (e.g., duplicate item number in EVR), the create is skipped.
    - Programs: JV142R, JV750R
    - File: VVARPF

35. **JV141R: Price Validity Check Before Hold/Release (switch 6.35)**
    - Logic: The subroutine sjekk_pris checks whether a valid (non-expired) price exists in JVPRPF for the item and supplier. If p_utgp = 1 (include expired prices), the check is bypassed and b_pris = *on unconditionally. Otherwise, JVPRPF is scanned for a price with JXTDAT either blank or not yet past today's date. If no valid price is found, b_pris = *off and the item is excluded from hold/release.
    - File: JVPRPF
    - Fields: JXTDAT (price to-date)
    - Condition: `p_utgp = 0 and all JVPRPF prices have jxtdat < today` → b_pris = *off, item skipped.

36. **JV160R: Søk (Search) with Filter Reset (switch 7.02)**
    - Logic: A "nullstill" (reset) button sets b2null=1, which clears all search filter fields (groups, free-text, supplier) and forces a full refresh of the subfile. This is a display-control mechanism with no direct database blocking, but ensures stale search criteria do not persist.
    - Condition: `b2null = 1` → all search variables cleared, b_reset = *on, forny (refresh) called.

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| RS760R | JV120R — supplier/price-giver validation | Returns blocked/password status for supplier | Blocks transfer if w_retk = 'P' or 'S' |
| JV121R | JV120R — on confirmed parameters | Batch transfer of new NOBB items to EVR | Reads JVLEPF, creates JVDTPF records |
| JV730R | JV130R — on confirmed parameters | Batch update of item info from NOBB to EVR | Updates VVARPF from JVARPF data |
| JV151R | JV150R — on confirmed parameters | Batch price code change on JVDTPF | Updates JVDPKO for all matching items |
| JV141R | JV140R — on confirmed parameters | Hold/release items by group | Updates JVDTPF.JVDHKO; skips expired items |
| JV142R | JV121R — valg=1 in subfile | Create new EVR item from NOBB | Creates VVARPF record |
| JV750R | JV121R — after item creation | Fetch NOBB price data | Populates JWPRPF (work price) for the new item |
| FL510R | JV160R — F14 key | Show customer/project information | Informational only |
| FR510R | JV160R — F15 key | Show rebate matrix for customer | Informational only |
| FP510R | JV160R — F16 key | Show special prices for customer | Informational only |
| VG510R | JV120R/JV130R/JV150R — F4 on over-group | Over-group lookup | Fills b1utxt from VOGRPF |
| VG511R | JV120R/JV130R/JV150R — F4 on main-group | Main-group lookup | Fills b1utxt from VHGRPF |
| VG512R | JV120R/JV130R/JV150R — F4 on sub-group | Sub-group lookup | Fills b1utxt from VUGRPF |
| JL500R | JV120R/JV130R/JV150R — F4 on supplier | Supplier lookup | Fills b1navn from JLEVPF |
| JU500R | JV150R — F4 on module | Module lookup | Fills b1mte1 from JMODPF |

---

## Reference Tables

| Table | Key | Purpose in NOBB/Item Maintenance |
|-------|-----|----------------------------------|
| JVARPF | JVVARE | NOBB item master — all item attributes, status, group codes, module |
| JVLEPF | JVLVNR + JVLLDO | NOBB item-supplier linkage — includes JVLUDA (expiry date), JVLLVA (supplier item number) |
| JVDTPF | JVDFIR + JVDVAR | NOBB item detail/staging — JVDHKO (hold code), JVDPKO (price code) used during EVR transfer |
| JVPRPF | JXVARE + JXLDOR | NOBB item price register — JXTDAT (price-to date) checked for expiry |
| JVALPF | JVBNOB + JVBTYP + JVBVAR | Alternate item number register for NOBB items |
| VVARPF | VVFIRM + VVVARE | EVR (internal) item master — checked to determine if item is already registered |
| VLAGPF | VLFIRM + VLLAGE + VLVARE | Warehouse item records |
| VOGRPF | VGFIRM + VGOOGR | Over-group register — used for filter validation |
| VHGRPF | VGFIRM + VGHOGR + VGHHGR | Main-group register — used for filter validation |
| VUGRPF | VGFIRM + VGUOGR + VGUHGR + VGUUGR | Sub-group register — used for filter validation |
| JLEVPF | JLLDOR | NOBB supplier register — blocking validation for all JV transfer programs |
| JMODPF | JUMODN | Module register — used for JV150R module validation |
| JSORPF | JSOSOR | Sortiment code register — used for JV120R sortiment validation |
| JSOKPF | Search key | Free-text search index for NOBB item descriptions |
| VLTPPF | VYFIRM + VYLETY | Delivery type register — used for JV120R delivery type validation |
| RA30PF | REFIRM + REDKOD | Price group register — used for JV120R price group validation |
| JWPRPF | JBVARE | Work price file — populated by JV750R during EVR transfer |
| AVALPF | Program + field | Field-level mandatory/optional configuration per program (JV101R) |
