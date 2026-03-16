# Business Logic for Purchasing and Invoicing Operations

The purchasing and invoicing module (ASLAGR system, LO prefix) maintains the LOHEPF purchase order header register and the LODTPF order line register, which together represent the full lifecycle of a purchase order from creation through invoicing and optional deletion. LO101R/LO102R are the interactive order header maintenance screens; LO103R handles invoice number and date entry; LO110R maintains order lines; LO300R copies orders; LO400R/LO401R perform cascade deletion; LO500R is the order search and dispatch screen; LO505R provides order view. Blocking rules centre on master data existence (supplier, warehouse, department, buyer, payment terms, currency), lock register checks, duplicate supplier invoice number prevention, deposit requirements, partial delivery match constraints, and Avalpf-driven mandatory field enforcement.

---

## Prerequisites / Master Data Requirements

- **RLEVPF** (supplier register): LO101R validates the supplier number (`loldor`) by chaining RLEVL1 (v6.35+). If `*in91 = *on` (supplier not found), indicator `*in34` is set on the screen and the order header cannot be saved. LO300R (copy) chains RLEVL1 to retrieve the new supplier's payment terms (`rlbetb`), VAT mode (`rlmkod`), currency (`rlvalk`), and discount category (`rlrabk`); if not found, these fields are not updated from supplier defaults.
- **LOHEPF** (purchase order header register): Must exist before LO103R (invoice entry), LO102R (delivery screen), LO505R (view), and LO300R (copy) can chain it. If the chain fails (`*in90 = *on` in LO103R), the screen is exited. LO500R checks LSTSPF for an existing lock before allowing order access.
- **LSTSPF** (lock register): LO500R chains LSTSPF by firm to check for order locks. If a lock exists and is owned by a different user, the order is presented as locked and certain operations (e.g., option 2=change) are blocked. V6.22: if the lock belongs to the same user (`laabrk = l_user`), access is allowed.
- **AVPF / AVALPF** (field validation register): LO102R reads AVALPF for program `LO102R` and firm. If parameter `u_lmat` is active and `d1lmat = 0` (delivery mode not entered), `*in38` is set and the order header cannot be saved. LO500R also reads AVALPF at v6.31+ for delivery mode validation on match operations.
- **SDEPL1** (deposit register): LO500R (v7.fb+) chains SDEPL1 by firm + order number before allowing certain operations. If a deposit exists for the order, a blocking flag is set and the requested operation cannot proceed without resolving the deposit.
- **LBHOPF** (electronic packing slip register): LO500R (v7.07+) checks LBHOPF for firm + order number + suffix. If an electronic packing slip exists for the order (`*in92 = *off`), option processing that would modify receipts is blocked.
- **SIHEPF** (supplier invoice header register): LO103R validates that the entered supplier invoice number (`d1binr`) has not already been used. It performs a SETLL/READ on SIHEL1 keyed by `firm + year + binr`. If a matching record is found (SHFIRM, SHAARR, and SHBINR all match), `*in34` is set and the screen is redisplayed with a duplicate error; the order cannot be invoiced with that number.
- **ANUMPF** (number series register): LO103R chains ANUML1 by `firm + fell + type + fast` to retrieve the valid invoice number range (`annfom`–`anntom`). If the entered invoice number (`d1binr`) is outside this range (`d1binr < wstart OR d1binr > wslutt`), `*in31` is set and entry is blocked. If ANUMPF has no record for the series, `wstart` and `wslutt` remain zero and any non-zero number passes the range check.
- **VVARPF** (item register): LO110R and LO300R use VVARL1 to look up item details for order lines. LO300R v6.30+ chains VVARL1 to determine the correct VAT code for each line at copy time via the `mva_kode` subroutine. If the item is not found, the VAT code defaults to `'1'`.
- **VLTYPF** (line type register): LO505R, LO300R, and LO110R chain VLTYL1 by firm + line type. The `valktx` field indicates whether the line is a text line (`valktx = 1`) or an item line. If VLTYPF has no record for the line type, text-line detection fails and item lines may be processed incorrectly.
- **VMVAPF** (VAT code register): LO300R v8.03 chains VMVAL1 by firm + VAT code when `lomkod = 2` to resolve `vamkui` (import VAT code). If not found, the VAT code on the copied line remains from the source.
- **FODTPF** (order line register, sales): LO400R (cascade delete) chains FODTPF lines linked to the purchase order line (`fdbenr`/`fdbsuf`/`fdblin`). If the sales order line is found, LO400R clears the back-order link fields (`fdbenr`, `fdbsuf`, `fdblin` = 0) before deleting the purchase line. If FODTPF is not found, the purchase line is still deleted but the sales order line may retain a stale link.

---

## Validation Rules

- **Supplier validation** (LO101R v6.35): Chains RLEVL1 on `firm + loldor`. If `*in91 = *on` (not found), the save is blocked with error indicator on supplier field. The order header is not written or updated.
- **Department validation** (LO101R): Chains the department code entered in the header. If `*in32 = *on` (not found), the screen is redisplayed with a department error and the save is blocked.
- **Buyer validation** (LO101R): Chains the buyer (internal user/employee) code. If `*in33 = *on` (not found), the save is blocked.
- **Warehouse validation** (LO101R subroutine sblagr): Chains the warehouse register for the entered warehouse code. If `*in34 = *on` (not found), the save is blocked. After a successful warehouse change, LO101R calls LO732R to handle warehouse-related side effects.
- **Payment terms validation** (LO101R): Chains the payment terms code. If `*in35 = *on` (not found), the save is blocked.
- **Currency validation** (LO101R): Chains the currency register for the entered currency code. If `*in36 = *on` (not found), the save is blocked.
- **Freight forwarder validation** (LO101R): Chains the freight forwarder register. If `*in37 = *on` (not found), the save is blocked.
- **Freight mode validation** (LO101R): Chains the freight mode (delivery mode) register. If `*in38 = *on` (not found), the save is blocked. LO102R additionally enforces: if AVALPF `u_lmat` is on and the delivery mode field is zero, `*in38` is set regardless of register lookup and the save is blocked.
- **Delivery terms validation** (LO101R): Chains the delivery terms register. If `*in40 = *on` (not found), the save is blocked.
- **VAT code change propagation** (LO101R v6.31): If the VAT mode (`lomkod`) changes on an existing order header, the `endr_mva` subroutine updates all LODTPF order lines with the new VAT code. If propagation is partially blocked (e.g., a line is locked), the remaining lines are still updated; there is no rollback.
- **Invoice number blank check** (LO103R): If `d1binr = *zero`, `*in35` is set and the screen is redisplayed; zero invoice numbers are not accepted.
- **Invoice number range check** (LO103R): `d1binr` must be within `wstart`–`wslutt` from ANUMPF. Numbers outside the range set `*in31` and block save.
- **Invoice number duplicate check — historical** (LO103R): SETLL/READ on SIHEL1 by `firm + year + binr`. If a matching SIHEPF record exists with same firm/year/binr, `*in34` is set and the screen is redisplayed; the duplicate invoice number blocks save.
- **Invoice number duplicate check — unposted** (LO103R): After the historical check, LO103R scans LOHELO (LOHEPF logical by order type `'IF'`) for any open order with the same `lobinr`. If found, `*in34` is set and the screen is redisplayed; using the same number on multiple open orders is blocked.
- **Invoice date validity** (LO103R): If `d1fkda <> 0`, TEST(D) is applied; if invalid date format, `*in32` is set and the screen is redisplayed. Same for due date `d1ffda` → `*in33`.
- **Delete dialog confirmation** (LO400R / LO500R): Cascade deletion is only executed after confirmation in the delete dialog. LO400R receives firm/order/suffix as parameters and proceeds immediately; the confirmation step is managed by the calling program (LO500R D1WIN).
- **Partial delivery block for match** (LO500R v8.01–v8.07): Option 55 (match/receive) first checks all LODTPF lines for the order. For each line with a back-order link to FODTPF (`ldornu + ldorsu + ldorli <> 0`), LO500R chains FODTPF. If the sales order line has a varesortiment (`fdsort`) and the purchase line's item does not belong to that sortiment, the match is blocked. No partial match is allowed for mixed-sortiment lines.
- **Electronic packing slip block** (LO500R v7.07): If LBHOPF has a record for firm + order + suffix, options that modify the received quantity are blocked for that order.
- **Deposit block** (LO500R v7.fb): If SDEPL1 has a deposit for the order, the operation requiring full payment resolution is blocked until the deposit is cleared.
- **Project validation** (LO101R v6.35): If a project code is entered in the order header, LO101R chains RP1PPF. If not found, `*in41` is set and the save is blocked.

---

## Configuration and Authorization Rules

- **Firm number from LDA** (positions 944–946): LO101R, LO102R, LO103R, LO110R, LO300R, LO400R, LO401R, LO500R, and LO505R all read `l_firm` from the LDA. All LOHEPF, LODTPF, RLEVPF, and LSTSPF lookups use this firm as the primary key.
- **Order lock ownership** (LO500R v6.22): LSTSPF lock records include the user ID. If the lock's user matches `l_user` (same user), the order is accessible for that user even while locked. A different user's lock prevents option 2 (change) and certain match operations.
- **Mandatory delivery mode** (LO102R AVALPF `u_lmat`): If the AVALPF parameter for LO102R sets `u_lmat = *on`, the delivery mode field (`d1lmat`) is mandatory. Zero value blocks save regardless of register lookup success. This is a firm-level configurable requirement.
- **Copy order — rabatt/price mode** (`p_xrab` in LO300R): The calling program passes `p_xrab` to LO300R: `0` = no recalculation, `1` = recalculate prices from supplier price list (calls LO730R with `w_enor = '1'`), `2` = zero discounts on copied lines. Incorrect `p_xrab` produces silently incorrect pricing on the copy.
- **Copy order — order reference retention** (`p_beor` v6.32 in LO300R): If `p_beor = 1`, the copied order retains the link to the originating sales order (FOHEPF `fobenr`/`fobsuf` updated; FODTPF lines updated with `fdbenr`/`fdbsuf`). If `p_beor = 0`, the sales order link is cleared on the copy. Incorrect flag produces orphaned or stale order references.
- **Copy text lines** (`p_xtxt` in LO300R): If `p_xtxt = 0`, text lines (LOTXPF) are not copied to the new order. If `p_xtxt <> 0`, text lines are copied. Existing target text line keys are skipped (no overwrite).
- **VAT mode on copy** (LO300R v6.33): If the target order has a different supplier from the source, LO300R re-chains RLEVL1 to get the new supplier's `rlmkod` (VAT mode) and propagates it to all copied lines via the `mva_kode` subroutine.
- **Post-print deletion** (LO401R): LO401R ("delete after print") is a separate program from LO400R. It deletes the order after a print run completes. LO401R also deletes LLOGPF log records only when no other LOHEPF suffix exists for the same order number (safe log cleanup). V8.01+: also deletes LOTIST (supplementary EDI register). V8.02+: writes to LDELPF (deleted orders archive) and LDESPF (deleted lines log) before deleting.

---

## Financial / Transactional Rules

- **Invoice total sign** (LO103R): `lototf` (total amount) is displayed negated if the order type's credit indicator (`vaokre`) equals `'-'`: `d1totf = lototf * -1`. This means credit orders display their total as positive on screen but are stored as negative in LOHEPF.
- **Line amount recalculation on copy** (LO300R): After copying lines, LO300R calls LO730R (price/total recalculation) if `p_xrab <> 0`. LO730R recomputes `lototk`, `lototr`, `lototi` from all copied lines. If LO730R is absent from the library list, the order header totals remain at their original values.
- **Inventory update on copy** (LO300R subroutine opplag): For each copied order line, if `laalag <> 0` (warehouse is active) AND `ldlety <> 1` (line type is not excluded from inventory), the `opplag` subroutine calls VL001R with a 128-byte data structure to update the warehouse inventory. If VL001R is absent or returns an error, the inventory is not updated but no error stops the copy.
- **Delivery date clearing on copy** (LO300R v8.02): Copied order lines have their delivery date (`ldldat`) set to `*loval` (low value = not yet set). Order header delivery date (`loldat`) is also cleared. This prevents carrying over past delivery dates to the new order.
- **Discount zeroing on copy** (LO300R when `p_xrab = 2`): `ldrab1`, `ldrab2`, and `ldsumr` are set to zero on each copied line before writing.
- **LDELPF archive write** (LO400R / LO401R v8.02+): Before deleting the LOHEPF header, if `p_type <> *blank` (a deletion type code is provided), LO401R writes one LDELPF record with full header data (supplier, totals, dates, references, user). Duplicate key writes are skipped (`not %found` check). This archive is permanent; deleted order headers are retrievable from LDELPF.
- **LDESPF line delete log** (LO400R / LO401R v8.02+): For each LODTPF line deleted, the `del_line` subroutine writes one LDESPF record capturing all line fields (item, quantity, price, discounts, VAT, GL account, activity, etc.), plus deletion timestamp, workstation, user, and program name (`'LO400R'`). Duplicate line keys are skipped.
- **LOTIST supplementary register** (LO400R / LO401R v8.01+): After deleting the LOHEPF header, both programs attempt to delete the corresponding LOTIST (supplementary EDI register) record by chaining LOTII2 by firm + order + suffix. If not found, deletion is skipped silently.

---

## Status and Lifecycle Rules

- **Order header lifecycle**: Created via LO101R (new screen). Updated via LO101R option 2 or LO102R/LO103R for supplementary fields. Deleted via LO400R (cascade) or LO401R (post-print). No soft-delete; physical deletion is immediate and irreversible unless archived in LDELPF.
- **Order lock lifecycle** (LSTSPF): LO500R places a lock on an order when a user begins editing. The lock is released when the user exits normally. An abnormal exit may leave stale locks in LSTSPF. V6.22: the same user can re-enter a locked order; other users are blocked until the lock is cleared.
- **Invoice number lifecycle** (LO103R): A supplier invoice number is entered on an already-received order. After entry, `lobinr` is stored in LOHEPF. Duplicate prevention checks SIHEPF (historical) and LOHEPF open orders (current). Once posted, the number appears in SIHEPF and prevents re-use.
- **Post-delete archival** (LO400R v8.02+): When an order is deleted with a type code (`p_type`), the header is archived to LDELPF and all lines are archived to LDESPF before physical deletion. If `p_type = *blank`, no archive records are written.
- **Copied order state**: A copied order is created with a new order number (`p_nyor`), today's date, cleared delivery date, cleared invoice fields, and cleared GL cross-references. The copy has no link to the original order in LOHEPF itself; the link to the source sales order is controlled by `p_beor`.

---

## Special Conditions (Program-Specific)

- **LO101R — order header screen 1**: Primary header maintenance. Validates supplier, department, buyer, warehouse, payment terms, currency, freight, delivery terms. V6.31 VAT code change propagates to all order lines. V6.35 validates supplier number and project. After warehouse change, calls LO732R for warehouse address updates.
- **LO102R — order header screen 2**: Delivery information (buyer, warehouse, freight mode). AVALPF `u_lmat` flag makes delivery mode mandatory. If warehouse name is blank after a warehouse change, copies the warehouse address to `lonavn`/`loadr1`/`losted` in the order header.
- **LO103R — invoice number and date entry**: Called for order type `'IF'` (supplier invoice). Validates invoice number against ANUMPF range, SIHEPF historical duplicates, and LOHEPF open order duplicates. Validates invoice date and due date. Updates LOHEPF with `lobinr`, `lofkda`, `loffda`, `lototf`. Writes tracking fields `loedat`, `loetim`, `loeusr` (v6.10+).
- **LO110R — order line maintenance**: Maintains LODTPF order lines. Opens RLEVL1 (supplier), VOTYL1 (order type), VHGRL1/VOGRL1/VUGRL1 (group hierarchies), VVARL1 (item), RA13L1 (market code) as reference files. Contact person lookup via search (v6.11+). Reference fields extended from 20 to 30 characters (v6.10+).
- **LO300R — copy order**: Copies text lines (LOTXPF), order lines (LODTPF), and header (LOHEPF) to a new order number. V6.21: clears order-line back-order links unless `p_beor = 1`. V6.22: does not update inventory flags (`lolopp = 0`). V6.30: applies VAT code logic per line. V6.33: re-reads supplier VAT mode when supplier changes. V8.01: writes LOTIST supplementary record for EDI. V8.02: clears delivery dates on copied lines.
- **LO400R — cascade delete**: Accepts firm + order + suffix + deletion type parameters. Phase 1: deletes LOTXPF text lines. Phase 2: deletes LODTPF order lines, clears FODTPF back-order links, calls opplag for inventory reversal (if warehouse active and line type not excluded). Phase 3: deletes LOHEPF header and LOTIST EDI record. Phase 4 (v6.20): deletes LLOGPF log records only if no other LOHEPF suffix remains for the order number. V8.02: writes LDELPF header archive and LDESPF line archives.
- **LO401R — delete after print**: Structurally similar to LO400R. Called automatically after order print completes. Adds LDELPF and LDESPF archival. The distinction from LO400R is the calling context (post-print automation vs. interactive user-confirmed deletion). V8.01: deletes LOTIST. V8.02: logs lines to LDESPF with `lrspgm = 'LO400R'` (note: uses LO400R name even when called from LO401R).
- **LO500R — order search and dispatch**: Complex 20+ option search screen. F14 shows supplier information (via LO575R). Validates delivery mode via AVALPF. Lock check via LSTSPF (v6.22 same-user pass). Deposit check via SDEPL1 (v7.fb). Electronic packing slip block via LBHOPF (v7.07). Partial delivery block for match (option 55, v8.01–v8.07). Varesortiment check for order lines in match.
- **LO505R — order line view**: Read-only subfile of LODTPF for a specific order. Chains LOHEPF at startup; if not found, exits immediately. Line type classification via VLTYPF (`valktx = 1` = text line). F14 calls LO575R for supplier information (v6.30+). Option 5 on a line calls LD505R for line detail view. V7.02: strips trailing `.` and `:` characters from text fields to prevent Newlook GUI misinterpretation.

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| LO730R | LO300R after copy when p_xrab <> 0 | Recalculates prices and totals for the copied order; w_enor='1' triggers price lookup from supplier price list | If absent, order totals remain from source; incorrect p_xrab causes silent pricing errors |
| VL001R | LO300R opplag subroutine per copied line (when warehouse active and line type not excluded) | Updates VLAGPF warehouse inventory with copied line quantity/price | If absent, program abends; inventory not updated |
| LO732R | LO101R after successful warehouse change | Handles warehouse address copy and related side effects | Must be in library list; absent program abends |
| LO575R | LO500R F14 and LO505R F14 (v6.30+) | Displays supplier information for the order's supplier | View-only; no blocking |
| LD505R | LO505R option 5 on order line | Displays order line detail view | View-only; no blocking |
| LO401R | Post-print automation | Delete order header + lines + text after successful print | Called automatically; same cascade as LO400R plus LDELPF/LDESPF archival |
| LO300R | LO500R option 3 (copy) | Copies order to new order number with configurable rabatt/text/order-reference flags | Destructive to inventory on warehouse-active firms; VAT re-resolves for new supplier |

---

## Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| LOHEPF (LOHEL*) | firm, numm, suff | Purchase order header register — core maintained by LO101R/LO102R/LO103R |
| LODTPF (LODTL*) | firm, numm, suff, line | Purchase order line register — maintained by LO110R |
| LOTXPF (LOTXL*) | firm, numm, suff, line | Purchase order text lines — deleted by LO400R/LO401R; optionally copied by LO300R |
| LLOGPF (LLOGL*) | firm, numm, suff, kode, date, time | Purchase order log — deleted by LO400R/LO401R when no other suffix remains |
| LDELPF (LDELL*) | firm, aarr, numm, suff | Deleted order header archive — written by LO400R/LO401R v8.02+ |
| LDESPF (LDESL*) | firm, aarr, numm, suff, line | Deleted order line archive — written by LO400R/LO401R v8.02+ |
| LOTIST (LOTII2) | firm, numm, suff | Supplementary EDI register — deleted by LO400R/LO401R v8.01+ |
| LSTSPF (LSTSL*) | firm | Lock register — checked by LO500R for order access control |
| RLEVPF (RLEVL*) | firm, levr | Supplier register — mandatory validation and defaults at order creation and copy |
| SIHEPF (SIHEL*) | firm, aarr, binr, numm, suff | Historical supplier invoice register — duplicate invoice number check in LO103R |
| ANUMPF (ANUML*) | firm, fell, type, fast | Number series register — invoice number range validation in LO103R |
| VLTYPF (VLTYL*) | firm, ltyp | Line type register — distinguishes text lines from item lines |
| VVARPF (VVARL*) | firm, vare | Item register — VAT code resolution in LO300R and LO110R |
| VMVAPF (VMVAL*) | firm, mkod | VAT code conversion register — resolves import VAT codes in LO300R v8.03 |
| FODTPF (FODTL*) | firm, numm, suff, line | Sales order line register — back-order link cleared by LO400R on purchase line delete |
| FOHEPF (FOHEL*) | firm, numm, suff | Sales order header register — updated with purchase order reference when p_beor=1 in LO300R |
| SDEPL1 | firm, numm | Deposit register — checked by LO500R v7.fb before match operations |
| LBHOPF | firm, numm, suff | Electronic packing slip register — blocks receipt modification if present (LO500R v7.07) |
| AVALPF | firm, pgm | Field validation rules — u_lmat flag enforces mandatory delivery mode in LO102R |
| RP1PPF (RP1PL*) | firm, pros | Project accounting register — project code validated by LO101R v6.35 |
