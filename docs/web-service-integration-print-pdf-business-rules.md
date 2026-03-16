# Web Service Integration / Print PDF — Business Rules

**Module:** 29 (AP prefix)
**Focus:** What blocks or prevents web service calls, transport integration, and PDF print parameter resolution

---

## 1. Prerequisites / Master Data

Before web service integration and PDF print operations can proceed, the following conditions must be satisfied:

- **Company number in LDA** (`l_firm` at pos 944–946) must be set. All AP programs read this value for firm-scoped operations. `l_figr` (pos 934–939) and `l_filg` (pos 937–939) additionally identify the user group and file group for IFS path construction and parameter lookup.
- **AFPSPF (system parameters / properties file)** keyed on `(AFUFIL, AFUIDE)` must contain entries for all required configuration keys. AP050R requires `cldPMUrl` (Profile Manager URL) and `CLOUD_API_KEY`. AP060R requires `WSSOKUSR`, `WSSOKPASS`, `cldD365Url`, `D365_reque` (request template name), `D365_KliId`, `D365_KliSc`, `D365_TenId`, `D365_LoUrl`. AP055R / AP056R / AP058R require the Lognet/Solwr transport service URL and API key entries. If any of these SQL SELECT statements returns no row, the corresponding variable is left blank, and the web service call will fail with `p_stat = 'FALSE'` or `p_stat = '8'`.
- **APOSEXTNST (company settings table)** must contain a row with `APOSFIRM = p_firm`, `APOSSYST = 'EGAPPS'`, `APOSCKEY = 'CLOUD_API_KEY'` for AP050R to retrieve the Profile Manager API key. Without this row, the token is blank and the Profile Manager call will be rejected by the remote API.
- **IFS directory** at `/home/<l_figr>/pmanager/` (AP050R), `/home/<l_figr>/D365/` (AP060R), or the Lognet/Solwr transport path (AP055R/AP056R/AP058R) must exist and be writable by the IBM i job user. If `chdir()` to the path returns a negative value, the request file cannot be created and the program aborts with `goto avslutt`.
- **AW702C (HTTP client CL program)** must be available in the library list. All AP web service programs call `AW702C` to execute the actual HTTP GET or POST. If AW702C is absent or its parameters change, the web service call cannot proceed.
- **AFORPF (print routine register)** must contain a row for the routine name passed to AP600R or AP605R. Without this row, no printer parameters can be resolved and the print job will proceed with system defaults.
- **AFPSPF `XMLPATH` entry** (keyed on `l_filg` + `'XMLPATH'`) must exist for AP607R / AP609R to resolve the IFS output path for XML/PDF files. Without this entry, the PDF output path is blank and archiving will fail silently.

---

## 2. Validation Rules

### AP050R — Profile Manager web service client

| Condition | Effect |
|-----------|--------|
| `chdir()` to the Profile Manager IFS path returns < 0 | Request file cannot be created; `goto avslutt` — **web service call is skipped** |
| `open()` on request file returns fd < 0 | `goto avslutt` — program exits without calling AW702C |
| `open()` on response file returns fd < 0 | `API_status = '9'` — AP051R is not called; no data written |
| PSSR error handler fires | `p_stat = '8'` — error indicator returned to caller |

### AP051R — Profile Manager response parser

| Condition | Effect |
|-----------|--------|
| JSON data-into `count = 0` (no records returned in JSON array) | `p_api_status = '7'` — no records processed; no PROMST writes |
| Profile `data.state <> 'PUBLISHED'` | Record is **skipped** — only published profiles are written to PROMST |
| Profile found in PROMST (CHAIN hits on firm + profile code) | New row is **not written** — existing PROMST record is preserved unchanged (insert-only logic) |
| Profile not found in PROMST | New row is **written** with `pmnoit` (number of articles), `pmpmtx` (profile name), and audit fields |

### AP055R / AP056R — Lognet transport service client

| Condition | Effect |
|-----------|--------|
| `chdir()` to Lognet IFS path returns < 0 | Request file creation fails; `goto avslutt` |
| `open()` on request file returns fd < 0 | `goto avslutt` |
| FOHEPF (order header) not found for the given order + suffix | Order data cannot be assembled; transport request is not sent |
| FLOGPF (order log) not found for the year + order + suffix | Log-based status check fails; readyForDispatch logic may default |
| v7.01: If order is fully picked (`FLOGPF` log code >= threshold) | `readyForDispatch` is set to `true` in the JSON request |
| Switch `VOT1PF` internal order flag is set for the warehouse | Internal orders are **not transmitted** to Lognet (v8.03) |
| RDS supplier linked to confirmed purchase order (`RA37PF` check) | Transport request is **not sent** (v8.21) |
| FTRPST (transport registration) not found after transport call | No transport ID is recorded |
| Response file `open()` returns fd < 0 | `p_stat` set to error indicator; no response processed |
| v8.26: Response file `open()` returns fd < 0 (file-arrival check) | `p_stat` set to indicate missing response — v8.26 explicit check |

### AP057R — Lognet transport cancellation client

| Condition | Effect |
|-----------|--------|
| Old shipment method has transport integration but new one does not | Cancellation (delete flag) is sent to Lognet for the old transport record |
| Order is of negative order type (credit note) | JSON is built with inverted amounts (v8.08) |

### AP058R — Solwr transport service client

| Condition | Effect |
|-----------|--------|
| All prerequisite URL/credential fields blank (from AFPSPF) | `p_stat = '5'` (no setup) — transport not sent |
| FOHEPF not found for order | Transport request cannot be assembled — `goto avslutt` |
| `chdir()` to Solwr IFS path returns < 0 | Request file cannot be created — `goto avslutt` |
| `open()` on request file returns fd < 0 | `goto avslutt` |
| v8.26: Response file not received (`open()` fd < 0 after AW702C call) | `p_stat` set to error; caller handles non-arrival |
| RKUNPF customer not found | Customer contact data defaults to blank; request proceeds without contact info |
| Invalid characters in item text (v8.36) | Text is scanned and sanitised before JSON serialisation to prevent parse failures on the remote API |
| v8.39: Parcel line (`JVPKPF`) contains non-numeric value in quantity field | Numeric conversion is skipped; quantity defaults to zero |

### AP060R — D365 customer ledger entries web service client

| Condition | Effect |
|-----------|--------|
| Any of `w_mal`, `w_lurl`, `w_url`, `w_tenid`, `w_kliid`, `w_klisc` is blank | `p_stat = 'FALSE'` — configuration incomplete; program exits before any HTTP call |
| `chdir()` to D365 IFS path fails | Login request file cannot be created — `goto avslutt` |
| `open()` on login request file returns fd < 0 | `goto avslutt` |
| v8.01: AP060C (CCSID converter) returns blank (`w_ifsf = *blanks`) | `p_stat = 'FALSE'` — response file could not be converted; no data parsed |
| `open()` on logon response file returns fd < 0 | `p_stat = 'FALSE'` — token cannot be extracted |
| v7.01: D365 transaction response length < 40 bytes | `p_stat = 'TOM  '` — response is too short to contain valid data |
| PSSR error handler fires | `p_stat = 'FALSE'` — caller handles as configuration failure |

### AP500R — Postal address lookup

| Condition | Effect |
|-----------|--------|
| No blocking validation defined — pure query screen | APOSPF is read sequentially; user selects a row to return `p_ponr` (postal code) and `p_sted` (town) |

### AP600R / AP605R — Print parameter resolver

| Condition | Effect |
|-----------|--------|
| AFORPF row not found for routine `p_ruti` | Printer parameters default to system standards from ASPSTD; no blocking |
| PDF mode: AFOPPF row not found for routine `p_ruti` | PDF parameters not resolved; `l_arch`, `l_skje`, `l_mail`, `l_exlp`, `l_land` remain at defaults |
| Output queue set to `*ARKIV` | Spool output is suppressed; document is archived only (v6.14 logic) |
| Excel or preview mode selected | Output queue is overridden to `DUMMY` or `XDUMMY` to suppress physical printing (v6.32/7.02) |
| v8.01: `b_papir` flag (no physical paper): | Selected output mode suppresses spooled print output entirely |
| `p_in03 = '1'` (F03/F12 pressed) | Caller is informed to abort the print job |

### AP607R / AP608R / AP609R — PDF parameter lookup

| Condition | Effect |
|-----------|--------|
| AP608R: AFPSPF row with `AFUIDE = 'STATUS'` not found | `p_pdf = '0'` — PDF mode considered inactive for this file group |
| AP608R: AFPSPF `STATUS` value is `'0'` | `p_pdf = '0'` — PDF system disabled globally |
| AP608R: AFOPPF row not found for routine | `p_pdf = '0'` — routine not configured for PDF |
| AP608R: AFOPPF `AFPPDF = 0` | `p_pdf = '0'` — PDF mode explicitly disabled on the routine |
| AP608R: `AFPPDF = 2` (department-level control) and AFAPPF row not found for firm + dept | `p_pdf = '0'` — no department-level PDF override found |
| AP608R: `AFPPDF = 2` and AFAPPF.AFAPDF = 0 | `p_pdf = '0'` — PDF disabled at department level |
| AP607R / AP609R: AFPSPF `XMLPATH` not found | XML output path is blank — PDF archiving will fail |
| AP607R: AFPSPF `FIFOFOLDER` not found | FIFO folder path is blank — FIFO-based document delivery is disabled |
| AP607R: AFPSPF `FAKTKOPI` not found | Invoice paper-copy flag defaults to `'0'` — no paper copy will be printed |
| AP607R: AFPSPF `DFTEMAIL` not found | Default email address is blank — email delivery will use blank recipient |

---

## 3. Configuration and Authorization Rules

- **LDA layout for print parameters**: AP600R writes resolved printer parameters into the LDA at fixed offsets: `l_utqm` (pos 585–594) output queue name, `l_bach` (pos 595–635) batch number, `l_arch` (pos 636) archive flag, `l_skje` (pos 637) screen preview flag, `l_land` (pos 638) landscape flag, `l_exlp` (pos 639) Excel export flag, `l_mail` (pos 640) mail flag. Downstream programs read these positions from the LDA to determine output mode.
- **AFPSPF keying**: All AP607R / AP608R / AP609R PDF parameter lookups use the composite key `(AFUFIL, AFUIDE)` where `AFUFIL` is the file group (`l_filg` from LDA, or `p_filg` parameter in v7.01+). The file group partitions settings by installation, allowing different customers sharing the same IBM i to have independent PDF configurations.
- **AW702C HTTP parameters**: All web service calls pass the following to AW702C: URL, request type (GET or POST), request file path (IFS), response file path (IFS), username, password, encoding (`ISO-8859-1` or `UTF-8`), and a status variable. AW702C is responsible for executing the curl or equivalent HTTP client command. No retries are implemented in the RPG programs.
- **IFS file cleanup**: AP050R and AP060R contain cleanup logic to delete request, response, and header files from the IFS after processing. In the production code, the cleanup code for some files is commented out (versioned as `9.xx` lines) to facilitate debugging. This means IFS files from web service calls may accumulate unless the commented-out `unlink()` calls are re-enabled.
- **AS100R sequence number counter**: AP050R and AP060R call `AS100R` with counter key `'NXTKLI'` to generate a unique file suffix for IFS request/response files. Without a valid counter, the files cannot be uniquely named, risking name collisions in concurrent job environments.

---

## 4. Financial / Transactional Rules

- **AP051R Profile Manager data**: Only profiles in `PUBLISHED` state are written to PROMST. Draft, archived, or otherwise non-published profiles are silently skipped. The insert is strictly append-only: if a profile code already exists in PROMST for the member (`promir_memb + promir_pkey`), no update is performed. Fields written: `PMMEMB` (member), `PMPKEY` (profile code), `PMNOIT` (number of affected articles), `PMPMTX` (profile name), and audit timestamp/user.
- **AP060R D365 token lifecycle**: AP060R first posts a login request to the D365 OAuth2 token endpoint using `client_credentials` grant type with `client_id`, `client_secret`, and `resource` from AFPSPF. The returned `access_token` is extracted by scanning for the literal `access_token` in the response body, then used as a Bearer token for the subsequent transaction request. No token caching is implemented; a new token is obtained on every call.
- **AP055R / AP056R transport readyForDispatch logic**: The `readyForDispatch` field in the Lognet JSON is set to `true` when the order log (`FLOGPF`) shows that all items have been picked (log code >= threshold, v7.01). If the order is linked to a confirmed purchase order from an RDS supplier (v8.21 check via RA36PF/RA37PF), the entire transport request is suppressed — no JSON is submitted to Lognet.
- **AP058R transport order type inversion**: For order types classified as negative (credit note or return) based on `VOTYPF.VAOTYP` accumulator code 2 (v8.28), the JSON request is built with inverted quantity/amount fields to correctly represent a return shipment to the transport provider.

---

## 5. Status and Lifecycle Rules

- **AP050R / AP051R PROMST (profile master)**: PROMST is an append-only table from the perspective of the AP050R/AP051R programs. Records are inserted but never updated. If a profile changes in the remote Profile Manager system, the change is not propagated to PROMST on subsequent runs because the CHAIN check prevents overwriting existing rows.
- **AP058R FTRPST (transport registration)**: After a successful transport request, AP058R updates FTRPST with the transport order ID (`FTROID`) received from Solwr. The FTRPST table tracks all transport registrations. If FTRPST already has a row for the order (from a previous call), it is updated (not inserted) to reflect the new transport ID.
- **AP060R response lifecycle**: The D365 logon response is parsed to extract the Bearer token, which is then used for a second HTTP call. Both response files remain on the IFS after processing (cleanup is commented out for debugging). Callers receive `p_path` and `p_resp` pointing to the last D365 transaction response file on the IFS, enabling the caller to parse the response independently.
- **AP600R number counter (ANUMPF)**: AP600R calls AS100R via `w_fell = 'ANUMPF'` to generate a batch run number for PDF archiving (`l_bach` LDA field). This counter is stored in ANUMPF. If the counter record is missing for the firm, a zero batch number will be used, potentially causing archiving conflicts.

---

## 6. Special Conditions

- **AP056R vs AP058R**: Both programs send transport orders but target different carriers: AP055R/AP056R targets Lognet; AP058R targets Solwr. The programs are structurally similar but differ in endpoint URLs, JSON structure, and IFS directory paths. The calling program (typically a CL wrapper) determines which transport program to invoke based on the order's configured shipment method.
- **AP058R invalid character scanning (v8.36)**: Before building the JSON request, AP058R scans item description text fields for characters that are invalid in JSON (control characters, unescaped double quotes, etc.) and replaces them with spaces. This prevents JSON parse failures on the Solwr API side.
- **AP060R response length guard (v7.01)**: After receiving the D365 transaction response, AP060R reads the first 1024 bytes and checks that the returned length is at least 40 bytes (`rdlen < 40 → p_stat = 'TOM  '`). This guards against empty or minimal responses (e.g. `{"value":[]}`) that would cause downstream JSON parsing to produce zero records without a meaningful error.
- **AP608R three-level PDF activation**: The PDF mode check in AP608R operates on three levels: (1) global file-group activation via `AFPSPF STATUS`, (2) routine-level activation via `AFOPPF AFPPDF`, and (3) optional department-level override via `AFAPPF`. All three must permit PDF before `p_pdf = '1'` is returned. Any level setting PDF to 0 disables PDF for that call, regardless of higher-level settings.
- **AP607R p_filg parameter (v7.01)**: In v7.01, AP607R was changed to accept `p_filg` as an explicit input parameter instead of reading `l_filg` directly from the LDA. This allows callers to override the file group for parameter lookup, enabling multi-tenant scenarios where a single job processes documents for multiple file groups.

---

## 7. Subprogram Calls

| Caller | Called Program | Purpose |
|--------|---------------|---------|
| AP050R | AS100R | Generate unique IFS file sequence number |
| AP050R | AW702C | Execute HTTP GET to Profile Manager API |
| AP050R | AP051R | Parse Profile Manager JSON response |
| AP056R / AP058R | AW702C | Execute HTTP POST to transport service API |
| AP056R | AP060C (v8.25) | Convert CCSID of transport response file |
| AP058R | AP060C | Convert CCSID of Solwr response file |
| AP060R | AS100R | Generate unique IFS file sequence number |
| AP060R | AW702C | Execute HTTP POST to D365 OAuth2 token endpoint |
| AP060R | AW702C | Execute HTTP POST to D365 transaction service |
| AP060R | AP060C | Convert CCSID of D365 response file |
| AP600R | (ANUMPF via AS100R) | Generate batch number for PDF archiving |
| AP607R / AP609R | (none — direct file reads) | Resolve PDF parameters from AFOPPF, AFAPPF, AFPSPF |
| AP608R | (none — direct file reads) | Check PDF activation status from AFPSPF + AFOPPF + AFAPPF |

---

## 8. Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| AFPSPF | AFUFIL + AFUIDE | System properties / configuration key-value store |
| APOSEXTNST | APOSFIRM + APOSSYST + APOSCKEY | Company settings: API keys, integration settings |
| PROMST | PMMEMB + PMPKEY | Profile Manager profile cache |
| AFORPF | Firm + routine code | Print routine register (printer assignments) |
| AFOPPF | Routine code | PDF print options per routine |
| AFAPPF | Routine + firm + department | Departmental overrides for PDF options |
| AUSRPF | Firm + user | User settings (printer, output queue) |
| AWSDPF | Workstation ID | Workstation-specific printer settings |
| FOHEPF | Firm + order number + suffix | Order header (for transport requests) |
| FODTPF | Firm + order + suffix + line | Order lines (for transport item data) |
| FLOGPF | Year + order number + suffix | Order log (pick status for readyForDispatch) |
| FTRPST | Firm + transport route ID | Transport registration (outbound transport IDs) |
| FTRLST | Firm + transport + line | Transport line details |
| RKUNPF | RKFIRM + RKKUND | Customer master (contact info for transport) |
| ANUMPF | Firm + series code | Number series counters (batch number for PDF archiving) |
| VOT1PF | Firm + order type | Order type parameters (internal order switch) |
| VOTYPF | Firm + order type | Order type definitions (accumulator code for negative types) |
| RA36PF / RA37PF | Firm + supplier code | RDS/EAN supplier reference tables |
