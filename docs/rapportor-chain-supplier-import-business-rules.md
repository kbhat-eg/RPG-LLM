# Rapportør Chain Supplier Import — Business Rules

**Module:** 30 (JR prefix)
**Focus:** What blocks or prevents supplier catalogue data from being imported and matched into the internal item register

---

## 1. Prerequisites / Master Data

Before Rapportør supplier catalogue import can proceed, the following conditions must be satisfied:

- **Company number in LDA** (`l_firm` at pos 944–946) must be set. All JR programs use this value to scope JRAPPF and related file accesses to the current firm.
- **JRAPPF (Rapportør register)** must contain a row for the requested Rapportør number. JR150R validates that the entered `b1rapp` exists in JRAPPF (keyed on `l_firm + jrrapp`). If the Rapportør number is zero or not found, the parameter screen blocks with *in31 or *in32.
- **JRAPPF.JRFILE** (input file name) and **JRAPPF.JRFLDR** (input folder) must be populated. JR151R's `sjekk_input` subroutine blocks if `b1file = *blank` with *in31. Without a valid file name, JR151C cannot locate the source catalogue file on the IFS.
- **JR151C (IFS file reader CL program)** must be available in the library list. JR151R calls JR151C to load the supplier catalogue file from the IFS. If JR151C is absent or fails, `b_feil = *on` is returned and the import is blocked with *in35.
- **JR152C (batch import dispatcher CL program)** must be available. JR151R calls JR152C to submit the actual import job. JR152C in turn calls JR153R (import format dispatcher), which routes to the format-specific import program (e.g. JR754R for JERNIA format). Without JR152C, no import job is submitted.
- **JVRCPF (record format register)** must contain a row for `JRAPPF.JRRFMT` (the record format code). JR100R validates this during Rapportør maintenance — if the format code is blank (*in35) or not found in JVRCPF (*in36), the Rapportør definition cannot be saved.
- **JLEVPF (Rapportør supplier / leverandør register)** must contain a row for `JRAPPF.JRLDOR` when the Rapportør is not a NOBB, S4, or S4/FELLESK format. JR100R validates the supplier number against JLEVPF; if not found (*in33), the Rapportør definition cannot be saved.
- **Format-specific input file** (the supplier catalogue) must be present in the IFS path defined by `JRAPPF.JRFLDR + JRAPPF.JRFILE`. If the file is absent, JR151C will return `b_feil = *on`, blocking the import.
- **RA30PF (price group register)** must contain a row for the price group code `b1prgr` if a price group is entered in JR151R. If the code is not found in RA30PF, `b_feil = *on` is set and the import is blocked.

---

## 2. Validation Rules

### JR100R — Rapportør definition maintenance

| Condition | Effect |
|-----------|--------|
| Rapportør name (`c2navn`) is blank | **Blocked** with *in31 |
| Supplier number (`c2ldor`) is zero AND record format (`c2rfmt`) is not 'NOBB', 'S4', or 'S4/FELLESK' | **Blocked** with *in32 — supplier number required for non-NOBB/S4 formats |
| Supplier number non-zero but JLEVPF row not found | **Blocked** with *in33 — supplier does not exist in Rapportør supplier register |
| Record format (`c2rfmt`) is blank | **Blocked** with *in35 |
| Record format non-blank but JVRCPF row not found | **Blocked** with *in36 — format code not defined |
| Input file name (`c2file`) is blank | **Blocked** with *in37 |
| Copy (option 3): target Rapportør number already exists in JRAPPF | **Blocked** with *in32 — duplicate prevention on copy |

### JR120R — Rapportør item-group reference maintenance

| Condition | Effect |
|-----------|--------|
| Description (`c2gbes`) is blank | **Blocked** with *in31 |
| Over-group (`c2gogr`), main-group (`c2ghgr`), or sub-group (`c2gugr`) is zero | **Blocked** with *in32 — all three group levels required |
| VUGRPF row not found for `(c2gogr, c2ghgr, c2gugr)` | **Blocked** with *in33 — sub-group combination does not exist |
| Copy (option 3): target group already exists in JRVGPF | **Blocked** with *in32 — duplicate on copy |

### JR130R / JR131R — Rapportør reception filter maintenance

| Condition | Effect |
|-----------|--------|
| JR130R: always calls JR131R regardless of screen content | No blocking in JR130R itself |
| JR131R: input filter group (`c1fvgr`) is blank | `goto end_c1bld` — new record creation is skipped silently |
| JR131R: copy (K1WIN): target group code already exists in JRFIPF | **Blocked** with *in32 — duplicate filter code |

### JR150R — Item import parameter screen 1 (Rapportør + date)

| Condition | Effect |
|-----------|--------|
| Rapportør number (`b1rapp`) is zero | **Blocked** with *in31 |
| Rapportør number not found in JRAPPF | **Blocked** — *in32 is set; import cannot proceed |
| Import date (`b1dato`) is zero | Defaults to current date (`w_udat`) — no blocking |
| Import date fails TEST(D) validation | **Blocked** with *in33 — invalid date |
| All validations pass | JR151R is called with firm, Rapportør number, and date |

### JR151R — Item import parameter screen 2 (file + price group)

| Condition | Effect |
|-----------|--------|
| Input file name (`b1file`) is blank | **Blocked** with *in31 |
| Price group (`b1prgr`) non-blank but RA30PF row not found for `(l_firm, b1prgr)` | `b_feil = *on` — import blocked |
| JR151C call returns `b_feil = *on` | *in35 is set — import blocked; file cannot be found or read on IFS |
| All validations pass | JR152C is called with the full parameter block (`d_list`); batch job is submitted |

### JR153R — Import format dispatcher

| Condition | Effect |
|-----------|--------|
| `d_prfm` (record format) does not match any of the known format codes (JERNIA, CHRIST, FARVERING, … FRAKTBETI) | `goto avslutt` — **no import program is called**; the batch job ends silently |
| `d_prfm = 'S4'` | JR763R is called, and additionally JR890R is called (order quantity update for S4 format) |
| All other recognised formats | Corresponding JR7xxR or JR8xxR program is called with `d_list` |

### JR754R — JERNIA format import (representative example)

| Condition | Effect |
|-----------|--------|
| `d_ekod` (change code) is blank on an input record | Record is **skipped** — header/trailer or unchanged rows are not processed |
| Item not found in JVARPF by supplier item number | New price is created using NOBB or EAN match if available; no price created if no match found |
| JLEVPF supplier not found for `d_ldor` | Price creation is **skipped** for that record |
| EAN number `d_eann` is blank | EAN-based matching is not attempted |
| NOBB number `d_nobb` is blank | NOBB-based matching is not attempted |

---

## 3. Configuration and Authorization Rules

- **LDA `l_user`** (pos 911–920): Written to audit fields `JREUSR`, `JRGESI`, `JRFESI` on JRAPPF, JRVGPF, and JRFIPF writes respectively. Without a valid user, audit fields will be blank.
- **JRAPPF.JRRFMT record format**: This field controls which format-specific import program JR153R calls. The mapping is hardcoded in the WHEN-chain of JR153R. Adding a new format requires a code change to JR153R and compilation of a new JR7xxR import program. There is no dynamic lookup — an unrecognised format code silently produces a no-op.
- **d_list parameter block structure**: The 32-byte parameter block passed between JR151R → JR152C → JR153R → JR7xxR is structured as: bytes 1–3 = firm (3 packed decimal digits), bytes 4–5 = Rapportør number (2 packed decimal digits), bytes 6–15 = record format code (10 characters), bytes 16–29 = import date (ISO date), bytes 30–31 = price group code (2 characters, v6.31+). Misalignment of this structure between caller and callee will silently corrupt import data.
- **JR151R price group (v6.31 / v8.01 PRG2)**: The price group field `b1prgr` was extended from 1 to 2 characters in v8.01 (PRG2 build). The overlay position in `d_list` for this field is offset 30 (bytes 30–31). RA30PF is keyed on `(l_firm, REDKOD)` where REDKOD is the price group code. Old programs compiled before PRG2 may only read 1 character, causing mismatched price group lookups.

---

## 4. Financial / Transactional Rules

- **JR754R (JERNIA) price update logic**: For each input record with a non-blank change code, the program attempts to find the item in JVARPF. If found by supplier item number (`JVARLDR` keyed on supplier + item), an existing price in JVPRPF is updated. If not found by supplier item number but found by NOBB or EAN, a new price entry is created on the matching item for the pricing supplier. The price fields written are: `d_inpr` (incoming price), `d_saim` (sales-in price), `d_antp` (quantity per unit), and EAN if present.
- **JR754R EAN update (v6.12)**: The EAN number from the supplier file is written to `JVLEPF` (item-supplier linking) for the pricing supplier. This allows subsequent imports to match by EAN even if the supplier item number changes.
- **JR754R GTIN search (v6.20)**: For items where the supplier EAN is blank, the program searches JVPKPF (item packaging) for a matching GTIN to establish the pricing supplier linkage on the packaging record.
- **JR153R S4 dual call**: For the S4 format (Felleskjøpet), JR763R is called for the standard price/item import, and then JR890R is called separately to update order quantities (`bestillingsantall`). The order quantity update is unique to S4 and reflects special purchasing terms.

---

## 5. Status and Lifecycle Rules

- **JRAPPF (Rapportør definition) lifecycle**: Created via JR100R (F6 + C1BLD). Maintained via option 2 (edit). Copied via option 3 (K1WIN). Deleted via option 4 (D1WIN — no cascade delete of related JRVGPF or JRFIPF rows). Option 7 calls JR120R for item-group reference maintenance; option 8 calls JR130R for reception filter maintenance.
- **JRVGPF (item-group reference) lifecycle**: Created and maintained through JR120R (called from JR100R option 7). Each row maps a Rapportør-specific group code (`JRGVGR`) to an internal over/main/sub-group combination. These mappings are used by format-specific import programs (JR7xxR) to classify imported items into the internal product hierarchy.
- **JRFIPF (reception filter) lifecycle**: Created and maintained through JR131R (called via JR130R from JR100R option 8). Each row defines a group code (`JRFVGR`) that acts as a reception filter. During import, items whose group code is in the reception filter are suppressed or specially handled, depending on the specific JR7xxR program logic.
- **Import batch job lifecycle**: When JR151R completes validation successfully, it calls JR152C, which submits JR153R as a batch job. The batch job runs JR153R → JR7xxR. The interactive screen shows a submission confirmation message (`b1sbm`). The import runs asynchronously. No progress feedback is provided to the interactive user after submission.

---

## 6. Special Conditions

- **JR100R blank supplier for NOBB/S4/S4/FELLESK**: These three record formats are catalogue formats that operate without a specific Rapportør supplier number (`c2ldor = 0` is permitted). All other formats require a non-zero supplier number linked to a valid JLEVPF row. The condition is coded as a combined check: `c2ldor = 0 AND c2rfmt <> 'NOBB' AND c2rfmt <> 'S4' AND c2rfmt <> 'S4/FELLESK'` → *in32.
- **JR153R silent no-op for unknown formats**: If a Rapportør is configured with a format code that is not in the WHEN-chain of JR153R (e.g. a typo or a format retired from the list), JR153R executes `goto avslutt` without calling any import program and without reporting an error. The batch job completes with RC=0, giving no indication that no data was processed.
- **JR151R IFS file check via JR151C**: Before submitting the batch import, JR151R calls JR151C to verify that the input file exists and can be read. This is a pre-flight check. If the file is absent at the time of JR151R execution but appears by the time the batch job runs, the batch job will succeed. Conversely, if the file is present at JR151R time but deleted before the batch job runs, the batch job will fail.
- **JR754R v7.01 price-not-found worksheet**: Since v7.01, JR754R writes an Excel-compatible file (LWEXPF / LEXCPF) listing all Jernia records where no matching item was found in JVARPF. This provides the buyer with a list of unmatched supplier items for manual reconciliation. The worksheet is populated regardless of whether any items were successfully matched.
- **v8.01 PRG2 two-character price group**: The price group code `b1prgr` was extended from 1 to 2 characters in a specific build tagged `PRG2`. This change affects JR151R, JR153R, and all JR7xxR programs that reference price group in their processing. Programs compiled before PRG2 and programs compiled after PRG2 use different overlay offsets for this field in the d_list parameter block. Mixed-version library lists can cause silent price group misassignment.

---

## 7. Subprogram Calls

| Caller | Called Program | Purpose |
|--------|---------------|---------|
| JR100R | JR120R | Maintain item-group references for a Rapportør (option 7) |
| JR100R | JR130R | Maintain reception filters for a Rapportør (option 8) |
| JR100R | JL500R | Supplier lookup (F4 on supplier field) |
| JR100R | JF500R | Record format lookup (F4 on format field) |
| JR130R | JR131R | Reception filter subfile maintenance |
| JR120R | VG510R | Over-group lookup (F4) |
| JR120R | VG511R | Main-group lookup (F4) |
| JR120R | VG512R | Sub-group lookup (F4) |
| JR150R | JR500R | Rapportør lookup (F4 on Rapportør field) |
| JR150R | JR151R | Parameter screen 2 (file, price group) |
| JR151R | JR151C | IFS file availability check (CL) |
| JR151R | JR152C | Submit batch import job (CL) |
| JR152C | JR153R | Import format dispatcher (batch) |
| JR153R | JR754R–JR840R | Format-specific import programs (one per supplier format) |
| JR153R | JR890R | S4 format: order quantity update (additional call after JR763R) |

---

## 8. Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| JRAPPF | Firm + Rapportør number | Rapportør definition: supplier, format code, IFS file/folder |
| JRVGPF | Firm + Rapportør + group code | Item-group reference: maps supplier group codes to internal product groups |
| JRFIPF | Firm + Rapportør + filter code | Reception filter: group codes to suppress or specially handle on import |
| JVRCPF | Record format code | Record format register: defines known import formats and their descriptions |
| JLEVPF | Rapportør supplier code | Rapportør supplier register: maps external to internal supplier numbers |
| JVARPF | Firm + item number | Internal item master |
| JVPKPF | Firm + item + unit | Item packaging data (GTIN, quantities) |
| JVPRPF | Firm + item + supplier | Item-supplier price register |
| JVLEPF | Firm + item + supplier | Item-supplier linking (EAN, NOBB, dates) |
| RA30PF | Firm + price group code | Price group reference table |
| VUGRPF | Firm + over + main + sub group | Sub-group product hierarchy reference |
| LWEXPF / LEXCPF | Sequential | Unmatched-items Excel worksheet (JR754R v7.01+) |
