# Item Unit Business Rules

**Module**: Item / Unit EAN / GTIN Resolution (VE prefix)
**System**: ASOFAK
**Source files analyzed**: VE710R, VE711R, VE712R, VE713R, VE714R, VE715R, VE716R, VE717R, VE718R, VE720R, VE721R, VE722R

---

## 1. Prerequisites / Master Data Requirements

The VE module resolves EAN/GTIN codes for items and units. For any resolution to succeed, the following master data must be present:

| Requirement | Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|---|
| Item must exist in VVARPF | Item Existence | All GTIN lookups start with a chain to VVARPF; if item not found, program exits with blank output | VVARPF | VVFIRM/VVVARE | Not found → goto avslutt, p_stat = blank |
| Sales unit 1 must exist in VVENPF | Unit-1 Existence for salgsenh | VE712R and VE720R chain to VVENPF with saen=1; if not found, GTIN fallback to VEANPF | VVENPF | VEFIRM/VEVARE/VESAEN | Not found → skip vveppf, try veanpf |
| NOBB supplier number required | NOBB Supplier Check | VE712R and VE720R use VVLDOR (NOBB supplier) as part of the VVEPPF key; if zero, no price-giver GTIN is found | VVARPF | VVLDOR | Zero → no match in VVEPPF |
| Firm must be valid | Firm Context | All programs receive p_firm from LDA or parameter; no validation of firm itself, but all keys include firm | All files | *FIRM fields | Incorrect firm returns blank |

---

## 2. Validation Rules

### VE710R — EAN to Item/Unit Resolution

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| EAN Number Must Be Non-Zero | v6.11: if p_eann = 0, program goes to avslutt immediately; no lookup performed | VEANPF/VVEPPF | VXEANN/VEPGTI | Zero EAN → p_stat = blank, no output |
| GTIN Found in VVEPPF (price-giver level) | Chains VVEPPF with key firm+GTIN; if found: p_vare and p_enhe populated, p_stat='1' | VVEPPF | VEPFIR/VEPGTI | Found → resolved; not found → try VEANPF |
| GTIN Found in VEANPF (item EAN register) | If VVEPPF chain fails, chain VEANPF with firm+EAN; if found: p_vare populated, p_stat='1' | VEANPF | VXFIRM/VXEANN | Found → resolved; not found → p_stat blank |

### VE711R — Item+Unit to EAN Resolution

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Item Must Exist in VVARPF | Chains VVARPF; if not found: goto avslutt, p_eann=0 | VVARPF | VVFIRM/VVVARE | Not found → no EAN returned |
| GTIN in VVEPPF Must Be Non-Zero | v6.12: even if VVEPPF record found, vepgti <> 0 must be true; if vepgti=0: ignore record | VVEPPF | VEPGTI | Zero GTIN in record → not accepted |
| Unit Must Match in VEANPF | v6.11/6.12: when reading VEANPF, p_enhe must equal vxenhe AND vxeann <> 0; if unit mismatch or zero EAN: skip | VEANPF | VXENHE/VXEANN | Unit mismatch or zero EAN → skip record |

### VE712R — Item to GTIN via Sales Unit 1

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Item Must Exist in VVARPF | v6.11: chains VVARPF; if not found: goto avslutt | VVARPF | VVFIRM/VVVARE | Not found → no GTIN returned |
| GTIN in VVEPPF Must Be Non-Zero | vepgti <> 0 required; if record found but GTIN is zero: ignore | VVEPPF | VEPGTI | Zero GTIN → ignored |
| VEANPF Unit Must Match Sales Unit 1 | v6.10: w_sae1 (from VVENPF saen=1 lookup) must equal vxenhe; if mismatch: skip | VEANPF | VXENHE | Unit mismatch → skip VEANPF record |
| VEANPF EAN Must Be Non-Zero | v6.12: vxeann <> 0 required even when unit matches | VEANPF | VXEANN | Zero EAN → skip record |

### VE720R — Item to GTIN (Alphanumeric Return with GTIN Type)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Item Must Exist in VVARPF | Chains VVARPF; if not found: goto avslutt, p_eann = blank | VVARPF | VVFIRM/VVVARE | Not found → blank return |
| GTIN in VVEPPF Must Be Non-Zero | vepgti <> 0 required | VVEPPF | VEPGTI | Zero → not accepted |
| GTIN Type Must Be Set in JVPKPF | v9.02: SQL query on JVPKPF fetches JWGTYP for item+unit; if GTYP is blank ('     '): no formatting applied | JVPKPF | JWGTYP | Blank type → p_eann not set |
| GTIN8 Extraction | If w_gtyp='GTIN8 ': p_eann = %subst(w_gtin:7:8) — rightmost 8 digits | Derived | w_gtin | Applies only when type matches |
| GTIN12 Extraction | If w_gtyp='GTIN12': p_eann = %subst(w_gtin:3:12) | Derived | w_gtin | Applies only when type matches |
| GTIN13 Extraction | If w_gtyp='GTIN13': p_eann = %subst(w_gtin:2:13) | Derived | w_gtin | Applies only when type matches |
| GTIN14 Extraction | If w_gtyp='GTIN14': p_eann = w_gtin (full 14 digits) | Derived | w_gtin | Applies only when type matches |

---

## 3. Configuration and Authorization Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| NOBB Supplier Governs VVEPPF Lookup | VVLDOR from VVARPF is used as the price-giver key (VEPLDO) in VVEPPF; this links the item to its NOBB supplier | VVARPF | VVLDOR | If VVLDOR=0: VVEPPF lookup finds no match |
| Sales Unit 1 Determines Default GTIN | In VE712R and VE720R, VVENPF is read with VESAEN=1 to find the primary sales unit; only this unit's GTIN is returned | VVENPF | VESAEN | If no unit with saen=1: no GTIN via VVEPPF |
| GTIN Formatting Controlled by JVPKPF | VE720R uses JVPKPF.JWGTYP to determine whether to return GTIN8/12/13/14 format; only programs calling VE720R get formatted output | JVPKPF | JWGTYP | Missing type code → blank p_eann |

---

## 4. Financial / Transactional Rules

The VE programs are pure lookup/resolution utilities; they do not modify any financial data. They are called by order processing and inventory programs to resolve EAN/GTIN codes for EDI, barcode scanning, and price lookup.

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| EAN Returned as Numeric (VE710R, VE711R, VE712R) | p_eann is typed like VXEANN (numeric packed); callers must handle numeric EAN | VEANPF | VXEANN | Numeric format; 0 indicates not found |
| EAN Returned as Alphanumeric 14-char (VE720R) | p_eann is declared as char(14) in VE720R; formatted EAN strips leading zeros based on GTIN type | Derived | p_eann char(14) | Alphanumeric format for EDI output |
| GTIN Used in VVEPPF is Packed Decimal | VEPGTI in VVEPPF stores GTIN as packed decimal; VE720R converts via %editc(w_eann:'X') before extracting substring | VVEPPF | VEPGTI | Zero GTIN explicitly rejected before formatting |

---

## 5. Status and Lifecycle Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| p_stat = '1' Means Success | All VE programs set p_stat='1' only when a valid non-zero EAN/GTIN is found and the unit condition is met | All VE programs | p_stat | Any other value (blank) means no EAN found |
| Priority: VVEPPF Before VEANPF | VE711R, VE712R, VE720R all try VVEPPF first (price-giver level GTIN), then fall back to VEANPF (item-level EAN) | VVEPPF/VEANPF | VEPGTI/VXEANN | VVEPPF match short-circuits VEANPF scan |
| Sequential Scan of VEANPF | When VVEPPF yields no result, VEANPF is read sequentially (SETLL+READE) for all units of the item; first matching unit (by saen=1 or p_enhe) stops the scan | VEANPF | VXENHE | First match accepted; remaining records skipped |

---

## 6. Special Conditions (Program-Specific)

### VE710R — EAN to Item/Unit

- Two-level lookup: VVEPPF (keyed by firm+GTIN) first, then VEANPF (keyed by firm+EAN).
- Returns both item number and unit; VVEPPF provides unit, VEANPF does not guarantee unit (unit is blank when only VEANPF matches).
- v6.11 guard: zero EAN immediately exits — prevents scanning barcodes that have not been assigned.

### VE711R — Item+Unit to EAN

- Requires both item number and unit to be supplied.
- VVEPPF lookup uses the full 4-field key: firm+item+unit+supplier (VVLDOR from VVARPF).
- Falls back to VEANPF keyed by firm+item only, but then checks vxenhe = p_enhe to confirm unit match.

### VE712R — Item to GTIN via Sales Unit 1

- Does not require the caller to supply a unit; resolves the GTIN for the item's primary sales unit.
- The NOBB supplier (VVLDOR) is the fourth key field in VVEPPF — if an item has no NOBB supplier, VVEPPF will never match.
- Falls back to sequential scan of VEANPF checking unit against the stored sales unit 1 (w_sae1).

### VE720R — Item to GTIN with Type-Aware Formatting

- Version v9.02 addition: adds GTIN type awareness via JVPKPF.JWGTYP.
- Returns p_eann as a character field formatted according to the GTIN standard (8, 12, 13, or 14 digits).
- The internal computation uses %editc with 'X' mask to convert packed decimal GTIN to 14-char zero-padded string, then substrings according to type.
- If JWGTYP is blank or the SQL fetch finds no row, p_eann remains blank even though w_eann was set.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Callee | Purpose | Effect on Blocking |
|---|---|---|---|
| VE720R | SQL on JVPKPF | Fetch GTIN type (JWGTYP) for item+unit | Blank type = blank p_eann output |
| HR130R | VE710R | EAN scan resolution for goods receipt | Not found = *in33 error indicator |
| HR701R | VE710R | (via calling pattern) EAN lookup in radio terminal ordering | Used for item resolution |
| Various order programs | VE711R | Resolve EAN for item+unit in order lines | p_stat='1' required by caller |
| Various EDI programs | VE720R | Format GTIN for EDI output | Formatted alphanumeric GTIN |

---

## 8. Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| VVARPF | Item master | VVFIRM, VVVARE |
| VVENPF | Item unit register (sales units) | VEFIRM, VEVARE, VESAEN |
| VVEPPF | Item unit price-giver GTIN register | VEPFIR, VEPVAR, VEPENH, VEPLDO |
| VEANPF | Item EAN register | VXFIRM, VXVARE, VXENHE |
| JVPKPF | NOBB packaging register (GTIN types) | JWFIRM, JWVARE, JWENHE |
