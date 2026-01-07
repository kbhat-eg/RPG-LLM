# RPG Program RB002R – Code Walkthrough and Explanation

## Overview

This ILE RPG source reads records from an input file (bovf - voucher file) and outputs customer and company information in a comma-separated (CSV-like) format, apparently for integration with an external business system (Visma Business). It also does lookups to find supplementary customer details and handles some logic around company/firm changes and formatting of address fields.

The program contains logic for output formatting, conditional field extraction, and some business-specific data cleansing, such as removing post codes from city/postal place fields based on external configuration ("switch").

### Header and Metadata

```rpg
h option(*nodebugio) datedit(*dmy)
h decedit('0.')
```
- Disables debug I/O.
- Sets date editing to DMY and decimal editing convention.

#### Change History Comments

The comments at the top record changes/versions and describe purpose:
- Reads lines from `bovf` (voucher file) and outputs comma-separated customer/business info.
- Removes post number from city field based on a configuration value.
- Various additions and cleanups over time.

---

## File Declarations

```rpg
fbovfl1    if   e           k disk
frkunl1    if   e           k disk    rename(rkunpfr:rkunl1r)
frb10l1    if   e           k disk    rename(rb10pfr:rb10l1r)
frv02pf    o    e           k disk
```
- **bovfl1:** Input file (likely vouchers).
- **rkunl1:** Input file, renamed; contains customer info.
- **rb10l1:** Input file, renamed; contains company-end info.
- **rv02pf:** Output file.

---

## Variable Declarations

```rpg
d                uds
d l_filg                931    933
d l_firm                944    946  0
d w_kund          s                   like(rkkund)
d w_biln          s                   like(bfbiln)
d w_firm          s                   like(bffirm)
d rkunl1_kund     s                   like(rkkund)
d tell            s              1  0
```
- **l_filg, l_firm:** Data area fields from the local data area; `l_firm` is the current firm number.
- **w_kund, w_biln, w_firm, rkunl1_kund:** Work variables for customer number, document number, firm, etc.
- **tell:** Counter, tracks if heading rows have been written.

### External Program Declarations

```rpg
dcl-s co402_verdi1  char(1);
dcl-s co402_verdi2  char(2048);
DCL-PR CO402R EXTPGM('CO402R');
  p_filgrp            char(3)     const;
  p_firm              packed(3:0) const;
  p_lager             packed(2:0) const;
  p_nokkel            char(50)    const;
  p_verdi1            char(1);
  p_verdi2            char(2048);
END-PR;
d u_post          s              1    inz(*off)
```
- Declares interface to external program `CO402R`, which is used to check if post number should be excluded from the post city.
- **u_post:** Switch for removing post number from city/postal place field.

---

## Main Logic

### Initial Setup

```rpg
eval w_firm = l_firm
```
- Copies the firm from LDA (local data area) to work variable.

### Main Record Reading Loop

```rpg
nyles: 
  read bovfl1 60
  if *in60 = '1'
    goto slutt
  endif
```
- Reads a record from `bovfl1`.
- If EOF (`*in60='1'`), jumps to end of processing.

#### Per Record: Firm Filtering

```rpg
if l_firm <> bffirm
  goto nyles
endif
```
- Only processes records where the firm from LDA matches the record’s firm field.

#### Heading/Setup for First Record

If `tell = 0` (meaning first time through for this firm), writes three heading records:

```rpg
eval rvdata = *blank
write rv02pfr

eval rvdata = %trim('@IMPORT_METHOD(1)')
write rv02pfr

eval rvdata = %trim('@ActoR(=CustNO,') +
                  ('Nm,Shrt,Ad1,Ad2,') +
                  ('PNo,PArea,Phone,') +
                  ('PrivPh,Fax,') +
                  ('InvoCust,FactNo)')
write rv02pfr

eval w_biln = bfbiln
```
- Blank line, import method, and field names.
- Stores first "bilagsnummer" (voucher number).

### Lookup and Write Address

```rpg
exsr finnadr
goto nyles
```
- Calls subroutine to fetch address and customer info and outputs formatted data.
- Loops back to read next record.

---

### End of Processing

```rpg
slutt:
eval rvdata = *blank
write rv02pfr

read rb10l1 90
eval rvdata = %trim('@FIRM_END(') +
               %trim(rafirb) +
               %trim(')')
write rv02pfr

eval *inlr = *on
```
- Finalizes processing: writes blank line, then a `@FIRM_END` line (reading company end-info from `rb10l1`), and ends the program.

---

## Subroutines

### `finnadr` – Find Customer Name, Address, etc.

```rpg
finnadr: begsr

if bfckto <> 0
  eval w_kund = bfckto
else
  eval w_kund = bfdkto
endif

eval rkunl1_kund = w_kund
chain rkunl1 90
```
- Chooses customer number (`w_kund`) from two possible fields; chains into customer file.

#### Special Logic for Stripping Postal Code

```rpg
if %found(rkunl1) and u_post = *off
   and %char(rkponr) = %subst(rksted:1:4)
  eval rksted = %subst(rksted:6:25)
endif
```
- If the postal code (rkponr) matches the beginning of the city field (rksted), and the switch is off, strips postal code from city. (The logic for u_post is switched via external program call.)

#### Output Customer Line

```rpg
if *in90 = *off
  eval rvdata = *blank
  eval rvdata = %trim('"') + 
                %char(w_kund) + '" "' +
                %trim(rknavn) + '" "' +
                %trim(rkalfa) + '" "' +
                %trim(rkgate) + '" "' +
                %trim(rkadr2) + '" "' +
                %char(rkponr) + '" "' +
                %trim(rksted) + '" "' +
                %trim(rkmobn) + '" "' +
                %trim(rktlfn) + '" "' +
                %trim(rktlfx) + '" "' +
                %char(rkfaku) + '" "' +
                %char(rkfacn) + '"'
  write rv02pfr
endif
endsr
```
- Outputs a CSV-style line with customer and address info.

### Subroutine: *INZSR (initialization)

```rpg
CallP CO402R(l_filg:w_firm:0:'ØKONOMI_EKSTERNT_POSTNR_POSTSTED':
             co402_verdi1:co402_verdi2);
if co402_verdi1 = '1';
  u_post = *on;
endif;
endsr
```
- Calls external program to check if the post code should be removed from city name; sets `u_post`.

---

## Key List for Customer Lookup

```rpg
rkunl1_key klist
  kfld w_firm
  kfld rkunl1_kund
```
- Key for customer file lookup: firm + customer number.

---

## In Summary

- **Reads** each voucher for the current firm.
- **On first record,** writes header lines.
- **For each record:**
  - Looks up corresponding customer data.
  - Optionally strips the postal code from the city field (depending on configuration).
  - Outputs a formatted, quoted, comma-separated line with customer and address info.
- **At end,** writes a trailer (`@FIRM_END`) with company number.

---

## Key Business Logic

- **Firm filtering:** process records for only the current firm.
- **Dynamic address formatting:** adjust city/postal place field based on system configuration.
- **Customer lookup:** retrieves enriched info for each voucher.
- **CSV-like output** for use by another system.

---

### Important Sections

- **Initialization (`*INZSR`):** Checks configuration for address formatting.
- **Record loop:** Reads, filters, writes.
- **`finnadr` subroutine:** Gathers and writes customer/address information per voucher.
- **End processing:** Writes finalization record.

---

This program is a batch export/report utility, tailored for an integration scenario, with flexible output formatting and system-configurable logic for sensitive formatting (postal code handling). The structure and naming reflect a typical Norwegian business context.