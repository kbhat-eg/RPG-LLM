```markdown
## Overview

This ILE RPG program, named **GETKIDD**, is designed to retrieve the "KID" (customer identification) number from various data sources for use in external economic systems. The program processes input parameters, searches for relevant KID data in database files, and outputs the found KID information. The comments and code structure indicate a focus on accounting/invoicing data, with Norwegian language conventions.

---

## Header Specifications

```rpg
h option(*nodebugio) datedit(*dmy)
h decedit('0.')
```
- `option(*nodebugio)`: Disables debug I/O.
- `datedit(*dmy)`: Sets date format to day/month/year.
- `decedit('0.')`: Sets decimal edit code (0 decimal separator).

---

## File Declarations

```rpg
fsohel1    if   e           k disk    rename(sohepfr:sohel1r)
```
- Declares a keyed database file `sohel1`, renamed from format `sohepfr` to record format `sohel1r`.

---

## Data Definitions

### Local Data Area

```rpg
d                uds
d l_firm                944    946  0
d l_user                911    920
```
- Data structure for accessing the Local Data Area (LDA).
- `l_firm`: 3-digit company/firm identification, LDA positions 944-946.
- `l_user`: User identification, LDA positions 911-920.

### Key Variables

```rpg
d sohel1_aarr     s                   like(soaarr)
d sohel1_binr     s                   like(sobinr)
```
- Variables for the `sohel1` file's key fields (likely year and voucher number).

### Working Variables

```rpg
d w_firm          s              3  0
d w_aarr          s              4  0
d w_kidr          s                   like(sokidr)
d p_biln          s                   like(sobinr)
d p_bdat          s               d   datfmt(*iso)
d p_kref          s                   like(sobinr)
d p_kidd          s                   like(sokidd)
d p_kida          s              9
```
- `w_firm`: Working variable for firm number.
- `w_aarr`: Working variable for year.
- `w_kidr`: Working variable for KID number.
- `p_biln`, `p_bdat`, `p_kref`, `p_kidd`, `p_kida`: Program parameters (voucher number, date, credit ref, KID found, edited KID).

---

## SQL Environment Setup

```rpg
c/Exec SQL
c+  Set Option DatFmt=*ISO, DlyPrp=*YES, SRTSEQ=*LANGIDUNQ, LANGID=NOR,
c+             commit=*none
c/End-Exec
```
- Sets SQL session options:
    - Date format: ISO.
    - Delay prepare: Yes.
    - Sort sequence: Unique language ID.
    - Language ID: Norwegian.
    - Commit control: None.

---

## Main Processing Logic

### Initialization

```rpg
c                   eval      p_kida = *blank
c                   eval      p_kidd = *blank
c                   eval      w_kidr = 0
```
- Clear output values.

---

### Step 1: Attempt to Retrieve KID from `brefpf` File

```rpg
c/Exec SQL
c+  select bxkidr
c+    into :p_kidd
c+  from brefpf
c+    where bxfirm = :w_firm and bxbiln = :p_biln
c/End-Exec
c                   if        p_kidd <> *blank
c                   goto      end_kid
c                   endif
```
- SQL SELECT attempts to fetch field `bxkidr` (KID) from `brefpf` where `bxfirm` and `bxbiln` match the working firm and provided voucher number.
- If a KID is found (`p_kidd` not blank), jump to the end.

---

### Step 2: Attempt to Retrieve KID from `sohel1` File Using Voucher Number

```rpg
c                   eval      w_aarr = %subdt(p_bdat:*years)
c                   eval      sohel1_aarr = w_aarr
c                   eval      sohel1_binr = p_biln

c     sohel1_key    setll     sohel1
c     sohel1_key    reade     sohel1
c                   dow       not %eof
c                   if        sokidd <> *blanks or
c                             sokidr <> 0
c                   eval      p_kidd = sokidd
c                   eval      w_kidr = sokidr
c                   goto      end_kid
c                   endif
c     sohel1_key    reade     sohel1
c                   enddo
```
- Sets up keys from input date and voucher number.
- Reads through matching records in `sohel1`.
- If either KID text (`sokidd`) or number (`sokidr`) is found, assigns to output and jumps to end.

---

### Step 3: Repeat Search Using Credit Reference

```rpg
c                   eval      w_aarr = %subdt(p_bdat:*years)
c                   eval      sohel1_aarr = w_aarr
c                   eval      sohel1_binr = p_kref

c     sohel1_key    setll     sohel1
c     sohel1_key    reade     sohel1
c                   dow       not %eof
c                   if        sokidd <> *blanks or
c                             sokidr <> 0
c                   eval      p_kidd = sokidd
c                   eval      w_kidr = sokidr
c                   goto      end_kid
c                   endif
c     sohel1_key    reade     sohel1
c                   enddo
```
- If previous searches failed, repeats the `sohel1` search using the credit reference (`p_kref`) instead of voucher number.

---

### Output Formatting

```rpg
c     end_kid       tag
c                   eval      p_kida = %EditC(w_kidr :'X')
```
- Formats the numeric KID (`w_kidr`) as a character string into `p_kida` with edit code 'X' (leading zero suppression).

---

### Program End

```rpg
c                   eval      *inlr = *on
c                   return
```
- Sets LR (last record) indicator to on, signaling program end and cleanup.
- Returns to caller.

---

## Subroutine: *INZSR (Initialization)

```rpg
c     *inzsr        begsr
...
c                   parm                    p_biln
c                   parm                    p_bdat
c                   parm                    p_kref
c                   parm                    p_kidd
c                   parm                    p_kida
...
c     sohel1_key    klist
c                   kfld                    w_firm
c                   kfld                    sohel1_aarr
c                   kfld                    sohel1_binr
...
c                   eval      w_firm = l_firm
...
c                   endsr
```
- *INZSR is run at program start.
- Receives parameters: voucher number, date, credit reference, output KID, edited KID string.
- Defines a key list for file lookups (`sohel1_key`).
- Initializes `w_firm` from the value in the LDA (`l_firm`).

---

## Summary of Business Logic

1. **Input Parameters:**
    - Voucher number, date, credit reference, and output fields for KID.

2. **Search Order:**
    1. Try to find a KID in `brefpf` by firm and voucher number.
    2. If not found, try `sohel1` by year (from date) and voucher number.
    3. If still not found, try `sohel1` by year and credit reference.

3. **Output:**
    - Set found KID (as both text and number; numeric version formatted for output).

---

## Typical Usage

This program is typically called from other RPG or CL programs, supplying accounting document information. It attempts to retrieve the KID, used for payment identification in Norwegian accounting, from different sources in a business-prioritized order, and returns the result for further processing.

---
```