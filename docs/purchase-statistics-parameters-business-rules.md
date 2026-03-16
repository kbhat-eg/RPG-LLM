# Purchase Statistics Parameters Business Rules

**Module**: Purchase Statistics Parameters (SR prefix)
**System**: ASSTAT
**Source files analyzed**: SR112R, SR113R, SR114R, SR115R, SR121R, SR122R, SR131R, SR141R, SR151R, SR152R

---

## 1. Prerequisites / Master Data Requirements

The SR module provides parameter-selection screens for purchase statistics. Each screen stores or updates a record in SLPMPF (parameter file) identified by firm+type+code+selection key. The following master data must exist before a parameter can be saved:

| Requirement | Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|---|
| Sub-group must exist (SR112R) | Sub-Group Existence | Chain to VAUGPF with firm+hgrp+ugrp key; if not found: *in32 = *on | VAUGPF | VGFIRM/VGHOGR/VGUGRP | Not found → save blocked |
| Over-group must exist (SR114R) | Over-Group Existence | Chain to VOGRPF with firm+ogrp key; if not found: *in32 = *on | VOGRPF | VGFIRM/VGOOGR | Not found → save blocked |
| Main-group must exist (SR115R) | Main-Group Existence | Chain to VHGRPF with firm+hhgr key; if not found: *in32 = *on | VHGRPF | VGFIRM/VGHHGR | Not found → save blocked |
| Supplier category must exist (SR121R) | Supplier Category Existence | Chain to RA14PF with firm+LKOD key; if not found: *in32 = *on | RA14PF | RAFIRM/RALKAT | Not found → save blocked |
| Discount category must exist (SR122R) | Discount Category Existence | Chain to RA06PF with firm+fkod key; if not found: *in32 = *on | RA06PF | RAFIRM/RAFKOD | Not found → save blocked |
| Order type must exist (SR151R) | Order Type Existence | Chain to VOTYPPF with firm+aork key; if not found: *in32 = *on | VOTYPPF | VAFIRM/VAOTYP | Not found → save blocked |
| Firm must be in LDA | Firm from LDA | All SR programs read dsfirm from LDA position 944–946; no explicit validation of firm | LDA | dsfirm | Incorrect LDA firm → wrong parameter written |

---

## 2. Validation Rules

### SR112R — Sub-Group Selection (kode=12)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Sub-Group Key Must Be Non-Zero | If entered sub-group key (wwugrp or combined hgrp+ugrp) = 0: *in31 = *on | Input | wwugrp | Zero key → save blocked |
| Sub-Group Must Exist in VAUGPF | Chain VAUGPF; if not found: *in32 = *on | VAUGPF | VGFIRM/VGHOGR/VGUGRP | Not found → save blocked |
| Stored Key is Packed hgrp+ugrp | SLPMPF.SRVALG stores the combined hgrp+ugrp value (packed); main-group+sub-group tuple identifies the selection | SLPMPF | SRVALG | Packed composite key |

### SR114R — Over-Group Selection (kode=14)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Over-Group Key Must Be Non-Zero | If b2ogrp = 0: *in31 = *on | Input | b2ogrp | Zero → save blocked |
| Over-Group Must Exist in VOGRPF | Chain VOGRPF; if not found: *in32 = *on | VOGRPF | VGFIRM/VGOOGR | Not found → save blocked |

### SR115R — Main-Group Selection (kode=15)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Main-Group Key Must Be Non-Zero | If b2hhgr = 0: *in31 = *on | Input | b2hhgr | Zero → save blocked |
| Main-Group Must Exist in VHGRPF | Chain VHGRPF; if not found: *in32 = *on | VHGRPF | VGFIRM/VGHHGR | Not found → save blocked |

### SR121R — Supplier Category Selection (kode=21)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Supplier Category Key Must Be Non-Zero | If LKOD = 0: *in31 = *on | Input | LKOD | Zero → save blocked |
| Supplier Category Must Exist in RA14PF | Chain RA14PF; if not found: *in32 = *on | RA14PF | RAFIRM/RALKAT | Not found → save blocked |

### SR122R — Discount Category Selection (kode=22)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Discount Category Key Must Be Non-Zero | If fkod = 0: *in31 = *on | Input | fkod | Zero → save blocked |
| Discount Category Must Exist in RA06PF | Chain RA06PF; if not found: *in32 = *on | RA06PF | RAFIRM/RAFKOD | Not found → save blocked |

### SR131R — From/To Item Number Ranges (kode=31)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| From-Item Must Be <= To-Item | For each of 10 from/to pairs (wwfvar/wwtvar arrays): if from > to, loop back silently with no error indicator | Input arrays | wwfvar(i)/wwtvar(i) | From > To → save loops back |
| To-Item Zero Means Row Not Stored | SLPMPF record for a pair is only written if wwtvar(i) <> 0 (non-blank to-value); zero to-value is silently skipped | SLPMPF | SRREST | Zero to-value → no record written |
| Both Zero Deletes Existing Record | If both wwfvar and wwtvar are zero/blank for the overall key: the existing SLPMPF record is deleted | SLPMPF | SRFIRM/SRTYPE/SRKODE/SRVALG | Both zero → delete |

### SR141R — From/To Period Ranges (kode=41)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| From-Period Must Be <= To-Period | If b2fper > b2tper: goto b2taga (loops back to input screen) | Input | b2fper/b2tper | From > To → input rejected |
| Period Format Must Be >= 195000 | If b2fper <> 0 and b2fper < 195000: goto b2taga — format YYYYPP, minimum period is 195001 (year 1950, period 1) | Input | b2fper | Too-small period → input rejected |
| Both Zero Deletes Existing Record | If b2fper=0 and b2tper=0: existing SLPMPF record is deleted | SLPMPF | SRFIRM/SRTYPE/SRKODE/SRVALG | Both zero → delete |
| New Write Only (No Update) | SR141R writes SLPMPF only when *in60 = *on (record not found); if record exists, it updates; if both zero and found, deletes | SLPMPF | All key fields | Standard SLPMPF maintenance pattern |

### SR151R — Order Type Selection (kode=51)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Order Type Must Be Non-Blank | If aork = *blank: *in31 = *on | Input | aork | Blank → save blocked |
| Order Type Must Exist in VOTYPPF | Chain VOTYPPF; if not found: *in32 = *on | VOTYPPF | VAFIRM/VAOTYP | Not found → save blocked |

---

## 3. Configuration and Authorization Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Parameter Key Structure | SLPMPF key: SRFIRM (firm) + SRTYPE (statistic type, passed as wptype) + SRKODE (sub-code, hardcoded per program) + SRVALG (selection value) | SLPMPF | SRFIRM/SRTYPE/SRKODE/SRVALG | Four-field composite key |
| Type and Code Hardcoded | Each SR screen hardcodes wwkode (e.g., SR112R: 12, SR114R: 14, SR141R: 41); the caller passes wptype; srsyst is set to 'L' on write | SLPMPF | SRKODE/SRSYST | Fixed by program; not user-enterable |
| F4 Lookup Available on All Fields | Each SR program calls a lookup program (VF511R, VG510R, VG511R, RA514R, RA506R, VV500R, VA550R) on F4 to assist selection | Various | Various | Lookup does not block; aids input accuracy |
| Firm from LDA | dsfirm (LDA 944–946) is used as wwfirm; the parameter record is stored under this firm | LDA | dsfirm | LDA firm defines which firm's statistics are parameterized |

---

## 4. Financial / Transactional Rules

The SR module stores user-defined selection criteria for statistical reports; it does not execute financial transactions. However:

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| SLPMPF as Parameter Store | SLPMPF records written by SR programs are read by the statistics batch programs to filter the data set | SLPMPF | All fields | Incorrect parameter = wrong data set in report |
| SRREST Field Carries Range Data | SR131R packs from/to item numbers into SRREST (ds: positions 1–30 text, 31–36 from-item, 37–42 to-item for SR141R); the format must match what the report reader expects | SLPMPF | SRREST | Layout is fixed; must match reader expectations |
| Text Description Set | SR141R sets SRREST.wwtext = 'Periode fra/til'; SR131R uses default text from caller | SLPMPF | SRREST (first 30 chars) | Human-readable label in parameter record |

---

## 5. Status and Lifecycle Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Delete on Zero Input | SR131R and SR141R both delete the SLPMPF record when all selection values are zero/blank | SLPMPF | SRFIRM/SRTYPE/SRKODE/SRVALG | Cleanup on clear |
| Update vs Insert Logic | All SR programs use chain-then-update-or-write pattern: if *in60=*off (found): update; if *in60=*on (not found): write new | SLPMPF | All key fields | Standard upsert |
| SRSYST Flag | srsyst is set to 'L' on write/update; meaning: this parameter was set interactively (L=lagret/saved) | SLPMPF | SRSYST | Flag distinguishes interactive vs programmatic params |

---

## 6. Special Conditions (Program-Specific)

### SR131R — Item Number Range with 10 Pairs

- Unique in the SR module: supports up to 10 from/to item number pairs stored as an array.
- Each pair has its own SLPMPF record (different SRVALG values) for the same srtype+srkode.
- Calls VV500R on F4 for item number lookup.
- The validation silently rejects pairs where from > to by looping back without an error indicator — the user sees the screen re-displayed without explanation.

### SR141R — Period Range

- Period format is YYYYPP (6-digit packed decimal, minimum 195000); this represents year and 2-digit period number.
- Periods below 195000 are invalid because no fiscal periods existed before 1950 in the Norwegian system.
- The program writes only on new record creation (chain returns not found); if record exists, it uses update path.

### SR112R — Sub-Group Uses Packed Combined Key

- SRVALG stores the combined hgrp+ugrp as a single packed decimal, not as two separate fields.
- The display shows separate fields (wwhhgr and wwugrp) but storage is combined; the reader must unpack accordingly.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Callee | Purpose | Effect on Blocking |
|---|---|---|---|
| SR112R | VF511R (F4) | Sub-group inquiry | Display only |
| SR114R | VG510R (F4) | Over-group inquiry | Display only |
| SR115R | VG511R (F4) | Main-group inquiry | Display only |
| SR121R | RA514R (F4) | Supplier category inquiry | Display only |
| SR122R | RA506R (F4) | Discount category inquiry | Display only |
| SR131R | VV500R (F4) | Item number inquiry | Display only |
| SR151R | VA550R (F4) | Order type inquiry | Display only |

---

## 8. Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| SLPMPF | Statistics parameter file | SRFIRM, SRTYPE, SRKODE, SRVALG |
| VAUGPF | Item sub-group register | VGFIRM, VGHOGR, VGUGRP |
| VOGRPF | Item over-group register | VGFIRM, VGOOGR |
| VHGRPF | Item main-group register | VGFIRM, VGHHGR |
| RA14PF | Supplier category register | RAFIRM, RALKAT |
| RA06PF | Discount category register | RAFIRM, RAFKOD |
| VOTYPPF | Order type register | VAFIRM, VAOTYP |
