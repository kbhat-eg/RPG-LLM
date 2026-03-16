# External Balance / Account Update — Business Rules

**Module:** 32 (RE / RX prefix)
**Focus:** What blocks or prevents external balance and account data from being imported into the NX system

---

## 1. Prerequisites / Master Data

Before external balance or account updates can be applied, the following conditions must be satisfied:

- **Company number in LDA** (`l_firm` at position 944–946) must match the firm number on the incoming data record. RE110R (account updater) checks `RHOVPF.RHFIRM <> l_firm`; RE110R (customer updater) checks `RKUNPF.RKFIRM <> l_firm`. A mismatch silently skips the record.
- **Chart-of-accounts range from RAA1PF**: RE109R/RE110R (account) load `RA1HKT` (highest account number) from RAA1PF on initialisation by chaining on `l_firm`. The account number `RHOVPF.RHKONT` must not exceed `RA1HKT` or the record is skipped.
- **Customer number range from RAA1PF**: RE110R (customer) loads `RA1HKT` (lower bound for customer numbers) and `RA1HRS` (upper bound) from RAA1PF. Incoming customer numbers `RKUNPF.RKKUND` must satisfy `rkkund > ra1hkt AND rkkund <= ra1hrs`; any customer number outside this range is silently skipped.
- **Payment terms default from RAA1PF**: `RA1BEK` (default payment terms) is used when the incoming `RKBETB` field is zero — both for new customer creation and for updates where payment terms are zero.
- **RHOVPF (account master) and RKUNPF (customer master)** must be accessible as update-eligible files for RE110R to write or update records.
- **VEINPF** (Visma Business balance input file) must be populated and accessible as a sequential read file for RX001R. If the file is empty, no updates occur.
- **RSCDPF** (job configuration) keyed on firm must exist for RX000R (FTP path lookup) and RX002R (repeat-run scheduling). Without RSCDPF row for the firm, path and job information are not returned.
- **AFTPPF** (FTP parameter) must have a row with `AFNAVN = RSCDPF.RCFTPA` and `AFLOCD <> ' '` for RX000R to resolve the local IFS path. If the FTP parameter is absent or AFLOCD is blank, the program sends an error message via AA005R.

---

## 2. Validation Rules

### RE109R / RE110R — Account updater from external system

| Condition | Effect |
|-----------|--------|
| `RHOVPF.RHFIRM <> l_firm` | Record is **skipped** (`goto slutt`) — firm mismatch |
| `RHOVPF.RHKONT > RA1PF.RA1HKT` | Record is **skipped** — account number exceeds chart-of-accounts upper limit |
| Account found in RHOVPF (CHAIN hits): all selective fields | Only non-blank/non-zero incoming values overwrite existing data (conditional updates per field) |
| Account not found in RHOVPF | New account row is **written** (INSERT) with audit fields set from LDA timestamp and user |

### RE110R — Customer updater from external system

| Condition | Effect |
|-----------|--------|
| `RKUNPF.RKFIRM <> l_firm` | Record is **skipped** (`goto slutt`) |
| `RKKUND <= RA1HKT OR RKKUND > RA1HRS` | Record is **skipped** — customer number outside valid customer range |
| `RKALFA = *blank` | Alpha search field is auto-populated from `RKNAVN` (customer name) and uppercased |
| Customer found in RKUNPF: `RKBETB = 0` | Payment terms are set to `RA1BEK` (default) |
| Customer not found in RKUNPF: new record | Financial fields initialised to zero (`RKSALD`, `RKKJOP`, `RKKTID`, `RKKJOF`, `RKGRAL`); `RKBETB` set to `RA1BEK` if incoming value is zero |
| New customer (not found): memo balance (RKMEPF) not found | New memo balance row is **created** with zero amounts |

### RE600R — Balance export parameter screen

| Condition | Effect |
|-----------|--------|
| Run date `A2PKDA` fails TEST(D) | **Blocked** with indicator *in33 — date is invalid |
| Accounting year `A2PRAR > *year` | **Blocked** with indicator *in32 — future year not allowed |
| Period `A2PPER <= 0` | **Blocked** with indicator *in35 — period must be positive |
| Period `A2PPER > 13` | **Blocked** with indicator *in35 — period 14+ not valid |
| Department validation: `A2PAVF = A2PAVT` (single department) and RS707R returns `p_stat='N'` | **Blocked** with indicator *in33 — department does not exist |
| Summarisation code `A2PSPL` not in ('N','J') | **Blocked** with indicator *in34 |
| Balance code `A2PSAL` not in ('N','J') | **Blocked** with indicator *in36 |

### RE120R — External posting parameter screen

| Condition | Effect |
|-----------|--------|
| Run date `A2PKDA` fails TEST(D) | **Blocked** with indicator *in33 |
| Accounting year `A2PRAR > *year` | **Blocked** with indicator *in32 |
| Period `A2PPER <= 0` | **Blocked** with indicator *in35 |
| Period `A2PPER > 13` | **Blocked** with indicator *in35 |

### RX001R — Customer balance updater from Visma Business

| Condition | Effect |
|-----------|--------|
| Record first character is not a digit (`< '0' OR > '9'`) | Record is **skipped** — header/trailer records are not processed |
| Parsed customer number field length `> 6` characters | Record is **skipped** — invalid customer number format |
| Balance value string contains `E-` (scientific notation) | Record is **skipped** — unprocessable floating-point value |
| Customer not found in RKUNPF | Balance update is **blocked** — no write occurs; record silently skipped |
| Incoming sperrekode (block code) = '1' AND `RKKRES <> 2` | `RKKRES` is set to `2` (credit block activated) |

### RX000R — FTP path resolver

| Condition | Effect |
|-----------|--------|
| RSCDPF row not found for firm | `p_bane`, `p_file`, `p_job` are returned blank — caller cannot proceed |
| AFTPPF row not found OR `AFLOCD = ' '` | AA005R is called with message "Lokal bane mangler i FTP-Prameter" — job aborts |

### RX002R — Repeat-run scheduler

| Condition | Effect |
|-----------|--------|
| RSCDPF row not found OR `RCREPP = ' '` | `p_min` is returned blank — no repeat scheduled |
| Current time + `RCREPP` minutes would exceed `RCSLUT` (end time) | `p_min` is set to blank — repeat scheduling is **suppressed** |

### RX732R — VAT base correction

| Condition | Effect |
|-----------|--------|
| `RHTRPF.RBRAAR < 2004` | Record is **skipped** — only transactions from 2004 onwards are eligible for VAT base correction |
| `RBNETO >= 0` or `RBMVAB >= 0` or `RBMVAG >= 0` for negative correction | Record is **skipped** — conditions must all match for sign-flip logic |
| `RBNETO <= 0` or `RBMVAB <= 0` or `RBMVAG >= 0` for positive correction | Record is **skipped** |

### RX733R — Factoring date reset

RX733R resets `RKTRPF.RNFADA` (factoring date) to `*loval` for all rows of the current firm. There are no blocking conditions — the operation runs unconditionally on all firm rows.

---

## 3. Configuration and Authorization Rules

- **LDA `l_user`** (pos 911–920): Used to populate audit fields `RHEUSR`, `RHOUSR`, `RKESIG`, `RKNSIG`, `RKEPGM`, `RK2DPT`. Without a valid user, tracking fields are blank.
- **RAA1PF system parameters** (`RA1HKT`, `RA1HRS`, `RA1BEK`): These are loaded once during program initialisation and govern account and customer number boundaries. If RAA1PF is not found for the firm (indicator 60 on), the program continues with zero limits, effectively passing all records.
- **RE600R system mode**: Initialisation parameter `w_syst` controls which screen indicators are set (PC, FINA, TOTA, MAES, MAX, AKEL). This determines which external system's layout is displayed but does not block processing.
- **AFTPPF local path**: Used by RX000R. The table `AFTPPF` is keyed on `(firm, AFNAVN)` where AFNAVN comes from `RSCDPF.RCFTPA`. The returned `AFLOCD` is the IFS directory where incoming balance files are placed by FTP.

---

## 4. Financial / Transactional Rules

- **RE110R account update**: `RHOVPF.RHTEXT` (account name) and `RHTXT2` are always overwritten from incoming data. All other fields (RHSPRK, RHMVAK, RHBNOK, RHPROJ, RHVALK, RHMVAO, RHMVAG, RHAVDK, RHMNGD, RHNOKK, RHALFA) are only overwritten when the incoming value is non-blank / non-zero.
- **RE110R customer update**: Name (`RKNAVN`), address (`RKGATE`), secondary address (`RKADR2`), postal code (`RKPONR`), town (`RKSTED`) are always overwritten. Over 35 additional fields are selectively overwritten (only when non-blank/non-zero). The alpha search field `RKALFA` is uppercased after update.
- **RX001R balance assignment**: The balance string is parsed as `integer_part + decimal_part/100`. The resulting value replaces `RKUNPF.RKSALD` unconditionally when the customer is found. There is no check for whether the balance has changed before updating.
- **RX732R VAT sign correction**: When net amount (`RBNETO`) is negative, VAT base (`RBMVAB`) is negative, but VAT code (`RBMVAG`) is positive, the sign of RBMVAG is flipped to negative. The reverse applies for the positive case. This corrects historical data where VAT base and VAT code amounts had inconsistent signs.

---

## 5. Status and Lifecycle Rules

- **RE110R account UPSERT**: A CHAIN to RHOVPF determines whether UPDATE or WRITE is performed. If found: UPDATE. If not found: WRITE (new row). Audit fields `RHEDAT`, `RHETIM`, `RHEUSR` are set on update. For new rows: `RHODAT`, `RHOTIM`, `RHOUSR`, and the edit fields are set.
- **RE110R customer UPSERT**: A CHAIN to RKUNPF determines UPDATE or WRITE. For new customers, after writing the RKUNPF row, RK750R is called to update the alpha search index. If the memo balance table (RKMEPF) row is also absent, a new RKMEPF row is created with zero amounts.
- **RX001R credit status**: The field `RKKRES` (credit status) is only escalated, never cleared, by RX001R. A credit block (set to 2) can be applied but the program does not remove credit blocks.
- **RX733R reset**: This is a bulk reset operation — no status checking is performed. All factoring dates for the firm are reset to `*loval` (low value = cleared). This is intended as a period-end maintenance operation.

---

## 6. Special Conditions

- **RE110R duplicate account prevention**: The CHAIN on RHOVPF uses key `(l_firm, RHKONT, RHSPES, RHAVDE)`. If the incoming record specifies a combination of account + specification + department that already exists, it is treated as an update. There is no explicit duplicate rejection — updates are always applied.
- **RE110R customer alpha field auto-fill**: If `RKALFA` is blank on incoming data and also blank on the existing RKUNPF record, it is copied from `RKNAVN` and uppercased. This ensures the alpha search register is always populated.
- **RX001R tab-separated input**: The balance file from Visma Business uses spaces (not tabs or semicolons) as separators. The program scans for `' '` and `;` to locate field positions. The code substitutes `;;;;` for a block of spaces to handle alignment issues in the numeric fields.
- **RX002R time window enforcement**: The repeat interval (`RCREPP` minutes) is added to the current time. If the result would exceed the configured stop time (`RCSLUT`), no repeat run is scheduled. This prevents the import job from running after close-of-business hours even if it finishes quickly.
- **RX732R year guard**: Only transactions with `RBRAAR >= 2004` are eligible for VAT sign correction. Earlier years are not corrected because the sign convention for VAT was changed in 2004 in the Norwegian accounting standards followed by this system.

---

## 7. Subprogram Calls

| Caller | Called Program | Purpose |
|--------|---------------|---------|
| RE110R (customer) | RK750R | Update customer alpha search index after INSERT |
| RE600R | RS707R | Validate that a given department code exists |
| RX000R | AA005R | Display error message when FTP local path is missing |

---

## 8. Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| RHOVPF | RHFIRM + RHKONT + RHSPES + RHAVDE | Account master (chart of accounts) |
| RKUNPF | RKFIRM + RKKUND | Customer master |
| RKMEPF | RKFIRM + RKKUND | Customer memo balance |
| RAA1PF | Firm | System parameters: account/customer number limits, default payment terms |
| RHTRPF | RBFIRM + RBRAAR + RBRPER | General ledger transactions (for VAT correction) |
| RKTRPF | RNFIRM + RNNUMM | Customer transaction register (factoring dates) |
| VEINPF | Sequential | Visma Business balance import file (customer balances) |
| RSCDPF | Firm | Job configuration: FTP parameter name, repeat interval, end time |
| AFTPPF | Firm + AFNAVN | FTP parameters: local IFS directory path, filename |
