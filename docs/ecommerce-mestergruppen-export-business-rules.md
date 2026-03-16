# eCommerce Mestergruppen Export – Business Rules

**Module prefix:** NM
**System:** ASOFAK
**Focus:** What blocks or prevents customer, item, or data-hub export to Mestergruppen's webshop and Blueridge data hub

---

## Prerequisites / Master Data Requirements

- **NM700R – Customer export:** The webshop configuration record must exist in `ANBLPF` (logical view `ANBLL3`, key functional group + firm). If not found → `p_stat = 'E'`; entire export aborted.
- **NM700R:** The webshop owner record must exist in `ANBHPF` (logical view `ANBHL1`), chained by the `aoleie` (owner/tenant) field from the `ANBLL3` record. If not found → `p_stat = 'E'`; export aborted.
- **NM700R – Individual customers:** Only customers with `rkprti > 0` (professional-portal access flag, sourced from `RKBPPF`) AND a non-blank email address (`rkemal <> *blank`) are included in the export loop. Customers without a pro-portal access entry in `RKBPPF` default to `rkprti = 0` and are excluded.
- **NM710R – Item export:** The webshop configuration record in `ANBLL3` must exist; missing record blocks the entire export (`p_stat = 'E'`). Items are selected from a master assortment (Mestergruppen-specific, using `VSDTPF`); only items belonging to the designated master assortment are candidates for export.
- **NM710R – Item groups:** For each item, the full product-group hierarchy must be resolvable — overgroup in `VOGRPF`, main group in `VHGRPF`, sub-group in `VUGRPF`. Missing any level excludes the item with a warning.
- **NM720R – Blueridge datahub export:** This program exports item and supplier data to the Blueridge datahub (`BRITST`/`BRSOST` target files). No ANBLL3 check is required; the program writes directly to DB2 target tables. Items flagged as deleted (expiry date passed with no open orders or purchase orders) are exported with a delete marker rather than being omitted.

---

## Validation Rules

### Customer Export (NM700R)

| Condition | Effect |
|-----------|--------|
| `ANBLL3` record not found | `p_stat = 'E'`; entire customer export aborted |
| `ANBHL1` owner record not found | `p_stat = 'E'`; entire customer export aborted |
| `rkprti = 0` (customer has no pro-portal access, from `RKBPPF`) | Customer excluded from export loop entirely |
| `rkemal = *blank` (no email address) AND `rkprti > 0` | Customer skipped; warning written: `'<name> mangler E-post adresse, og utelates'`; `p_stat = 'R'` |
| `%subst(rksted:6:25) = *blank` (no postal address) | Customer skipped; warning written: `'<name> mangler postadresse, og utelates'`; `p_stat = 'R'` |
| `rkftnr = 0` (missing organisation/VAT number) | Customer exported as private customer; warning written: `'<name> mangler org-nr. Kunden opprettes som privatkunde.'`; `p_stat = 'R'` |

### Item Export (NM710R)

| Condition | Effect |
|-----------|--------|
| `ANBLL3` record not found | `p_stat = 'E'`; entire item export aborted |
| Item not in master assortment (`VSDTPF`) | Item not selected for export |
| Item type not `'L'` or `'S'` in any warehouse | Item excluded |
| Expiry date set AND no stock on hand | Item excluded |
| `vvtek1` starts with `'*'` | Item excluded (internal/blocked marker) |
| Overgroup missing in `VOGRPF` | Item excluded with warning |
| Main group missing in `VHGRPF` | Item excluded with warning |
| Sub-group missing in `VUGRPF` | Item excluded with warning |

### Blueridge Datahub Export (NM720R)

| Condition | Effect |
|-----------|--------|
| Item type changed from `'L'`/`'S'` to other (warehouse type change) | Delete marker sent for item (version 8.12) |
| Item expiry date passed AND no open orders AND no open purchase orders | Delete marker sent for item (version 8.10) |
| PK1 warehouse items | Excluded from export (version 8.02) |
| Warehouse-local supplier exists | Overrides the main item supplier for that warehouse's export row (version 8.05) |

---

## Configuration and Authorization Rules

- The firm number is read from the Local Data Area (LDA positions 944–946, field `l_firm`). All file reads are scoped to that firm.
- The functional group (`l_fgr`, LDA positions 931–933) is the first key field for `ANBLL3` lookup. Different functional groups within a firm can have separate Mestergruppen webshop configurations.
- **NM700R differs from NN700R** in the customer access model: while NN700R filters on `rkebiz = 1` (business customer flag), NM700R uses the `RKBPPF` professional-portal register. Only customers with a record in `RKBPPF` having `rkprti > 0` are included. This is a Mestergruppen-specific access control layer.
- **NM710R** selects items exclusively from a master assortment, not from the full item register. This constrains the Mestergruppen export to a curated product list, unlike the standard NN710R which reads all stock items.
- **NM720R** uses a newer free-format RPG style (`H DftActgrp(*No) Actgrp(*NEW)`) and writes to DB2 tables (`BRITST` for items, `BRSOST` for suppliers) rather than flat files.

---

## Financial / Transactional Rules

- NM700R exports customer bonus data from `EKBTPF` (logical view `EKBTIR`, keyed by firm + customer). The bonus balance (`d_cusbsal`) and bonus date (`d_cusbdat`) are summed across all bonus records for the customer. If none exist both fields default to zero.
- NM700R exports `d_cuscre = 0` unconditionally — credit limits are not propagated via this export.
- NM720R (version 8.01+) calculates prices as of the next day (`p_dato + 1`), not the current date. This ensures forward-looking pricing is exported to the Blueridge hub.
- NM720R (version 8.02) extended the price group field from 1 to 2 positions. Price group codes longer than 1 character are now correctly included in the export.
- NM720R exports `valutakode` (currency code) and `landkode` (country code) for each item/supplier combination (version 8.04).

---

## Status and Lifecycle Rules

- **Item active status (NM710R):** The active status flag in the exported record is set to `'N'` (inactive) when:
  - `vvnuda <= today` (item's "new until" date has passed)
  - `vvudat <= today` (item's expiry date has passed)
  - Warehouse-specific expiry date `<= today`
  Items default to active unless one of these date conditions triggers.
- **Delete markers (NM720R):** Rather than omitting discontinued items, NM720R sends a delete marker record to the Blueridge hub when an item's expiry date has passed and there are no open sales orders or purchase orders. This ensures the hub's item register stays synchronised.
- **Customer project export (NM700R):** Customer projects (CPR segment) are included only when the project end date (`fltdat`) is on or after the current run date. Expired projects are excluded. Projects with `flkpro > 900000` are flagged as diverse projects (`d_cprdiv = 'D'`).
- **NOBB numbers (NM710R, version 6.32):** One export line is written per unique NOBB number associated with an item. Items with multiple NOBB numbers in `CSODIR` generate multiple export rows.
- **GTIN (NM710R, version 9.01):** The GTIN (barcode) is retrieved via a new alphanumeric GTIN program rather than the legacy numeric field, allowing 13-digit barcodes with leading zeros to be exported correctly.

---

## Special Conditions

- **NM700R vs NN700R:** NM700R is a Mestergruppen-specific extension of NN700R. The key difference is the customer filter: NM700R requires pro-portal access (`rkprti > 0` from `RKBPPF`) instead of the generic business-customer flag (`rkebiz = 1`). All other export logic (EDIFACT format, project export, contact person override, bonus data) is identical.
- **Contact person email override (NM700R):** If a contact person in `RUKPPF` has role `'EHANDEL'`, that person's email address (`rumail`) overrides the customer's own email (`rkemal`) in the CUS export segment. This ensures the webshop login email is the eCommerce contact's address, not the billing address.
- **Organisation ID propagation (version 9.02):** Both CUS (`d_cusorgid`) and CPR (`d_cprorgid`) segments include the webshop owner identifier (`aoheie` from `ANBHPF`), tying all exported entities to the correct Mestergruppen webshop tenant.
- **NM710R trelast assortment (version 6.30/6.31):** Extended with a timber (trelast) assortment selection — items in the timber assortment are included in the export through an additional assortment code path.
- **NM720R Blueridge architecture:** This program writes directly to DB2 tables, making it structurally different from NM700R/NM710R which write to flat file export structures. The delete-marker logic (versions 8.10 and 8.12) is specific to the hub's expected synchronisation protocol.

---

## Subprogram Calls Affecting Logic

| Caller | Called Program | Purpose | Blocking Effect |
|--------|---------------|---------|-----------------|
| NM700R | (RKBPPF chain) | Check pro-portal access per customer | `rkprti = 0` → customer excluded |
| NM710R | (VOGRPF/VHGRPF/VUGRPF chains) | Group hierarchy validation | Missing group → item excluded |
| NM710R | GTIN program (9.01) | Alphanumeric GTIN retrieval | — |
| NM700R | (EKBTIR read) | Bonus balance aggregation | No bonus records → fields default to zero |
| NM700R | (CKGRPF/CKKLPF chain) | Customer category / chain category | Missing category → fields left blank, not blocking |

---

## Reference Tables

| Table | Logical View | Key Fields | Usage |
|-------|-------------|-----------|-------|
| ANBLPF | ANBLL3 | functional group + firm | Webshop configuration; missing record blocks NM700R and NM710R |
| ANBHPF | ANBHL1 | owner (aoleie) | Webshop owner; missing record blocks NM700R |
| RKBPPF | RKBPLR | firm + customer | Pro-portal access register; `rkprti` controls customer inclusion in NM700R |
| RKUNPF | RKUNL1 | firm | Customer master; sequential read in NM700R |
| RUKPPF | RUKPL6 | firm + customer | Contact persons; role `'EHANDEL'` overrides customer email |
| FKPRPF | FKPRL1 | firm + customer | Customer projects; read within customer loop in NM700R |
| FKMAPF | FKMALR | firm + customer + project | Cadastral/property info for customer projects |
| EKBTPF | EKBTIR | firm + customer | Bonus balance and date |
| APOSPF | — | postal code | Postal place name lookup for customer projects |
| VVARPF | VVARL1 | firm | Item master; item type, expiry, text fields for NM710R |
| VSDTPF | CSODI2 | firm + assortment | Master assortment detail; item selection source for NM710R |
| VOGRPF | VOGRL1 / VOGRLR | firm + overgroup | Item overgroup; missing → item excluded in NM710R |
| VHGRPF | VHGRL1 / VHGRLR | firm + maingroup | Item main group; missing → item excluded in NM710R |
| VUGRPF | VUGRL1 | firm + subgroup | Item subgroup; missing → item excluded in NM710R |
| VPKOPF | VPKOL1 | firm + item | Package information for item export |
| VVENPF | VVENL1/VVENL2 | firm + item + unit | Unit/conversion data for item export |
| CKGRPF | CKGRL2 | firm + customer category | Customer category to chain-category mapping |
| CKKLPF | CKKLLR | firm + chain category | Chain category description |
