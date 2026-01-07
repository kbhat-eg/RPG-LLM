```markdown
## Overview

This RPG program, named **ADRARAPP**, is designed to log the start and end times of the latest update of payments from the Adra system. It works by updating or inserting a record in a "log" physical file (`adrdlu`) based on input parameters and the user information found in the local data area (LDA).

---

### 1. Header and Program Options

```rpg
h option(*nodebugio) datedit(*dmy)
```
- `*nodebugio`: Disables certain debug features for I/O operations.
- `*dmy`: Sets default date editing (day/month/year) for date fields.

---

### 2. File Declaration

```rpg
fadrdlu    uf a e           k disk    rename(adrdpfr:adrdlur)
```
- Defines a user-open, full procedural file `adrdlu` for update (`uf`), with keyed access (`k`), based on the record format `adrdpfr`, but referred to in the program as `adrdlur`.

---

### 3. Local Data Area (LDA) Usage

```rpg
d                uds
d l_user                911    920
d l_firm                944    946  0
```
- Declares usage of the program’s User Data Space (UDS) as a structure.
- `l_user`: 10-character field from positions 911–920 in LDA.
- `l_firm`: 3-digit field from positions 944–946 in LDA.

---

### 4. Key Variable Declarations

```rpg
d adrdlu_user     s                   like(aduser)
```
- Declares a variable `adrdlu_user` with the same type as field `aduser` (from the file).

---

### 5. Parameter Handling

```rpg
d p_par           s              1
```
- A single character parameter to control program flow:
  - `'S'`: Log Start time
  - `'E'`: Log End time

---

### 6. Main Logic

#### a. Set Key Variable

```rpg
c                   eval      adrdlu_user = l_user
```
- `adrdlu_user` is set to the user value from LDA.

#### b. Chain to File

```rpg
c     adrdlu_key    chain     adrdlu
```
- Tries to read a record from the log file (`adrdlu`) using the user key.

#### c. Handle Start/End Logging

```rpg
c                   if        p_par = 'S'
c                   time                    addato
c                   time                    adfrat
c                   endif
c                   if        p_par = 'E'
c                   time                    adtilt
c                   endif
```
- If `'S'` (start): Set fields `addato` and `adfrat` in the record to current system time.
- If `'E'` (end): Set field `adtilt` in the record to current system time.

#### d. Update or Insert

```rpg
c                   if        %found(adrdlu)
c                   update    adrdlur
c                   else
c                   eval      aduser = l_user
c                   write     adrdlur
c                   endif
```
- If the keyed record was found: Update it.
- Else: Set the `aduser` field in the buffer to `l_user`, then write a new record.

---

### 7. Program End

```rpg
c                   eval      *inlr = *on
```
- Set the last record indicator to ON, closing files and ending program cleanly.

---

### 8. Initialization Subroutine (*INZSR)

- Handles parameter passing and key definition.

```rpg
c     *inzsr        begsr
c     *entry        plist
c                   parm                    p_par
c     adrdlu_key    klist
c                   kfld                    adrdlu_user
c                   endsr
```
- Declares a parameter list with `p_par` as an incoming parameter.
- Defines the key list for file I/O (`adrdlu_key`) as `adrdlu_user`.

---

## Summary Table

| Section             | Description                                                    |
|---------------------|----------------------------------------------------------------|
| File                | `adrdlu` log file, update/insert by user key                   |
| LDA Usage           | Gets user ID (`l_user`) for logging                            |
| Parameter           | `p_par = 'S'` (start) / `'E'` (end)                            |
| Start action        | Updates date/time fields for start in log                      |
| End action          | Updates date/time field for end in log                         |
| File I/O            | Updates record if found, else writes new record                |
| Initialization      | Handles parameter and file key definition                      |
| Program End         | Ensures proper closure and resource release                    |

---

## Typical Usage Example

- **To log a start:** Call `ADRARAPP` with `'S'` parameter. It writes start times in the log for the current user.
- **To log an end:** Call `ADRARAPP` with `'E'` parameter. It writes end time in the same log record.

---

## Onboarding Tips

- Update field calculations (`addato`, `adfrat`, `adtilt`) depend on current system time.
- The program uses the LDA to get "current user" for logging purposes—ensure LDA is populated correctly.
- Only a single-character parameter is expected on call.
- If customizing, adjust field/record names to match your physical file definitions.

```
