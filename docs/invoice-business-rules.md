# Business Logic for Invoice Processing

This module covers the complete lifecycle of invoice processing in the ASOFAK system, from order selection and consolidation through final print, cost-price posting, and post-print cleanup. The programs involved are FO614R (order selection/sorting), FO615R (project final-invoice utility), FO616R (core invoice update), FO617R (cost-price transaction builder), FO618R (post-print cleanup), FO619R (RF picking remainder handler), and FO621R (invoice-basis review and consolidated-invoice preparation). NXKORR overrides exist for FO614R, FO616R, FO617R, and FO619R.

---

## Prerequisites / Master Data Requirements

1. **Order Selection Register Must Contain a Valid Entry**
   - Logic: FO614R reads FPSOPF (order selection register) to determine which orders are candidates for invoicing. If no matching selection record exists for the company (`FKPFIR`), the order is silently skipped.
   - File: FPSOPF
   - Field: FKPFIR
   - Condition: `FKPFIR ≠ LDA.l_firm` causes the order to be skipped (switch 5.44).

2. **Order Type Must Exist in VOTYPPF**
   - Logic: FO617R calls `sjekk_otyp` to verify the order type from sales statistics (SSTAPF) against VOTYPPF. If the order type is not found, `b_feil = *on` and the cost-posting line is blocked.
   - File: VOTYPPF
   - Field: VAOTYP (order type key)
   - Condition: `not %found(VOTYPPF)` → `b_feil = *on`.

3. **Print Queue Selection Must Succeed**
   - Logic: FO614R calls AP600R to select a print queue. If the user pressed F3 inside AP600R (`w_in03 = '1'`), processing aborts with `goto xslutt` (switch 8.11).
   - File: n/a (interactive program call)
   - Condition: `w_in03 = '1'` after returning from AP600R → entire invoice run is aborted.

4. **Factoring Configuration**
   - Logic: FO616R reads FNUFPF for factoring KID generation and supports multiple factoring providers (Sparebank1, DNB, NORDEA, NORDEANY, Svea, SAP). The switch `b_samf`/`b_forf` controls consolidated invoicing behavior. If factoring is active, specific KID-number formats must be resolvable.
   - File: FNUFPF
   - Condition: Factoring provider code must match a known scheme; unrecognized factoring code results in standard KID generation.

---

## Validation Rules

5. **Company Number Filter (FO614R)**
   - Logic: Each FPSOPF record is checked against the logged-on company from the LDA. Records belonging to a different company are skipped.
   - File: FPSOPF
   - Field: FKPFIR vs. LDA position 944-946 (`l_firm`)
   - Condition: `FKPFIR ≠ l_firm` → skip record (switch 5.44).

6. **Order Type Filter (FO614R)**
   - Logic: The order type on the selection register (`fk2oty`) must match the order's actual type (`fkpoty`), or the order is skipped. Exception: if `fk2oty = '99'` (combined invoice mode), only orders with `VAOAKK = 3` (invoice/credit-note stage) pass.
   - File: FPSOPF, VOTYPPF
   - Fields: FKPOTY, FK2OTY, VAOAKK
   - Condition: `fkpoty cabne fk2oty` → skip. For combined: `if fk2oty='99' and VAOAKK ≠ 3` → skip (switch 8.01).

7. **Workstation Filter (FO614R)**
   - Logic: If switch `fk2vg1 = 1`, only orders belonging to the current workstation ID (`fkpwsd`) are selected.
   - File: FPSOPF
   - Fields: FK2VG1, FKPWSD, FK2WSD
   - Condition: `fk2vg1=1 and fkpwsd ≠ fk2wsd` → skip.

8. **User Filter (FO614R)**
   - Logic: If switch `fk2vg2 = 1`, only orders owned by the current logged-on user are selected.
   - File: FPSOPF
   - Fields: FK2VG2, FKPUSR, FK2USR
   - Condition: `fk2vg2=1 and fkpusr ≠ fk2usr` → skip.

9. **Packing Slip Already Written (FO614R)**
   - Logic: If VAOAKK = 1 (picking list stage) and FOHEPF.FOKSOB = 1 (packing slip already written), the order is skipped.
   - File: FOHEPF
   - Fields: VAOAKK, FOKSOB
   - Condition: `vaoakk=1 and foksob=1` → goto tag300 (skip).

10. **Credit Packing Slip Hold Code (FO614R)**
    - Logic: If the order type is at packing-slip stage (`vaoakk=2`) and is a credit order (`vaokre='-'`), a hold code is applied to the order header (`FOITYP = w_hkod`). This does not block selection but prevents further progression until the hold is cleared.
    - File: FOHEPF, VOTYPPF
    - Fields: VAOAKK, VAOKRE, FOITYP
    - Condition: `vaoakk=2 and vaokre='-'` → `FOHEPF.FOITYP = w_hkod` (switch 7.02).

11. **Item Type Must Be Stock ('L') for Cost Posting (FO617R)**
    - Logic: `sjekk_vtyp` subroutine: if both the warehouse item type (`VLAGPF.VLVTYP`) and the item master type (`VVARPF.VVVTYP`) are not 'L' (stock/lager item), the cost posting is blocked.
    - File: VLAGPF, VVARPF
    - Fields: VLVTYP, VVVTYP
    - Condition: `VLVTYP ≠ 'L' and VVVTYP ≠ 'L'` → `b_feil = *on`.

12. **Cost Price Must Exist for Cost Posting (FO617R)**
    - Logic: In `finn_kost` subroutine, if the resolved cost price (`w_kost`) is zero after all lookup attempts, the cost line is blocked.
    - File: SSTAPF (sales statistics)
    - Field: SFGJKP (average sliding cost) or SFSUMK (total cost)
    - Condition: `w_kost = 0` → `b_feil = *on`.

13. **POS/Retail Transactions Excluded from Cost Posting (FO617R)**
    - Logic: If the sales statistics record contains a cash-register bong reference (`sfbong ≠ ' '`), the record is skipped (POS transactions do not generate cost postings).
    - File: SSTAPF
    - Field: SFBONG
    - Condition: `sfbong ≠ ' '` → `goto les_neste`.

14. **Order Header Line Code Filter (FO618R)**
    - Logic: FO618R only processes records where the selection code type is 'H' (header-level). Non-header records are skipped.
    - File: FFAKST (order summation/selection register)
    - Field: FKSKOD
    - Condition: `FKSKOD ≠ 'H'` → `goto XSLUTT`.

15. **RF Picking Register Must Exist (FO619R)**
    - Logic: FO619R checks for an active RF picking register entry (LWPLPF) for the order/suffix combination. If no record is found, the program terminates without creating a rest order.
    - File: LWPLPF (RF picking register)
    - Fields: Order number + suffix composite key
    - Condition: `not %found(lwpll1)` → `goto avslutt`.

---

## Configuration and Authorization Rules

16. **`POST_KOST` Switch Controls Which Item Types Get Cost Postings (FO617R)**
    - Logic: CO402R is called with key `'POST_KOST'`. If `verdi1 = ' '` (blank, default), only stock items (`VLVTYP = 'L'`) receive cost postings. If the switch is set to a non-blank value (`u_alle` is set), all item types receive cost postings.
    - File: CO402R (switch/configuration lookup)
    - Field: verdi1 from CO402R key 'POST_KOST'
    - Condition: `u_alle = *blank` → only stock items posted; `u_alle ≠ *blank` → all items posted.

17. **Cost Price Source Selection (FO617R)**
    - Logic: CO402R with key controlling cost type (`w_ktyp`). If `w_ktyp = 'G'`, the average sliding cost (SFGJKP) is used. Otherwise, the accumulated cost (SFSUMK) is used.
    - File: SSTAPF
    - Fields: SFGJKP, SFSUMK
    - Condition: `w_ktyp = 'G'` → use SFGJKP; else use SFSUMK.

18. **Rounding Control for Invoice Lines (FO616R)**
    - Logic: Switch `b_round` in FO616R controls whether final invoice totals are rounded. EHF electronic invoices require rounded FDSUMS values (switch 6.1a).
    - Condition: `b_round = *on` → apply half-adjust rounding to line totals.

19. **AOR (External Order System) Integration (FO619R)**
    - Logic: After creating a rest order, if the original order came from the AOR external system (`FOFSYS = 'AOR'`), the system fields FOFSYS and FONREF are cleared on the rest order so it does not retain the external reference.
    - File: FOHEPF
    - Fields: FOFSYS, FONREF
    - Condition: `FOFSYS = 'AOR'` → clear FOFSYS and FONREF on rest order (switch 8.07).

20. **Transport Integration Gate (FO619R)**
    - Logic: Before sending a rest order to the transport system (AP058R), both the transport flag on the order (`RA2FLT = 1`) and the system transport flag (`FTLFLT = 1`) must be set. If either is 0, the transport call is skipped.
    - Fields: RA2FLT, FTLFLT
    - Condition: `RA2FLT = 1 and FTLFLT = 1` required for transport integration.

---

## Financial / Transactional Rules

21. **Invoice Number Assignment (FO616R)**
    - Logic: FO616R increments `t_biln` (bill number) from a number register and writes it to FFAKST records. The invoice number is also used to build KID payment references for different factoring providers.
    - File: FFAKST, FNUFPF
    - Fields: FKSONR (order number), t_biln (generated invoice number)

22. **GL Postings Written to FOVFPF (FO616R, FO617R)**
    - Logic: FO616R writes sales entries to FOVFPF (accounts-payable/receivable voucher file). FO617R separately writes cost-price entries to FOVFPF along with log records in RLOHST/RLODST/RLOBST.
    - Files: FOVFPF, RLOHST, RLODST, RLOBST
    - Condition: Both programs only write if `b_feil = *off`.

23. **Sales Statistics Updated (FO616R)**
    - Logic: FO616R updates SSTAPF (sales statistics by period) and SST3PF (customer sales statistics) as part of the invoice posting.
    - Files: SSTAPF, SST3PF

24. **Customer Credit Memo Update (FO619R)**
    - Logic: After creating a rest order, FO619R calls FO190R to update the customer's credit memo balance (RKMEPF) to reflect the reduced open order amount.
    - Program: FO190R
    - Condition: Called unconditionally after rest order creation.

25. **Consolidated Invoicing Logic (FO621R)**
    - Logic: FO621R reads FPSOPF and checks RKUNPF.RKSAMF (consolidated invoicing flag) for each customer. If the customer should NOT be consolidated (switch 8.03), no summation post (FFAKST) is written. Switch 8.05 verifies that the order number exists before writing. Switch 8.06 also sums "normal" orders to catch those that need order-type switching for consolidated billing.
    - Files: RKUNPF, FPSOPF, FFAKST
    - Fields: RKSAMF, order type, VAOAKK

26. **Project Final Invoice (FO615R)**
    - Logic: Reads previously invoiced lines from SODTPF. Identifies on-account (akonto) lines by checking SDTEK1 for prefixes 'Akonto', 'AKONTO', 'A-KONT', 'A-Kont'. Only lines where SOPROJ/SOUPRO/SOAKTI match the final order header are included. Writes "- Tidligere Fakturert" (previously invoiced) summary lines and "S L U T T F A K T U R A" (final invoice) marker lines to FODTPF.
    - Files: SODTPF (previous invoices), FODTPF (order detail)
    - Fields: SDTEK1, SOPROJ, SOUPRO, SOAKTI

---

## Status and Lifecycle Rules

27. **Order Deleted After Invoicing (FO618R)**
    - Logic: When VAOAKK = 3 (invoice/credit-note stage), FO618R calls FO410R with delete type 'FAKT' to delete the order from FOHEPF/FODTPF after the invoice has been printed.
    - Program: FO410R
    - Condition: `VAOAKK = 3` → call FO410R('FAKT').

28. **Order Selection Code Cleared After Invoicing (FO618R)**
    - Logic: After invoice print, FO618R clears the order's selection fields on FOHEPF: FOKODE = 0, FOKODA = *loval, FOKOTI = *loval, FOKOUS = *blank, FOKOWS = *blank. This removes the order from the "pending invoice" queue.
    - File: FOHEPF
    - Fields: FOKODE, FOKODA, FOKOTI, FOKOUS, FOKOWS

29. **FFAKST Summation Records Deleted After Processing (FO618R)**
    - Logic: FO618R deletes all FFAKST records keyed by FKSSRT/FKSONR/FKSSUF, cleaning up the invoice summation register.
    - File: FFAKST
    - Condition: Deleted after successful invoice-run post-processing.

30. **Credit Order Sign Inversion for Cost Posting (FO617R)**
    - Logic: If `VAOKRE = '-'` (credit order type), `b_negp = *on` is set, and the cost amount is negated when posting to FOVFPF.
    - File: VOTYPPF
    - Field: VAOKRE
    - Condition: `VAOKRE = '-'` → negate cost posting sign.

---

## Special Conditions (Program-Specific)

31. **Rest Order Warehouse Code Preservation (FO619R)**
    - Logic: When creating the rest order via FO709R, a new FOHEPF record is written with FOKODE = 3 (rest order hold code) and FOREST = 1 (rest order flag).
    - File: FOHEPF
    - Fields: FOKODE, FOREST

32. **Order-Type Switch for Combined Invoice (FO621R, switch 8.06)**
    - Logic: FO621R sums "normal" orders together with consolidated candidates to detect orders that need their order type changed before invoicing (e.g., a normal order customer who should receive a consolidated invoice).
    - Condition: Switch 8.06 active in FO621R.

33. **FO616R Batch Number and Consolidated Invoicing (FO616R)**
    - Logic: `t_bunt` (batch number) is used to group related invoices. `b_samf` flag indicates consolidated invoicing is in progress. `b_forf` flag indicates a "forfalt" (due) invoice run variant. These flags alter the order in which customers are processed and how the FAKT register is populated.
    - Fields: t_bunt, b_samf, b_forf

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| AP600R | FO614R startup | Print queue selection | If user presses F3, entire invoice run aborts (`w_in03='1'`) |
| FO410R | FO618R, VAOAKK=3 | Delete order after invoicing | Order removed from FOHEPF/FODTPF |
| FO709R | FO619R, rest order creation | Create new rest order header | New FOHEPF written with FOKODE=3, FOREST=1 |
| FO190R | FO619R, after rest order | Update customer credit memo | RKMEPF adjusted for reduced open order |
| FO577R | FO614R (indirect via FO102R) | Check S-type items on warehouse change | Warning displayed if items cannot move to new warehouse |
| FO732R | FO102R/FO103R | Update order lines when warehouse changes | FODTPF lines updated with new warehouse |
| CO402R | FO617R | Fetch `POST_KOST` switch | Controls which item types receive cost postings |
| CO402R | FO616R | Various invoice configuration switches | Round control, factoring type selection |
| VL001R | FO619R (implied) | Warehouse stock update | Inventory adjusted for rest quantities |
| FU900R | FO616R | Order status/log write | Status log entries created per invoice |
| RS760R | FO617R (indirect) | Supplier validation | Validates supplier number before cost posting |

---

## Reference Tables

| Table | Key | Purpose in Invoice Processing |
|-------|-----|-------------------------------|
| FPSOPF | FKPFIR + selection key | Order selection register — which orders are queued for invoicing |
| FOHEPF | FOFIRM + FONUMM + FOSUFF | Order header — status, type, customer, amounts |
| FODTPF | FDFIRM + FDNUMM + FDSUFF + FDLINE | Order detail lines — items, prices, quantities |
| FFAKST | FKSSRT + FKSONR + FKSSUF | Invoice summation register — accumulates totals per invoice batch |
| VOTYPPF | VAFIRM + VAOTYP | Order types — VAOAKK stage, VAOKRE credit indicator |
| SSTAPF | SFFIRM + SFPAAR + SFVARE | Sales statistics — source for cost-price lookup in FO617R |
| FOVFPF | Voucher key | GL voucher postings — both sales and cost entries |
| FNUFPF | Firm | Number register — invoice number sequence |
| LWPLPF | LWFIRM + order/suffix | RF terminal picking register — required by FO619R |
| RKUNPF | RKFIRM + RKKUND | Customer master — RKSAMF consolidated invoicing flag |
| VLAGPF | VLFIRM + VLVARE + VLLAGE | Warehouse items — VLVTYP item type for cost-posting filter |
| VVARPF | VVFIRM + VVVARE | Item master — VVVTYP item type backup for cost-posting filter |
| SODTPF | Previous order detail | Source for project final-invoice line accumulation (FO615R) |
| RLOHST/RLODST/RLOBST | Log header/detail/bilags | Cost posting audit log written by FO617R |
