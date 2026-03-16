# FTP Transfer — Business Rules

## Introduction

The FTP Transfer module (module prefix **AF**) manages automated file transfer jobs between the IBM i and external servers. It maintains a registry of FTP job definitions (`AFTPPF`), tracks execution logs (`AFTHPF`, `AFTLPF`), manages file group configurations (`AFGRST`), and provides supporting functions for SMS XML generation and directory monitoring. The module ensures that FTP jobs are not run concurrently, that servers are restricted to an approved whitelist, and that transfer parameters are fully configured before execution.

---

## Prerequisites and Master Data Requirements

| Requirement | Table | Key Fields | Used By |
|---|---|---|---|
| FTP job must exist in FTP job parameter file | `AFTPPF` | job name | AF700R, AF703R |
| File group must exist in AFGRST before FTP parameters can be built | `AFGRST` | firm + file group | AF700R, AF820R |
| Server name must be in the approved whitelist | `AFGRST.AGSERV` | — | AF120R, AF820R |
| SMS XML path must be configured in `AFPSPF` | `AFPSPF` | file group + `'SMS'` / `'XMLPATH'` / `'DATAQ'` | AF725R |
| Post code register entry must exist for AF010R maintenance | `APOSPFR` | firm + post code | AF010R |

---

## Validation Rules

### VR-01 — FTP Job Must Exist (AF700R, AF703R)

When building FTP parameters or setting job status:

**AF700R**:
```
chain to AFTPPF; if not %found → p_status = 'N'
```

**AF703R**:
```
chain to AFTPPF; if not %found → p_status = 'N'
```

If the job name does not exist in `AFTPPF`, the status parameter is set to `'N'` (not found) and the calling program must treat this as an error. No FTP commands are built and no lock is set.

### VR-02 — FTP Job Already Locked / Busy (AF703R)

```
if p_status = 'B' and afstat = 'B' → p_status = 'L'
```

`AF703R` is called with `p_status = 'B'` (request to mark as busy). If the job is already in status `'B'` in `AFTPPF`, the returned status is set to `'L'` (locked/busy conflict). The calling scheduler must interpret `'L'` as "another instance is already running" and abort the new execution.

*Effect*: Concurrent execution of the same FTP job is prevented. Only one instance of a given job name may run at a time.

### VR-03 — QY Password Authentication Failure Blocks FTP (AF700R)

If the FTP job uses QY password authentication (`afqyps = 'J'`):

1. `AF709R` is called to retrieve the password.
2. If `AF709R` returns a non-blank status (`w_status <> *blank`), then `p_status = 'A'` (authentication failure).

*Effect*: If password retrieval fails, the FTP parameter block is not built and the job cannot proceed.

### VR-04 — Server Name Must Be in Approved Whitelist (AF120R, AF820R)

**AF120R** (file group registry maintenance):
```
if server_name not in whitelist → *in32 = *on
```

The server name field (`AGSERV`) is validated against a hardcoded whitelist of approved server names. If the entered server name is not in the list, `*in32` is set and the record cannot be saved.

**AF820R** (CSV update of AFGRST):
Same whitelist check applies when importing file group definitions from CSV.

Approved server names (as of current source):
`MGRTST01`, `EGUTVK01`, `EGTEST`, `XLBYGG`, `OES`, `STASKOV`, `BYGGMAKK`, `BYGGNIB`, `OPTIMERA`, `MESTERGR`, `ASDRIFT`, `MAXBO`

*Effect*: FTP jobs can only be configured to connect to pre-approved servers. Adding a new server requires a code change to update the whitelist.

### VR-05 — File Group Field Required (AF120R)

```
if file_group = *blank → *in31 = *on
```

The file group field (`AGFILG`) must not be blank. A file group record without a group code is rejected.

### VR-06 — SMS XML Paths Must Be Configured (AF725R)

```
if XML_path = *blanks or OUT_path = *blanks → goto avslutt
```

Before generating an SMS XML file, both the XML output path and the data-output path must be configured in `AFPSPF`. If either path is blank, the program exits immediately without generating any output.

The paths are looked up from `AFPSPF` using the keys:
- `file_group + 'SMS'` — base configuration
- `file_group + 'XMLPATH'` — XML output path
- `file_group + 'DATAQ'` — data queue name

If any of these lookup keys return blank values, `XML_path` or `OUT_path` will be blank and the program exits.

### VR-07 — Firm Mismatch on Log Screens (AF110R, AF111R)

FTP log header and log line views check the session firm against the record firm:

- **AF110R**: `if adhfir <> w_firm → skip record`
- **AF111R**: `if adlfir <> w_firm → skip record`

Records from other firms are not displayed.

---

## Configuration and Authorization Rules

### CA-01 — Delete Log Header Cascades to Log Lines (AF110R)

Deleting a log header record from `AFTHPF` triggers cascade deletion of all associated log lines in `AFTLPF` for the same log key. There is no confirmation beyond the standard screen delete confirmation.

### CA-02 — Active File Group Count Threshold (AF820R)

When importing file group definitions from CSV (`LWEXPF`):

```
if w_anta < 50 → afgakt = 1 (active)
else            → afgakt = 0 (inactive)
```

File groups with fewer than 50 associated records are marked as active (`AFGAKT = 1`); those with 50 or more are marked inactive. This threshold is hardcoded.

### CA-03 — Environment Field from CSV Column 3 (AF820R, v7.01)

From version 7.01, the environment field (`AGMILJ`) in `AFGRST` is populated from column 3 of the CSV file (`LWEXPF`). If the CSV file has fewer than 3 columns, the environment field will remain blank.

### CA-04 — FTP Command Construction Rules (AF700R)

The FTP parameter block (`AFPAPF`) is built with the following command logic:
- `namefmt 1` — always set
- `sendpasv` — passive mode
- `binary` — binary transfer mode
- `lcd` / `cd` — local and remote directory change commands
- `GET`, `PUT`, `MPUT`, `MGET` — depending on job transfer direction

If `afslet = 'J'` (delete after download), a `DEL` command is appended after each GET.
If `aflibt = 'J'` (library save/restore), library backup/restore commands are prepended and appended.

Counter fields in `AFTPPF` are incremented each time the parameter block is built.

---

## Financial and Transactional Rules

There are no financial or monetary calculations in this module. The module is a pure infrastructure/transport layer.

---

## Status and Lifecycle Rules

### SL-01 — FTP Job Status Values (AF703R)

| Status value | Meaning |
|---|---|
| `'B'` | Busy (job is running) |
| `'L'` | Locked (duplicate execution attempted) |
| `'N'` | Not found (job does not exist in AFTPPF) |
| `'A'` | Authentication failure (QY password error) |
| `' '` (blank) | Available / idle |

### SL-02 — Log Writing Always Succeeds (AF720R)

`AF720R` reads the FTP output spool file (`AFLOPF`) and writes all lines to `AFTHPF` (log header) and `AFTLPF` (log lines) unconditionally. There are no blocking rules in the log writer. Log records are always written regardless of transfer success or failure.

### SL-03 — Directory Monitor Alerts on Stale Files (AF730R)

`AF730R` uses the stored procedure `SP_ALLE_AKTIVE_FILGRP` to get all active file groups. For each group it checks:
- If the group is in BUSY status.
- If the BYGGDOK last-processed date is stale.
- If COBUILDER2 file age exceeds the configured threshold.

If any condition is met, an email alert is sent via `AF732C`. The monitor does not block operations; it is a notification-only process.

---

## Special Conditions

### SC-01 — Horizontal Scroll in Log Line View (AF111R)

The log line detail screen (`AF111R`) supports horizontal scrolling controlled by `w_hors`. Different scroll positions show different columns of the log data. This is a display behaviour, not a blocking rule.

### SC-02 — FTP Job Selection Popup Returns One Job (AF500R)

The FTP job selection popup reads `AFTPPF` and returns the selected job name to the calling program. Only one job can be selected per popup invocation. The popup has no firm filter because `AFTPPF` does not have a firm field; all jobs for all firms are visible.

### SC-03 — SMS Generation Calls Batch Number Service (AF725R)

The SMS XML file generator calls:
- `AB700R` / `AB705R` — job tracking services
- `AS100R` — batch number allocation

If `AS100R` cannot allocate a batch number, the batch number field in the SMS XML will be zero or blank, which may cause the receiving system to reject the file.

---

## Subprogram Calls Affecting Logic

| Program | Called Sub-Program | Purpose | Failure Effect |
|---|---|---|---|
| AF700R | `AF709R` | QY password retrieval | Status = 'A'; job blocked |
| AF703R | (none) | Status lock check; no sub-calls | — |
| AF725R | `AB700R` / `AB705R` | Job tracking | Job not tracked |
| AF725R | `AS100R` | Batch number allocation | Batch number = 0 in XML |
| AF730R | `AF732C` | Email alert for stale files | Alert not sent |
| AF730R | `SP_ALLE_AKTIVE_FILGRP` | Get active file groups | Monitor cannot run |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| `AFTPPF` | FTP job parameter definitions | job name |
| `AFTHPF` | FTP execution log headers | firm + log key |
| `AFTLPF` | FTP execution log lines | firm + log key + sequence |
| `AFGRST` | File group registry | firm + file group |
| `AFPSPF` | FTP path/parameter settings | file group + parameter key |
| `AFPAPF` | FTP command parameter block (work file) | job name |
| `AFLOPF` | FTP output spool (source for log writing) | — |
| `APOSPFR` | Post code register | firm + post code |
| `LWEXPF` | CSV import file for file group update | — |
