# Project and Unit Maintenance Business Rules

## Introduction

This document describes the business rules enforced by the ASOKON Project Maintenance module (RP prefix programs). The module manages the project hierarchy: projects (RP1PPF), sub-projects (RP2UPF), activities (RP3APF), budgets/orders (RP4OPF), project reporting structures (RPRRPF), and salary transactions (RHTRPF). All rules described below represent conditions that **block or prevent** an operation from completing. Fields use DB2 TABLE.COLUMN notation where tables are referenced by their physical-file name.

---

## Prerequisites and Master Data Requirements

| Master Data | Table | Key Fields | Used By |
|---|---|---|---|
| Project master | RP1PPF | RP1FIRM + RP1PRO | RP100R, RP110R, RP120R |
| Sub-project | RP2UPF | RP2FIRM + RP2PRO + RP2UPR | RP110R |
| Activity | RP3APF | RP3FIRM + RP3PRO + RP3UPR + RP3AKT | RP120R |
| Budget/order | RP4OPF | RP4FIRM + RP4PRO | RP100R |
| Customer | RKUNPF | RKFIRM + RKKUND | RP100R |
| Project leader / employee | ILONPF | ILFIRM + ILRKLI + ILRLNR | RP100R, RP110R |
| Cost center code | RA07PF | RAGFIR + RAGKOD | RP100R |
| General code | RA09PF | RAIFIR + RAIKOD | RP100R (v6.30+) |
| Project report structure | RPRRPF | RPHFIR + RPHRAP + RPHLIN + RPHULI | RP200R |
| Salary transactions | RHTRPF | RBFIRM + RBHPRO | RP100R |
| User access | AUSRPF / FUSRPF | Firm + user | RP100R |
| Project categories | RPKOPF | Firm + category | RP100R (v9.02+) |
| Number series | ANUMPF | Firm + series | RP100R (v9.03+) |
| VA codes / activation codes | RA21PF | RAUFIRM + RAUKOD | RP120R |

---

## Validation Rules

### RP100R — Project Maintenance (RP1PPF)

The following conditions block saving a project record:

| Field | Condition | Version | Effect |
|---|---|---|---|
| Project number (RP1PPF.RP1PRO) | Blank | v6.31+ | Save blocked; project number is mandatory |
| Start date (RP1PPF.RP1FSD) | Zero / not set | v8.01+ | Save blocked; start date is mandatory |
| Budget entry | Attempted when project is divided into sub-projects OR activities | v6.35 | Budget cannot be entered at project level if sub-projects or activities exist |
| Project accounting phase 2 fields | Used without phase 2 activation (v9.01/9.02 flag) | v9.01+ | Phase 2 fields are blocked until module activation |

**Project leader source (v6.30):** The project leader is resolved based on the `w_prlslg` variable (initialised to `'S'`):
- `'S'` — project leader is taken from the seller/salesperson field
- `'P'` — project leader is taken from the project leader field

This controls which lookup (against ILONPF or the sales representative register) is used to populate the project leader at creation time.

**Budget entry restriction (v6.35):** Budget amounts (RP4OPF) cannot be entered at the main project level if the project has been subdivided into sub-projects (RP2UPF records exist) or activities (RP3APF records exist). This prevents budget double-counting.

**Project number autoincrement (v9.03):** When project number autoincrement is enabled via ANUMPF, the system generates the project number automatically. Manual entry of a project number in this mode is not permitted.

### RP110R — Sub-project Maintenance (RP2UPF)

Sub-project maintenance operates as a subordinate screen called from RP100R (F-key navigation from v6.36). The following conditions apply:

| Condition | Effect |
|---|---|
| Parent project (RP1PPF) does not exist | Sub-project cannot be created; parent must exist first |
| Sub-project number (RP2UPF.RP2UPR) is blank | Save blocked |
| Activity flag (RP2UPF.RP2AKF) is set and activities already exist for this sub-project in RP3APF | Budget cannot be entered at sub-project level; must be entered at activity level |

**Navigation:** If the sub-project screen is reached from RP100R via the "proceed to sub-project" path (`c2rup = 'J'`), the program passes control back to RP100R on completion. If the "proceed to activity" flag is set (`c2rak = 'J'`), control flows further to RP120R.

### RP120R — Activity Maintenance (RP3APF)

Activity maintenance operates as the third level in the project hierarchy. The following conditions apply:

| Condition | Effect |
|---|---|
| Parent project and sub-project must exist in RP2UPF | Activity cannot be created without a valid parent sub-project |
| Activity code (RP3APF.RP3AKT) is blank | Save blocked |
| Invalid VA code (RA21PF) | Activities requiring a VA activation code block save if the code is not found in RA21PF |

From version 6.36, activities can be reached directly from the project screen when `c2rak = 'J'`, bypassing the sub-project screen. The parameters RP3PRO (project), RP3UPR (sub-project), and RP3AKT (activity) must all be valid before a save is permitted.

### RP200R — Project Report Structure Maintenance (RPRRPF)

RP200R manages the reporting hierarchy in RPRRPF. The following conditions apply:

| Condition | Effect |
|---|---|
| Report number (RPRRPF.RPHRAP) is zero | Save blocked |
| Description (RPRRPF.RPHTXT) is blank | Save blocked |

The report structure supports both sequential (by report number) and alphabetical (by description text) positioning. Copy operations are supported across file groups (from version 6.20), meaning report structures can be duplicated from one company file group to another.

---

## Configuration and Authorization Rules

### Project Accounting Phase 2 (v9.01/9.02, RP100R)

Project accounting phase 2 introduced full accounting integration for projects. Once activated (controlled by a system switch in AMODPF or AFPSPF):
- Additional fields for accounting are enabled on the project maintenance screen.
- Transaction fields for income (RPSUMI), cost (RPSUMK), budget income, and budget cost become active.
- Prior to activation, these fields are protected and cannot be entered.

### File Group Support (RP200R, v6.20)

From version 6.20, the report structure maintenance program reads the file group from the LDA (l_filg at positions 931–933). Report structures can be copied between file groups, enabling shared reporting hierarchies across multiple company entities.

### User Filtering (RP100R)

The project list can be filtered by:
- Department (b2vavd) — filters against RP1PPF.RP1AVD
- Project leader (b2vprl) — filters against RP1PPF.RP1PRL (or seller code depending on `w_prlslg`)
- Active/all filter (b2aktiv) — toggles between active and all project statuses

These filters do not block operations; they limit which projects are displayed in the subfile.

---

## Financial and Transactional Rules

### Budget and Actual Tracking (RP100R, RP110R)

The project maintenance screen shows four financial columns rotated with the F11 key:
1. Budget income vs. budget cost (and actual income vs. actual cost)
2. Budget income vs. actual income (remaining and % complete)
3. Budget cost vs. actual cost (remaining and % complete)
4. From date, to date, department, project leader

The financial columns are calculated from RP4OPF (budget/order records) and RPRRPF/RHTRPF (reporting and salary transactions). No blocking conditions apply to reading these values, but budget entry is blocked at project level if sub-projects or activities exist (as described in the VG100R section above).

### Salary Transaction Integration (RHTRPF)

Salary transaction records (RHTRPF keyed by firm + project) are read and included in project totals. These are populated by payroll processes and are read-only within the RP module. No blocking conditions apply to reading salary transactions.

---

## Status and Lifecycle Rules

### Project Status Filter

The project list filter `b2aktiv` controls whether only active projects or all projects (including closed ones) are shown. The field RP1PPF.RP1STS holds the project status. Only active projects (typically RP1STS = active code) are shown by default. This is a display filter only and does not block any operations.

### Project Closure

When a project is closed (status updated), it remains in RP1PPF but is excluded from the default active filter. Budget and actual figures remain accessible via the all-projects view. Deletion of a project with associated sub-projects or activities requires cascading deletes through RP2UPF and RP3APF.

---

## Special Conditions

### Navigation Flow (v6.36)

From version 6.36, after creating or updating a project, the system can automatically navigate to sub-project or activity maintenance depending on the flags `c2rup` (proceed to sub-project) and `c2rak` (proceed to activity):
- If `c2rup = 'J'`: calls RP110R with the new project number.
- If `c2rak = 'J'` (and not going to sub-projects): calls RP120R directly.
- These flags are set on the project detail screen (C2BLD) by the user, not automatically.

### Lookup Programs (RP500R, RP600R, RP700R)

These programs provide search/lookup and report parameter entry functionality:
- RP500R: Read-only project search.
- RP600R: Report parameter entry for project reports (calls RP600C).
- RP700R: Additional reporting functions.

No blocking conditions specific to master data validation have been identified in these programs beyond the date and range validations common to all report parameter programs.

---

## Subprogram Calls

| Calling Program | Called Program | Purpose |
|---|---|---|
| RP100R | RP110R | Sub-project maintenance (via F-key, v6.36) |
| RP100R | RP120R | Activity maintenance (via F-key, v6.36) |
| RP100R | RP151R | Project status inquiry (option 1 in subfile) |
| RP110R | RP120R | Activity maintenance from sub-project screen |
| RP600R | RP600C | Batch project report |

---

## Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| RP1PPF | Project master | RP1FIRM + RP1PRO |
| RP2UPF | Sub-project register | RP2FIRM + RP2PRO + RP2UPR |
| RP3APF | Activity register | RP3FIRM + RP3PRO + RP3UPR + RP3AKT |
| RP4OPF | Budget and order register | RP4FIRM + RP4PRO |
| RPRRPF | Project reporting structure | RPHFIR + RPHRAP + RPHLIN + RPHULI |
| RHTRPF | Salary transactions | RBFIRM + RBHPRO |
| RKUNPF | Customer master | RKFIRM + RKKUND |
| ILONPF | Employee register | ILFIRM + ILRKLI + ILRLNR |
| RA07PF | Cost center codes | RAGFIR + RAGKOD |
| RA09PF | General codes | RAIFIR + RAIKOD |
| RA21PF | VA activation codes | RAUFIRM + RAUKOD |
| RPKOPF | Project categories | Firm + category |
| ANUMPF | Number series | Firm + series key |
| AUSRPF | User access control | Firm + user |
| FUSRPF | User rights | Firm + user |
