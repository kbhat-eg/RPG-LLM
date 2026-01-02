```markdown
# Program Overview  
This ILE RPG program reads a semicolon-separated “spreadsheet” file (`FALINJ`), parses each line into fields, looks up related master‐data (items, prices, vendors), and creates detailed transaction records (`FODTLUR`). It also updates invoice headers (`FOHELU`).  

---

## 1. File Definitions (F-Specs)  
- **FALINJ** (input, unformatted, disk)  
  - Contains lines from the spreadsheet, each in a 256‐char field `FATRAN`.  
- **Various Master Files** (`VVARL1`, `JVVARL1`, `VPGRL1`, `VMVAL1`, `RLEVL1`, …)  
  - Used to look up item master, price groups, tax codes, vendors, etc.  
- **FODTLUR** (update detail)  
  - Where parsed and enriched transaction lines are written/updated.  
- **FOHELU** / **FOHELUR** (update header)  
  - Invoice/order header files.

---

## 2. Local Data Area / System Variables  
- `l_user`, `l_firm` … taken from system local data area (UDS).  
- Work fields for date/time stamping, user, error flags.

---

## 3. Key Data Areas (D-Specs)  
- *Key fields* for chain/read operations (e.g. `fodtlu_key`, `vvarl1_key`).  
- Separate sets for detail line positioning, item lookups, price lookups, vendor lookups.

---

## 4. Working Variables  
- **Input parsing**  
  - `w_tran` (256 chars): raw line read from `FALINJ`.  
  - Arrays of delimited fields: `d_type`, `d_vare`, `d_tek1`, `d_tek2`, `d_anta`, `d_enhe`, `d_levr`, `d_pris`.  
- **Numeric conversions**  
  - `w_anta` (decimal quantity), `w_pris` (decimal price).  
  - Character checks (`c_digi`, `c_dig2`) to ensure numeric fields.  
- **Lookup results**  
  - Item master fields: `vvvare`, `vvtek1`, `vvenhe`, `vvnobb`, …  
  - Price lookup parameters / return: `p_sapr` (sales price), `p_kopr` (cost price), etc.  
- **Totals and flags**  
  - `w_salg` accumulates line totals.  
  - Error messages: `w_m1`, `w_m2` and calls to `AA007R`.  

---

## 5. Initialization (INZSR)  
- Parameters `w_numx`, `w_sufx` → `w_numm`, `w_suff` (invoice number).  
- Initialize key lists for all file lookups (detail lines, items, price groups, tax, vendors).  
- Set default firm code from `l_firm`.

---

## 6. Main Logic Flow  

1. **Determine last used detail line**  
   - Chain `FODTLU` for highest `line` number for this firm/invoice.  
   - If found, set `w_line = fdline`, else `0`.

2. **Open input and header**  
   - `OPEN FALINJ` & chain `FOHELU` to read header (to get `fotots` = amount so far).

3. **Read/Process loop**  
   - `READ FALINJ` → loop until `%EOF`.  
   - Skip lines where first char ≠ `'V'` (only process value lines).  

4. **Clean-up raw line**  
   - Mask out 5 chars after first blank‐to‐blank scan using `;` delimiter.  
   - Replace various extended characters via `%XLATE` (fix encoding).  

5. **Parse fields one by one**  
   For each semicolon‐separated field:  
   - **Field 1 (`d_type`)**: transaction type (`T`/`V`/`TX`).  
   - **Field 2 (`d_vare`)**: item number; right‐justify/pad with zeros if needed.  
   - **Field 3 & 4 (`d_tek1`, `d_tek2`)**: item text lines.  
   - If not text line (`d_vare <> 'TX'`):  
     - **Field 5 (`d_anta`)** → `w_anta` (numeric qty).  
     - **Field 6 (`d_enhe`)** → unit of measure.  
     - **Field 7 (`d_levr`)** → vendor number (validate via `RLEVL1`).  
     - **Field 8 (`d_pris`)** → `w_pris` (override price from sheet).  

6. **Vendor validation**  
   - If `d_levr` not blank, `CHAIN RLEVL1` → if not found, call error subroutine `AA007R`.

7. **Build detail record (`FODTLUR`)**  
   - Clear work fields, increment `w_line` by 10 for each new line.  
   - Set `fdnumm`, `fdsuff`, `fdline`, `fdotyp`.  
   - If text‐only line (`d_vare = 'TX'`): set `fdltyp='TX'`, copy `d_tek1/2`.  
   - Else (item line):  
     - **Lookup item master**: `CHAIN VVARL1`; if not found, try `JVARL1`.  
     - If still not found, call external program `FD120R` to fetch/purge master.  
     - **Tax code** via `VMVAL1` lookup; default to `w_mvak`.  
     - **Group codes** (`fdogrp`, `fdhgrp`, `fdugrp`).  
     - **Pricing**: call `VP700R` with date/time, item, unit, price group → returns `p_sapr` & `p_kopr`.  
       - If sheet‐provided `w_pris>0` and match in `FODTLU` staging table, override `fdsapr=w_pris`, flag `fdkpri='F'`.  
   - Compute `w_salg += fdsapr * fdanta`.  
   - Timestamp/stamp `FDETIM`, `FDEUSR`, `FDODAT`, `FDOOUS`.  
   - `WRITE FODTLUR` (or `UPDATE` when overriding existing).  

8. **Loop back** → `READ FALINJ` until end.

---

## 7. Totals & Header Updates (Commented Out)  
The latter section (under `*` specifications) contains code to:  
- Sum detail lines by net/gross, compute discounts.  
- Call `AA922R` to update memo‐balance.  
- Update header (`FOHELUR`) with new totals.  
- Display coverage contribution (`FO522R`).  
- These blocks are currently commented out (`s` specs) pending further use.

---

## 8. End of Program  
- Set `*INLR = *ON` and `RETURN`.

---

### Key Points & Pedagogical Tips  
- **Field Delimiter Logic**: uses `%SCAN(';')` + `%SUBST` to extract each field.  
- **Numeric Validation**: `%CHECK(c_dig2:d_anta)=0` ensures only digits, comma, dash allowed before conversion.  
- **Master‐Data Chaining**: always `CHAIN` on `l_firm` + key fields to fetch item/vendor/price info.  
- **Modular Error Handling**: unified calls to `AA007R` for unknown vendors/items, passing line info.  
- **Price Override**: special logic (revision 7.02) reads price from excel and updates detail record immediately.  
- **Extensibility**: commented template to re‐activate total/update logic when required.  
- **Encodings**: `%XLATE` used to map stray extended chars.  

This layered structure—parse → validate → lookup → enrich → write—simplifies debugging and allows incremental feature enhancements.