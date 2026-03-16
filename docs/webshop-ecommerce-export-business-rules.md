# Webshop / eCommerce Export – Business Rules

**Module prefix:** NN
**System:** ASOFAK
**Focus:** What blocks or prevents customers, items, or shopping lists from being exported to the webshop platform

---

## Prerequisites / Master Data Requirements

- All three export programs (NN700R, NN710R, NN720R) require that the webshop configuration record exists in `ANBLPF` (logical view `ANBLL3`, key functional group + firm). If this record is not found the program sets `p_stat = 'E'` and terminates immediately — no export is written.
- NN700R additionally requires that the webshop owner record exists in `ANBHPF` (logical view `ANBHL1`), chained by the `aoleie` (owner/tenant) field from the `ANBLL3` record. If not found → `p_stat = 'E'`, export aborted.
- NN720R (shopping list export) also requires that the assortment header record exists in `VSHEPF`. Only assortments of type `'H'` (handleliste/shopping list) are processed.
- For item export (NN710R), the item's top-level item-group hierarchy must be fully populated:
  - Overgroup must exist in `VOGRPF`
  - Main group must exist in `VHGRPF`
  - Sub-group must exist in `VUGRPF`
  - Any missing level causes the item to be excluded with a warning written to the report file.

---

## Validation Rules

### Customer Export (NN700R)

| Condition | Effect |
|-----------|--------|
| `ANBLL3` record not found | `p_stat = 'E'`; entire export aborted |
| `ANBHL1` owner record not found | `p_stat = 'E'`; entire export aborted |
| `rkebiz <> 1` (customer is not flagged as business customer) | Customer record excluded from export |
| `rkemal = *blank` (no email address) | Customer skipped; warning written: `'<name> mangler E-post adresse, og utelates'`; `p_stat = 'R'` |
| `%subst(rksted:6:25) = *blank` (no postal address) | Customer skipped; warning written: `'<name> mangler postadresse, og utelates'`; `p_stat = 'R'` |
| `rkftnr = 0` (missing organisation number) | Customer exported as private customer; warning written: `'<name> mangler org-nr. Kunden opprettes som privatkunde.'`; `p_stat = 'R'` |

### Item Export (NN710R)

| Condition | Effect |
|-----------|--------|
| `ANBLL3` record not found | `p_stat = 'E'`; entire export aborted |
| Item type not `'L'` or `'S'` in any warehouse (`b_vtyp = *off`) | Item excluded |
| `b_udat = *on` AND `b_behl = *off` (expired date set, no stock on hand) | Item excluded |
| `vvtek1` starts with `'*'` (internal/blocked item marker) | Item excluded |
| Overgroup missing in `VOGRPF` | Item excluded with warning |
| Main group missing in `VHGRPF` | Item excluded with warning |
| Sub-group missing in `VUGRPF` | Item excluded with warning |

### Shopping List Export (NN720R)

| Condition | Effect |
|-----------|--------|
| `ANBLL3` record not found | `p_stat = 'E'`; entire export aborted |
| `ANBHPF` header not found | `p_stat = 'E'`; entire export aborted |
| Item type `w_vtyp <> 'L'` (not a stock item) | Item line excluded from shopping list |
| Expiry date set AND `w_dato >= w_udat` (item expired) | Item line excluded |
| `vvtek1` starts with `'*'` | Item line excluded |

---

## Configuration and Authorization Rules

- The firm number is read from the Local Data Area (LDA positions 944–946, field `l_firm`). All reads are scoped to that firm.
- The functional group (`l_fgr`, LDA positions 931–933) is used as the first key field when looking up the webshop configuration in `ANBLL3`. Different functional groups can have separate webshop configurations within the same firm.
- Country code defaults to `'NO'` (Norway) for all customers; overridden if `RKUNPF.rkland` is non-blank.
- The export file format is a semicolon-delimited EDIFACT-like structure with segments: `UNH` (message header), `BGM` (message start, type `IMLCUSIMP`), `ORG` (organisation), `CUS` (customer), `CPR` (customer project), `UNT` (message trailer). The message type is fixed at compile time and cannot be configured.

---

## Financial / Transactional Rules

- Customer export writes `d_cusbsal` (bonus balance) and `d_cusbdat` (bonus date) fields sourced from `EKBTPF` (logical view `EKBTIR`, keyed by firm + customer). If no bonus records exist, both fields are exported as zero.
- Customer export writes `d_cuscre = 0` unconditionally — the credit indicator is always sent as zero; credit limits are not propagated to the webshop via this export.
- Item price data for NN710R is sourced from the assortment detail (`VSDTPF`) and price list files; the exact price logic is complex (8 key-combination lookups in VD700R for item selection). Items with no price in the relevant assortment may still be exported without a price.

---

## Status and Lifecycle Rules

- **Item active status:** NN710R sets the active status flag to `'N'` (inactive) in the exported record when any of the following is true:
  - `vvnuda <= today` (item's "new until" date has passed)
  - `vvudat <= today` (item's expiry date has passed)
  - Warehouse-specific expiry date `<= today`
- Items are exported as active (`'Y'`) by default unless one of the above date conditions triggers.
- Customer projects (CPR segment) are exported only when the project date (`fltdat`) is on or after the current run date (`w_dato`). Past-dated projects are silently excluded from the export.
- A customer project is flagged as a "diverse project" (`d_cprdiv = 'D'`) when `flkpro > 900000`.

---

## Special Conditions

- **Contact person override (NN700R):** If a contact person in `RUKPPF` has role `'EHANDEL'`, that person's email address (`rumail`) overrides the customer's own email address (`rkemal`) in the CUS segment export field `d_cusmai`. This is an important override — the webshop login email may differ from the billing email.
- **Project start date (9.02 fix):** If `flfdat = *loval` (no start date set on a customer project in `FKPRPF`), the exported start-date field `d_cprfda` is set to `0` rather than a garbled low-value date. This prevents the webshop from rejecting records with invalid dates.
- **Organisation-level ID (9.02 fix):** Both CUS and CPR segments include `d_cusorgid` / `d_cprorgid` set to `aoheie` (the webshop owner identifier from `ANBHPF`). This ties all exported entities to the correct webshop owner.
- NN720R calls `VD700R` to build the item selection file. The same item-exclusion rules applied in VD700R (warehouse type, expiry, text markers) apply before items reach the shopping list export.
- Items with `vvtek1` starting with `'*'` are excluded across all three export programs — this is a system-wide convention for items that must not be exposed externally.

---

## Subprogram Calls Affecting Logic

| Caller | Called Program | Purpose | Blocking Effect |
|--------|---------------|---------|-----------------|
| NN700R | AA005R (via indirect) | Error logging | — |
| NN720R | VD700R | Builds item selection from assortment | Item not meeting VD700R criteria is excluded |
| NN710R | (item group lookups) | VOGRPF, VHGRPF, VUGRPF chain | Missing group → item excluded |

---

## Reference Tables

| Table | Logical View | Key Fields | Usage |
|-------|-------------|-----------|-------|
| ANBLPF | ANBLL3 | functional group + firm | Webshop configuration; missing record blocks all exports |
| ANBHPF | ANBHL1 | owner (aoleie) | Webshop owner; missing record blocks customer and shopping-list export |
| RKUNPF | RKUNL1 | firm | Customer master; sequential read in NN700R |
| RUKPPF | RUKPL6 | firm + customer | Contact persons; role `EHANDEL` overrides customer email |
| FKPRPF | FKPRL1 | firm + customer | Customer projects; sequential read within customer loop |
| FKMAPF | FKMALR | firm + customer + project | Cadastral/property info for customer projects |
| EKBTPF | EKBTIR | firm + customer | Bonus balance and date |
| RKBPPF | RKBPLR | firm + customer | Customer access/pro-portal flag (`rkprti`); only customers with `rkprti > 0` are included |
| APOSPF | — | postal code | Postal place name lookup for customer projects |
| VVARPF | VVARL1 | firm | Item master; item type, expiry, text fields for NN710R |
| VOGRPF | VOGRL1 | firm + overgroup | Item overgroup; missing → item excluded in NN710R |
| VHGRPF | VHGRL1 | firm + maingroup | Item main group; missing → item excluded in NN710R |
| VUGRPF | — | firm + subgroup | Item subgroup; missing → item excluded in NN710R |
| VSHEPF | — | firm + assortment | Shopping list header; type `'H'` only in NN720R |
| VSDTPF | — | firm + assortment | Assortment detail; item lines for NN720R |
| CKGRPF | CKGRL2 | firm + customer category | Customer category to chain-category mapping |
| CKKLPF | CKKLLR | firm + chain category | Chain category description |
