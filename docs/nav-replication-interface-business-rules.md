# NAV Replication Interface — Business Rules

**Module:** 31 (IT prefix)
**Focus:** What blocks or prevents data from being replicated to or from NAV/Dynamics

---

## 1. Prerequisites / Master Data

Before the NAV replication interface can operate, the following master data and configuration must be present:

- **Company number** must be set in the Local Data Area (LDA) at position 944–946 (`l_firm`). All queue writes and reads are filtered on `RLOBST.RLBFIR` / `ITNAST.ITFIRM` / `IFNAST.IFFIRM` matching this value.
- **Outbound queue table ITNAST** must exist and be accessible. IT001R writes unconditionally to this table. If the table is unavailable, the PSSR error handler fires and returns `p_stat='5'`.
- **Inbound queue table IFNAST** must exist and be accessible for IT100R to read inbound NAV messages.
- **Customer master RKUNPF** must contain the customer record before IT110R can update balances or before IT111R can perform UPDATE rather than INSERT.
- **RLOBST (batch log)** keyed on `RLBFIR + RLBBIL + RLBSLT + RLBOMK` must contain a row for the given voucher number (`w_binr`) before IT123R can update receipt status. RLBSLT and RLBOMK must both be blank (primary, unallocated records).
- **RLOHST (batch header)** must have a row keyed on `RLHFIR + RLHTS1` (the batch timestamp) for IT123R to update header summaries.
- **Success log IUPDST** and **error log IERRST** must be writable. IT100R writes to these after each inbound message is processed.
- **Reference tables** RA03PF, RA07PF, RA08PF, RA09PF, RA12PF, and RA18PF must be populated with firm data for outbound replication programs (IT011R–IT014R, IT010R, IT120R) to have rows to send.

---

## 2. Validation Rules

### IT100R — Inbound queue dispatcher

| Condition | Effect |
|-----------|--------|
| `IFNAST.IFPROG` does not match any of the handled program names (NexstepOppdaterKundebalanse, NexstepOppdaterKunde, NexstepOppdaterLevbetingelse, …) | No subprogram is called; the record is deleted silently (treated as ignored) |
| After a subprogram call: `w_stat = ' '` (blank) | IT100R writes the record to IERRST (error log) before deleting from IFNAST |
| After a subprogram call: `w_stat = '9'` | IT100R deletes from IFNAST silently (acknowledged but not logged) |

### IT110R — Customer balance updater (NAV → NX)

| Condition | Effect |
|-----------|--------|
| Customer number (`RKUNPF.RKKUND` keyed on `l_firm + rkunlu_kund`) not found in RKUNPF | Balance update is **blocked**; `p_stat` remains blank (`' '`); IT100R writes to IERRST |
| Balance string contains scientific notation (`E-` substring, e.g. `1.2345E-12`) | Record is skipped entirely (`goto end_oppd`) — cannot be converted to decimal |
| Field length of parsed customer number exceeds 6 digits | Record is skipped (`goto end_oppd`) |
| PSSR error handler fires | `p_stat` set to `' '`; `*inlr = *on`; IT100R treats as error |

### IT111R — Customer master updater (NAV → NX)

| Condition | Effect |
|-----------|--------|
| Command = 'DELETE': customer not found in RKUNPF | Delete silently skipped; `p_stat='1'` still returned (idempotent) |
| `RKUNPF.RKSPFA <> 0` | RKAVDE (department) is **not overwritten** by NAV data on UPDATE — department is protected when a split-billing override exists |
| PSSR error handler fires | `p_stat = ' '`; treated as error by IT100R |

### IT120R — Delivery terms updater (NAV → NX)

| Condition | Effect |
|-----------|--------|
| Command = 'INSERT' and record already exists in RA18PF | Write is **blocked** — duplicate prevention; no update performed; `p_stat='1'` |
| Command = 'UPDATE' and record not found in RA18PF | Update is **blocked** — only existing records can be updated; `p_stat='1'` |
| Command = 'DELETE' and record not found in RA18PF | Delete is silently skipped; `p_stat='1'` |

### IT123R — Return receipt updater on batch log (NAV → NX, NXKORR)

| Condition | Effect |
|-----------|--------|
| `p_inlr = '1'` | Program sets `*inlr = *on` and exits immediately (LR-close pattern for performance) |
| `w_binr = 0` after parsing the parameter string (voucher number could not be extracted) | Execution aborted; `p_stat = '9'` (silent ignore) |
| RLOBST row not found (`sqlcod = 100`) for given `p_firm + w_binr + RLBSLT=' ' + RLBOMK=' '` | `b_feil = *on`; execution aborted; `p_stat = '9'` |
| RLOHST row not found after RLOBST lookup | `b_feil = *on`; execution aborted; `p_stat = '9'` |
| Status `w_stat` = 'OVERF' (transfer): SQL UPDATE rowcount = 0 | `p_stat = '9'` |
| Status `w_stat` = 'KLADD' (draft): SQL UPDATE rowcount = 0 | `p_stat = '9'` |
| Status `w_stat` = 'JOURN' (journalised): SQL UPDATE rowcount = 0 | `p_stat = '9'` |
| Status `w_stat` = 'FEIL ' (error): records written with zero amounts and error code/text | IT100R writes to IUPDST (success path) |
| Status `w_stat` does not match any of OVERF / KLADD / JOURN / FEIL | No update performed; `p_stat` stays at initial value |
| PSSR error handler fires | `p_stat = '5'`; `*inlr = *on` |

---

## 3. Configuration and Authorization Rules

- **LDA `l_user`** (pos 911–920): Written to `RKEUSR`, `IUPDST`, `IERRST`, and `RLOBST` tracking fields. Without a valid user in LDA, audit fields are populated with blanks.
- **Outbound programs IT010R–IT014R**: These programs replicate reference tables unconditionally. There is no user-level authorization check within the RPG logic; access control is via OS/400 object authority on the calling menu or submitted job.
- **IT100R closing optimization**: At the end of a batch run, IT100R calls IT110R and IT123R with `p_inlr='1'` to close their open cursors (LR-mode close). If this call is not made, DB2 cursors remain open, preventing subsequent runs from positioning correctly.

---

## 4. Financial / Transactional Rules

- **IT110R balance update**: When a customer is found, `RKUNPF.RKSALD` is overwritten unconditionally with the incoming balance. The incoming balance string is parsed as: integer part before `.`, decimal part after `.`; negative sign handling: if the balance starts with `-`, it is negated. Fields updated: `RKSALD`, `RKEDAT`, `RKETIM`, `RKESIG`, `RK2DPT`.
- **IT110R credit block**: If the incoming sperrekode field = '1' and `RKUNPF.RKKRES <> 2`, then `RKKRES` is set to `2` (credit block from NAV). This blocks the customer from receiving new orders in the NX system.
- **IT123R JOURN status with SHOP type**: When `RLOHST.RLHTYP = 'SHOP'`, the jounal amounts in `RLOBST.RLBSD4` / `RLBSK4` are **accumulated** (additive `+= w_sumd`). For all other batch types, the amounts are overwritten (non-additive `= w_sumd`). This allows multiple SAP/NAV journal postings against a single shop batch.
- **IT123R batch header summation**: After updating the detail log row (RLOBST), the program always recalculates the batch totals by summing all RLOBST rows for the same batch timestamp, then updates RLOHST with the aggregated amounts.
- **IT123R FEIL status**: Sets `RLOBST.RLBSD4 = 0`, `RLBSK4 = 0`, `RLBAN4 = 0`, writes error code (`RLBFF4`) and error text (`RLBFT4`). Also updates RLOHST header with zero amounts and error user.

---

## 5. Status and Lifecycle Rules

- **Outbound queue (ITNAST)**: IT001R writes new rows unconditionally. The queue is consumed by an external NAV process.
- **Inbound queue (IFNAST)**: IT100R reads rows sequentially for the current firm. After processing each row:
  - `p_stat = '1'` (success) → write to IUPDST and delete from IFNAST
  - `p_stat = ' '` (error) → write to IERRST and delete from IFNAST
  - `p_stat = '9'` (ignored) → delete from IFNAST without logging
- **IT111R INSERT/UPDATE logic**: RKUNPF is always chained first. If not found, a new record is written (INSERT path) with zero financial balances. If found, selective field updates are applied (fields are only overwritten when the incoming value is non-zero/non-blank).
- **IT120R delivery terms**: Only the RA18PF table (delivery terms reference) is managed. All three DML commands (INSERT / UPDATE / DELETE) are supported. The success indicator `p_stat = '1'` is returned in all handled cases.
- **IT123R receipt status transitions**: Valid status values received from NAV are: `OVERF` (transferred to NAV), `KLADD` (draft in NAV), `JOURN` (journalised in NAV), `FEIL ` (error in NAV). The timestamp `w_tims` (converted from ISO/UTC format) is written to the corresponding timestamp column (`RLBTS2` for OVERF, `RLBTS3` for KLADD, `RLBTS4` for JOURN/FEIL).

---

## 6. Special Conditions

- **IT123R voucher number with dash**: Voucher numbers containing a dash (e.g. `12345-0`) are handled by extracting the numeric portion before the dash. Pure numeric strings up to 8 digits are accepted directly.
- **IT123R parameter timestamp parsing**: The incoming parameter string uses a mixed ISO/UTC format. Characters `T`, `Z`, spaces, and colons are replaced to build a DB2 timestamp. If parsing fails, the program will trap in PSSR and return `p_stat='5'`.
- **IT110R E-notation guard**: Balances returned from Visma Business with values like `1.2345E-12` (from floating-point rounding) are detected by scanning for the substring `E-` and skipped entirely rather than causing a decimal conversion error.
- **IT111R RKSPFA protection**: The field `RKKSPFA` (split-billing flag) acts as a protection gate for `RKAVDE` (department code). If the customer has a non-zero split-billing designation, NAV cannot overwrite the department code. This prevents NAV from disrupting revenue-splitting configurations.
- **LR-close performance pattern**: IT110R and IT123R both check `p_inlr = '1'` at entry. When IT100R sends this signal at the end of a batch, the programs execute `*inlr = *on` and return, forcing all open DB2 cursors and file handles to close. This is required for correct operation on subsequent batch runs in the same job.

---

## 7. Subprogram Calls

| Caller | Called Program | Purpose |
|--------|---------------|---------|
| IT100R | IT110R | Update customer balance from NAV inbound queue |
| IT100R | IT111R | Update customer master data from NAV |
| IT100R | IT120R | Update delivery terms from NAV |
| IT100R | IT123R | Update batch return receipt status from NAV |
| IT010R | IT001R | Write country codes to NAV outbound queue (ITNAST) |
| IT011R | IT001R | Write payment terms to NAV outbound queue |
| IT012R | IT001R | Write departments to NAV outbound queue |
| IT013R | IT001R | Write districts to NAV outbound queue |
| IT014R | IT001R | Write salespeople to NAV outbound queue |

---

## 8. Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| ITNAST | ITFIRM | Outbound queue: NX → NAV commands |
| IFNAST | IFFIRM | Inbound queue: NAV → NX commands |
| IUPDST | — | Success audit log for processed inbound messages |
| IERRST | — | Error audit log for failed inbound messages |
| RKUNPF | RKFIRM + RKKUND | Customer master — balance and profile |
| RA18PF | Firm + delivery terms code | Delivery terms reference table |
| RA03PF | Firm | Payment terms reference |
| RA07PF | Firm | Department reference |
| RA08PF | Firm | District reference |
| RA09PF | Firm | Salesperson reference |
| RA12PF | Firm | Country codes reference |
| RLOBST | RLBFIR + RLBBIL + RLBSLT + RLBOMK | Batch detail log (receipt status) |
| RLOHST | RLHFIR + RLHTS1 | Batch header log (aggregated totals) |
