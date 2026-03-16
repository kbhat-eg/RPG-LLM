# Business Logic for Sales Order Processing

This module covers the creation, validation, modification, and selection of sales orders and quotes (tilbud) in the ASOFAK system. The primary programs are FO100R (order/quote registration — main screen), FO101R (order header detail — delivery method, deposit, card), FO102R (supplementary order info — warehouse/salesperson/delivery method), FO103R (extended order info — department/project/warehouse), FO500R (order inquiry and selection), FO614R (order sorting and selection for print/invoicing), and FO730R (order line recalculation). NXKORR overrides exist for FO100R and FO614R.

---

## Prerequisites / Master Data Requirements

1. **Customer Must Exist in RKUNPF**
   - Logic: FO100R chains RKUNPF using the customer number entered. If not found, the order cannot be created.
   - File: RKUNPF
   - Field: RKKUND
   - Condition: `not %found(RKUNPF)` → error indicator set, order blocked.

2. **Order Type Must Exist in VOTYPPF**
   - Logic: FO100R validates the order type against VOTYPPF. If not found, the order cannot proceed.
   - File: VOTYPPF
   - Field: VAOTYP
   - Condition: `not %found(VOTYPPF)` → order type error.

3. **Salesperson Must Exist in AUSRPF (When Required)**
   - Logic: FO102R and FO103R call RS709R to validate the salesperson code. If not found, indicator `*in33 = *on` blocks save.
   - File: AUSRPF (via RS709R)
   - Field: FOSELG (salesperson code)
   - Condition: `w_retk ≠ *blank` after RS709R call → `*in33 = *on`.

4. **Warehouse Must Exist (FO102R, FO103R)**
   - Logic: Warehouse code is validated via FÅ720R. If `w_retk = '1'` (not found), indicator `*in34 = *on` blocks save.
   - Field: FOLAGE (warehouse code)
   - Condition: `w_retk = '1'` → `*in34 = *on`.

5. **Project Must Exist in Project Register (FO103R)**
   - Logic: RS880R validates the project number. If `w_retk = 'X'`, a closed-project indicator (`*in45`) is set. If `w_retk ≠ *blank` for other errors, `*in35 = *on` blocks save.
   - Program: RS880R
   - Fields: FOPROJ, FOUPRO (sub-project)
   - Condition: `w_retk = 'X'` → *in45 (closed project); `w_retk ≠ *blank` → *in35.

---

## Validation Rules

6. **Cash Customer (Kontantkunde) May Only Create Quote-Type Orders**
   - Logic: FO100R checks if the customer is a cash customer (kontantkunde, `w_feil_kund = 'D'`). If so, and the selected order type is not a quote type (VAOAKK ≠ 0), the order creation is blocked.
   - File: RKUNPF, VOTYPPF
   - Fields: VAOAKK (order type stage; 0 = quote/tilbud)
   - Condition: `w_feil_kund = 'D' and VAOAKK ≠ 0` → blocked (switch u_til / 8.16).

7. **Credit Limit Validation**
   - Logic: FO100R compares RKUNPF.RKMEMS (current memo balance including VAT) against RKUNPF.RKGRAL (credit limit). If the balance exceeds the limit, a warning is shown. Switch `u_734` (8.16) forces credit-limit checking even for quote-type orders.
   - File: RKUNPF
   - Fields: RKMEMS, RKGRAL
   - Condition: `RKMEMS > RKGRAL` → credit limit warning; order may still be forced by user unless RKSPKR credit block is active.

8. **Credit Block Flag Prevents Order Actions (FO500R)**
   - Logic: FO500R displays a credit-block indicator when RKUNPF.RKSPKR is set. Specific order actions (e.g., picking to purchase, Valg 18) are blocked for credit-locked customers.
   - File: RKUNPF
   - Field: RKSPKR
   - Condition: `RKSPKR ≠ 0` → operations blocked; also prevents copying a quote to a credit-locked customer (switch 3.39).

9. **NGN-Sourced Order Cannot Be Manually Unlocked (FO500R)**
   - Logic: Valg 9 (unlock locked order) is blocked if the order was created by the NGN external system (`FOFSYS = 'NG'`).
   - File: FOHEPF
   - Field: FOFSYS
   - Condition: `FOFSYS = 'NG'` → Valg 9 blocked (switch 8.01).

10. **Manual Order Number Entry Blocked (FO500R)**
    - Logic: Manual entry of order numbers is only allowed if FAAMAN = 1 (system setup flag). If FAAMAN = 0, the field is protected.
    - File: FSTSPF (status codes / system setup)
    - Field: FAAMAN
    - Condition: `FAAMAN = 0` → manual number entry blocked (switch 6.29).

11. **Pick-to-Purchase Blocked for Cash Customers (FO500R)**
    - Logic: Valg 18 (pick to purchase / bestillingspunkt) is blocked for kontantkunde orders.
    - File: RKUNPF
    - Condition: kontantkunde flag active → Valg 18 blocked (switch 8.11).

12. **Pick-to-Purchase Requires Deposit to Be Paid (FO500R)**
    - Logic: If an order has a deposit requirement (SDEPPF record exists) and the deposit has not been paid, Valg 18 (pick to purchase) is blocked.
    - File: SDEPPF (deposit register)
    - Condition: Deposit record present but not paid → Valg 18 blocked.

13. **Direct Delivery Orders Cannot Be Picked to Purchase (FO500R)**
    - Logic: Direct-delivery (direkte levering) orders cannot be picked to purchase via Valg 18.
    - Condition: Direct delivery flag active on order → Valg 18 blocked.

14. **Card Order Sent to Card Company Cannot Be Released (FO500R)**
    - Logic: If a card payment order has already been transmitted to the card company (RKSJPF record exists), the order cannot be released (switch 6.30).
    - File: RKSJPF
    - Condition: RKSJPF record found for order → release blocked.

15. **Purchase Already in Progress Blocks Pick-to-Purchase (FO500R)**
    - Logic: If `b_dell` flag is set (indicating a purchase order already exists for this sales order), Valg 18 is blocked to prevent duplicate purchasing.
    - Condition: `b_dell = *on` → Valg 18 blocked.

16. **Delivery Date Mandatory Based on Order Type (FO100R)**
    - Logic: If the order type's treatment code VABKOD > 0, a delivery date (FOBDAT) must be entered before the order can be saved.
    - File: VOTYPPF
    - Field: VABKOD
    - Condition: `VABKOD > 0 and FOBDAT = 0` → delivery date required.

17. **Department (Avdeling) Required When Switch 8.01 Is Active (FO100R)**
    - Logic: If system switch 8.01 is active and the order type requires a department code, the department field (FOAVDE) must be populated.
    - Fields: FOAVDE, switch 8.01
    - Condition: `switch 8.01 = on and FOAVDE = 0` → blocked.

18. **Delivery Method Mandatory (FO102R, FO103R)**
    - Logic: If AVALPF contains an entry for program 'FO102R'/'FO103R' with field 'FOLMAT' and `avaval = 1`, then a delivery method (FOLMAT) is required. If missing, `*in38 = *on` blocks save.
    - File: AVALPF (field-level validation configuration)
    - Fields: FOLMAT, avaval
    - Condition: `u_lmat = *on and FOLMAT = 0` → `*in38 = *on` (switch 6.30).

19. **Salesperson Required When Switch Active (FO102R, FO103R)**
    - Logic: If AVALPF contains field 'FOSELG' with avaval = 1 for the program, the salesperson (FOSELG) field is mandatory.
    - File: AVALPF
    - Fields: FOSELG, avaval
    - Condition: `u_selg = *on and FOSELG = 0` → `*in33 = *on` (switch 6.31).

20. **Delivery Method Must Be Valid When Entered (FO102R, FO103R)**
    - Logic: If a delivery method code is entered and it differs from the existing value, RS717R validates it. If invalid, `*in38 = *on`.
    - Program: RS717R
    - Condition: `w_retk ≠ *blank` after RS717R → invalid delivery method, blocked.

21. **Direct Delivery Method Sets Delivery Type (FO102R)**
    - Logic: Switch 7.01 — if the delivery method code resolves to a "direct delivery" type in RA17PF (`raqdir = 1`), `b_dirl = *on` is set and `FOLETY = 1` is forced on the order header.
    - File: RA17PF
    - Field: RAQDIR
    - Condition: `raqdir = 1` → `FOLETY = 1` on FOHEPF.

22. **S-Type Items Shown When Warehouse Changes (FO102R, FO103R)**
    - Logic: If the order has a warehouse change AND `faalbe > 0` (skaffevare/S-item handling is active), FO577R is called to display S-type (special order) items that may not be stocked at the new warehouse. If `w_retk = '1'` and `faalbe > 1`, the warehouse change is blocked.
    - Program: FO577R
    - File: FSTSPF
    - Field: FAALBE
    - Condition: `w_retk = '1' and faalbe > 1` → goto d1tagb (warehouse change blocked, switch 6.24).

23. **Order Lines Updated When Warehouse Changed (FO102R, FO103R)**
    - Logic: When the warehouse on the order header changes, FO794R is called to present a selection screen asking which lines to update. If the user presses F3 (`w_in03 = '1'`), the change is discarded and the screen is re-displayed. After confirmation, FO732R is called to physically update the order lines.
    - Programs: FO794R, FO732R
    - Condition: `w_in03 = '1'` after FO794R → goto d1tagb (change cancelled).

24. **Department Change Propagates to Order Lines (FO103R)**
    - Logic: If the department (FOAVDE) is changed on the order header, subroutine `sjk_lines` updates FODTPF lines where either the line has no department (fdavde=0, version 8.03) or the line's department matches the old header department.
    - File: FODTPF
    - Fields: FDAVDE, FOAVDE
    - Condition: `FOAVDE ≠ s_avde` → `sjk_lines` called (switch 8.02/8.03).

---

## Configuration and Authorization Rules

25. **Forced Internal Order Type (FO100R)**
    - Logic: If customer flag `RKAVGF = 1` and `RKSPFA = 1`, the order type is forced to the internal order type regardless of what the user selects.
    - File: RKUNPF
    - Fields: RKAVGF, RKSPFA
    - Condition: `RKAVGF = 1 and RKSPFA = 1` → override to internal order type.

26. **Copy-to-New-Order-Type: Target Must Have VASYKOD=0 (FO500R)**
    - Logic: When copying an order to a new order type (Valg 5.03), the target order type must have VASYKOD = 0 (not a system-reserved type).
    - File: VOTYPPF
    - Field: VASYKOD
    - Condition: `VASYKOD ≠ 0` → copy blocked (switch 5.03).

27. **Wrong Invoice Customer Blocks Operations (FO100R)**
    - Logic: `w_feil_kund` flags: 'A' = wrong invoice customer (the order customer doesn't match the invoice customer profile), 'B' = wrong item customer (item is customer-specific and doesn't match), 'C' = internal customer (cannot place external orders), 'D' = cash customer (kontantkunde restrictions apply). Each flag triggers specific blocks on order creation or modification.
    - File: RKUNPF, VVARPF (item-customer restrictions)
    - Condition: Each flag value maps to specific blocking conditions.

---

## Financial / Transactional Rules

28. **Order Line Recalculation (FO730R)**
    - Logic: FO730R recalculates prices and discounts for all non-text order lines. Text lines (VLTYPF.VALKTX ≠ 0) are skipped. Price is fetched via VP900R using customer, item, price group, and delivery type. If a valid sale price (hosapr > 0.01) is returned and the price code is not 'F' (fixed), the line is updated.
    - File: FODTPF, VVARPF, VLTYPF
    - Fields: FDSAPR, FDKOPR, FDRAB1, FDRAB2, FDKPRI
    - Condition: `hosapr > 0.01 and fdkpri ≠ 'F'` → price updated; `fdkpri = 'F'` → price preserved.

29. **D-Type Item Price Not Recalculated (FO730R)**
    - Logic: If the item is of type 'D' (direct delivery / direkte levering) and exists in the item master (b_evar = *on), the price is not fetched anew — these lines are manually priced and recalculation would set the price to zero.
    - File: VVARPF
    - Field: VVVTYP = 'D'
    - Condition: `b_evar = *on and vvvtyp = 'D'` → goto no_pris, skip price fetch (switch 8.02).

30. **Almenningsrabatt (Community/Commons Discount) Boundary Check (FO730R)**
    - Logic: The commons discount amounts (s_brr1, s_brr2 — hobrr1, hobrr2) are accumulated separately in `w_suma`. After all lines, this total is compared to the previously accumulated `fobrra` (commons discount on header). The difference is the adjustment written back via FA922R.
    - File: FOHEPF
    - Field: FOBRRA
    - Condition: Always updated after recalculation.

31. **Customer Memo Balance Updated After Recalculation (FO730R)**
    - Logic: After recalculating all lines, FO730R calls FA922R to update the customer's memo balance (RKMEPF) with the difference `w_sald = fototr - fotots + w_tots - w_totr`.
    - Program: FA922R
    - Condition: Called unconditionally in `oppdat_hode` subroutine.

32. **Returgebyr (Return Fee) Applied to Invoice Total (FO730R)**
    - Logic: If CO402R key `'AKTIVER_RETURGEBYR'` returns `verdi1 = '1'`, a return fee (`u_rgeb = *on`) is activated. For each line, FOD2PF is checked for a return fee definition (fd2ber = 1 and fd2sat > 0). If found, the fee percentage is applied to the net line amount.
    - File: FOD2PF (order line supplementary / sidetabell)
    - Fields: FD2BER, FD2SAT
    - Condition: `u_rgeb = *on and fd2ber = 1 and fd2sat > 0` → `w_retb` added to totals (switch 7.01).

---

## Status and Lifecycle Rules

33. **Order Status Codes (FO500R)**
    - Logic: FOKODE on FOHEPF controls the order's status in the queue. Value 0 = open, 3 = rest/partial order. FOKODE is set/cleared by various programs. FO500R displays the current status and allows certain transitions based on business rules.
    - File: FOHEPF
    - Field: FOKODE

34. **VAOAKK Stage Progression**
    - Logic: VOTYPPF.VAOAKK defines the processing stage: 0 = quote/tilbud (no picking, no invoice), 1 = picking list (plukkliste), 2 = packing slip (følgeseddel), 3 = invoice/credit note (faktura/kreditnota). Orders can only be invoiced when at stage 3.
    - File: VOTYPPF
    - Field: VAOAKK
    - Condition: Stage 0-2 orders cannot be directly invoiced.

35. **Deposit Record Management (FO101R)**
    - Logic: Deposit records in SDEPPF are validated by year. Stale records (year mismatch) are deleted. If VAOAKK = 0 (quote type), the deposit field is write-protected (`*in50 = *on`, switch 8.03). Deposit amount cannot exceed the order total including VAT (switch 6.32).
    - File: SDEPPF
    - Fields: Deposit year, order total, VAT
    - Condition: `deposit > order_total_incl_vat` → blocked.

36. **Card Type Validation (FO101R)**
    - Logic: If a card type (korttype / RUKSPF) is entered, it must exist in the card type register. Additionally, if the customer has a registered card (RUKRPF), the card type must match.
    - Files: RUKSPF (card types), RUKRPF (customer cards)
    - Condition: Invalid card type → error indicator set.

---

## Special Conditions (Program-Specific)

37. **FO614R: Combined Invoice Mode (fk2oty = '99')**
    - Logic: Order type '99' is a special combined-invoice selector. In this mode, VOTYPPF is re-read for the order's actual type and only orders with VAOAKK = 3 qualify. This allows a single print run to cover all invoice-stage orders across multiple order types.

38. **FO100R: ASOKON Required for F4 Inquiry**
    - Logic: The F4 (inquiry) key for salesperson, warehouse, and delivery method lookup is only active if `l_syst = 'A'` (ASOKON system is installed). This prevents error calls when the accounts-receivable module is not licensed.
    - Field: LDA position 999 (`l_syst`)
    - Condition: `l_syst ≠ 'A'` → F4 inquiry blocked.

39. **FO103R: Sub-project lookup via RP601R**
    - Logic: When the user presses F4 on the sub-project (FOUPRO) field, RP601R is called with an extended parameter list (from version 7.01) including customer, date range, and salesperson filters. The returned project replaces the current values.

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| RS709R | FO102R/FO103R salesperson validation | Validate FOSELG against salesperson register | `w_retk ≠ *blank` → `*in33 = *on`, blocks save |
| RS717R | FO102R/FO103R delivery method validation | Validate FOLMAT against delivery method register | `w_retk ≠ *blank` → `*in38 = *on`, blocks save |
| FÅ720R | FO102R/FO103R warehouse validation | Validate FOLAGE | `w_retk = '1'` → `*in34 = *on`, blocks save |
| FO577R | FO102R/FO103R on warehouse change | Show S-type items on new warehouse | `w_retk = '1' and faalbe > 1` → blocks warehouse change |
| FO794R | FO102R/FO103R on warehouse change | Select which lines to update | `w_in03 = '1'` → change discarded |
| FO732R | FO102R/FO103R after warehouse change confirmed | Update order lines with new warehouse | FODTPF lines updated |
| FA922R | FO730R after recalculation | Update customer memo balance | RKMEPF adjusted by `w_sald` |
| VP900R | FO730R per item line | Fetch sale price and discount | Price, discount applied to FODTPF |
| FL710R | FO730R | Check discount category override | If override exists, customer/project zeroed for price lookup |
| VL712R | FO730R | Fetch price group for warehouse/department | w_prgr populated for VP900R call |
| CO402R | FO730R | Key 'AKTIVER_RETURGEBYR' | Activates return fee processing if set to '1' |
| RS880R | FO103R project validation | Validate project number | `w_retk = 'X'` → closed project; other error → `*in35` |
| RP601R | FO103R F4 sub-project lookup | Interactive sub-project selection | Selected project/sub-project returned to screen |

---

## Reference Tables

| Table | Key | Purpose in Order Processing |
|-------|-----|------------------------------|
| FOHEPF | FOFIRM + FONUMM + FOSUFF | Order header — status, type, customer, amounts, warehouse |
| FODTPF | FDFIRM + FDNUMM + FDSUFF + FDLINE | Order detail lines — items, prices, quantities, discounts |
| RKUNPF | RKFIRM + RKKUND | Customer master — credit limit, block flags, invoice type |
| VOTYPPF | VAFIRM + VAOTYP | Order types — VAOAKK stage, VAOKRE credit indicator, VABKOD treatment |
| FSTSPF | FAFIRM | System status/configuration — FAAMAN manual number flag, FAALBE S-item handling |
| AVALPF | AVFIRM + AVAPGM | Field-level validation configuration — delivery method mandatory, salesperson mandatory |
| SDEPPF | Firm + order + customer | Deposit register — deposit amounts per order |
| VLTYPF | VLFIRM + VALTYP | Line types — VALKTX flag identifies text lines |
| VVARPF | VVFIRM + VVVARE | Item master — VVVTYP item type, customer-specific item flags |
| VLAGPF | VLFIRM + VLVARE + VLLAGE | Warehouse items — item type per warehouse |
| RKSJPF | Card key | Card-company transmission register — prevents order release after card sent |
| FOD2PF | FDFIRM + order + suffix + line | Order line supplementary — return fee definitions |
| RA17PF | RAFIRM + delivery method | Delivery method attributes — RAQDIR direct delivery flag |
