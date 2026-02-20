# NexStep ERP — JV Item and Product Maintenance: Business Rules

---

## 1. Introduction

The JV module is the central item and product registry (vareregister) for NexStep ERP. It stores and maintains every product sold or distributed through the system: physical dimensions, supplier relationships, pricing, packaging, classification, NOBB numbers, environmental data, and lifecycle status. All other modules — order processing, purchasing, logistics, general ledger — consume item data from this registry.

### Purpose of the JV Programs

| Program | Norwegian Description | English Function |
|---|---|---|
| JV100R | Vare, vedlikehold | Item list maintenance — search and selection |
| JV101R | Vedlikehold av vareregister ny/endring | Item master create and edit |
| JV113R | Alternativt varenr, vedlikehold | Alternative item numbers maintenance |
| JV114R | Avgifter vedlikehold | Tax/fee codes maintenance |
| JV115R | NRF koder vedlikehold | NRF code maintenance |
| JV116R | Oppsalg/Mersalg (BM info) | Up-sell and cross-sell maintenance |
| JV117R | Spørring — Vare består av | Bill of materials component query |
| JV118R | Vare–Leverandør spørring | Item–supplier relationship query |
| JV119R | Vare–Leverandør spørring II | Item–supplier relationship query (dual key) |
| JV120R | Overføring til EVR — parametre | New item transfer to EVR — parameter screen |
| JV125R | Prisoppdatering — parametre | Price and item update — parameter screen |
| JV130R | Vareinformasjon oppdatering — parametre | Item information update to EVR — parameters |
| JV140R | EVR statuskoder overføring | EVR status codes transfer (shell) |
| JV145R | Va info, kalkulering av kostpris | Item info with cost price calculation |
| JV150R | Endre priskode — parametre | Price code change — parameter screen |
| JV151R | Endre priskode — oppdatering | Price code change — batch update |
| JV155R | Endre varetype — parametre | Item type change — parameter screen |
| JV160R | Vare, registrering fra ordre | Item search for order line entry |
| JV165R | Vare, registrering fra ordre — kalkulering | Order item registration with price calculation |
| JV170R | Vare, spørring/kalkulering fra EVR | Item inquiry from EVR |
| JV171R | Vare, spørring/kalkulering ordrelinje | Item inquiry for order line |
| JV180R | Vedlikehold — slettede varer | Maintenance of items deleted in external register |
| JV190R | Egenskaper, spørring | Item properties/characteristics inquiry |

### Field Prefix Conventions

All field definitions originate in the reference file JFRF.MBR. Tables and their field prefixes:

| Table | Prefix | Norwegian Name | English Name |
|---|---|---|---|
| JVARPF | JV | Vare | Item master |
| JVDTPF | JVD | Vare-detalj | Item detail per company |
| JLEVPF | JL | Leverandør | Supplier master |
| JW packaging table | JW | Vare-pakning | Item packaging |
| JX pricing table | JX | Vare-pris | Item price |
| JVL supplier link table | JVL | Vare–Leverandør | Item–supplier link |
| JVB alt numbers table | JVB | Vare alternative varenr | Alternative item numbers |
| JVN NRF data table | JVN | Vare NRF | NRF external data |
| JVO cross/up-sell table | JVO | Vare Oppsalg/Mersalg | Up-sell / cross-sell |
| JVFFPF file group release | JVF | Vare-frigitt pr. filgruppe | Item release by file group |
| JZ outer packaging table | JZ | Forpakning | Outer packaging / unit hierarchy |

### Norwegian Terminology

| Norwegian | English |
|---|---|
| Vare | Item / Product |
| Varenummer | Item number |
| Nobb / Nobbnr | NOBB number — Norwegian Building Products Database identifier |
| Leverandør | Supplier |
| Pakning | Packaging / package unit |
| Prisenhet | Price unit |
| Forpakning | Outer packaging |
| Overgruppe | Overgroup (top level of product classification) |
| Hovedgruppe | Main group (second level) |
| Undergruppe / Subgruppe | Subgroup (third level) |
| Sortimentskode | Assortment code |
| Utgår dato | Discontinuation / obsolete date |
| Aktiv dato | Active / availability date |
| Varetekst | Item description text |
| Produsent | Producer / manufacturer |
| Enhet | Unit of measure |

---

## 2. Database Schema Overview

### JVARPF — Item Master (prefix JV)

Key fields: JVARPF.JVVARE (item number, primary key). Accessed via logical files keyed by item number (JVARL1), description text (JVARL2), NOBB number (JVARL3), supplier item number (JVARL4/JVARL8), and product groups.

| Field | Type | Size | Norwegian Name | English Description |
|---|---|---|---|---|
| JVVARE | A | 15 | Varenr | Item number (primary key) |
| JVOGRP | S | 2,0 | Overgruppe | Product overgroup |
| JVHGRP | S | 2,0 | Hovedgruppe | Product main group |
| JVUGRP | S | 3,0 | Undergruppe | Product subgroup |
| JVEOGR | S | 2,0 | Egen overgruppe | Own overgroup (company override) |
| JVEHGR | S | 2,0 | Egen hovedgruppe | Own main group (company override) |
| JVEUGR | S | 3,0 | Egen undergruppe | Own subgroup (company override) |
| JVTEK1 | A | 35 | Varetekst 1 | Item description line 1 |
| JVTEK2 | A | 35 | Varetekst 2 | Item description line 2 |
| JVETE1 | A | 35 | Egen varetekst 1 | Own description line 1 (local override) |
| JVETE2 | A | 35 | Egen varetekst 2 | Own description line 2 (local override) |
| JVLTXT | A | 100 | Lang varetekst | Long item text |
| JVFTXT | A | 256 | Fritekst | Free text / long notes |
| JVNOBB | S | 8,0 | Nobbnr | NOBB number (external product ID) |
| JVMODN | S | 8,0 | Modulnr | Module number |
| JVEAN1 | S | 14,0 | EAN-nr | EAN/GTIN barcode number |
| JVSTRE | A | 15 | Andre strekkoder | Other barcode / item codes |
| JVPROD | S | 6,0 | Produsent | Producer number (links to JPROPF) |
| JVPVAR | A | 15 | Prod. varenr | Producer's own item number |
| JVLDOR | S | 6,0 | Leverandør | Primary supplier number (links to JLEVPF) |
| JVLVAR | A | 20 | Lev. varenr | Supplier item number |
| JVSLDO | S | 6,0 | Sortimentsleverandør | Assortment supplier |
| JVSORT | A | 10 | Sortimentskode | Assortment code |
| JVPENH | A | 3 | Prisenhet | Price unit of measure |
| JVSENH | A | 3 | Statistikkenhet | Statistics unit of measure |
| JVTYPE | A | 2 | Nobb varetype | NOBB item type (ST/SS/SP/DS) |
| JVLEVE | S | 1,0 | Leveringstype | Delivery type (0=Normal, 1=On Order) |
| JVSTAT | S | 1,0 | Varestatus | Item status (0=Manual, 1=Machine, 2=New, 4=Deleted) |
| JVADAT | L | — | Aktiv dato | Active date — item visible from this date |
| JVUDAT | L | — | Utgår dato | Discontinuation date — item blocked from new orders |
| JVUARB | S | 1,0 | Under arbeid | Under construction flag |
| JVLFOR | S | 1,0 | Lagerført | In stock / stockholding flag |
| JVHDAT | S | 1,0 | Holdbarhetsdato | Expiry date tracking flag |
| JVTFRO | S | 1,0 | Tåler frost | Frost tolerant flag |
| JVHPSE | S | 1,0 | Har PSE | PSE marking required flag |
| JVDOKI | A | 20 | Dokumentasjonsindikator | Documentation indicator (8 document type flags) |
| JVUTKO | A | 1 | Utstillingskode | Display/exhibition code |
| JVKATE | S | 3,0 | Kategoriansvarlig | Category responsible number (links to JKATPF) |
| JVGSTS | S | 1,0 | Godkjenningsstatus | Approval status |
| JVBRAN | A | 35 | Brand | Brand name |
| JVMMK1 | A | 5 | Miljømerke 1 | Environmental mark 1 |
| JVMLK1 | A | 3 | Miljø landkode 1 | Environmental country code 1 |
| JVMMK2 | A | 5 | Miljømerke 2 | Environmental mark 2 |
| JVMLK2 | A | 3 | Miljø landkode 2 | Environmental country code 2 |
| JVUNNR | A | 11 | UN-nr | UN hazardous materials number |
| JVFANR | A | 11 | Farenr | Hazard identification number |
| JVFABS | A | 50 | Farebeskrivelse | Hazard description text |
| JVEMBM | A | 50 | Embalasjemerking | Packaging marking text |
| JVEMGR | A | 30 | Emballasjegruppe | Packaging group classification |
| JVTONO | A | 15 | Tollkode norsk | Norwegian customs tariff code |
| JVTOEU | A | 15 | Tollkode EU | EU customs tariff code |
| JVENOB | S | 8,0 | Erstatt. av nobb | Replacement NOBB number |
| JVNODA | L | — | NOBB opprettelsesdato | NOBB creation date |
| JVNBED | L | — | NOBB endringsdato | NOBB last change date |
| JVNSDA | L | — | NOBB slettet dato | NOBB deleted date |
| JVODAT | L | — | Opprettet dato | Record creation date |
| JVEDAT | L | — | Endringsdato | Record last change date |
| JVETIM | T | — | Endringstidspunkt | Record last change time |
| JVEUSR | A | 10 | Endret av | User who last changed the record |

### JVDTPF — Item Detail per Company (prefix JVD)

Key fields: JVDTPF.JVDFIR + JVDTPF.JVDVAR. One row per company per item. Stores company-specific pricing behaviour for an item.

| Field | Type | Size | Norwegian Name | English Description |
|---|---|---|---|---|
| JVDFIR | S | 3,0 | Firma | Company number (part of key) |
| JVDVAR | A | 15 | Varenr | Item number (part of key) |
| JVDPKO | A | 1 | Priskode | Price code (blank=default, N=Normal, B=Buy-in) |
| JVDSKO | S | 1,0 | Skaffe-kode | Procurement code (0=Standard, 1=Special order) |
| JVDHKO | S | 1,0 | Holde-kode | Supplier hold code (0=Normal, 1=Held back) |
| JVDNOD | L | — | NOBB opprettelsesdato | NOBB creation date (propagated from item master) |
| JVDNED | L | — | NOBB endringsdato | NOBB change date |
| JVDNSD | L | — | NOBB slettet dato | NOBB deleted date |

### JLEVPF — Supplier Master (prefix JL)

Key field: JLEVPF.JLLDOR (supplier number). Accessed via logical file JLEVL1.

| Field | Type | Size | Norwegian Name | English Description |
|---|---|---|---|---|
| JLLDOR | S | 6,0 | Leverandørnr | Supplier number (primary key) |
| JLNAVN | A | 35 | Navn | Supplier name |
| JLADR1 | A | 35 | Adresse 1 | Address line 1 |
| JLADR2 | A | 35 | Adresse 2 | Address line 2 |
| JLPONR | A | 15 | Postnr | Postal code |
| JLSTED | A | 35 | Poststed | City |
| JLTLFN | A | 15 | Telefon | Phone number |
| JLMOBN | A | 15 | Mobil | Mobile number |
| JLFAXN | A | 15 | Fax | Fax number |
| JLFAXO | A | 15 | Fax ordre | Fax for orders |
| JLMAIL | A | 50 | E-mail | Email address |
| JLURLA | A | 50 | URL | Web address |
| JLORGN | S | 9,0 | Organisasjonsnr | Norwegian organization number |
| JLEANL | S | 14,0 | EAN-lokasjonsnr | EAN GLN location number |
| JLVALU | A | 3 | Valuta | Currency code |
| JLLAND | A | 20 | Land | Country |
| JLDTYP | S | 1,0 | Deltakertype | Participant type / delivery type code |
| JLKVAV | S | 1,0 | Vareavtale | Item agreement active flag (1=yes) |
| JLKMOD | S | 1,0 | Modulkode | Module code |
| JLKKAL | S | 1,0 | Kalkylekode | Calculation method code |
| JLFKAL | S | 1,0 | Fraktkode | Freight code |
| JLGRPK | S | 1,0 | Grønt punkt | Green dot environmental compliance |
| JLMMRK | A | 10 | Miljømerke | Environmental certification mark |
| JLBGIR | S | 11,0 | Bankgiro | Bank giro number |
| JLPGIR | S | 11,0 | Postgiro | Post giro number |
| JLENUM | S | 6,0 | Eget lev.nr | Supplier's own internal number |
| JLRSTS | S | 1,0 | Reg. status | Registration status (0=Active, 1=Inactive) |
| JLPRIO | S | 1,0 | Prioritert lev. | Priority / preferred supplier flag (1=yes) |
| JLNODA | L | — | NOBB opprettelsesdato | NOBB creation date |
| JLNEDA | L | — | NOBB endringsdato | NOBB change date |
| JLNSDA | L | — | NOBB slettet dato | NOBB deleted date |
| JLODAT | L | — | Opprettet dato | Record creation date |
| JLEDAT | L | — | Endringsdato | Record last change date |
| JLETIM | T | — | Endringstidspunkt | Record last change time |
| JLEUSR | A | 10 | Endret av | User who last changed |

> Contact person fields JLKPRS, JLKTLF, JLKMOB, JLKFAX, JLKMAI (all deprecated since v6.11) have been replaced by a separate contact person table (JKP).

### JW Packaging Table (prefix JW)

Key fields: JW table keyed by item number + supplier + unit code.

| Field | Type | Size | Norwegian Name | English Description |
|---|---|---|---|---|
| JWVARE | A | 15 | Varenr | Item number |
| JWLDOR | S | 6,0 | Leverandør | Supplier number |
| JWPAKN | A | 3 | Enhetskode | Packaging unit code |
| JWENHE | A | 3 | Enhet | Unit of measure |
| JWOFAK | S | 14,6 | Omr.fakt. volum | Volume conversion factor |
| JWOFAP | S | 12,6 | Omr.fakt. pris | Price conversion factor |
| JWBRED | S | 6,0 | Bredde mm | Gross width in millimetres |
| JWLENG | S | 6,0 | Lengde mm | Gross length in millimetres |
| JWHOYD | S | 6,0 | Høyde mm | Gross height in millimetres |
| JWVEKT | S | 9,0 | Vekt gram | Gross weight in grams |
| JWVOLU | S | 9,5 | Volum m3 | Gross volume in cubic metres |
| JWNBRE | S | 6,0 | Netto bredde mm | Net width in millimetres |
| JWNLEN | S | 6,0 | Netto lengde mm | Net length in millimetres |
| JWNHOY | S | 6,0 | Netto høyde mm | Net height in millimetres |
| JWNVEK | S | 9,0 | Netto vekt gram | Net weight in grams |
| JWNVOL | S | 9,5 | Netto volum m3 | Net volume in cubic metres |
| JWGTIN | S | 14,0 | GTIN-nr | GTIN barcode number |
| JWGTYP | A | 6 | GTIN-type | GTIN type code |
| JWANTP | S | 14,6 | Antall i pakning | Quantity per package |
| JWPKLA | A | 1 | Pakningsklasse | Packaging class |
| JWPAKS | A | 3 | Pakning sammensatt | Assembled / composite packaging code |
| JWPAPP | S | 6,0 | Pall pakn/lag | Pallet — packages per layer |
| JWPMSB | S | 6,0 | Pall maks stbl.vekt | Pallet — maximum stacking weight |
| JWBEST | S | 1,0 | Bestillbar | Orderable flag (1=yes) |
| JWLFOR | S | 1,0 | Lagerført | Stockholding flag |
| JWPSEN | S | 1,0 | PSE | PSE marking flag |
| JWEPSE | S | 1,0 | Er prisenhet | Is price unit flag |
| JWRSTS | S | 1,0 | Reg. status | Registration status |
| JWFDAT | L | — | Fra dato | Valid from date |
| JWTDAT | L | — | Til dato | Valid to date |
| JWNODA | L | — | NOBB opprettelsesdato | NOBB creation date |
| JWNEDA | L | — | NOBB endringsdato | NOBB change date |
| JWNSDA | L | — | NOBB slettet dato | NOBB deleted date |
| JWODAT | L | — | Opprettet dato | Record creation date |
| JWEDAT | L | — | Endringsdato | Record last change date |
| JWETIM | T | — | Endringstidspunkt | Record last change time |
| JWEUSR | A | 10 | Endret av | User who last changed |

### JX Pricing Table (prefix JX)

Key fields: item number + supplier (+ optional supplier item number).

| Field | Type | Size | Norwegian Name | English Description |
|---|---|---|---|---|
| JXVARE | A | 15 | Varenr | Item number |
| JXLDOR | S | 6,0 | Leverandør | Supplier number |
| JXLVAR | A | 20 | Lev. varenr | Supplier item number |
| JXPRIS | S | 9,2 | Pris | Supplier price |
| JXGPRI | S | 9,2 | Gammel pris | Previous price |
| JXGDAT | L | — | Gyldig fra dato | Price valid from date |
| JXTDAT | L | — | Gyldig til dato | Price valid to date |
| JXVALU | A | 3 | Valuta | Currency code |
| JXFBEL | S | 9,2 | Fraktbeløp | Freight amount |
| JXCRDO | S | 1,0 | Cross docking | Cross-docking flag (1=yes) |
| JXEANN | S | 14,0 | EAN foreløpig | Preliminary EAN number |
| JXRSTS | S | 1,0 | Reg. status | Registration status |
| JXNODA | L | — | NOBB opprettelsesdato | NOBB creation date |
| JXNEDA | L | — | NOBB endringsdato | NOBB change date |
| JXNSDA | L | — | NOBB slettet dato | NOBB deleted date |
| JXODAT | L | — | Opprettet dato | Record creation date |
| JXEDAT | L | — | Endringsdato | Record last change date |
| JXETIM | T | — | Endringstidspunkt | Record last change time |
| JXEUSR | A | 10 | Endret av | User who last changed |

### JVL Item–Supplier Link Table (prefix JVL)

Key fields: JVLVNR (item number) + JVLLDO (supplier).

| Field | Type | Size | Norwegian Name | English Description |
|---|---|---|---|---|
| JVLVNR | A | 15 | Varenr | Item number |
| JVLLDO | S | 6,0 | Leverandør | Supplier number |
| JVLVEI | S | 1,0 | Er vareeier | Item owner flag (1=this supplier owns the item) |
| JVLLVA | A | 20 | Lev. varenr | Supplier's item number |
| JVLMR1 | A | 15 | Merking 1 | Supplier marking / label 1 |
| JVLMR2 | A | 15 | Merking 2 | Supplier marking / label 2 |
| JVLMR3 | A | 15 | Supplier marking 3 | Supplier marking / label 3 |
| JVLUDA | L | — | Utgår dato | Supplier discontinued date for this item |
| JVLODA | L | — | Opprettet dato | Record creation date |
| JVLOTI | T | — | Opprettet tidspunkt | Record creation time |
| JVLOUS | A | 10 | Opprettet av | User who created the record |
| JVLEDA | L | — | Endringsdato | Record last change date |
| JVLETI | T | — | Endringstidspunkt | Record last change time |
| JVLEUS | A | 10 | Endret av | User who last changed |

---

## 3. Item Master Prerequisites

Before an item can be created or saved in JV101R, the following master data must exist in the system.

### Product Group Hierarchy (VUGRPF)

**What is checked:** When an overgroup (JVARPF.JVOGRP), main group (JVARPF.JVHGRP), or subgroup (JVARPF.JVUGRP) is entered, JV101R chains to the VUGRPF lookup (via VUGRL1 logical file) to confirm the combination is valid.

**Why:** The three-level product hierarchy controls item classification, pricing surcharges, assortment management, and reporting. An invalid group code would orphan the item from all group-based processes.

**What happens:** If any of the three group levels is not found in VUGRPF, indicator *in34 is set and the screen redisplays with an error. The group description text (c2utxt) is also populated from this lookup — if the text is blank, the screen re-presents even if the validation technically passed.

### Unit of Measure (VENHPF)

**What is checked:** The price unit (JVARPF.JVPENH) and statistics unit (JVARPF.JVSENH) must exist in the unit master VENHPF (accessed via VENHL1). This is validated in JV101R screen C3.

**Why:** Price calculations, packaging conversion factors, and EDI messages all reference the unit code. An unknown unit would cause pricing failures downstream.

**What happens:** If the unit is not found, an error indicator is set and the user must correct the entry before the record can be saved.

### Category Responsible (JKATPF)

**What is checked:** If a category responsible code is entered (JVARPF.JVKATE), JV101R chains to JKATPF (via JKATL1) to retrieve the category area name.

**Why:** Category managers are responsible for product assortment decisions. The link ensures only valid category codes are assigned.

**What happens:** If the category code exists, the category name (c2knav) is displayed next to the field. The validation is informational — the name is shown but the code itself is not hard-blocked if lookup fails in all versions.

### Supplier Must Exist (JLEVPF)

**What is checked:** The primary supplier number (JVARPF.JVLDOR) is mandatory (non-zero) and must be found in JLEVPF via JLEVL1.

**Why:** Every item in the NexStep registry must have a responsible supplier. Supplier attributes (currency, freight codes, calculation methods) drive all procurement and pricing calculations for the item.

**What happens:** If JVARPF.JVLDOR is zero, indicator *in41 is set. If the supplier number is non-zero but not found in JLEVPF, indicator *in42 is set. In both cases the screen redisplays with an error and the record is not saved.

### NOBB Registration Requirements

**What is checked:** If a NOBB number (JVARPF.JVNOBB) is entered, it must be a valid 8-digit number (>= 10,000,000 — i.e., exactly 8 digits). JV101R also checks that no other active item already carries the same NOBB number (JVARPF.JVVARE ≠ c2vare and JVARPF.JVUDAT = *LOVAL, meaning not discontinued).

**Why:** NOBB is the Norwegian Building Products Database primary key. Duplicate NOBB assignments would corrupt all NOBB-driven data synchronisation and price downloads.

**What happens:** If the NOBB number is below 10,000,000, indicator *in31 is set. If a duplicate active item is found with the same NOBB, indicator *in32 is set. Both conditions redisplay the screen with an error.

### User Permission Controls (AVALPF)

**What is checked:** JV101R reads the field-level write-protection table AVALPF (via AVALL1) for the program JV101R. If a field has protection code AVABES=1, the corresponding screen field is made read-only.

**Why:** Different users or user groups have different authorisation levels. Field-level protection prevents unauthorised changes to NOBB numbers, group classifications, supplier assignments, and environmental marks.

**What happens:** Protected fields are displayed with indicator *in60–*in74 set, making them non-enterable on the screen. The user can view but not change protected fields.

### Item Number Format (JV100R / JV101R)

**What is checked:** JV100R manages the item list. When creating a new item (valg=3 on JV100R launching JV101R), the item number is initially blank. A new item number is generated automatically by incrementing the counter JAAVSI in the status file JSTSPF.

**Why:** Item numbers are the primary key across all JV module tables. Auto-generation ensures uniqueness. Items in the 90000000–99999999 range are treated as "9000-series" supplier-specific items with different supplier-link handling.

**What happens:** On saving a new item, JV101R reads and increments JAAVSI in JSTSPF (via JSTSLUR), then writes the item master record. If the item number supplied on screen already exists in JVARPF and it is active (JVARPF.JVUDAT = *LOVAL), indicator *in43 is set for the duplicate supplier item number check.

---

## 4. Item Master Validation Rules

All validation is performed in JV101R subroutine sjekk_input before any database write.

### Item Number (JVARPF.JVVARE)

**What is checked:** For new item creation, the item number is generated from the auto-counter. When copying an existing item (valg=3), the item number field on screen is cleared and a new number is assigned. For supplier item number (JVARPF.JVLVAR) uniqueness, JV101R chains JVARL8 keyed by supplier (JVLDOR) + supplier item number (JVLVAR). If a match is found where JVARPF.JVVARE ≠ the current item and JVARPF.JVUDAT = *LOVAL (not discontinued), the combination is a duplicate.

**Why:** Two active items cannot have the same supplier item number for the same supplier, because the supplier item number is used to identify items in EDI messages and price lists.

**What happens:** Indicator *in43 is set and the screen redisplays with an error. The record is not written.

### Description (JVARPF.JVTEK1 / JVARPF.JVTEK2)

**What is checked:** JVARPF.JVTEK1 (primary description line 1) must not be blank. If the second alternative text line (JVARPF.JVETE2) is filled, then the first alternative text line (JVARPF.JVETE1) must also be filled — a second line cannot exist without a first.

**Why:** Item descriptions are used in all customer-facing documents: orders, invoices, labels, and EDI messages. A blank primary description would produce blank line items in all outputs.

**What happens:** If JVTEK1 is blank, indicator *in37 is set. If JVETE2 is filled but JVETE1 is blank, indicator *in39 is set. Both prevent saving and redisplay the screen with an error.

### NOBB Item Type (JVARPF.JVTYPE)

The NOBB item type defines how this item is handled in the NOBB product database:

| Code | Norwegian | English Description |
|---|---|---|
| ST | Standard | Standard product — direct NOBB listing |
| SS | Sammensatt | Assembled product — composed of other items |
| SP | Spesialvare | Special product — non-standard NOBB entry |
| DS | Display | Display unit — retail display configuration |

JV101R sets the description text (c2vttx) based on JVTYPE. If JVTYPE is blank or unrecognised the description is left blank.

### Item Status (JVARPF.JVSTAT)

| Value | Norwegian | English Description |
|---|---|---|
| 0 | Manuell | Manually registered item |
| 1 | Maskin | Machine-processed (from NOBB sync or EDI) |
| 2 | Ny | New item (awaiting activation) |
| 4 | Slettet | Deleted item |

**What is checked:** JV100R sets indicator *in80 when JVARPF.JVSTAT = 4. Deleted items are shown with the marker "** SLETTET **" appended to the item text in the subfile list.

**Why:** Deleted items must remain in the database for history and audit, but must be visually distinguished to prevent re-use.

**What happens:** Deleted items are displayed with a visual marker. The item remains selectable for inquiry but is normally excluded from new ordering and purchasing processes.

### Active Date (JVARPF.JVADAT)

**What is checked:** If an active date is entered on screen (c2adat ≠ 0), JV101R validates it as a valid date in DMY (day-month-year) format.

**Why:** The active date controls when an item becomes visible and orderable. An invalid date would either block a valid item indefinitely or make an item available prematurely.

**What happens:** If the date is not a valid DMY date, indicator *in54 is set and the screen redisplays with an error.

### Discontinuation Date (JVARPF.JVUDAT)

**What is checked:** If a discontinuation date is entered (c2udat ≠ 0), JV101R validates it as a valid DMY date. In item search programs (JV160R, JV170R), items where JVARPF.JVUDAT is not *LOVAL and is in the past are excluded from order entry search results.

**Why:** Discontinued items must not appear on new orders. The date field enables soft deletion — the item history is preserved while the item is excluded from new business.

**What happens:** Invalid date format → indicator *in55, screen redisplays. Valid date → item becomes invisible in order search once the date passes.

### Under-Construction Flag (JVARPF.JVUARB)

**What is checked:** When JVARPF.JVUARB = 1, the item is flagged as under construction. JV101R uses field-level protection (AVALPF) to restrict editing of designated fields on items in this state.

**Why:** Items being set up by one user or received from NOBB should not be changed by other users until the setup is complete.

**What happens:** Protected fields are displayed read-only. The flag is cleared by the responsible user when setup is finalised.

### Alternative Item Numbers (JVB Table, via JV113R)

**What is checked:** JV113R validates that each alternative item number entry has a type code of 'T' (TUN — Trade Unit Number) or 'F' (FINFO — product information reference). The alternative number field (JVBVAR) cannot be blank. Duplicate records (same type + same number) are blocked by a CHAIN check against JVALLR before inserting.

**Why:** Alternative item numbers (TUN codes, FINFO identifiers) are used for EDI barcode scanning and product information exchange with trading partners. Duplicates would cause ambiguous lookups.

**What happens:** Invalid type or blank number → error indicator set, screen redisplays. Duplicate found → error indicator set, insert blocked.

---

## 5. Item Classification and Grouping Rules

### Three-Level Product Hierarchy

Items are classified using a three-level group structure maintained in VUGRPF:

| Level | Field | Size | Norwegian | Description |
|---|---|---|---|---|
| 1 | JVARPF.JVOGRP | 2S 0 | Overgruppe | Top-level product category |
| 2 | JVARPF.JVHGRP | 2S 0 | Hovedgruppe | Main product group within overgroup |
| 3 | JVARPF.JVUGRP | 3S 0 | Undergruppe | Subgroup within main group |

**Hierarchy enforcement (JV101R, JV120R, JV125R, JV130R):**

- If JVOGRP = 0 and JVHGRP ≠ 0, the main group cannot exist without an overgroup — indicator *in31 set.
- If JVHGRP = 0 and JVUGRP ≠ 0, the subgroup cannot exist without a main group — indicator *in32 set.
- Each supplied level is validated against VUGRPF before acceptance.

**Why:** The three-level hierarchy is used in pricing surcharges (JCPRGR markup table), assortment management, reporting, and item transfer to EVR. An orphaned group assignment produces incorrect surcharge calculations and missing report lines.

**What happens:** Invalid or incomplete hierarchy → error indicator set, screen redisplays. All three levels must form a valid chain before the record is saved.

### Own-Group Overrides

| Field | Size | Norwegian | Description |
|---|---|---|---|
| JVARPF.JVEOGR | 2S 0 | Egen overgruppe | Company-specific overgroup override |
| JVARPF.JVEHGR | 2S 0 | Egen hovedgruppe | Company-specific main group override |
| JVARPF.JVEUGR | 3S 0 | Egen undergruppe | Company-specific subgroup override |

**What is checked:** These own-group fields follow the same hierarchy enforcement as the primary groups. If JVEOGR is zero and JVEHGR is non-zero, an error is raised. If JVEHGR is set, JVEOGR must also be set. JV101R validates the own-group combination against VUGRPF separately (indicator *in36 on failure).

**Why:** Some companies in a multi-company installation classify items differently from the NOBB/national classification. The own-group fields allow a local override without altering the master NOBB classification.

**What happens:** Own-group override description (c2eutx) is populated from the lookup. If the text is blank after validation, the screen re-presents.

### Assortment Code and Assortment Supplier

| Field | Description |
|---|---|
| JVARPF.JVSORT | Assortment code — indicates which product assortment the item belongs to |
| JVARPF.JVSLDO | Assortment supplier — the supplier responsible for this item's assortment |

**What is checked:** Both are optional fields. JV120R and JV125R accept an assortment sort code and validate it against JSORL1 if provided.

**Why:** Assortment codes group items for bulk transfer operations, pricing updates, and EDI exchange with specific trading partners.

### Delivery Type (JVARPF.JVLEVE)

| Value | Norwegian | English |
|---|---|---|
| 0 | Normal | Normal delivery from stock |
| 1 | Bestillingsvare | Delivered on order only |

**What is checked:** JV101R screen displays the delivery type as an editable field. JV120R validates it against the delivery type table (VLTPL1) via VY500R inquiry.

**Why:** On-order items cannot be picked from stock and require special order handling. Procurement processes check this flag to determine lead time and sourcing method.

**What happens:** If delivery type 1 is set, the item will not be committed from warehouse stock on orders — a purchase order must be raised.

### Category Responsible (JVARPF.JVKATE)

**What is checked:** The category responsible code links the item to a category manager registered in JKATPF. JV101R populates the category name (c2knav) by chaining JKATL1 on JVKATE. The field is optional.

**Why:** Category managers in the building products industry negotiate supplier agreements and are responsible for item data quality for their assigned categories.

**What happens:** If JVKATE is set and JKATPF has a matching entry, the category name is displayed alongside the code. If not found, the name field remains blank but the code is still saved.

### Documentation Indicator (JVARPF.JVDOKI)

JVARPF.JVDOKI is a 20-character field where each character position represents a specific document type flag. Eight document type positions are defined, each set to '1' (present) or '0' / blank (absent). This controls which documentation types exist for the item (for example: data sheet, installation guide, safety data sheet, CE declaration).

**Why:** Document type flags drive the document management module, controlling which documents must be registered before an item can be marked as fully approved.

---

## 6. NOBB and External Database Integration

### NOBB Number (JVARPF.JVNOBB)

**What is checked:** NOBB number must be exactly 8 digits (≥ 10,000,000) if provided. It must be unique across all active items (items where JVARPF.JVUDAT is not set).

**Why:** NOBB (Norsk Byggevaredatabase) is the national database for Norwegian building products. The NOBB number is the primary key used for automated price synchronisation, product data exchange, and EDI transactions with suppliers and buying groups.

**What happens:** On save, JV101R writes JVNOBB to the item master. On NOBB synchronisation runs, JVARPF.JVSTAT is set to 1 (machine-processed) and JVNODA/JVNBED are updated with the NOBB creation and change dates received from the external feed.

### NOBB Dates

| Field | Description |
|---|---|
| JVARPF.JVNODA | Date this item was first registered in the NOBB database |
| JVARPF.JVNBED | Date the NOBB record was last changed externally |
| JVARPF.JVNSDA | Date the NOBB record was deleted from the external database |

These dates are written during automated NOBB data import. They are displayed in JV101R for information and are read-only in normal maintenance. The same date pattern is propagated to JVDTPF (JVDNOD/JVDNED/JVDNSD) and JLEVPF (JLNODA/JLNEDA/JLNSDA) to track when the corresponding supplier or company-detail records were last synchronised.

### Replacement NOBB (JVARPF.JVENOB)

**What is checked:** If JVARPF.JVNSDA is set (item deleted in NOBB) and a replacement NOBB number is available, it is stored in JVARPF.JVENOB.

**Why:** When a NOBB product is discontinued, the database typically specifies a successor product. Storing the replacement NOBB allows the system to guide users to the correct successor item.

**What happens:** JV101R displays JVENOB on screen. When the discontinuation date (JVUDAT) is set and JVENOB is populated, users searching for the old item can navigate directly to the replacement.

### NRF Data (JVN Table, via JV115R)

The JVN table stores NRF (Norsk Rørleggernes Forening — Norwegian Plumbers Association) classification data linked to the item's NOBB number.

| Field | Description |
|---|---|
| JVNNOB | NOBB number (links to JVARPF.JVNOBB) |
| JVNNRF | NRF number — the NRF system's product identifier |
| JVNSTS | Status code for the NRF record |
| JVNGRP | NRF group number |
| JVNLDO | NRF supplier identifier |
| JVNTX1/TX2/TX3 | NRF description text lines 1–3 |

**What is checked (JV115R):** The NRF number (JVNNRF) must be non-zero. Duplicate NRF number entries for the same item are blocked by a chain check on JVNRLR.

**Why:** Some product categories (particularly plumbing and HVAC) use NRF numbering in addition to NOBB for trade-specific procurement systems.

**What happens:** On save, JVNTS, JVNGRP, and the three text fields are written. Timestamps JVNODA/JVNEDA/JVNUSR are updated.

### Alternative Product Numbers (JVB Table, via JV113R)

The JVB table stores TUN (Trade Unit Number) and FINFO product identifiers linked to the NOBB number.

| Field | Description |
|---|---|
| JVBTYP | Type: 'T' = TUN (Trade Unit Number), 'F' = FINFO |
| JVBNOB | NOBB number |
| JVBVAR | Alternative item/product number |
| JVBTXT | Product identifier name / description |
| JVBNOD/JVBNED/JVBNSD | NOBB date tracking (creation/change/deleted) |

**What is checked (JV113R):** Type must be 'T' or 'F'. Number (JVBVAR) cannot be blank. No duplicate type + number combination allowed.

**Why:** TUN codes are GTIN-based identifiers used in retail barcode scanning and EDI. FINFO codes are used for product information exchange. Multiple codes per NOBB number are common in the building products industry.

### EAN/GTIN (JVARPF.JVEAN1)

**What is checked:** EAN number is a 14-digit numeric field on the item master. It is also stored on packaging records (JWGTIN) and preliminary on pricing records (JXEANN).

**Why:** EAN/GTIN is the global trade item number used for barcode scanning at goods receipt, pick and pack, and point of sale. It must match the physical barcode on the product.

### Producer Number (JVARPF.JVPROD)

**What is checked:** If JVARPF.JVPROD is non-zero, JV101R chains JPROPF (via JPROL1) to retrieve the producer name. As of version 8.02, a missing producer record no longer blocks saving (the name field is simply left blank).

**Why:** Producer/manufacturer information is required for documentation, certification tracking, and compliance reporting.

### Module Number (JVARPF.JVMODN)

**What is checked:** The module number is an 8-digit numeric field. It links the item to a product module in the NOBB/BM (Byggevareprodusentenes Forening) module structure.

**Why:** Modules group related products for bulk ordering, assortment management, and module-based price downloads from the NOBB database.

---

## 7. Item Pricing Rules

### JX Pricing Table

Each item can have one or more price records in the JX pricing table, one per supplier. The price record holds the supplier's purchase price, valid dates, currency, and freight amount.

**What is checked (JX100R, called from JV101R):**

- JXPRIS (supplier price): the published purchase price from the supplier.
- JXGDAT (valid from date) and JXTDAT (valid to date): define the period during which this price is active. In JV145R, a price is considered active only if the lookup date falls between JXGDAT and JXTDAT.
- JXVALU (currency): the currency in which the supplier quotes the price.
- JXFBEL (freight amount): any freight surcharge included in the supplier's price terms.
- JXCRDO (cross-docking flag): if set to 1, the item is delivered directly from the supplier to the customer without passing through the warehouse.

**Why:** Accurate purchase prices are the foundation of all cost price and sales price calculations. Validity dates ensure that price lists are applied for the correct period, especially during price transitions.

**What happens:** When a pricing calculation is requested (JV145R, JV165R, JV171R), the system retrieves the price record where the current date falls within JXGDAT–JXTDAT. If the price has expired (JXTDAT is set and is earlier than the lookup date), no price is returned and the calculation is based on alternative methods.

### Price Unit (JVARPF.JVPENH)

**What is checked:** The price unit defines the unit in which the supplier price is quoted (e.g., piece, metre, kilogram). It is validated against VENHPF on save. Packaging records (JW) store a price conversion factor (JWOFAP) which converts between the price unit and the packaging unit.

**Why:** A supplier may quote a price per kilogram while the item is sold per piece. The price unit and conversion factor together enable the system to calculate the correct price for each unit of measure.

### Price Code per Company (JVDTPF.JVDPKO)

| Value | Norwegian | English |
|---|---|---|
| blank | Standard | Default pricing behaviour |
| N | Normal | Normal purchase price channel |
| B | Betingelse (kjøp) | Conditions-based buy-in price |

**What is checked:** JV101R validates JVDPKO against the allowed values (blank, 'N', 'B'). Invalid codes set indicator *in48 and block the save.

**Why:** The price code tells the purchasing module which pricing channel to use when generating purchase orders for this item at this company. 'B' (buy-in) triggers condition-based pricing via the JP conditions table.

### Procurement Code (JVDTPF.JVDSKO)

| Value | Norwegian | English |
|---|---|---|
| 0 | Standard | Standard stock item |
| 1 | Spesialbestilling | Special order / non-stock item |

**What is checked:** JV101R validates JVDSKO as 0 or 1. Invalid values set indicator *in49.

**Why:** The procurement code tells the purchasing module how to source the item. Special-order items are not replenished automatically — a purchase order must be manually created against a sales order demand.

### Hold Code (JVDTPF.JVDHKO)

| Value | English |
|---|---|
| 0 | Normal — no hold |
| 1 | Held back from this company's operations |

**What is checked:** On creation of a new item detail record, JV101R defaults JVDHKO = 1 (held). This means a newly registered item starts in a held state.

**Why:** New items should not be immediately available for ordering until they have been reviewed and activated by the product manager.

**What happens:** The hold code is cleared (set to 0) when the product manager confirms the item is ready for sale.

### Price Code Change (JV150R / JV151R)

JV150R provides a parameter screen allowing a user to change the price code (JVDTPF.JVDPKO) for a range of items in one operation, filtered by supplier, module number, or product group hierarchy.

**Validation in JV150R:**
- Price code must be blank, 'N', or 'B'.
- Either supplier (b1ldor) or module (b1modn) must be provided — not both zero.
- If supplier is given, it must exist in JLEVPF.
- Group hierarchy rules apply: subgroup requires main group; main group requires overgroup.

JV151R executes the actual update, iterating through items linked to the supplier (via JVLEL3) or to the module (via JVARLG), and updating JVDTPF.JVDPKO for each matching item.

### Price Update Parameters (JV125R)

JV125R is the parameter screen for bulk price updates. It governs which items receive updated prices from supplier price lists.

**Key validation rules:**
- Update type (b1otyp) is mandatory.
- The price change effective date must be a valid date and cannot be earlier than today.
- The price application date, if different from the change date, must also be valid and not in the past.
- Either a supplier or a pricing supplier must be specified.
- A supplier with status 'P' (passive) or 'S' (suspended) cannot be used for bulk updates.
- The user cannot simultaneously select "skip internal price update", "skip cost price update", and "skip sales price update" — at least one price type must be updated.
- The user cannot combine "no correspondence" with "skip sales price update" in the same run.

**Date logic:** If only a change date is provided and it is in the future, the price application date defaults to the change date. Otherwise the price application date defaults to today.

---

## 8. Item Packaging Rules

### JW Packaging Table

Each item can have multiple packaging records — one per packaging unit code (JWPAKN). Packaging records describe the physical dimensions and logistical properties of each unit of packaging for the item.

### Orderable Flag (JWBEST)

**What is checked:** JWBEST = 1 means this packaging unit can be ordered from the supplier and can appear on purchase orders. JWBEST = 0 means the packaging configuration is for reference only.

**Why:** An item may be physically available in multiple configurations (single piece, inner pack, outer case, pallet), but only certain units are valid for purchasing.

### Dimensions and Weight

| Field | Unit | Description |
|---|---|---|
| JWBRED | mm | Gross width |
| JWLENG | mm | Gross length |
| JWHOYD | mm | Gross height |
| JWVEKT | g | Gross weight in grams |
| JWVOLU | m3 | Gross volume in cubic metres |
| JWNBRE | mm | Net width |
| JWNLEN | mm | Net length |
| JWNHOY | mm | Net height |
| JWNVEK | g | Net weight in grams |
| JWNVOL | m3 | Net volume in cubic metres |

**Why:** Gross dimensions are used for freight cost calculation and pallet planning. Net dimensions are used for product data exchange with retail systems and customer product catalogues.

### GTIN (JWGTIN)

**What is checked:** JWGTIN is a 14-digit number. Each packaging unit (each JWPAKN) can have its own GTIN. JWGTYP describes the GTIN type.

**Why:** Different packaging levels carry different GTIN numbers (item GTIN, inner pack GTIN, case GTIN). Correct GTIN assignment is required for retail scanning and EDI ASN (advance shipping notice) transactions.

### Quantity per Package (JWANTP)

**What is checked:** JWANTP (14,6 decimal) holds the quantity of base units contained in this packaging level. It is used as the volume conversion factor between the packaging unit and the base unit.

**Why:** When a customer orders 5 boxes and each box contains 10 pieces, the system must know JWANTP = 10 to calculate warehouse picks in pieces.

### Pallet Information

| Field | Description |
|---|---|
| JWPAPP | Number of packages per pallet layer |
| JWPMSB | Maximum pallet stacking weight in grams |

**Why:** Pallet configuration data is used for logistics planning, warehouse slotting, and transport cost estimation.

### PSE Flag (JWPSEN)

**What is checked:** JWPSEN = 1 on a packaging record indicates that this packaging unit requires PSE (Particular Sensitive Equipment) marking. This ties to the item-level JVARPF.JVHPSE flag.

**Why:** PSE items require specific handling, labelling, and documentation under Norwegian product safety regulations.

### Packaging Class (JWPKLA)

The packaging class code categorises the type of packaging (e.g., consumer unit, inner pack, transport unit) for logistics and EDI purposes.

### Outer Packaging (JZ Table)

The JZ table defines the standard packaging hierarchy for groups of items, specifying which sales units (JZENH1/JZENH2/JZENH3), purchase units (JZENI1/JZENI2/JZENI3), and price book units (JZENP1/JZENP2/JZENP3) apply to items within a given overgroup/main group/subgroup/supplier/module combination.

**Why:** Rather than manually setting unit configurations for every item, the JZ outer packaging defaults are applied to new items based on their group and supplier, reducing data entry effort and ensuring consistency.

### Conversion Factors (JWOFAK / JWOFAP)

| Field | Description |
|---|---|
| JWOFAK | Volume conversion factor — converts packaging unit to base volume unit |
| JWOFAP | Price conversion factor — converts packaging unit to price unit |

**Why:** Pricing is often quoted per price unit (e.g., per 1000 pieces or per kilogram). Ordering is in packaging units (e.g., boxes). The price conversion factor maps from the packaging unit to the price unit so that order line prices are calculated correctly.

---

## 9. Item Supplier Rules

### JVL Item–Supplier Link Table

One row per supplier per item. A single item can have multiple supplier relationships — different suppliers, each with their own item number, markings, and discontinued status.

### Product Owner Flag (JVLVEI)

**What is checked:** JVLVEI = 1 marks this supplier as the item owner — the primary source of truth for this item's product data. Only one supplier per item should be the item owner.

**Why:** NOBB and product data synchronisation processes update item master data only from the item owner's feed. Having a clear item owner prevents data conflicts when multiple suppliers carry the same product.

### Supplier Item Number (JVLLVA)

**What is checked:** The supplier's own item number is stored in JVLLVA (20 characters). For 9000-series items (JVARPF.JVVARE ≥ '90000000'), JV101R writes JVLLVA to the JVLEL1 supplier link record when saving. For other items, the supplier item number is stored directly on the item master as JVARPF.JVLVAR.

**Why:** Supplier item numbers are needed for EDI purchase orders and for resolving incoming supplier price lists to internal item numbers.

### Supplier Discontinuation Date (JVLUDA)

**What is checked:** When a supplier marks an item as discontinued in their catalogue, JVLUDA is set on that supplier's link record. JV100R checks JVLELR (logical file on JVL) and sets flag b_akti when a discontinued date exists for the active supplier.

**Why:** If the primary supplier has discontinued the item, purchasing must be redirected to an alternative supplier or the item must be obsoleted. The item search programs (JV160R, JV170R) use JVLUDA to filter out supplier links where the supplier has discontinued the item.

**What happens:** In JV100R and JV180R, items where all supplier links have a passed JVLUDA are flagged as effectively obsolete and can be highlighted in the subfile (shown in red in v7.37+).

### Supplier Markings (JVLMR1 / JVLMR2 / JVLMR3)

Each supplier link can carry three free-text marking codes (15 characters each). These are used for supplier-specific product classification codes or category references that the supplier assigns to the item in their own system.

### Supplier Media (JVM Table)

The JVM table (Vare Leverandør Media) stores URLs and media type codes for supplier-provided product images, data sheets, and technical documentation. Each JVL supplier link can have multiple media records.

**Why:** Rich product content (images, spec sheets) is required for e-commerce channels and customer product catalogues. Storing supplier media links against the item–supplier relationship ensures the correct supplier's content is displayed.

### Primary Supplier on Item Master (JVARPF.JVLDOR / JVARPF.JVLVAR)

The item master stores the primary (default) supplier number and supplier item number directly on JVARPF. These are the supplier and number used when no other context specifies a preference.

**What is checked (JV101R):** JVLDOR must be non-zero (mandatory) and must exist in JLEVPF. JVLVAR, combined with JVLDOR, must be unique across active items.

### Supplier Validation for Bulk Operations (JV120R, JV125R, JV130R)

When transferring items to EVR or running bulk price updates, the selected supplier is validated by calling program RS760R:

- Supplier status 'P' (passive) → operation blocked.
- Supplier status 'S' (suspended) → operation blocked.

**Why:** Items belonging to passive or suspended suppliers should not be transferred or price-updated, as the commercial relationship with those suppliers is not active.

---

## 10. Environmental and Compliance Rules

### Environmental Marks (JVARPF.JVMMK1 / JVMMK2)

Each item can carry up to two environmental certification marks, each with an associated country code:

| Field | Description |
|---|---|
| JVARPF.JVMMK1 | Environmental mark 1 code (5A) — e.g., Svanen (Nordic Swan), FSC |
| JVARPF.JVMLK1 | Country code for mark 1 (3A) |
| JVARPF.JVMMK2 | Environmental mark 2 code (5A) |
| JVARPF.JVMLK2 | Country code for mark 2 (3A) |

**Why:** Norwegian building regulations and public procurement requirements demand declared environmental certifications. These fields are reported on product data sheets and exported to NOBB.

### Hazardous Goods (JVARPF.JVUNNR / JVFANR / JVFABS)

| Field | Description |
|---|---|
| JVARPF.JVUNNR | UN number (11A) — international hazardous substance classification number |
| JVARPF.JVFANR | Hazard identification number (11A) — Kemler code |
| JVARPF.JVFABS | Hazard description (50A) — plain-text description of the hazard |

**Why:** Items classified as hazardous materials require special transport documentation, safety data sheets (SDS), and warehouse handling procedures. The UN number triggers additional handling in the logistics module and is required on shipping documents.

### Packaging Marking (JVARPF.JVEMBM / JVEMGR)

| Field | Description |
|---|---|
| JVARPF.JVEMBM | Packaging marking text (50A) — required labels on the package |
| JVARPF.JVEMGR | Packaging group classification (30A) |

**Why:** Certain product categories require specific packaging markings under Norwegian product safety law (for example, child-warning symbols, recycling codes, or CE marking text). These fields store the required marking text for label printing.

### PSE Marking Flag (JVARPF.JVHPSE)

**What is checked:** JVARPF.JVHPSE = 1 indicates the item must carry a PSE (Particular Sensitive Equipment) mark. The JW packaging table (JWPSEN) also carries this flag at the packaging unit level.

**Why:** PSE items require specific handling instructions and documentation under Norwegian electrical and safety product regulations.

### Frost Tolerance (JVARPF.JVTFRO)

**What is checked:** JVTFRO = 1 means the item tolerates frost (can be stored and transported below 0 °C). JVTFRO = 0 means the item must not be frozen.

**Why:** Frost-sensitive items must be flagged so that warehouse temperature zones and transport arrangements respect the storage requirement. Ignoring this leads to product spoilage or damage.

### Expiry Date Tracking (JVARPF.JVHDAT)

**What is checked:** JVHDAT = 1 indicates that individual batch/lot expiry dates must be tracked for this item in the warehouse.

**Why:** Items with limited shelf life (adhesives, sealants, paints, food contact materials) require batch expiry tracking to prevent dispatch of expired stock and to support recall procedures.

### Norwegian Tariff Code (JVARPF.JVTONO) and EU Tariff Code (JVARPF.JVTOEU)

| Field | Description |
|---|---|
| JVARPF.JVTONO | Norwegian customs tariff code (15A) — TARIC for import/export declarations |
| JVARPF.JVTOEU | EU customs tariff code (15A) |

**Why:** Customs declarations for cross-border shipments require tariff codes. Norwegian companies importing from outside the EEA and exporting to non-EEA markets must include the correct TARIC code on customs documents.

### Approval Status (JVARPF.JVGSTS)

**What is checked:** JVGSTS is set during item setup to indicate the item's approval state. It is written from the screen and displayed in JV101R.

**Why:** Items may require internal or supplier approval before they can be sold. The approval status prevents premature activation of items that have not completed the product introduction process.

### Brand Name (JVARPF.JVBRAN)

**What is checked:** JVBRAN (35A) stores the commercial brand name of the product, which may differ from the manufacturer's technical product name.

**Why:** Retail and commercial channels often classify products by brand. The brand field is used in product data exports to e-commerce platforms and customer catalogues.

---

## 11. Item Status and Lifecycle Rules

### Status Lifecycle (JVARPF.JVSTAT)

Items progress through a defined lifecycle:

| Status | Trigger | Effect |
|---|---|---|
| 2 — New | Item created manually or received from NOBB | Item is visible in maintenance but not yet activated for ordering |
| 0 — Manual | Set by user when item is approved manually | Available for normal ordering |
| 1 — Machine | Set by NOBB synchronisation process | Machine-maintained; some fields may be write-protected via AVALPF |
| 4 — Deleted | User sets deletion or item removed from NOBB | Item remains in database; shown as "** SLETTET **" in list; excluded from order searches |

**What happens when deleted:** JV100R displays the "** SLETTET **" marker. Item search programs (JV160R, JV170R) exclude items with JVSTAT = 4. The item record is not physically removed from JVARPF.

### Active Date (JVARPF.JVADAT)

**What is checked:** Item must have a valid JVADAT to be visible in order entry. The active date represents the date from which the item is commercially available.

**Why:** Product launches are planned for specific dates. Setting JVADAT in the future allows the product to be set up in the system before it becomes orderable, without it appearing prematurely on order entry screens.

**What happens:** Before JVADAT, the item is not returned in order entry item searches (JV160R, JV170R). After JVADAT, it becomes visible and orderable.

### Discontinuation Date (JVARPF.JVUDAT)

**What is checked:** Once JVUDAT is set to a past or current date, the item is treated as discontinued.

**Why:** Discontinued items must not appear on new orders but must remain available for inquiry (open orders, invoice history, warranty claims).

**What happens:** Order entry searches (JV160R, JV170R) exclude items where JVUDAT is not *LOVAL and has passed. Existing orders referencing the item can still be processed. The item can still be viewed in JV100R.

### Under-Construction Flag (JVARPF.JVUARB)

**What is checked:** JVUARB = 1 restricts editing of designated fields. JV101R enforces this through AVALPF field-level protection.

**Why:** During product setup, multiple users may be involved. The under-construction flag locks the item against concurrent changes until setup is complete.

**What happens:** Fields are displayed read-only. The product manager clears JVUARB when setup is finalised, releasing the item for normal maintenance.

### In-Stock Flag (JVARPF.JVLFOR)

**What is checked:** JVLFOR = 1 means the item is a stockholding line — it is stocked in the warehouse. JVLFOR = 0 means the item is not normally held in stock.

**Why:** Stockholding classification affects replenishment planning, warehouse slotting, and minimum/maximum stock calculations. Non-stock items are sourced on demand.

### Product Release per File Group (JVFFPF)

The JVFFPF table records which file groups (JVFFGR) an item has been released to:

| Field | Description |
|---|---|
| JVFFGR | File group code (3A) — identifies the target distribution channel or system |
| JVFVAR | Item number |
| JVFOVF | Transfer flag (1S 0) — set to 1 when item has been transferred to scanner register |

**What is checked:** Before an item can be used in a specific channel (e.g., a retail scanner system), it must be released to the corresponding file group. JV120R manages the initial release to EVR.

**Why:** Items may be valid in NexStep but not yet ready for all channels. The file group release provides a controlled activation gate per distribution channel.

### Product Linkage (JVK / JVQ Tables)

The JVK table (header) and JVQ table (detail) store logistics item linkages, connecting a logistics item number to its component items:

| Table | Description |
|---|---|
| JVK | Logistics item link header — keyed by logistics item number |
| JVQ | Logistics item link detail — component items within the logistics bundle |

**Why:** Some products are sold as a logistics unit (e.g., a pallet of mixed items) that maps to multiple individual item numbers. The JVK/JVQ structure allows the system to resolve logistics orders into individual pick lines.

JV100R displays the logistics item number (w_jvqvnr) and quantity (w_jvquno) when browsing linked items.

### Cross-Sell and Up-Sell Links (JVO Table, via JV116R)

The JVO table stores commercial relationship data between items:

| Field | Description |
|---|---|
| JVONOB | NOBB number of the source item |
| JVOTYP | Type: 'O' = Oppsalg (up-sell), 'M' = Mersalg (cross-sell) |
| JVOTX1 | Description text for the recommended product (100A) |

**What is checked (JV116R):** Two records per item are maintained — one type 'O' and one type 'M'. The text field is split for display across two lines on screen. Timestamps (JVOODA/JVOETI/JVOEUS) are updated on every change.

**Why:** Up-sell and cross-sell recommendations drive sales intelligence features. Displaying a higher-specification alternative (up-sell) or a complementary accessory (cross-sell) at order entry increases order value.

### Maintenance of Deleted Items (JV180R)

JV180R handles items that have been deleted in an external register (VA/EVR) but still exist in JVARPF:

**What is checked:**
- Items in the external VA register (VVARL1) are compared against JVL supplier link records for the selected supplier.
- If all supplier links for an item have a passed JVLUDA (supplier discontinued date), the item is shown as fully obsolete.
- The user can select items to mark as deleted by setting JVUDAT (discontinuation date) on the item master.

**What happens:** On confirmation, JVARPF.JVUDAT is set to the expired date for the selected items. Warehouse register records (VLAGPF) are also updated with the expiration date (v6.21+).

### Item Properties (JVARPF via JV190R)

JV190R maintains item property records (EGENSKAPER) in the JVEGST table, accessed via JVEGI1 logical file:

- Properties are keyed by NOBB number and a property identifier (WGUI — web GUI identifier).
- Each property has a description of up to 256 characters.
- Full CRUD operations: create, view, edit, copy, delete.

**Why:** Item properties store structured attribute data (size ranges, material grades, colour options, compatibility codes) needed for product configurators and parametric search in e-commerce and ERP.

---

## 12. Field Reference

### JVARPF — Item Master Complete Field List

All fields from JFRF.MBR. Type: A=Character, S=Packed numeric (S size,decimal), L=ISO Date, T=Time.

| Field | Type | Size | Norwegian Name | English Description |
|---|---|---|---|---|
| JVVARE | A | 15 | Varenr | Item number — primary key |
| JVOGRP | S | 2,0 | Overgruppe | Product overgroup (level 1) |
| JVHGRP | S | 2,0 | Hovedgruppe | Product main group (level 2) |
| JVUGRP | S | 3,0 | Undergruppe | Product subgroup (level 3) |
| JVEOGR | S | 2,0 | Egen overgruppe | Own overgroup — company override of level 1 |
| JVEHGR | S | 2,0 | Egen hovedgruppe | Own main group — company override of level 2 |
| JVEUGR | S | 3,0 | Egen undergruppe | Own subgroup — company override of level 3 |
| JVTEK1 | A | 35 | Varetekst 1 | Primary item description (mandatory) |
| JVTEK2 | A | 35 | Varetekst 2 | Secondary item description |
| JVETE1 | A | 35 | Egen varetekst 1 | Own description line 1 — local text override |
| JVETE2 | A | 35 | Egen varetekst 2 | Own description line 2 — requires JVETE1 |
| JVLTXT | A | 100 | Lang varetekst | Long item text |
| JVFTXT | A | 256 | Fritekst | Free text / extended notes |
| JVNOBB | S | 8,0 | Nobbnr | NOBB number — 8-digit external product ID (≥ 10000000) |
| JVMODN | S | 8,0 | Modulnr | Module number |
| JVEAN1 | S | 14,0 | EAN-nr | EAN/GTIN barcode number (14 digits) |
| JVSTRE | A | 15 | Andre strekkoder | Other barcode / item codes |
| JVPROD | S | 6,0 | Produsent | Producer / manufacturer number (→ JPROPF) |
| JVPVAR | A | 15 | Prod. varenr | Producer's own item number |
| JVLDOR | S | 6,0 | Leverandør | Primary supplier number — mandatory (→ JLEVPF) |
| JVLVAR | A | 20 | Lev. varenr | Primary supplier item number |
| JVSLDO | S | 6,0 | Sortimentsleverandør | Assortment supplier number |
| JVSORT | A | 10 | Sortimentskode | Assortment code |
| JVPENH | A | 3 | Prisenhet | Price unit of measure (→ VENHPF) |
| JVSENH | A | 3 | Statistikkenhet | Statistics unit of measure |
| JVTYPE | A | 2 | Nobb varetype | NOBB item type: ST=Standard, SS=Assembled, SP=Special, DS=Display |
| JVLEVE | S | 1,0 | Leveringstype | Delivery type: 0=Normal, 1=On Order |
| JVSTAT | S | 1,0 | Varestatus | Item status: 0=Manual, 1=Machine, 2=New, 4=Deleted |
| JVADAT | L | — | Aktiv dato | Active from date — item not visible before this date |
| JVUDAT | L | — | Utgår dato | Discontinuation date — blocks new orders after this date |
| JVUARB | S | 1,0 | Under arbeid | Under-construction flag: 1=locked for editing |
| JVLFOR | S | 1,0 | Lagerført | Stockholding flag: 1=stock item |
| JVHDAT | S | 1,0 | Holdbarhetsdato | Expiry date tracking: 1=track batch expiry |
| JVTFRO | S | 1,0 | Tåler frost | Frost tolerance: 1=frost tolerant, 0=frost sensitive |
| JVHPSE | S | 1,0 | Har PSE | PSE marking required: 1=yes |
| JVDOKI | A | 20 | Dokumentasjonsindikator | Documentation indicator — 8 document type flags |
| JVUTKO | A | 1 | Utstillingskode | Display/exhibition code |
| JVKATE | S | 3,0 | Kategoriansvarlig | Category responsible number (→ JKATPF) |
| JVGSTS | S | 1,0 | Godkjenningsstatus | Approval status |
| JVBRAN | A | 35 | Brand | Commercial brand name |
| JVMMK1 | A | 5 | Miljømerke 1 | Environmental certification mark 1 |
| JVMLK1 | A | 3 | Miljø landkode 1 | Country code for environmental mark 1 |
| JVMMK2 | A | 5 | Miljømerke 2 | Environmental certification mark 2 |
| JVMLK2 | A | 3 | Miljø landkode 2 | Country code for environmental mark 2 |
| JVUNNR | A | 11 | UN-nr | UN hazardous materials number |
| JVFANR | A | 11 | Farenr | Hazard identification number (Kemler code) |
| JVFABS | A | 50 | Farebeskrivelse | Hazard description |
| JVEMBM | A | 50 | Embalasjemerking | Packaging marking text |
| JVEMGR | A | 30 | Emballasjegruppe | Packaging group classification |
| JVTONO | A | 15 | Tollkode norsk | Norwegian customs tariff code |
| JVTOEU | A | 15 | Tollkode EU | EU customs tariff code |
| JVENOB | S | 8,0 | Erstatt. av Nobb | Replacement NOBB number for this item |
| JVNODA | L | — | NOBB opprettelsesdato | Date item was created in NOBB database |
| JVNBED | L | — | NOBB endringsdato | Date item was last changed in NOBB database |
| JVNSDA | L | — | NOBB slettet dato | Date item was deleted in NOBB database |
| JVODAT | L | — | Opprettet dato | Date this record was created in NexStep |
| JVEDAT | L | — | Endringsdato | Date this record was last changed |
| JVETIM | T | — | Endringstidspunkt | Time this record was last changed |
| JVEUSR | A | 10 | Endret av | User ID who last changed this record |

### JVDTPF — Item Detail per Company Complete Field List

| Field | Type | Size | Norwegian Name | English Description |
|---|---|---|---|---|
| JVDFIR | S | 3,0 | Firma | Company number — part of primary key |
| JVDVAR | A | 15 | Varenr | Item number — part of primary key |
| JVDPKO | A | 1 | Priskode | Price code: blank=default, N=Normal, B=Buy-in |
| JVDSKO | S | 1,0 | Skaffe-kode | Procurement code: 0=Standard, 1=Special order |
| JVDHKO | S | 1,0 | Holde-kode | Supplier hold code: 0=Normal, 1=Held (default on create) |
| JVDNOD | L | — | NOBB opprettelsesdato | NOBB creation date (propagated from item master sync) |
| JVDNED | L | — | NOBB endringsdato | NOBB last change date |
| JVDNSD | L | — | NOBB slettet dato | NOBB deleted date |
