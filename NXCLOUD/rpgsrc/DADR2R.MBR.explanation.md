# RPG Program: DADR2R — Explanation

This RPG (Report Program Generator) IV (ILE RPG) program is designed for IBM i (AS/400) and is part of the ASBANK system. The program's main function is to handle selection and maintenance of records (probably account links) for a module called "ADRA AccountMatch."

Below you'll find an in-depth explanation, section by section, of the source code, including key logic, data structures, and the use of indicators.

---

## 1. **Header and Program Description**
The initial comments provide information on:

- **System**: ASBANK
- **Program Name**: DADR2R
- **Purpose**: Extraction for ADRA AccountMatch; handles subfile screens for selecting, creating, changing, viewing, and deleting account links.
- **Indicators**: Lists the use of RPG indicators for user interface control (keys, file status, subfile control, field protection, etc.).
- **Screen Layouts**: Documents which display records and subfiles (B1SFL, B2CTL, etc.) are used for which functions.

---

## 2. **File Declarations**
```rpg
fdadtl1    uf a e           k disk
fdadrl1    uf a e           k disk
fdadr2d    cf   e             workstn
f                                     sfile(b1sfl:srrn01)
```

- **dadtl1**: Update-capable, full procedural, keyed disk file.
- **dadrl1**: Same as dadtl1, perhaps for parameter records ("r" for parameter, "l" for list?).
- **dadr2d**: Workstation display file, includes subfile (`sfile` keyword).
- **b1sfl**: Subfile, controlled by `srrn01` (relative record number).

---

## 3. **Data and Field Declarations**
```rpg
d mld             s             11    dim(4) ctdata perrcd(1)
```
- **mld**: Array of 4 elements, probably used for messages loaded at compile time.

### Local Data Area (LDA) Fields
```rpg
d l_l1ko          2
d l_l1fi          2
d l_fgrp          931    933
d l_firm          944    946
```
- These relate to job environment (file group, firm number) and are mapped from Job LDA.

### Subfile and Window Field Naming Conventions
- **srrn01**: Subfile relative record number
- **spge01**: Subfile page size
- **ssrn01**: Last record number on the page
- **sfrn01**: First record number on the page
- **svrn01**: Used to determine which page contains a particular record

### Working Fields
These hold currently selected file group, firm, account, department, plus various subfile, window, and parameter control fields.

---

## 4. **Mainline Logic**

### **Subfile Display and User Command Handling**
The program is mainly a loop that shows a subfile, processes key commands (F3, F5, F12 etc., as documented), and allows selection of items, refreshing, rolling through pages, and entering maintenance windows.

#### **b2taga / Main Tag (Label)**
- Initializes row/col for window.
- Displays command window (`write b2cmd`).

#### **b2tagb**
- Calls `xdsp01` (display routine for subfile entries).
- Checks for exit (F3), add (F5), or other function keys.
- Handles new connection, page roll, or extraction call.
- Jumps back to b2taga as needed.

#### **Subfile Record Processing**
- Loops through all entries in subfile (`readc b1sfl` with loop from 1 to `ssrn01`).
- Handles selection (b1valg = 1) and deletion (b1valg = 4) of entries via subroutine calls.

---

## 5. **Subroutines**

### **xc1win** — "New" / "Edit" / "Show" Window Handler
- Handles the input of new account links or editing existing ones.
- Clears or initializes the entry fields.
- Validates input (file group and account number).
- If key fields are valid, writes the record to file, and refreshes the subfile.

### **xc3win** — "Delete" Handler
- Handles the deletion confirmation window.
- Deletes the relevant parameter record if confirmed.
- Clears the "marked for delete" flag in the subfile after deletion.

### **xoppdat** — "Update parameter file with account selection"
- Updates parameter file (`dadrpfr`) with selected/deselected entries.
- Toggles 'Valgt' (Selected) status on entries.
- Deletes or writes parameter records as needed, and updates the subfile.

### **xclr01** — Clear the subfile
- Sends control record with clear indicator to remove all subfile records.

### **xnysub** — Fill up the subfile after changes
- Clears parameter file and subfile, resets working fields, reloads items to the subfile by calling `xcrt01`.

### **xcrt01** — Populate subfile with records from file
- Pages through records from `dadtl1`, adds them to the subfile (up to page size).
- Copies file group, firm, account, dept fields into each subfile entry.
- Maintains subfile paging variables.

### **xdsp01** — Display Subfile
- Sets indicators for display.
- Presents the subfile control format.

### **xclear** — Wipes the parameter file (`dadrpfr`)
- Reads and deletes all records in the parameters file.

---

## 6. **Initialization**
**`*inzsr`** — Program Initialization Subroutine
- Initializes page/cursor/working fields to default values.
- Loads fields from LDA if needed.
- Prepares key list for `dadtl1` file operations.
- Deletes all parameter records (via `xclear`).
- Prepares the subfile for display by loading from `dadtl1`.

---

## 7. **Indicators and Branching**
- RPG indicators are used for controlling function keys, file status, screen display state, and more.
- Examples:
    - *INKC: F3=Exit
    - *INKF: F5=new
    - *IN10: Roll
    - *IN14/15: Subfile clear/end
    - *IN16: End of subfile readc

---

## 8. **Special Comments and Lookups**
At the end, there's a comment block for codes displayed on screen:
- Ny post (new)
- Endring (change)
- Vise (view)
- Slettet (deleted)

---

# **Summary for New Developers**

This program is a traditional ILE RPG application for interactive maintenance of account links. It relies heavily on subfiles, screens, and user function keys for navigation and record selection. The key logic revolves around building, displaying, and updating a subfile based on user actions, with clear modular separation (subroutines) for tasks such as clearing files, updating parameters, and handling window displays.

**Key Learning Points:**
- Understand RPG indicators and their assigned functions.
- Subfile handling (pagination, update, clear, display).
- Application structure: Mainline tags + subroutines for each logical function.
- File operations: CHAIN, SETLL, READ, WRITE, UPDATE, DELETE, READE.
- Use of LDA and working fields for context.
- Norwegian comments are mainly descriptive and do not affect logic.

**Tip:** If you're new to RPG or the AS/400, pay special attention to the use of indicators, file operations, and subfile management, as these are central to interactive RPG programming.