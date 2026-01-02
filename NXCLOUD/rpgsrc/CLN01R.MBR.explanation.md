# Explanation of RPG Program `CLN01R`

This documentation aims to bring a developer up to speed on how the program works and how it is structured.

---

## **General Purpose**

The program is used for maintenance of supplier categories (nobbkategorier) for a system named "Ehandel", specialized for "Mestergruppen". It is mainly a subfile-based interactive program that allows users to view, add, update, or delete supplier category records.

---

## **Header and Program Attributes**

- `h option(*nodebugio) datedit(*dmy)`  
  Specifies compile options:
  - `*nodebugio`: Excludes I/O debug information for performance.
  - `datedit(*dmy)`: Date fields use Day/Month/Year format.

---

## **File Declarations**

- Several files are defined, all of which are keyed physical files:
    - `clnki1`, `clnkir`, `clnkiu`: different access paths over the same table (`clnkst`), renamed for clarity.
- Display file:
    - `cln01d` is the display file, using record-level access (`workstn`), with a subfile (`b1sfl`), and an INFDS (`dspfbk`) for screen feedback.

---

## **Data Structures and Fields**

### **Field Usage from Local Data Area (LDA)**
- `l_user`, `l_firm`, `l_fnav`: User, firm number, and navigation options, respectively, are read from the LDA.

### **Display Feedback Data Structure**
- `dspfbk`: Used to capture feedback from the workstation, like current screen record number (`d_fcrn`).

### **Key Variables**
- `clnkir_nkka`, `clnkiu_nkka`, `clnki1_nkka`: Used to hold key values for accessing the above declared files.

### **Other Variables**
- Variables beginning with `w_`: Work variables for page control, subfile record numbers, flags, etc.
- `b_oppr`, `b_forn`, etc.: Boolean flags controlling program flow.
- Constants like `c_sfil` (subfile page size, default 14).

---

## **Indicator Usage**

The comments give an extensive indicator mapping, e.g.:
- `LR`: Last record, program end.
- `*in10`, `*in11`, etc.: Function key indicators for Rollup, Rolldown, etc.
- `*in12`, `*in13`, `*in14`, `*in15`: Special subfile control indicators (show, control, clear, end-of-list).
- `*in21`, `*in22`, `*in30`: Cursor and field protection control.
- `31-79`: Error/messaging.
- `80-99`: Work indicators.

---

## **Main Loop (Program Flow)**

### **Tags—Main Loop Control**
- `b2taga` and `b2tagb`: Control points for the main menu and subfile maintenance.
- Displays command key screen (`b2cmd`) and then jumps into subfile maintenance.

### **Function Key Handling**
Inside a `SELECT` block:
- `*inkc` or `*inkl`: Exit program.
- `*inke`: "Forny" (refresh or renew screen), calls `forny` subroutine.
- `*inkf`: Go to create new entry subroutine (`xc1bld`).
- `*in10`, `*in11`: Roll up/down through subfile pages.
- `*in21`: Home key, sets cursor relocation.
- If user has entered something in the position field (b2nkka), program calls `posisjoner` for positioning in the subfile.

### **Subfile Handling**
- Main interaction happens via subfiles, allowing insert, update, delete, review of supplier categories.

---

## **Subroutines**

### **forny**
- Refreshes the subfile, re-reads data from the file starting at the current subfile record.

### **posisjoner**
- Positions the subfile at a specific record, based on user entry.
- Clears and refills the subfile from the new position.

### **subfile**
- Reads and handles each subfile record selected on the screen.
- Allows user to:
    - **Edit:** (option 2)
    - **Delete:** (option 4)
    - **View:** (option 5)
  Each option triggers its corresponding screen and update routine.

### **xc1bld**
- Screen for adding a new entry.
- Handles validation to prevent duplicates.
- On valid entry, calls `xc2bld` for maintenance.

### **xc2msg**
- Displays an info screen when a duplicate entry is attempted.

### **xc2bld**
- Handles the screen for creation/edit/viewing of a category.
- Updates or writes records, depending on whether the record already exists.

### **xd1win**
- Handles delete confirmation window and performs delete.

### **clr_subfile**
- Clears the subfile (indicator 14 and resets counters).

### **crt_subfile**
- Fills the subfile with records up to the page size.

### **bck_subfile**
- Allows flipping back a page in the subfile (using `readp` - read previous).

### **dsp_subfile**
- Handles subfile display logic, sets indicators for display or re-display.

### **\*inzsr (Initialization)**
- Sets key lists for the various logical file accesses.
- Reads values from the LDA and initializes the subfile.

---

## **Key Lists**

- For all physical/logical file access, key lists (`klist`) are defined for each:  
  Each corresponding access path requires both the firm and the key (category).

---

## **Screen Record Formats**

These are referenced in comments and in the code:
- `b1sfl`: Subfile record.
- `b2ctl`: Subfile control record.
- `b2cmd`: Command key screen.
- `c1bld`: Add-entry screen.
- `c1msg`: Info screen for `c1bld`.
- `c2bld`: Create/Edit/View screen.
- `d1win`: Delete confirmation window.

---

## **Special/Norwegian Terms**

- `Vedlikehold`: Maintenance
- `Leverandør`: Supplier
- `Kategori`: Category
- `Nobb`: Norwegian building product register

---

## **Summary**

- **Purpose**: The program manages supplier category (nobbkategorier) entries in a database, with full subfile-based CRUD (Create, Read, Update, Delete) capability.
- **Interaction**: It is an interactive green-screen program using subfiles for list-entry display, with options for next/previous page, add, edit, delete, view.
- **Flow**: The program’s main loop branches based on function keys and subfile option field entries, calling subroutines as needed.
- **Extensibility**: The clear separation into subroutines for each functional area makes it fairly maintainable.

A developer onboarding this code should focus on:
- Understanding the subfile mechanics (especially `crt_subfile`, `clr_subfile`, `bck_subfile`).
- The indicator usage for controlling display and options.
- The logical file/key structures for accessing category data.
- The various screen formats (DSPF/member `CLN01D`) for possible extensions or changes.