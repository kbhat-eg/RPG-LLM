# Discount Matrix Business Rules

**Module**: Discount Matrix (FR prefix)
**System**: ASOFAK
**Source files analyzed**: FR100R (NXCLOUD + NXKORR override), FR160R, FR190R, FR192R, FR410R, FR510R, FR600R, FR601R, FR602R, FR610R, FR700R

---

## 1. Prerequisites / Master Data Requirements

The discount matrix (FRABPF) uses a multi-dimensional key covering price group, discount category, customer category, customer, project, item groups, supplier, module, item, unit, and line type. The following master data must exist before a discount entry can be created or passed:

| Requirement | Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|---|
| Discount category must exist | Discount Category Required | FR100R chains RA06PF; if not found and field entered: *in31 or validation error | RA06PF | RAFIRM/RAFKOD | Not found → error |
| Price group must exist (v5.70+) | Price Group Required | FR100R chains RA30PF if prgr entered; if not found: *in32 | RA30PF | REFIRM/REDKOD | Not found → blocked |
| Customer category must exist | Customer Category Required | FR100R chains RA11PF; if kkat entered and not found: error | RA11PF | RAKKOD | Not found → blocked |
| Customer must exist | Customer Required | FR100R chains RKUNPF; if kund entered and not found: error | RKUNPF | RKFIRM/RKKUND | Not found → blocked |
| Customer project must exist | Project Required | FR100R chains FKPRPF with kund+kpro; if kpro entered and not found: error | FKPRPF | FLFIRM/FLKUND/FLKPRO | Not found → blocked |
| Item groups must exist | Item Group Required | FR100R validates VOGRPF (over-group), VHGRPF (main-group), VUGRPF (sub-group) if entered | VOGRPF/VHGRPF/VUGRPF | vgoogr/vghhgr/vgugrp | Not found → blocked |
| Supplier must exist | Supplier Required | FR100R chains RLEVPF if ldor entered; if not found: error | RLEVPF | RLFIRM/RLLEVR | Not found → blocked |
| Item must exist | Item Required | FR100R chains VVARPF if vare entered; if not found: error | VVARPF | VVFIRM/VVVARE | Not found → blocked |
| Unit must exist | Unit Required | FR100R chains VENHPF if enhe entered; if not found: error | VENHPF | VAEFIRM/VAEENHE | Not found → blocked |
| Line type must exist | Line Type Required | FR100R chains VLTPPF if lety entered; if not found: error | VLTPPF | VYFIRM/VYLETY | Not found → blocked |
| NOBB item must exist | NOBB Item Required | FR100R chains JVARPF if nobb item entered from VA; if not found: error | JVARPF | JVFIRM/JVVARE | Not found → blocked |

---

## 2. Validation Rules

### FR100R — Main Discount Matrix Maintenance

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| F8 Excel Import Authorization | v8.05: before processing F8 (Excel import), AD005R is called with user+routine='FR100R'+function='F08'; if status <> 'Ok': goto b2tagb (blocked) | AUSRPF (via AD005R) | User+routine+function | Unauthorized user cannot import from Excel |
| Master User for Price Group Blank (NXKORR v9.01) | b_mast flag set from FUSRPF/FBKO19; if b_mast=*off: user cannot create or edit discount entries with prgr=' ' (blank price group) | FUSRPF | FBKO19 | Non-master user blocked from editing global (blank prgr) discounts |
| Price Group Restriction (NXKORR v9.01) | For non-master users (b_mast=*off): can only see and edit entries for their own price group; cannot select blank price group unless also master | FUSRPF | User's price group | Restricts view and edit to own prgr |
| Date Consistency | All five discount date pairs (rab1/ra1d, rab2/ra2d, rabn/rand, rabb/rabd, rabt/ratd) and the DG date pair (dekn/dekf/dekt) must have from <= to; invalid dates blocked | FRABPF | fra/til date fields | From > To → blocked |

### FR190R — Copy Discount Agreement

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Source Must Be rkat OR kund, Not Both | If source rkat=0 AND kunf=0: *in31 (no source specified) | Input | b1rkaf/b1kunf | Both zero → blocked |
| Source Cannot Have Both rkat AND kund | If source rkat<>0 AND kunf<>0: *in32 (ambiguous source) | Input | b1rkaf/b1kunf | Both non-zero → blocked |
| Source rkat Must Exist | Chain RA06PF; if rkat entered and not found: *in33 | RA06PF | RAFKOD | Not found → blocked |
| Source kund Must Exist | Chain RKUNPF; if kund entered and not found: *in34 | RKUNPF | RKKUND | Not found → blocked |
| Target Must Be rkat OR kund, Not Both | If target rkat=0 AND kunt=0: *in35 | Input | b1rkat/b1kunt | Both zero → blocked |
| Target Cannot Have Both rkat AND kund | If target rkat<>0 AND kunt<>0: *in36 | Input | b1rkat/b1kunt | Both non-zero → blocked |
| Target rkat Must Exist (or creates via FR192C) | Chain RA06PF or calls FR192C; if not found: *in37 | RA06PF | RAFKOD | Not found → blocked |
| Target kund Must Exist (or creates via FR194C) | Chain RKUNPF or calls FR194C; if not found: *in38 | RKUNPF | RKKUND | Not found → blocked |
| Source Must Not Equal Target | If source key = target key: *in39 | Input | Derived | Same source/target → blocked |

### FR600R — Print Selection Screen

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Valid Print Selection | valg must be 1 or 2; if not: *in31 = *on | Input | valg | Invalid selection → blocked |

### FR601R — Print Matrix Parameter Screen

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Discount Category From <= To | b1rkaf > b1rkat: *in31 | Input | b1rkaf/b1rkat | From > To → blocked |
| Customer Category From <= To | b1kkaf > b1kkat: *in32 | Input | b1kkaf/b1kkat | From > To → blocked |
| Customer From <= To | b1kunf > b1kunt: *in33 | Input | b1kunf/b1kunt | From > To → blocked |
| Project From <= To | b1kprf > b1kprt: *in34 | Input | b1kprf/b1kprt | From > To → blocked |
| Item Group From <= To | Composite ogr+hgr+ugr from <= to: *in35 | Input | Composite group fields | From > To → blocked |
| Supplier From <= To | b1ldof > b1ldot: *in36 | Input | b1ldof/b1ldot | From > To → blocked |
| Module From <= To | b1mof > b1mot: *in37 | Input | b1mof/b1mot | From > To → blocked |
| Item From <= To | b1varf > b1vart: *in38 | Input | b1varf/b1vart | From > To → blocked |

### FR610R — Customer Letter Parameter Screen

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Price Group Must Exist (v5.70) | Chain RA30PF; if not found: *in40 | RA30PF | REDKOD | Not found → blocked |
| rkat OR kund Required, Not Both | rkat=0 AND kund=0: *in31; rkat<>0 AND kund<>0: *in32 | Input | b1rkat/b1kund | Both zero or both non-zero → blocked |
| rkat Must Exist | Chain RA06PF; if rkat entered and not found: *in33 | RA06PF | RAFKOD | Not found → blocked |
| kund Must Exist | Chain RKUNPF; if kund entered and not found: *in34 | RKUNPF | RKKUND | Not found → blocked |
| Project Requires Customer | If kpro entered but kund=0: *in35 | Input | b1kpro/b1kund | Project without customer → blocked |
| Project Must Exist | Chain FKPRPF; if kpro+kund not found: *in36 | FKPRPF | FLKUND/FLKPRO | Not found → blocked |
| Item Assortment Must Exist | Chain VSHEPF; if varesortiment not found: *in37 | VSHEPF | Sortiment key | Not found → blocked |
| Date Must Be Valid | Invalid date format: *in38 | Input | Date field | Invalid → blocked |
| Output Selection Must Be 0 or 1 | valg not 0 or 1: *in39 | Input | valg | Invalid → blocked |

---

## 3. Configuration and Authorization Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Excel Import via AD005R (v8.05) | F8 import requires explicit user authorization in ADB library via AD005R; checks user+routine+function key='F08' | ADB library (via AD005R) | User+routine+F08 | Unauthorized → F8 blocked |
| Master User Flag (NXKORR v9.01) | b_mast flag loaded from FUSRPF field FBKO19; determines if user can edit blank price group entries | FUSRPF | FBKO19 | b_mast=*off → blank prgr restricted |
| Copy Function Key (NXKORR v9.03) | F9 (inki) triggers copy_matrix subroutine; available to all users but restricted by prgr access | FRABPF | prgr key | Follows same prgr access rules |
| Price Group Auto-Load (v7.01) | FR610R calls VL711R on startup to load user's price group automatically | FUSRPF | User's price group | Pre-populates price group field |
| Project-Level Discounts Protected | FR410R: when deleting customer discounts by kund, only records with frkpro=0 are deleted; project-specific discounts are preserved | FRABPF | FRKPRO | Project discounts survive customer-level delete |

---

## 4. Financial / Transactional Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| DG Calculation via FR700R | FR700R receives all FRABPF discount fields and returns p_dggj (5:2 decimal gross margin percentage); called by FR602R for each matrix line in print | FR700R | p_dggj | Returned only if DG date range is valid |
| DG Field Priority | FR700R: if frdekn/frdekd (DG numerator/denominator) is set AND today is within frdekf–frdekt: return DG immediately, skip discount rate resolution | FRABPF | FRDEKN/FRDEKD/FRDEKF/FRDEKT | DG field takes priority over discount rates |
| Date-Controlled Discount Rates | FR700R resolves 5 discount fields (frrab1, frrab2, frrabn, frrabb, frrabt) each with from/to dates; the rate whose date range includes the query date is applied | FRABPF | FRRA1D/FRRA2D/FRRAND/FRRABD/FRATD | Only rate with valid date range applied |
| Copy Preserves All Rates | FR190R copies all FRABPF fields from source to target; all five rate fields and dates are copied | FRABPF | All frrab/fra/frt fields | Full copy; no field omitted |
| Redundancy Removal (FR160R) | FR160R deletes lower-level discount entries that duplicate a higher-level entry; checks all 24 discount and DG fields for equality before deleting | FRABPF | All 24 frrab/fra/frt/frdek fields | Full field equality required before delete |

---

## 5. Status and Lifecycle Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Delete by Discount Category | FR410R reads FRABPF by rkat (FRABLD index) and deletes all matching records | FRABPF | FRRKAT | All entries for rkat removed |
| Delete by Customer (Preserves Projects) | FR410R reads FRABPF by kund (FRABL3 index); only deletes records where frkpro=0 | FRABPF | FRKUND/FRKPRO | Project-specific entries not deleted |
| Audit Trail | FR100R writes FREUSR (modified by user), FREDAT/FRETIM (modification timestamp) on every update | FRABPF | FREUSR/FREDAT/FRETIM | Maintained on every write/update |
| 12 Logical Views | FRABPF is accessed via 12 logical files (frabl1–frablc, frablr, frablu) with different key orderings; each view provides a different sorting/access path | FRABPF | All key combinations | Logical views enable flexible search |

---

## 6. Special Conditions (Program-Specific)

### FR100R — Main Discount Matrix Maintenance (NXKORR version)

The NXKORR version extends the NXCLOUD version with:

- **v9.01**: Master user control via FUSRPF.FBKO19 (b_mast flag). Non-master users: read-only for blank price group entries; can only copy from blank prgr to their own prgr; can delete/edit entries with their own prgr.
- **v9.02**: Blank price group filter can be excluded by non-master users viewing their own prgr.
- **v9.03**: F9 function key (copy_matrix subroutine) added for copying between price groups.
- **v9.04**: Price group inquiry program added on F4 for prgr field.
- **v8.04**: F8 Excel import function added (NXS-12523).
- **v8.05**: F8 requires AD005R authorization check (NXS-12523 security extension).
- **v7.01 (K00)**: Combined rkat+kkat (discount category + customer category) in the matrix is allowed (NXS-4138).

### FR160R — Remove Redundant Discounts

- Sequential scan: reads all FRABPF records.
- For item-level entries: looks up item in VVARPF to find ugr/hgr/ogr; then checks if identical discount exists at the sub-group level first, then at main-group level, then at over-group level.
- The equality check covers all 24 discount/DG fields; a single field difference means the record is NOT redundant.
- Designed to clean up after mass updates where item-level details duplicate group-level rules.

### FR700R — DG Calculator

- Receives all FRABPF discount fields as parameters.
- Exits immediately if DG fields (frdekn/frdekd) are set and today is within the DG date range.
- Otherwise resolves discount rate by checking which of the 5 rate slots has a valid date; the level (vare/modn/ldor/vgrp) determines which rate field is primary.
- Returns p_dggj as a 5:2 decimal percentage.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Callee | Purpose | Effect on Blocking |
|---|---|---|---|
| FR100R | AD005R | Authorization check for F8 (Excel import) | Not 'Ok' → F8 blocked |
| FR100R | VL711R | Auto-load user's price group | Pre-populates prgr field |
| FR190R | FR192C | Validate/create target discount category | Cross-firm category validation |
| FR190R | FR194C | Validate/create target customer | Cross-firm customer validation |
| FR190R | FR195C | Delete target entries before copy | Clears target before copy |
| FR190R | FR196C | Copy source to target | Performs the actual copy |
| FR192R | RA06PF | Check discount category exists | Returns p_feil=*on if not found |
| FR410R | (direct FRABPF delete) | Delete discount entries by rkat or kund | Project entries preserved |
| FR602R | FR700R | Calculate DG for each printed line | Called per line; result shown in print |
| FR610R | VL711R | Auto-load price group | Pre-populates prgr |

---

## 8. Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| FRABPF | Discount matrix | FRFIRM, FRPRGR, FRRKAT, FRKKAT, FRKUND, FRKPRO, FROGRP, FRHGRP, FRUGRP, FRLDOR, FRMODN, FRVARE, FRENHE, FRLETY |
| RA06PF | Discount category register | RAFIRM, RAFKOD |
| RA30PF | Price group register | REFIRM, REDKOD |
| RA11PF | Customer category register | RAFIRM, RAKKOD |
| RKUNPF | Customer register | RKFIRM, RKKUND |
| FKPRPF | Customer project register | FLFIRM, FLKUND, FLKPRO |
| VOGRPF | Over-group register | VGFIRM, VGOOGR |
| VHGRPF | Main-group register | VGFIRM, VGHHGR |
| VUGRPF | Sub-group register | VGFIRM, VGUGRP |
| RLEVPF | Supplier register | RLFIRM, RLLEVR |
| VVARPF | Item master | VVFIRM, VVVARE |
| VENHPF | Unit register | VAEFIRM, VAEENHE |
| VLTPPF | Line type register | VYFIRM, VYLETY |
| JVARPF | NOBB item register | JVFIRM, JVVARE |
| VSHEPF | Item assortment register | Sortiment key |
| FUSRPF | User register (price group, flags) | FBFIRM, FBUSER |
