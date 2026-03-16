# Radio Terminal Business Rules

**Module**: Radio Terminal Transactions (HR prefix)
**System**: ASLAGR
**Source files analyzed**: HR110R, HR111R, HR112R, HR120R, HR121R, HR130R, HR140R, HR150R, HR151R, HR200R, HR701R

---

## 1. Prerequisites / Master Data Requirements

The HR module supports radio/handheld terminal transactions for warehouse operations: purchase order proposals (HR110R/HR111R), stock counting (HR120R/HR121R), goods receipt (HR130R), location registration (HR140R), picking (HR150R/HR151R), and item quantity calculation (HR200R). The following must exist:

| Requirement | Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|---|
| Order type must be valid (HR110R) | Order Type Valid | Chain VOTYPPF; if not found: *in31=*on; and vaosys must = 1 for purchase-type | VOTYPPF | VAFIRM/VAOTYP | Not found or wrong system → blocked |
| Supplier must exist (HR110R, HR130R) | Supplier Required | Chain RLEVPF; if not found: *in32=*on | RLEVPF | RLFIRM/RLLEVR | Not found → blocked |
| Supplier must not be blocked or passive (v6.11, v6.30) | Supplier Active | rlsprk='J' (blocked) OR rlpass='J' (passive): *in40=*on, b_feil=*on | RLEVPF | RLSPRK/RLPASS | Blocked/passive → transaction blocked |
| Warehouse must be valid (HR110R, HR120R, HR130R, HR140R) | Warehouse Valid | Calls FA720R; if b_feil=*on returned: error indicator set | LSTSPF/LSMPF | Warehouse key | Invalid warehouse → blocked |
| Batch must exist for counting (HR120R) | Batch Exists | Chain LBCHPF; if not found: *in31=*on | LBCHPF | LCFIRM/LCBATC | Not found → blocked |
| Batch must not be blocked (v6.20) | Batch Not Locked | If lcsper=1: *in34=*on, b_feil=*on | LBCHPF | LCSPER | =1 → blocked |
| Purchase order must exist (HR130R) | Order Exists | Chain LOHEPF; if not found: *in31=*on | LOHEPF | LOFIRM/LONUMM/LOSUFF | Not found → blocked |
| Purchase order must not be locked (v6.30) | Order Not Locked | If lokode <> 0: *in34=*on | LOHEPF | LOKODE | Non-zero lock code → blocked |
| Order type must allow goods receipt (v6.21) | Order Type Accumulation | vaoakk must <= 1 (not fully processed); if > 1: *in31=*on | VOTYPPF | VAOAKK | >1 → blocked (already goods-receipted) |
| Location must exist (HR140R) | Location Exists | Chain MDLOPF with warehouse+location; if not found: *in32=*on | MDLOPF | MDFIRM/MDLAGE/MDLOKA | Not found → blocked |
| Only one location at a time (HR140R v6.11) | Single Location Rule | Only one of b1loka, b1pla1, b1pla2 may be non-blank; if count > 1: *in36=*on | Input | b1loka/b1pla1/b1pla2 | Multiple filled → blocked |
| At least one location required (HR140R v6.12) | Location Required | If none of b1loka/b1pla1/b1pla2 is filled: *in32=*on | Input | b1loka/b1pla1/b1pla2 | All blank → blocked |
| Item must exist for picking (HR150R/HR151R) | Item Must Exist | Sales orders listed via FOHEPF; order type must have vaoakk=1 | VOTYPPF | VAOAKK | !=1 → order not shown in list |
| Item exists by EAN/supplier number/NOBB (HR130R) | Multi-Key Item Resolution | Item resolved via EAN (VE710R), logistics number (SQL on VVLEPF), supplier number (VVARPF by lvar), NOBB number (VVARPF by nobb), or own number | VVARPF/VVLEPF | Multiple | Not found by any key → *in33=*on |

---

## 2. Validation Rules

### HR110R — Purchase Order Proposal Parameters

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Order Type Must Be for System 1 | After finding order type, vaosys must = 1; if not: *in37=*on | VOTYPPF | VAOSYS | System type mismatch → blocked |
| Supplier Not Blocked | rlsprk='J': *in40=*on; b_feil=*on | RLEVPF | RLSPRK | Blocked supplier → blocked |
| Supplier Not Passive (v6.30) | rlpass='J': *in40=*on; b_feil=*on | RLEVPF | RLPASS | Passive supplier → blocked |
| From-Date Valid | TEST(D) on b1bdat; if invalid: *in34=*on, b_feil=*on | Input | b1bdat | Invalid date → blocked |
| To-Date Valid | TEST(D) on b1ldat; if invalid: *in36=*on, b_feil=*on | Input | b1ldat | Invalid date → blocked |
| To-Date Not Before From-Date | If both dates set and w_ldat < w_bdat: *in35=*on, b_feil=*on | Input | b1bdat/b1ldat | To before From → blocked |
| Department Must Be Valid | Calls RS707R; if w_stat <> blank: *in38=*on, b_feil=*on | RSAVDPF (via RS707R) | Department key | Invalid department → blocked |
| Salesperson Must Be Valid | Calls RS709R; if w_stat <> blank: *in39=*on, b_feil=*on | RSSELGPF (via RS709R) | Salesperson key | Invalid salesperson → blocked |

### HR120R — Stock Counting Parameters

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Batch Must Exist | Chain LBCHPF; if not found (*in90=*on): *in31=*on | LBCHPF | LCFIRM/LCBATC | Not found → blocked |
| Batch Must Not Be Locked (v6.20) | lcsper=1: *in34=*on, b_feil=*on | LBCHPF | LCSPER | Locked batch → blocked |
| Warehouse Must Be Valid | Calls FA720R; if b_feil returned: *in32=*on | LSTSPF | Warehouse key | Invalid → blocked |

### HR130R — Goods Receipt Parameters

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Order Number Must Exist | Chain LOHEPF; if not found: *in31=*on | LOHEPF | LONUMM/LOSUFF | Not found → blocked |
| Order Must Not Be Locked (v6.30) | lokode <> 0: *in34=*on | LOHEPF | LOKODE | Locked order → blocked |
| Order Type Accumulation Check (v6.21) | Chain VOTYPPF by lootyp; vaoakk must be <= 1; if > 1: *in31=*on | VOTYPPF | VAOAKK | Already processed → blocked |
| Supplier Must Exist (when searching by supplier) | Chain RLEVPF; if not found: *in32=*on | RLEVPF | RLLEVR | Not found → blocked |
| Item Not Found by Any Key | If all lookup attempts fail (EAN, logistics, supplier, NOBB, own): *in33=*on | VVARPF/VVLEPF | Multiple | No match → blocked |
| EAN Numeric Detection | v6.12: length > 8 and all characters are digits → treated as EAN; calls VE710R | VE710R | VXEANN | Non-numeric or short → not treated as EAN |

### HR140R — Location Registration Parameters

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Warehouse Must Be Valid | Calls FA720R; if b_feil returned: *in31=*on | LSTSPF | Warehouse key | Invalid → blocked |
| Only One Location Field Filled (v6.11) | If more than one of b1loka/b1pla1/b1pla2 is non-blank: *in36=*on | Input | b1loka/b1pla1/b1pla2 | Multiple filled → blocked |
| At Least One Location Required (v6.12) | If w_test=0 (none filled): *in32=*on | Input | b1loka/b1pla1/b1pla2 | All blank → blocked |
| Main Location Must Exist | Chain MDLOPF by warehouse+loka; if not found: *in32=*on | MDLOPF | MDFIRM/MDLAGE/MDLOKA | Not found → blocked |
| Alt Location 1 Must Exist (v6.11) | Chain MDLOPF by warehouse+pla1; if not found: *in34=*on | MDLOPF | MDFIRM/MDLAGE/MDLOKA | Not found → blocked |
| Alt Location 2 Must Exist (v6.11) | Chain MDLOPF by warehouse+pla2; if not found: *in35=*on | MDLOPF | MDFIRM/MDLAGE/MDLOKA | Not found → blocked |
| Override Flag Must Be J or N | b1vars must be 'J' or 'N'; if neither: *in33=*on | Input | b1vars | Invalid value → blocked |

### HR150R — Picking Order Selection

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Order Must Exist | Chain FOHEPF; if not found: *in31=*on | FOHEPF | FOFIRM/FONUMM/FOSUFF | Not found → blocked |
| Order Must Be Ready for Picking | When faakjk=1 (delivery office): calls FU706R; p_kode must = '4'; if not: *in32=*on | FOHEPF/FSTSPF | FOKODE/FU706R result | Not ready → blocked (with message) |
| Without Delivery Office: Order Not In-Use | If faakjk<>1: fokode must not = 1 (already in use); if = 1: *in32=*on | FOHEPF | FOKODE | In-use → blocked |
| Order Type Must Be Picking Type | vaoakk must = 1 (picking order type); if not: order excluded from list | VOTYPPF | VAOAKK | != 1 → filtered from subfile |
| Assigned Picker Must Match (v5.60) | If foplse <> 0 and foplse <> fbselg (user's salesperson): order excluded from list | FOHEPF | FOPLSE | Picker mismatch → filtered out |

---

## 3. Configuration and Authorization Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Firm from LDA | l_firm from LDA 944–946 used as working firm for all HR programs | LDA | l_firm | Incorrect LDA = wrong firm |
| Warehouse from LDA (HR110R, HR120R, HR200R) | l_lage from LDA 1001–1002 used as default warehouse | LDA | l_lage | Pre-populated from user session |
| Salesperson Auto-Loaded (HR110R v6.10) | FUSRPF chained by l_user; fbselg loaded as default salesperson | FUSRPF | FBSELG | Default salesperson from user register |
| Delivery Office Mode (HR150R) | FSTSPF.faakjk=1 enables delivery office (kjørekontor) mode; changes order-ready check from fokode to FU706R result | FSTSPF | FAAKJK | Mode switch changes readiness validation |
| Status Tracking via HR701R Switches | HR701R uses CO402R to check switch 'BESTF_STATUS6'; if set, status code 6 (existing purchase order) can be overridden to '0' for items linked to an order | CO402R | 'BESTF_STATUS6' | Switch controls status 6 suppression |
| Salesperson Assignment Switch (HR701R v8.06) | HR701R checks switch 'HR701_806' via CO402R; if set (u_selg=*on), supplier's saksbehandler (rlsaks) is NOT auto-assigned as salesperson on proposal | CO402R | 'HR701_806' | Switch controls saksbehandler assignment |

---

## 4. Financial / Transactional Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Purchase Order Proposal Created (HR701R) | HR701R reads LWHBPF (handheld buffer) and creates LFHEPF (proposal header) + LFDTPF (proposal lines); deletes LWHBPF records after transfer | LFHEPF/LFDTPF | LHNUMM/LFNUMM | Next number from AS100R via hent_bnumm |
| Price from VP700R and VP753R (HR701R) | Purchase price fetched via VP753R; if available, used for proposal line value (lflbel); sale price via VP700R for comparison | VP700R/VP753R | w_kopr/w_inpr | If w_inpr <> 0: w_kopr replaced with purchase price |
| Status Codes on Proposal Lines (HR701R) | Status 0=normal, 4=quantity not matching standard pack quantity, 5=quantity not divisible by minimum, 6=item already on existing purchase order, 7=cross-docking item | Derived | lfstat | Based on pack size, existing orders, cross-dock flag |
| Proposal Quantity Unit Conversion (HR701R v6.33) | finn_enhet subroutine converts scanned unit to primary purchase unit using unit conversion factors (veomre) from VVENPF | VVENPF | VEOMRE/VEENHE | Mismatched units converted before writing |
| Weight Calculated Per Line (HR701R v8.02) | w_vekt set from VVENPF.VEVEKT for the matching unit; stored in lfdtpf.lfvekt | VVENPF | VEVEKT | Zero if unit not found |
| Multi-Supplier Split (HR112R) | When supplier not specified in terminal (myldor=0), HR112R splits proposal into separate headings per supplier found on proposal lines | LFHEPF/LFDTPF | LHLDOR/LFLDOR | Automated supplier assignment from VVARPF.VVLDOR |
| Internal Order Type (HR701R v8.04) | finn_otyp: if lldipf record found with lpform='INTERN' for firm+supplier+warehouse: use lpotyp as order type; else default 'B' | LLDIPF | LPFORM/LPOTYP | Internal orders get special order type |

---

## 5. Status and Lifecycle Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Proposal Header Created Once | HR701R b_forst flag prevents duplicate headers within the same batch; set to *on after first header write | LFHEPF | b_forst | Already-created header not duplicated |
| Proposal Status Updated by LH714R | After creating/updating lines, HR701R and HR112R call LH714R to recalculate proposal header status, total amount, and weight | LFHEPF | LHSTAT/LHTBEL/LHVEKT | Consistent header from line recalculation |
| Handheld Buffer Deleted After Transfer | HR701R deletes LWHBPF records after successfully writing to LFHEPF/LFDTPF | LWHBPF | MYFIRM/MYKODE/MYTIME/MYLINE | Buffer cleared; one-way transfer |
| Stock Count Lines in LBCHPF | HR121R reads and updates LBCHPF batch counting lines; each scan updates the counted quantity | LBCHPF | LCBATC/LCLAGE | Counted quantity accumulated per scan |
| Goods Receipt in LOHEPF | HR131R writes goods receipt lines to LOHEPF/LODTPF; triggers stock level updates | LOHEPF/LODTPF | LONUMM/LOSUFF | Receipt recorded against original order |
| Audit Timestamps | HR112R records lhdate/lhtime/lhousr on header creation and lhedat/lhetim/lheusr on header modification; lfodat/lfotim/lfousr on line creation, lfedat/lfetim/lfeusr on modification | LFHEPF/LFDTPF | *date/*tim/*usr fields | Full audit trail on all writes |

---

## 6. Special Conditions (Program-Specific)

### HR110R — Purchase Order Proposal Parameters

- Initializes order type to 'B ' (purchase order) and loads warehouse/department/salesperson from LDA.
- After validation, calls HR111R with all parameters to process the proposal lines.
- v6.10: Salesperson is loaded from FUSRPF.FBSELG based on the logged-in user.
- v6.11/v6.30: Blocked and passive suppliers are both rejected.

### HR111R — Purchase Proposal Line Entry (77.6KB — preview only)

- Large program handling individual item scanning via barcode reader.
- Resolves items via EAN (VE710R/VE711R), NOBB number, supplier number.
- v5.70+: Price group and item type used for pricing.
- v6.20: Scaffold item warning shown.
- v6.21: Supplier order number and last log code displayed on screen.
- v6.30: Timber assortment extended.
- Calls HR112R after completion to handle multi-supplier splitting.

### HR121R — Stock Counting Line Entry (54.4KB — preview only)

- Large program handling item scanning for batch counting.
- v6.11: EAN extended to 14 digits using standard lookup.
- v6.21: Scaffold item warning.
- v6.30: Timber assortment.
- Supports scanning by EAN, NOBB, supplier number, or own number.
- On item not found (v5.61): writes to JVSCAPF (scan exceptions file) for later reconciliation.

### HR130R — Goods Receipt Parameters

- Three search methods: direct order number, by supplier (calls LO562R), by item (calls LO552R).
- First shows order details (finn_onr: fills order number, reference, log codes).
- Then calls HR131R for the actual item-level receipt.
- v6.21: Prevents receipt of already-goods-receipted orders (vaoakk > 1).

### HR140R — Location Registration Parameters

- Supports up to two alternative locations (b1pla1, b1pla2) in addition to the primary location (b1loka).
- v6.11: Only one location field allowed per transaction.
- v6.12: At least one location required.
- b1vars = 'J'/'N' controls whether existing location data is overridden.

### HR150R — Picking Order Selection

- Subfile screen showing all open picking orders for the warehouse.
- Option 1: launches HR151R for picking details.
- Option 5: shows order information (customer, delivery date, project).
- Filtered by: order type must have vaoakk=1; picker assignment (foplse) if set.
- When delivery office mode (faakjk=1): FU706R status code '4' required for picking readiness.

### HR200R — Item Calculator (Unit Conversion)

- Shows up to 3 sales units for an item; F1 cycles through units.
- Unit conversion uses veomre (conversion factor) from VVENPF.
- v6.21: Loads preferred supplier from warehouse via VL721R.
- Returns p_anta (quantity in the selected unit).

### HR701R — Build Purchase Proposal from Handheld Buffer

- Called programmatically; no interactive screen.
- Reads LWHBPF (handheld batch, code='B') and creates formal proposal records.
- Status codes: 4 (pack qty mismatch), 5 (min qty mismatch), 6 (existing order), 7 (cross-dock).
- v8.01: Switch 'BESTF_STATUS6' can suppress status 6 when item is linked to a sales order (ldornu <> 0).
- v8.06: Switch 'HR701_806' controls whether supplier's saksbehandler overrides salesperson.
- v8.04: Internal purchase order type resolved via LLDIPF.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Callee | Purpose | Effect on Blocking |
|---|---|---|---|
| HR110R | FA720R | Validate warehouse | b_feil=*on → blocked |
| HR110R | RS707R | Validate department | w_stat<>blank → blocked |
| HR110R | RS709R | Validate salesperson | w_stat<>blank → blocked |
| HR110R | HR111R | Process proposal lines | Main processing after param validation |
| HR111R | VE710R | EAN to item resolution | p_stat='1' required |
| HR112R | LH714R | Recalculate proposal header status | Updates lhstat/lhtbel/lhvekt |
| HR112R | AS100R | Get next proposal number | Required before write |
| HR120R | FA720R | Validate warehouse | b_feil=*on → blocked |
| HR120R | HR121R | Process counting lines | Main processing after param validation |
| HR130R | LO562R | Find order by supplier | Returns order number |
| HR130R | LO552R | Find order by item | Returns order number |
| HR130R | VE710R | EAN resolution for goods receipt | Item found from EAN |
| HR130R | HR131R | Process goods receipt lines | Main goods receipt after param validation |
| HR150R | FU706R | Check picking status of order | p_kode='4' required when faakjk=1 |
| HR150R | HR151R | Picking detail processing | Called on option 1 or direct order entry |
| HR701R | VP700R | Get sale price for proposal | w_sapr populated |
| HR701R | VP753R | Get purchase price for proposal | w_inpr replaces w_kopr if non-zero |
| HR701R | AS100R | Get next proposal number | Required before header write |
| HR701R | LH714R | Recalculate header after lines written | lhstat/lhtbel updated |
| HR701R | HR112R | Split proposal by supplier | Called when myldor=0 |
| HR701R | CO402R | Read switches | 'BESTF_STATUS6' and 'HR701_806' |
| HR701R | VL711R | Get user's price group | w_prgr populated for price lookup |

---

## 8. Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| LWHBPF | Handheld terminal buffer | MYFIRM, MYKODE, MYTIME, MYLINE |
| LFHEPF | Purchase proposal header | LHFIRM, LHNUMM |
| LFDTPF | Purchase proposal lines | LFFIRM, LFNUMM, LFLINE |
| LOHEPF | Purchase order header | LOFIRM, LONUMM, LOSUFF |
| LODTPF | Purchase order lines | LDFIRM, LDNUMM, LDSUFF |
| LBCHPF | Stock counting batch | LCFIRM, LCBATC |
| FOHEPF | Sales order header | FOFIRM, FONUMM, FOSUFF |
| MDLOPF | Warehouse location register | MDFIRM, MDLAGE, MDLOKA |
| LSTSPF | Warehouse status/configuration | LSFIRM |
| VVARPF | Item master | VVFIRM, VVVARE |
| VVENPF | Item unit register | VEFIRM, VEVARE, VEENHE |
| VLAGPF | Item warehouse register | VLFIRM, VLVARE, VLLAGE |
| RLEVPF | Supplier register | RLFIRM, RLLEVR |
| VOTYPPF | Order type register | VAFIRM, VAOTYP |
| FUSRPF | User register (salesperson, price group) | FBFIRM, FBUSER |
| FSTSPF | Firm status / configuration (faakjk) | FSFIRM |
| LLDIPF | Internal order distribution | LPFIRM, LPLDOR, LPLAGE |
