# SAF-T XML / Altinn Export — Business Rules

**Module:** 33 (AX prefix)
**Focus:** What blocks or prevents SAF-T XML generation and Altinn submission

---

## 1. Prerequisites / Master Data

Before the SAF-T XML export can be generated, the following conditions must be satisfied:

- **Company number in LDA** (`l_firm` at pos 944–946) must be set for all AX programs. Every database read is filtered on this firm value.
- **Data area AX000PAR** (100 bytes, `*dtaara`) must exist and be readable. AX000R writes the export parameters to this area; all AX section programs (AX001R, AX002R, AX310R, AX400R) read it during initialisation. If the data area is absent or inaccessible, the programs cannot resolve the report period or section flags.
- **AFIRPF (company register)** must contain a row for the firm. AX000R reads `AAFFTN` (company registration number) from this table and stores it at offset 90–98 of AX000PAR. AX001R also reads AFIRPF directly to supply values such as company name, address, and currency for the HEADER section.
- **RXMLLPF / RXMMPF (XML template files)** must be populated. AX001R reads line templates from RXMLLPF filtered on `axhead = 'HEADER'`. AX002R, AX310R, and AX400R read their corresponding section templates. If these files are empty or missing, no XML output is produced.
- **AXMLUT (XML output file)** must be writable. All AX section programs write their generated XML lines to AXMLUT. If AXMLUT is not accessible, a system error will occur.
- **RHOVPF (account master)** must be populated for the firm. AX002R iterates all accounts for the firm. An empty chart of accounts produces an empty KONTOPLAN section.
- **RHSTPF (standard account table)** must contain rows for all accounts referenced by `RHOVPF.RHREFN`. If any account's `RHREFN` is zero or the corresponding standard account is not found, AX002R aborts with an error message (see Section 2).
- **RHSAPF (account balance per period)** must contain balance rows for the selected period range for correct opening and closing balance computation in AX002R.
- **RHTRPF / RKTRPF / RLTRPF (GL, customer, and supplier transaction tables)** must contain records for the export period for AX310R to produce transaction entries.
- **RMVXPF (VAT register per code)** must be populated for the period for AX400R to generate the VAT report section.
- **AMODPF (add-on module register)** must contain a row for the SAF-T module. AX700R chains on the module name; if not found, the module is considered unlicensed and AX700R returns `p_stat = '1'`. Calling programs that check p_stat before proceeding will block the export.

---

## 2. Validation Rules

### AX000R — SAF-T parameter screen

| Condition | Effect |
|-----------|--------|
| Month entered (positions 1–2 of the 6-character period string) is not in '01'–'12' | **Blocked** — invalid month |
| Year entered (positions 3–6 of the period string) is outside the range of last 10 years (v7.01 onwards) | **Blocked** — invalid year |
| No section flag is checked (none of the checkboxes for HEADER, KONTOPLAN, HOVDBOK, MVA_MELD is selected) | **Blocked** — at least one section must be selected |
| AFIRPF row not found for firm | Company registration number `AAFFTN` is written blank to AX000PAR offset 90 — export proceeds but registration number will be empty in XML |

### AX001R — SAF-T HEADER section generator

| Condition | Effect |
|-----------|--------|
| RXMLLPF template line has `axtype = 'X'` | Template line is **skipped** — marked as not ready for production use |
| RXMLLPF line numbers 5, 6, and 99000 | Lines are **skipped** unconditionally regardless of content |
| Variable substitution type not recognised | Substitution marker `?V?` is left as-is or produces blank — XML may be malformed |
| GETVERSJOF / GETVERSJON service programs not available | Version information cannot be retrieved; *PSSR fires, program returns with error |

### AX002R — SAF-T KONTOPLAN (chart of accounts) section generator

| Condition | Effect |
|-----------|--------|
| `RHOVPF.RHREFN = 0` for an account | **Hard abort** — AA007R is called with message "Konto X mangler Standard Konto" and program terminates; no further accounts are processed |
| RHSTPF not found for `RHOVPF.RHREFN` | **Hard abort** — AA007R is called with message "Konto X finnes ikke i Standard Kontoplan" and program terminates |
| RHSAPF contains no balance rows for the report period | Opening and closing balances are computed as zero — no blocking, but balances will be incorrect |

### AX310R — SAF-T HOVDBOK (general ledger transactions) section generator

| Condition | Effect |
|-----------|--------|
| Period range contains no RHTRPF / RKTRPF / RLTRPF rows | Section is generated with no transaction entries — no blocking |
| Blank VAT code on transaction (`RBNMVA = ' '`) | Treated as code `'0'` (v8.04 change) — no error |
| RA04PF (account limits) row not found | Account limits cannot be checked — processing continues with zero values |
| RAT5PF (MVA translation table) row not found for a given VAT code | SAF-T MVA code mapping is skipped; the transaction still appears in output but without VAT mapping |

### AX400R — SAF-T MVA_MELD (VAT report) section generator

| Condition | Effect |
|-----------|--------|
| RMVXPF contains no rows for the period | VAT report section is empty — no blocking |
| RAS5PF (SAF-T code register) row not found for a VAT code | VAT description in XML will be blank |
| RA05PF (firm VAT codes) row not found | Firm VAT code description is blank |
| Period name label not in the predefined array (l_mper positions 1–6) | Bimonthly period label defaults to blank |
| Special characters (Æ, Ø, Å) in text fields | Transliterated in `spstgn` subroutine to E, O, A respectively; unknown hex codes become `%` |

### AX700R — Add-on module license checker

| Condition | Effect |
|-----------|--------|
| AMODPF row not found for the requested module name | `p_stat = '1'` — module not installed |
| AMODPF row found | `p_stat = '0'` — module installed |
| `p_disp = '1'` and module not found | AA005R is called with message "Tilleggsmodul må installeres. Kontakt EG Norge" |

### AX100R — SAF-T module maintenance

| Condition | Effect |
|-----------|--------|
| Module code `c1modu` is blank | **Blocked** with *in31 — module code is required |
| Description `c2mtxt` is blank | **Blocked** with *in31 — description is required |
| Copy (option 3): target module code already exists in AMODPF | **Blocked** with *in32 — duplicate module code on copy |

---

## 3. Configuration and Authorization Rules

- **LDA `l_user`** (pos 911–920): Used to populate audit fields on writes to AXMLUT and other tables. Without a valid user, audit fields will be blank.
- **AX000PAR offsets**: Positions 1–2 = from-month, 3–6 = from-year, 7–8 = to-month, 9–12 = to-year, 13–62 = comment text, 63–69 = section flags (one char per section type), 90–98 = company registration number from AFIRPF. The section flags govern which AX section programs are executed by the calling job.
- **RXMLLPF template control**: The `axtype` field controls line inclusion. Value `'X'` suppresses a template line entirely. This allows individual XML elements to be disabled in the template without removing them, enabling safe configuration changes without program recompilation.
- **XML entity encoding**: AX001R encodes all five XML special characters (`&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`, apostrophe → `&apos;`, `"` → `&quot;`) before writing to AXMLUT. The encoding is applied to variable substitution values; fixed template text is assumed to be pre-encoded.

---

## 4. Financial / Transactional Rules

- **AX002R opening balance**: For each account, the opening balance is computed by reading RHSAPF for `ritype = 1` (balance rows), `riraar = p_pfa` (from year), and summing fields `sa00 + sa01 + ... + sa[from_period - 1]`. A positive total is output as `INNGSALDOD` (debit); a negative total as `INNGSALDOC` (credit).
- **AX002R closing balance**: Computed by extending the opening balance sum with additional periods up to the to-period. A positive result is output as `UTGSALDOD`; negative as `UTGSALDOC`.
- **AX310R SHOP batch accumulation**: When `RLOHST.RLHTYP = 'SHOP'`, journal amounts `RLOBST.RLBSD4` / `RLBSK4` are accumulated (additive) across multiple postings against the same batch. For all other batch types, amounts overwrite the previous value.
- **AX400R VAT totals**: AX400R performs two passes over RMVXPF. The first pass accumulates the total VAT amount across all VAT codes. The second pass generates one XML element per SAF-T MVA code, including the code's share of the total.
- **AX400R sign convention**: VAT amounts are written as absolute values. The debit/credit classification in the XML is determined by the sign: positive amounts go to the debit element; negative amounts go to the credit element.

---

## 5. Status and Lifecycle Rules

- **AX000R section flags**: After the user confirms parameters on the AX000R screen, the program writes one flag character per section type at positions 63–69 of AX000PAR. A calling CL program or job control reads these flags to decide which AX section programs to submit.
- **AXMLUT line-by-line accumulation**: Each AX section program appends its XML lines to AXMLUT sequentially. The file is cleared before a new export run by the calling job. The final AXMLUT content represents the complete SAF-T file.
- **AX700R licensing gate**: AX700R is called at the start of the SAF-T export job. If `p_stat = '1'` is returned, the calling program must abort the export. AX700R itself does not abort; the caller is responsible for acting on the returned status.
- **AX002R abort behaviour**: When AX002R encounters a missing standard account mapping (RHREFN = 0 or RHSTPF not found), it calls AA007R and then exits with `*inlr = *on`. This means the AXMLUT output file will be incomplete — the KONTOPLAN section is truncated at the first invalid account. No rollback or cleanup of previously written AXMLUT lines occurs.

---

## 6. Special Conditions

- **AX000R period format**: The period is entered as a 6-character string formatted MMYYYY (2-char month followed by 4-char year), not MMYY. The year comparison uses the last 10 calendar years as the valid lower bound.
- **AX001R version retrieval**: The system version string is retrieved by calling the service programs `GETVERSJOF` (format) and `GETVERSJON` (version number). These are bound directory programs. If they are not available in the library list, a bind error at compile time or a runtime PSSR trap will occur.
- **AX002R RHREFN = 0 is a hard abort**: The `goto NextKont` instruction that previously bypassed the abort check has been removed (commented out with `s.ss` prefix). This means that in the production code, any account with `RHREFN = 0` causes an unrecoverable abort of the entire KONTOPLAN section.
- **AX400R Norwegian period naming**: Bimonthly period labels (periods 1–6) are defined internally in AX400R using an array. Periods outside this range produce a blank label. The labels correspond to Norwegian bimonthly period names (January-February through November-December).
- **AX400R special character transliteration**: The `spstgn` subroutine converts Norwegian letters (Æ, Ø, Å and their lowercase equivalents) and other non-ASCII characters to their closest ASCII equivalents for XML compatibility. Unknown characters are replaced with the `%` character.
- **XML substitution markers**: In RXMLLPF templates, the marker `?V?` identifies a position where a variable value is to be substituted. The marker `?F?` identifies a fixed text position that is substituted from a constant source. Any template line with neither marker is written verbatim to AXMLUT.

---

## 7. Subprogram Calls

| Caller | Called Program | Purpose |
|--------|---------------|---------|
| AX000R | (none — writes AX000PAR directly) | Stores parameters in data area |
| AX001R | GETVERSJOF | Get system version format string |
| AX001R | GETVERSJON | Get system version number |
| AX002R | AA007R | Display error when standard account mapping is missing |
| AX700R | AA005R | Display "module not installed" message when p_disp = '1' |

---

## 8. Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| AX000PAR | *dtaara (data area) | SAF-T export parameters: period range, section flags, company reg. number |
| AXMLUT | Sequential | Output XML file for SAF-T export |
| RXMLLPF | axhead + line number | XML template lines with substitution markers |
| RXMMPF | Template key | Variable value sources for XML substitution |
| AFIRPF | Firm | Company register: name, address, registration number |
| RHOVPF | RHFIRM + RHKONT + RHSPES + RHAVDE | Account master (chart of accounts) |
| RHSTPF | Standard account number | Standard account mapping table (SAF-T account codes) |
| RHSAPF | RHFIRM + account + year | Account balances per period |
| RHTRPF | RBFIRM + RBRAAR + RBRPER | General ledger transactions |
| RKTRPF | RNFIRM + RNNUMM | Customer transaction register |
| RLTRPF | Firm + transaction key | Supplier transaction register |
| RA04PF | Firm + account range | Account limits reference |
| RAT5PF | VAT code | MVA code translation to SAF-T format |
| RMVXPF | Firm + period + VAT code | Calculated VAT amounts per code |
| RAS5PF | SAF-T code | SAF-T MVA code descriptions |
| RA05PF | Firm + VAT code | Firm-specific VAT code descriptions |
| AMODPF | Module name | Add-on module license register |
