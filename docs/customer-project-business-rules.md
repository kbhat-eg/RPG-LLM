# Business Logic for Customer Project Creation and Order Integration

The customer project module (ASOFAK system, FL prefix) maintains the FKPRPF register, which links customers (RKUNPF) to named projects (flkpro), tracking project dates, addresses, contacts, billing type, and optional external project numbers. Projects are created from the FL100R maintenance screen or directly from order entry when an order requires a project not yet in the system. The RP prefix programs handle the parallel project accounting register (RP1PPF / ASOKON), which supports project leaders, sub-projects, activities, and budget integration. Blocking rules centre on mandatory customer existence, duplicate project name prevention, date validity, and integration constraints with Byggdok/Cobuilder modules.

---

## Prerequisites / Master Data Requirements

- **RKUNPF** (customer register): A valid customer number `flkund` must exist in RKUNPF with a matching firm key before any customer project can be created or viewed. FL105R chains RKUNPF (`rkunl1_key`) before displaying the project detail screen. If the chain fails (`not %found`), indicator `*in32` is set and the screen redisplays with an error; the project record is not created.
- **FUSRPF** (user register): The logged-in user must exist in FUSRPF. FL105R chains FUSRPF by `l_user` to retrieve the user's default salesperson code (`fbselg`), which is then used to pre-fill `c2selg`. If not found, the salesperson field starts blank.
- **AVALPF** (field validation register, v6.35+): If validation rules for program FL105R exist in AVALPF, fields `c2adr1` (address line 1) and `c2ponr` (postal code) may be declared mandatory by firm-level parameter flags `u_adr1` and `u_ponr`. If these flags are on and the fields are blank, the screen is re-displayed with an error.
- **AVALPF for Cobuilder** (v6.36): If `w_cobu = *on` and `c2adr1 = *blank` or `c2ponr = 0`, an absolute block is enforced: the record cannot be saved.
- **AMODPF** (module register, v6.32): The BYGGDOK and COBUILDER modules must be registered in AMODPF for the corresponding functionality to activate. If AMODPF has no record for `'BYGGDOK'` or `'COBUILDER'`, the respective `w_bdok` / `w_cobu` flags remain `*off` and those fields are hidden or inactive.
- **AFPSPF** (parameter register, v6.32): Even if the module exists in AMODPF, the firm-level parameter `'BYGGDOK'` or `'COBUILDER'` must also be active in AFPSPF (`afpslr_ufil = l_filg` and `afpslr_uide = 'BYGGDOK'`). If the parameter record is not found, the module is treated as inactive.
- **RP1PPF** (project register for ASOKON): A project `rp1pro` must exist in RP1PPF before sub-projects (RP2UPFR) or activities (RP3APFR) can be registered. RP100R validates that project number is not blank (v6.31 fix).
- **RPKOPF** (project category codes, v9.01): If a project category (`rp1pkat`) is entered in RP100R, it must exist in RPKOPF for the firm and type. If not found, an error message is issued via AA007R.
- **ILONPF** (payroll register): The project leader code entered in RP100R must exist in ILONPF. RP100R chains `ilonl1_key` by `il_rkli + il_rlnr`. If not found, the project cannot be saved.
- **Start date** (v8.01): RP100R enforces that a project start date `rp1fsd` must be entered when creating a new project. If blank, the record is blocked.

---

## Validation Rules

- **Customer existence** (FL100R / FL104R): Before calling FL105R, the calling program chains RKUNPF. If `not %found`, indicator `*in32` is set and the operation is aborted. The project detail screen (FL105R / FL105D) is never reached.
- **Customer number mandatory** (FL104R): If `c1kund = 0` after the user presses Enter, the program `goto avslutt` and returns without creating anything. The customer number field is required and non-zero.
- **Duplicate project name** (FL105R v1.00): Chains FKPRL4 (logical by firm + customer + name). If a record with the same `flnavn` already exists for the same customer, a duplicate error is raised and the save is blocked.
- **Project start date** (FL105R v6.24/6.25): For new projects, `c2fdat` (from-date) must not be in the past by more than 30 days. The program computes `w_fdat = *today - 30` and if `c2fdat < w_fdat`, an error indicator is set and the screen redisplays. Existing projects may update the from-date without this check.
- **Cobuilder address requirement** (FL105R v6.36): If `w_cobu = *on`, both `c2adr1` (address) and `c2ponr` (postal code) must be non-blank/non-zero. Missing either field blocks the save.
- **Byggdok flag** (FL105R v6.22): If `w_bdok = *on` and `c2bdok = 1`, then `c2epro` (external project number) must be non-blank; otherwise an error is set and the save is blocked.
- **Project category** (RP100R v9.01): If `b2pkat` is entered and chains RPKOPF with no match, program AA007R is called with an error message code; the project record is not saved.
- **Project leader** (RP100R): Chain ILONPF by `il_rkli + il_rlnr`. If not found, the save is blocked and a validation error indicator is set.
- **Project number non-blank** (RP100R v6.31): If the user attempts to enter a blank project number in the create screen, the system prevents saving.

---

## Configuration and Authorization Rules

- **Firm number from LDA** (positions 944–946): FL100R, FL500R, FL510R, and RP100R all read `l_firm` from the LDA. This firm is used as the primary key component for all FKPRPF, RKUNPF, and RP1PPF lookups. No cross-firm project access is possible.
- **File group / library authorization** (FL100R v7.01): FL100R calls CO402R with key `'rabatt-spes'` to check if the user has access to discount matrix (FR100R) and special price (FP100R) functions. If `co402_verdi1 <> *on` (or `u_s701 = *off`), the F13 and F14 keys for special prices and discount matrix are suppressed; pressing them has no effect.
- **Salesperson default**: If FUSRPF has a salesperson code (`fbselg`) for the logged-in user, it is pre-filled in `c2selg` when creating a new project. This is a default only; the user may override it.
- **Expired projects filter** (FL100R v5.62): F23 toggles indicator `*in80` to show/hide expired projects. When `*in80 = *on`, FKPRPF records with `fltdat < *today` are excluded from the subfile. This is a display filter only and does not affect record integrity.

---

## Financial / Transactional Rules

- **Billing type and terms inheritance** (FL105R): When a new project is created, `c2faty` (billing type), `c2mkod` (payment condition), `c2ekgb` (own reference), `c2betb` (payment terms), `c2neto` (net days), `c2fagb` (invoice discount), `c2samf` (collection invoice flag), and `c2sekv` (invoice sequence) are all copied from RKUNPF fields `rkfaty`, `rkmkod`, etc. These are defaults; the user may override them at project level.
- **GL account for shrinkage** (FL105R v7.02): Fields `f2ktou` (expense account) and `f2ktsd`/`f2ktsc` (sub-account variants) are read from FKPRPF extended table (FKP2PF). If the account is non-zero, RS740R is called to resolve the account text. If FKP2PF has no record or the account is zero, the text is set blank and no validation block applies.
- **Payment terms description** (FL105R v6.30): If `f2betb` (payment terms code) is non-zero, RS703R is called to retrieve the payment terms text. If RS703R returns `p_stat = 'N'` (not found), `f2ctxt` is cleared. No blocking occurs; a missing terms description is accepted.
- **Order linkage update** (FL105R v6.34): When billing type (`flfaty`), payment condition (`flmkod`), etc. are changed on an existing project, and if `w_endr = *on` (change flag), the program propagates these changes to open order headers (FOHEPF) for all orders linked to this project. If FOHEPF is locked by another user, the update is skipped silently.
- **Change logging** (FL105R v7.05): Changes to `flfaty` (billing type) and `flopda` (pricing date) are written to SLOGPF (change log). The log record includes user, timestamp, old and new values.

---

## Status and Lifecycle Rules

- **Project expiry**: A project is considered expired when `fltdat` (to-date) is in the past. FL100R can filter these out with F23/indicator `*in80`. Expired projects can still be edited but are excluded from new order project lookup (FL501R) unless the caller explicitly requests expired projects.
- **Deletion cascade** (FL400R): When a project is deleted via option `4` → D1WIN confirmation, FL400R is called with firm/customer/project as parameters. It deletes:
  1. The FKPRPF main record (FKPRLUR)
  2. The FKPSPF search text record (FKPSLUR)
  3. All FRABPF discount matrix records (FRABL3 — all rows for firm+customer+project)
  4. All FPRIPF special price records (FPRIL3 — all rows)
  5. The FKP3PF supplementary record (FKPIU — v8.01+)
  - If the FKPRPF record is not found in step 1, `*in90 = *on`; the delete is skipped but remaining steps still execute.
- **Copy project** (FL100R option `3` → K1WIN): Calls FL105R with an existing project's data pre-filled. FL105R creates a new FKPRPF record with the new project number. If the copy target already exists (duplicate name), the save is blocked per the duplicate name rule.
- **Project view** (FL100R option `5`): Calls FL505R (view-only). No updates occur. FL510R provides a combined customer+project view including memo balance (RKMEPF) and credit limit (RKUNPF.rkgrns).

---

## Special Conditions (Program-Specific)

- **FL104R — project creation from order entry**: Called when an order needs a project but the customer is unknown. Accepts firm/customer/project as parameters, displays a customer selection screen, then delegates to FL105R. If `w_kpro = 0` after FL105R returns, no project was created and `p_kund`/`p_kpro` are not populated in the return parameters. The calling order program must handle this case.
- **FL501R — date-filtered project lookup**: Called from order entry with firm, customer, and date as parameters. Shows non-expired projects for the customer by default; F3 toggles to include expired. Returns selected project number and full address/contact data (including `p_upro`/`p_akti` for sub-project and activity from RP2UPFR, v7.01+). If no project is found for the customer and date, the list is empty and the user must either select an expired project or exit without selection.
- **FL505R / FL510R — view programs**: Read-only. FL510R calls RS205R to compute the VAT percentage for memo balance (v6.32+) and FO763R to compute the extended memo balance including orders (Mestergruppen feature, v7.01). If CO402R returns `u_s701 = *off`, F13 (special prices) and the memo balance extension are suppressed.
- **FL581R / FL710R / FL750R / FL751R / FL891R**: Supporting programs for route management (flkjor/flkjnr), project reporting, and project document printing. No blocking rules identified beyond standard firm key matching.
- **RP100R — project accounting register**: Separate from FKPRPF. Creates RP1PPF records for project accounting. V9.03 links to ANUMPF for automatic number assignment. When sub-projects are used (`w_upro`), option `8` transfers to sub-project maintenance (RP110R) after initial project creation. V6.35: prevents budget entry if project is divided into sub-projects/activities.

---

## Subprogram Calls Affecting Logic

| Program | Trigger | Logic | Impact |
|---------|---------|-------|--------|
| FL105R | F6 (new) in FL100R; option 2 (change); called from FL104R | Maintains FKPRPF detail; validates customer, dates, address, external project; handles Byggdok/Cobuilder flags | Central creation/update gate; all validation blocking occurs here |
| FL400R | Option 4 confirmed in FL100R | Deletes FKPRPF + FKPSPF + FRABPF + FPRIPF + FKP3PF for the project | Irreversible; cascade delete; called only after D1WIN confirmation |
| FL505R | Option 5 in FL100R subfile | View-only display of project detail | No updates; uses firm+customer+project key |
| FL510R | Called from order module and FL505R | View customer+project summary with memo balance, credit limit, credit cards | Calls RS205R and optionally FO763R |
| FL501R | Called from order entry | Subfile of non-expired projects for a customer on a given date | Returns project fields + sub-project/activity to caller |
| RK500R | F4 on customer field in FL104R | Customer lookup | Returns customer number and name |
| RS205R | FL510R (v6.32+) | Retrieves current VAT percentage for VAT code '3' | Used only for memo balance display; no blocking |
| FO763R | FL510R (v7.01+, Mestergruppen) | Computes extended memo balance with orders for last N days | Requires CO402R parameter; suppressed if u_s701 = *off |
| CO402R | FL100R, FL105R, FL510R | Reads firm-level feature flags from AFPSPF | Controls access to discount matrix, special prices, Byggdok, Cobuilder |
| RS703R | FL105R (v6.30+) when payment terms are non-zero | Retrieves payment terms description text | Returns blank if not found; no blocking |
| RS740R | FL105R (v7.02+, v7.10) when GL account is non-zero | Retrieves GL account description | Returns blank if not found; no blocking |
| AA007R | RP100R (v9.01+) for project category validation | Issues error message for invalid category code | Blocks project save if category invalid |
| RP601R | Called from FL501R (v7.01+) | Fetches sub-project and activity details from RP2UPFR | Provides p_upro, p_akti to order entry |

---

## Reference Tables

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| FKPRPF (FKPRL*) | firm, kund, kpro | Customer project master — main register maintained by FL100R/FL105R |
| FKPSPF | firm, kund, kpro | Search text for project name lookup |
| FKP2PF | firm, kund, kpro | Extended project data (GL accounts for shrinkage, v7.02+) |
| FKP3PF | firm, kund, kpro | Supplementary project data (v8.01+, e.g. profile text, number of items) |
| RKUNPF | firm, kund | Customer register — mandatory for project creation |
| FRABPF | firm, kund, kpro, ... | Discount matrix — deleted cascade when project is deleted |
| FPRIPF | firm, kund, kpro, ... | Special prices — deleted cascade when project is deleted |
| FOHEPF | firm, numm, suff | Order headers — updated when project billing terms change (v6.34) |
| SLOGPF | firm, register, ... | Change log — receives billing type and price date changes (v7.05) |
| AVALPF | firm, pgm | Field validation rules — mandatory field enforcement (v6.35) |
| AMODPF | module | Module activation register (Byggdok, Cobuilder) |
| AFPSPF | filg, uide | Firm-level parameter flags for module activation |
| FUSRPF | firm, user | User register — provides default salesperson code |
| APOSPF | ponr | Postal code/location register — used for postal code lookup |
| RP1PPF (RP1PL*) | firm, pros | Project register (ASOKON) — project leader, dates, budget |
| RP2UPFR | firm, pros, upro | Sub-project register |
| RP3APFR | firm, pros, upro, akti | Activity register |
| RP4OPFR | firm, pros, arbo | Budget order links |
| RPKOPF | firm, type, kode | Project category codes (v9.01+) |
| ILONPF | kli, lnr | Payroll/employee register for project leader validation |
| ANUMPF | firm, fell, type | Automatic number series for project numbers (RP100R v9.03) |
