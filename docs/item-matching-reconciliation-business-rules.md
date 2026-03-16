# Item Matching Reconciliation — Business Rules

**Module:** 34 (JO prefix)
**Focus:** What blocks or prevents item matching between supplier catalogues and the internal item register

---

## 1. Prerequisites / Master Data

Before item matching can proceed, the following master data must be present:

- **Company number in LDA** (`l_firm` at pos 944–946) must be set. All JO programs use this value to scope database reads to the current firm.
- **JMATPF (item matching table)** must exist and be readable. JO101R reads JMATPF to load the subfile of items requiring matching. Without this table, no matching operations can be performed.
- **RLEVPF (supplier master)** must contain a row for the supplier number entered on the JO100R parameter screen. If the supplier is not found, the import is blocked with *in31.
- **JLEVPF (Rapportør supplier register)** must contain a row for `b1ldo2` (the Rapportør supplier code) when running JO650R. If `b1ldo2 = 0` after entry, the update to EVR is blocked with *in31. If `JLENUM` (own supplier number mapped to the Rapportør supplier) is zero, the update is further blocked with *in39 or *in40.
- **VOGRPF (over-group), VHGRPF (main-group), VUGRPF (sub-group)** must contain the entered group codes for filtering JMATPF. If a group code is entered but not found in the reference table, a `b_feil = *on` flag is set and the import is blocked.
- **JMATPF.JOKODE** status field governs what operations are possible. For JO750R (reverse matching), the item must be in a matched state (`JOVKOD <> 0`). An unmatched item (JOVKOD = 0) blocks the reverse operation with *in33.

---

## 2. Validation Rules

### JO100R — Item matching parameter screen

| Condition | Effect |
|-----------|--------|
| Supplier number (`b1ldo1`) is not purely numeric | **Blocked** with *in31 — supplier number must be numeric |
| Supplier number entered but RLEVPF row not found | **Blocked** with *in31, `b_feil = *on` |
| Over-group (`b1vogr`) entered but VOGRPF row not found | `b_feil = *on` — import blocked |
| Main-group (`b1vhgr`) entered but VHGRPF row not found | `b_feil = *on` — import blocked |
| Sub-group (`b1vugr`) entered but VUGRPF row not found | `b_feil = *on` — import blocked |
| Item-from (`b1var1`) > item-to (`b1var2`) | **Blocked** with *in35 — invalid item range |
| All validations pass | JO101R is called with all filter parameters |

### JO101R — Item matching subfile (core matching engine)

| Condition | Effect |
|-----------|--------|
| Option 1 (match): JV501R (item selector) returns no selected item | No update to JMATPF — silent no-op |
| Option 1 (match): JV501R returns selected item | JMATPF.JOVAR2 and JMATPF.JOLDO2 are updated; JMATPF.JOKODE is set to 1 (matched) |
| Option 4 (unmatch): selected row in JMATPF | JOVAR2 and JOLDO2 are cleared to blank/zero; JOKODE is set to 0 (unmatched) |
| Option 5 (view): selected row | JV510R is called for display — no data change |
| JMATPF row not found for a given firm + filter combination | Subfile is empty; no matching can be performed |

### JO600R — Matched items print parameter screen

| Condition | Effect |
|-----------|--------|
| Rapportør supplier (`b1ldo1`) entered but JLEVPF row not found | `b_feil = *on` — print blocked |
| Over-group, main-group, or sub-group entered but reference table row not found | `b_feil = *on` — print blocked |
| All validations pass | JO600C (batch print program) is called with the full parameter block |

### JO650R — Update matched items to EVR parameter screen

| Condition | Effect |
|-----------|--------|
| Rapportør supplier `b1ldo2 = 0` (blank) | **Blocked** with *in31 — Rapportør supplier code is required |
| JLEVPF row not found for `b1ldo2` | **Blocked** with *in31 |
| `JLENUM` (own supplier number from JLEVPF) = 0 | **Blocked** with *in39 or *in40 — own supplier number must be mapped |
| Pricing supplier (`b1ldo3`) is optional; if non-zero, it is validated against JLEVPF | If not found: blocked with *in31 |
| `b1behp` not in (0, 1) | Blocked — price handling code must be 0 or 1 |
| `b1beht` not in (0, 1) | Blocked — text handling code must be 0 or 1 |
| `b1behg` not in (0, 1) | Blocked — group handling code must be 0 or 1 |
| All validations pass | JO650C is called with the full parameter block |

### JO700R — Update EVR from matching result parameter screen

| Condition | Effect |
|-----------|--------|
| No explicit blocking conditions defined in JO700R | Program calls JO701R unconditionally |
| `b1slet = 1` (delete expired items) | JO702R is additionally called to delete expired item entries |

### JO750R — Reverse matching parameter screen

| Condition | Effect |
|-----------|--------|
| `b1var2` (item number to reverse) is blank | **Blocked** with *in31 |
| JMATPF row not found for the entered item number | **Blocked** with *in32 — item not in matching table |
| `JMATPF.JOVKOD = 0` (item is already unmatched) | **Blocked** with *in33 — item must be in a matched state to reverse |
| All validations pass | JO751R is called with firm and item number; matching is reversed |

---

## 3. Configuration and Authorization Rules

- **LDA `l_user`** (pos 911–920): Written to audit fields on all JMATPF updates. Without a valid user in LDA, audit trail fields will be blank.
- **JMATPF.JOKODE match status codes**: The matching status field uses the following values: 0 = unmatched, 1 = matched (by user via JO101R option 1), 2 = manually entered, 3 = matched by supplier item number (leverandørs varenummer), 4 = matched by EAN number, 5 = matched by NOBB number. Only codes 0 and 1 are directly set by the interactive JO programs; codes 2–5 are set by batch import processes.
- **JLEVPF.JLENUM protection**: The `JLENUM` field on the Rapportør supplier entry is the bridge between the external Rapportør supplier code and the internal supplier number. If this mapping is absent (zero), no EVR update can be submitted through JO650R. This prevents updates reaching the item-price register without a valid internal supplier linkage.
- **Filter parameters in JO100R**: All filter parameters (supplier, over-group, main-group, sub-group, item range) are optional except that at least one must typically be populated for a meaningful import run. The program does not enforce a minimum, but an empty filter will pass all JMATPF rows, which may be unintentionally broad.

---

## 4. Financial / Transactional Rules

- **JO101R option 1 (match) JMATPF update**: When a user selects option 1 and JV501R returns a matched item, JMATPF is updated with: `JOVAR2` = the matched internal item number, `JOLDO2` = the internal supplier's own supplier number, `JOKODE` = 1. The original external item number `JOVAR1` and external supplier number `JOLDO1` are preserved unchanged.
- **JO101R option 4 (unmatch) JMATPF update**: Clearing the match sets `JOVAR2 = *blanks`, `JOLDO2 = *zero`, and `JOKODE = 0`. The price records associated with the original match are not automatically removed by JO101R; price cleanup is handled separately by JO700R/JO702R.
- **JO650C EVR update**: When invoked, JO650C updates the EVR (external item register) with price and item data from JMATPF for all matched items of the given supplier. The update scope is controlled by `b1behp` (price handling), `b1beht` (text handling), and `b1behg` (group handling) flags.
- **JO702R expired item deletion**: Called from JO700R when `b1slet = 1`. Deletes EVR entries for items that existed in a previous import but are absent from the current matching result for the given supplier and record format.

---

## 5. Status and Lifecycle Rules

- **JMATPF lifecycle**: An item enters JMATPF through the supplier catalogue import process (JR series programs). It remains in an unmatched state (JOKODE = 0) until a user performs matching via JO101R or a batch process matches it by EAN/NOBB/supplier-item-number. Once matched (JOKODE >= 1), it can only be unmatched through JO101R option 4 or JO750R.
- **JO750R reverse matching**: Once reversed, the item is set back to JOKODE = 0. JO751R performs the actual reversal by updating JMATPF. No cascade deletion of price records occurs at the time of reversal.
- **JO700R / JO701R EVR update trigger**: JO700R is the interactive entry point. JO701R performs the actual batch update. The split ensures the interactive session completes promptly while the heavy update runs as a submitted job.
- **Copy blocking in JO100R parameter copy (JR120R)**: When copying a Rapportør item-group reference (option 3 in JR120R), the target group code is checked against JRVGPF. If the target already exists, *in32 is set and the copy is blocked. This prevents accidental overwrite of existing group reference mappings.

---

## 6. Special Conditions

- **Numeric supplier number enforcement in JO100R**: The supplier number field is validated for purely numeric content using a numeric test before the RLEVPF CHAIN. Non-numeric values (e.g. alpha characters) are rejected with *in31. This is necessary because the supplier number is stored as a numeric field in RLEVPF but may be entered as a character string on the screen.
- **JO750R requires prior matching**: JO750R can only reverse a match that exists. An attempt to reverse an item with JOVKOD = 0 (already unmatched) is blocked. This prevents double-reversal and ensures the unmatched state is not incorrectly "reset" when it is already in the correct state.
- **JO650R three-supplier model**: JO650R supports three distinct supplier roles: `b1ldo2` (the Rapportør supplier — the source of the catalogue), `JLENUM` (the internal own-supplier number — mandatory for EVR update), and `b1ldo3` (the pricing supplier — the supplier whose prices take precedence). Each role is independently validated. If the pricing supplier is not provided (zero), the Rapportør supplier's prices are used as the default.
- **JO101R JMATPF key structure**: JMATPF is keyed on `JOFIRM + JOLDO1 + JOVAR1` (firm + external supplier + external item number). This composite key ensures that the same external item number from different suppliers is treated as separate entries. Matching results are stored in the non-key fields JOVAR2, JOLDO2, and JOKODE.

---

## 7. Subprogram Calls

| Caller | Called Program | Purpose |
|--------|---------------|---------|
| JO100R | JO101R | Display and process item matching subfile |
| JO101R | JV501R | Interactive item selector (lookup for matching target) |
| JO101R | JV510R | View item details (option 5) |
| JO600R | JO600C | Batch print of matched items |
| JO650R | JO650C | Batch update of matched items to EVR |
| JO700R | JO701R | Batch update of EVR from matching result |
| JO700R | JO702R | Delete expired EVR items (when b1slet = 1) |
| JO750R | JO751R | Perform reverse matching on a single item |

---

## 8. Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| JMATPF | JOFIRM + JOLDO1 + JOVAR1 | Item matching table: maps external items to internal items |
| RLEVPF | Firm + supplier number | Supplier master for filtering matched items |
| JLEVPF | Firm + Rapportør supplier code | Rapportør supplier register: maps external to internal supplier numbers |
| VOGRPF | Firm + over-group | Over-group (top-level product group) reference |
| VHGRPF | Firm + main-group | Main-group product group reference |
| VUGRPF | Firm + over-group + main-group + sub-group | Sub-group product group reference |
| JVARPF | Firm + item number | Internal item master |
| JVPKPF | Firm + item + unit | Item packaging/unit data |
| JVPRPF | Firm + item + supplier | Item-supplier price register |
| JVLEPF | Firm + item + supplier | Item-supplier linking (with EAN, NOBB, dates) |
