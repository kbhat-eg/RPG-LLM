# RPG Program: BOOPOR — Order Register Update after Store Processing

This program, `BOOPOR`, is part of the ASSHOP system and is responsible for updating the order register (`ORDRE-HODE-REGISTER`) after processing orders in-store. Below, you’ll find an explanation of its structure and business logic, focusing on onboarding developers new to RPG or this codebase.

---

## File Declarations

```rpg
ffohelu    uf   e           k disk    rename(fohepfr:fohelur)
```
- **fohelu**: A physical file (likely the order header register), used for update (*) and keyed access (`k`).
- **rename(fohepfr:fohelur)**: The record format `fohepfr` is renamed to `fohelur` within this program.

---

## Data Area and Parameter Definitions

```rpg
d                uds
d l_wsid                901    910
d l_user                911    920
```
- Defines the User Data Structure (UDS) for local data area, with workspace/user ID fields.

**Parameter Variables (passed into the program):**
- `p_aarr` : 4-digit numeric, probably a year or similar code.
- `p_firm` : Firm/company code.
- `p_behd` : Processing code.
- `p_bong`, `p_numm` : Order numbers.
- `p_suff` : Order suffix.
- `p_wsid` : Workstation ID.

**Key Variables (for file access):**
- `w_firm`, `fohelu_numm`, `fohelu_suff`: Used together to access/update an order in the order header file.

**General Working Variables:**
- Short comment fields, type fields, and operation flag `w_opdt`.

---

## Main Logic

### 1. Begin/Initialization Section

- Program begins with /TITLE and commented headers including change history (e.g., change 5.01).

### 2. Parameter Handling

```rpg
c     *entry        plist
c                   parm                    p_behd
c                   parm                    p_firm
c                   parm                    p_aarr
c                   parm                    p_wsid
c                   parm                    p_bong
c                   parm                    p_numm
c                   parm                    p_suff
```
- The program expects those parameters to be passed from the caller (another program/job).

### 3. Key Preparation and Variable Initialization

```rpg
c                   eval      w_firm = p_firm
c                   eval      fohelu_numm = p_numm
c                   eval      fohelu_suff = p_suff
```
- Local key variables are set from parameters for consistent usage in file operations.

---

## Order Processing

### A. Case 0: No Processing Ongoing

```rpg
c                   if        p_behd = *zero
c     fohelu_key    chain     fohelur                            90
c                   if        *in90 = *off
c                   eval      fokode = *zero
c                   update    fohelur
c                   endif
c                   endif
```
- If `p_behd` is zero, meaning "no processing ongoing":
  - Chain (find by key) the order header record.
  - If found (`*in90 = *off`), set the order code field (`fokode`) to zero and update the record.
  - This *unmarks* any in-store processing on the order.

---

### B. Case 1: Difference Deleted

```rpg
c                   if        p_behd = '1'
* Call subprogram to delete order
c                   eval      w_type = 'SHOP'
c                   call      'FO410R'
c                   parm                    w_firm
c                   parm                    fohelu_numm
c                   parm                    fohelu_suff
c                   parm                    w_type
c                   parm                    w_kom1
c                   parm                    w_kom2
5.01 c                   parm                    w_opdt
c                   endif
```
- If `p_behd` is `'1'`, "delete the difference":
  - Sets `w_type` to `'SHOP'`.
  - Calls external RPG program `FO410R` to handle order record deletion/modification.
  - Passes necessary parameters, including keys and operation type.

---

### C. (Implied) Case 2: Difference Retained

- Not directly coded in the snippets shown, but documentation mentions:
    - Code=2 → Difference retained, i.e., backorder.

---

## Program End and Subroutines

### Program Exit

```rpg
c     xslutt        tag
c                   eval      *inlr = *on
c                   return
```
- Standard RPG/400 idiom for clean program exit (`*inlr = *on` closes files and ends activation group).

### Initialization Subroutine

```rpg
c     *inzsr        begsr
* Null out fields
c                   eval      w_type = *blank
c                   eval      w_kom1 = *blank
c                   eval      w_kom2 = *blank
c                   eval      w_alf1 = *blank
* Setup key list for fohelur
c     fohelu_key    klist
c                   kfld                    w_firm
c                   kfld                    fohelu_numm
c                   kfld                    fohelu_suff
c                   endsr
```
- Initializes working variables to blanks.
- Sets up a key list (`klist`) for convenient keyed access to the order header file.

---

## Summary

- **Purpose:** Updates order records after store processing by either clearing status, deleting, or potentially retaining (backordering) items.
- **Parameters:** Key order data, firm, and processing code.
- **File Operations:** Keyed updates to the order header file (`fohelu`/`fohelur`).
- **Subprogram Calls:** Invokes external logic as needed.
- **Fields and Workflow:** Sets/clears process status, uses keyed file access, cleanly initializes and exits.

---

**Onboarding Note:**  
Understanding the logic relies heavily on knowledge of business rules (how `p_behd` is intended to be used), the structure of `fohelu` and the behavior of `FO410R`. Most processing is straightforward RPG III/400 style, with a minor ILE flavor (subroutines, parameter handling, etc.). Comments are partly in Norwegian; “Ferdigbehandlet i butikk” means “Processed in store.”