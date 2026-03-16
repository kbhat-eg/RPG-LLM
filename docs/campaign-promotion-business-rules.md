# Campaign and Promotion Business Rules

**Module**: Campaign / Promotion (JM prefix)
**System**: ASVADM / ASPMAL
**Source files analyzed**: JM100R, JM110R, JM111R, JM112R, JM113R, JM114R, JM115R, JM116R, JM119R, JM500R, JM510R, JM591R

---

## 1. Prerequisites / Master Data Requirements

Before a campaign header or campaign price can be created or modified, the following master data must exist and be valid:

| Requirement | Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|---|
| Price group must exist | Price Group Validity | If price group (c1prgr) is entered, it must exist in RA30PF | RA30PF | REDKOD | If not found: *in32 = *on, reject |
| Campaign code must be non-blank | Campaign Code Required | If c1kamp = *blank on the creation screen, the record is not saved | JKAHPF | JMKAMP | Blank campaign code skips save |
| Campaign description required | Description Required | c2besk must not be blank; if blank: *in31 = *on | JKAHPF | JMBESK | Blank blocks write |
| From-date required | From-Date Required | c2fdat must be non-zero; if zero: *in32 = *on | JKAHPF | JMFDAT | Zero blocks write |
| To-date required | To-Date Required | c2tdat must be non-zero; if zero: *in34 = *on | JKAHPF | JMTDAT | Zero blocks write |
| Responsible person required | Responsible Required | c2ansv must not be blank; if blank: *in37 = *on | JKAHPF | JMANSV | Blank blocks write |
| Item must exist in VVARPF | Item Master Required | Called via JM111R; item is looked up in VVARPF by own item number, supplier item number, and NOBB number | VVARPF | VVVARE | Not found blocks campaign price status |
| NOBB packaging entry required | NOBB Packaging Required | JM111R checks JVPKPF for item+unit combination; if no packaging found, unit is flagged invalid | JVPKPF | JWENHE | Not found blocks unit acceptance (v6.30+) |

---

## 2. Validation Rules

These rules check the consistency of data before saving a campaign header or campaign price line.

### Campaign Header (JM100R)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Valid From-Date Format | c2fdat is tested with TEST(D); if invalid date format: *in33 = *on | JKAHPF | JMFDAT | Invalid date blocks save |
| Valid To-Date Format | c2tdat is tested with TEST(D); if invalid date format: *in35 = *on | JKAHPF | JMTDAT | Invalid date blocks save |
| From-Date Before To-Date | w_fdat must be <= w_tdat; if from > to: *in36 = *on | JKAHPF | JMFDAT/JMTDAT | From > To blocks save |
| Price Group Exists on Filter | Filter screen (f1win) validates price group against RA30PF; if not found: *in34 = *on | RA30PF | REDKOD | Invalid price group blocks filter apply |
| Duplicate Campaign Key on Copy | When copying (xk1win), target campaign code must not already exist in JKAHPF; if found: *in32 = *on | JKAHPF | JMKAMP | Duplicate blocks copy |
| Target Campaign Code Non-Blank on Copy | k1kamp must not be blank; if blank: *in31 = *on | JKAHPF | JMKAMP | Blank target blocks copy |

### Campaign Prices (JM111R - Status Check)

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Item Exists by Own Number | VVARPF is chained by item number; if not found, the item lookup tries NOBB number and supplier number before failing | VVARPF | VVVARE | Not found in any lookup = unresolved item |
| Unit Valid for NOBB Item | JM111R checks JVPKPF for matching item+unit; b_enhe_ok must be *on; if no valid unit found and b_enhe_en=*off (v6.36): status fails | JVPKPF | JWENHE | No valid unit blocks status return |
| NOBB Packaging Date Range | JM111R checks that today's date (w_udat) falls within package from/to dates in JVPKPF (v6.32) | JVPKPF | JWDFRA/JWDTIL | Out-of-range date means packaging is not current |
| Campaign Price <= Suggested Retail | JM111R compares campaign price against VP700R sale price; campaign price must not exceed suggested retail | Computed via VP700R | Derived | Campaign price > retail flagged (v5.50+) |
| Cost Price Check (Optional) | v7.01 switch u_701: if off, cost price = 0 is allowed; if on, cost price must be > 0 | JKAVPF | JMVKPR | Zero cost blocks save when switch on |

---

## 3. Configuration and Authorization Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Price Group Auto-Load | On startup JM100R calls VL711R to load the user's active price group (w_prgr) from FUSRPF or related; this pre-filters the campaign list | FUSRPF | FBPRGR | User without price group sees all campaigns (blank filter) |
| Price Group Filtering | The filter window (f1win) can restrict display to a specific price group; invalid price group returns *in34 | RA30PF | REDKOD | Unrecognized price group blocks filter |
| Campaign Price Access | Access to JM110R (campaign price maintenance) is via option 7 on the campaign header list; no separate authorization check beyond menu access | JKAHPF | — | Standard menu-level control |

---

## 4. Financial / Transactional Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Campaign Price Stored as Numeric | Campaign prices in JKAVPF are stored as packed decimal (JMVKPR); validated via JM111R price lookup | JKAVPF | JMVKPR | Zero allowed only with v7.01 switch |
| Purchase Price from VP753R | JM111R calls VP753R to fetch purchase price for the item+price group+unit; if purchase price available, it is stored for cost comparison | Derived via VP753R | — | Used for margin display; does not block save |
| Sale Price from VP700R | JM111R calls VP700R to get the recommended sale price; campaign price comparison is made against this value | Derived via VP700R | — | Used for validation of campaign price level |
| Delete Campaign Cascades to Prices | When a campaign header is deleted (xd1win in JM100R), all associated JKAVPF records for that campaign+price group are deleted sequentially | JKAVPF | JMVKAM/JMVPRG | Header delete purges all price lines |
| Copy Campaign Includes Prices | When a campaign is copied (xk1win), all JKAVPF records from the source campaign+price group are duplicated under the new key | JKAVPF | JMVKAM/JMVPRG | Price lines carried forward on copy |

---

## 5. Status and Lifecycle Rules

| Rule Name | Logic | File | Field | Condition |
|---|---|---|---|---|
| Campaign Header Dates Control Validity | JKAHPF fields JMFDAT (from-date) and JMTDAT (to-date) define when a campaign is active; no automatic expiry enforcement in JM100R, but date range is validated on save | JKAHPF | JMFDAT/JMTDAT | From > To is blocked at save |
| First Transfer Date Recorded | v6.30: JMFODA (first transfer date) is set when the campaign is first exported; shown with 'X' indicator in list | JKAHPF | JMFODA | Read-only once set |
| Modification Timestamp | JMEDAT/JMETIM/JMEUSR are updated on every save (JM100R c2bld write/update) | JKAHPF | JMEDAT/JMETIM/JMEUSR | Audit trail maintained |
| Campaign Status via JM111R | JM111R (called by JM110R) returns p_stat: 0=ok but no unit found, 1=unit valid, other values indicate pricing anomalies | Returned by JM111R | p_stat | Status code drives display indicators in JM110R |

---

## 6. Special Conditions (Program-Specific)

### JM100R — Campaign Header Maintenance

- On creation (F6), if the campaign code already exists, a confirmation message (c1msg screen) is shown; if user cancels (F12/F3), the duplicate is not overwritten.
- The list can be sorted by campaign code (default), from-date, responsible user, or price group.
- v8.02: Price group extended from 1 to 2 positions; RA30PF lookup uses REDKOD as the key.

### JM111R — Status Resolution for Campaign Prices

- JM111R resolves item identity through three fallback lookups: own item number (VVARPF by VVVARE), NOBB number (VVARPF by VVNOBB), supplier item number (VVARPF by VVLVAR).
- v6.35: For logistics items, an SQL query on VVLEPF is used to find the item number from the logistics article number before performing VVARPF lookup.
- v6.36: If at least one packaging entry exists for the unit in JVPKPF (b_enhe_en=*on), the unit is accepted even if the exact item+unit+supplier combination is not found.
- v7.01: VA campaign (NOBB database) extended with price group dimension.

### JM110R — Campaign Price Detail Maintenance

- JM110R is a large program (79.5KB) handling item-level campaign prices stored in JKAVPF.
- v6.30: Extended with purchase date from/to.
- v6.31: Supplier number included in call to JW100R.
- v7.01: Cost price zero is allowed when switch u_701 is off (v7.02 fixed inverted flag).
- v6.36: Allows changing the price-giver (prisgiver) on an existing line.

---

## 7. Subprogram Calls Affecting Logic

| Caller | Callee | Purpose | Effect on Blocking |
|---|---|---|---|
| JM100R | VL711R | Load user's active price group | Pre-filters display; does not block |
| JM100R | RA530R (F4) | Price group inquiry | Display only |
| JM100R | JM110R (opt 7) | Navigate to campaign price maintenance | Passes firm+campaign+price group |
| JM111R | VP700R | Fetch recommended sale price | Campaign price comparison |
| JM111R | VP753R | Fetch purchase/cost price | Cost display and margin computation |
| JM110R | JM111R | Status check for item+unit in JKAVPF | Returns p_stat; drives validation indicator |
| JM110R | JW100R | NOBB item data | Fetch packaging and unit details |

---

## 8. Reference Tables

| Table | Description | Key Fields |
|---|---|---|
| JKAHPF | Campaign header | JMFIRM, JMKAMP, JMPRGR |
| JKAVPF | Campaign prices (item level) | JMVFIR, JMVKAM, JMVPRG, JMVVAR, JMVENHE |
| RA30PF | Price group register | REFIRM, REDKOD |
| VVARPF | Item master | VVFIRM, VVVARE |
| JVARPF | NOBB item register | JVFIRM, JVVARE |
| JVPKPF | NOBB packaging register | JWFIRM, JWVARE, JWLDOR, JWENHE |
| VVENPF | Item unit register | VEFIRM, VEVARE, VEENHE |
| FUSRPF | User register (price group) | FBFIRM, FBUSER |
