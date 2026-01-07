# RPG Program XS500R – Code Explanation

This ILE RPG program (XS500R) is a typical "Work with" type application for managing some kind of codes (most likely delete codes) with subfile browsing. It employs subfiles to display lists, allows user interaction via function keys, and implements user authority checks. Below is a structured explanation of the code's components and logic.

---

## 1. **Header & Metadata**

```rpg
h option(*nodebugio) datedit(*dmy)
```
- Suppresses debug I/O and sets date edit to day/month/year.

Comment block describes program name (XS500R), description (Slettekoder, spørring = Delete codes, inquiry), and maintenance history. There is also a detailed list of indicator usage.

---

## 2. **File Declarations**

```rpg
fhentifsw  if   f  132        disk
fhentifsd  cf   e             workstn sfile(b1sfl:w_srrn)
```
- `hentifsw`: Input file, disk-based, 132 bytes, most likely contains records to be displayed.
- `hentifsd`: Workstation file, supports subfile `b1sfl` controlled by variable `w_srrn`.

---

## 3. **Data Area and Variables**

### Local Data Area (LDA)
```rpg
d                uds
d l_user                911    920
d l_firm                944    946  0
d l_fnav                951    980
```
- Fields overlaid on the LDA for user, firm, and firm name.

### Parameters/Variables
```rpg
d p_bane          s             60
d p_valg          s              1
d p_in03          s              1
d p_fil           s             10
d p_lib           s             10
```
- Program parameters: path, selection, indicator, file, and library.

### Working Variables and Subfile Counters
```rpg
d w_stel          s              4  0
d w_spge          s              4  0
d w_srrn          s              4  0
d w_sfrn          s                   like(w_srrn)
d w_ssrn          s                   like(w_srrn)
...
d b_valg_ok       s              1    inz(*off)
...
d c_sfil          s              2  0 inz(10)
```
- Subfile state: counters for subfile row number, page, from/to record numbers.
- `b_valg_ok`: Boolean flag for OK selection.
- `c_sfil`: Constant for page size (default 10).

### Authority Check Variables (AD)
```rpg
d w_tilgko        s             10
d w_rutine        s             10
d w_adv           s              3
d w_status        s             10
```
- For authority validation, used with external program 'AD005R'.

---

## 4. **Input Specifications**

```rpg
Ihentifsw  NS  01
I                                  1  132  h_rec
I                                  2   21  h_bane
I                                 26   29  h_type
```
- Input field mapping from `hentifsw`: full record, path (`h_bane`), and type (`h_type`).

---

## 5. **Comments on Subfile Variables, Screens, and Usage**

- Detailed explanations for subfile-related counters.
- Lists screen formats used: subfile record, subfile control, and command screen.

---

## 6. **Main Control Logic (Cycle Level)**

### a. Entry Tags and Display
- Write the command screen (`b2cmd`).
- Subfile handling starts (exsr dsp_subfile).

### b. Cursor and Page Calculation
```rpg
if w_curn > 0
    w_sfrn = (w_curn / c_sfil) * c_sfil + 1
endif
```
- Determines display page based on cursor.

### c. Function Key Processing

- SELECT block handles function keys:
    - *INKC or *INKL: Path manipulation (for wildcards and navigation), sets p_in03 and exits.
    - *INKE: Calls subroutine `forny` to renew subfile.
    - *IN10: Calls `crt_subfile` to create next page.
    - *IN11: Calls `bck_subfile` to go back a page.
    - *IN21: Home key pressed, sets cursor place and jumps.
- Handles new file selection logic (`b2nyfi`).

### d. Subfile Processing
- Calls subfile logic (`exsr subfile`), exits if valid selection.

### e. Program Exit
- At `avslutt`, sets LR indicator to end program.

---

## 7. **Subroutines**

### **forny**
- (Commented out legacy code for positioning/chain and subfile clearing)
- Intended for renewing the subfile display.

### **posisjoner**
- (Commented code for setting key and clearing/creating subfile)
- Intended for positioning in subfile.

### **subfile**
Handles user interaction with subfile records:
- Toggles *IN22 indicator (used for cursor tracking/page position).
- Loops through records on the current subfile page:
    - Reads subfile record (`readc b1sfl`).
    - Authority check by calling 'AD005R', disables selection if access is "Sperret".
    - Handles directory (`DIR`) or stream file (`STMF`) types:
        - For DIR: builds new path, sets selection flags, exits loop.
        - For STMF: similar, but different path manipulation.

### **clr_subfile**
- Sets indicator to clear subfile (*IN14), writes control record, and resets counters.

### **crt_subfile**
- Creates/fills a subfile page:
    - Sets page/end counters.
    - Reads records from base file (`hentifsw`) up to page size.
    - Only includes `DIR` or `STMF` types.
    - Writes each item to the subfile.
    - Updates first/last record numbers.

### **bck_subfile**
- Handles paging backwards:
    - Reads previous records from `hentifsw` file up to a page size.
    - Skips unwanted records.
    - Resets/clears and creates the new previous page subfile display.

### **dsp_subfile**
- Handles display of subfile screen.
    - If subfile has records, sets appropriate indicators and formats screen.

### **Initialization Subroutine (`*inzsr`)**
- Processes entry parameters.
- Sets up initial subfile display.
- Prepares screen variables.

---

## 8. **Authority Check Integration**

```rpg
call      'AD005R'
parm                    w_tilgko
parm                    w_rutine
parm                    w_adv
parm                    w_status
```
- Calls external program to check user authority before allowing selection of a record.

---

## 9. **Key Concepts**

- **Subfile Pagination:** Manages a window over many records using counters for page and row within the subfile.
- **Function Key Mapping:** Each function key has its specific navigation or data operation, typical in green-screen UIs.
- **Authority Checks:** Ensures only allowed users can perform certain actions.
- **Dynamic Path Manipulation:** For navigating a filesystem-like structure in the UI.
- **Use of Indicators:** Controls display, protection, error states, and navigation logic.

---

## 10. **Summary**

This program is a classic ILE RPG subfile-driven browser/inquiry tool, tailored for a code/record management process, with authority checks and navigation similar to IBM i's native file viewers. It is organized with clear subroutines (for page view, navigation, refresh, and display), leverages RPG's indicator model, and interacts with external validation logic for security.

Developers onboarding will mostly interact with:
- Subfile page size and navigation logic.
- Authority integration with 'AD005R'.
- Path and selection field management in the context of the screen.
- Understanding how indicators direct program flow for the green-screen UI.

**Tip**: Modernization would suggest improving maintainability by refactoring indicator usage and moving to RPG IV (free format), but the logic is still easily traceable for RPG III/IV-trained developers.