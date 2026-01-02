# RPG Program AT121R – Detailed Explanation

This ILE RPG program is designed for IBM i (AS/400, iSeries) systems and appears to be a customer-tailored configuration maintenance interface, using subfiles to display, add, update, and delete records in a table called AKTIST (likely a configuration or item table). Below, we break down its major sections and logic:

---

## 1. Header and Comments

- **`h option(*nodebugio : *srcstmt) datedit(*dmy)`**  
  Sets compiler options:  
  - `*nodebugio`: Disables I/O debug code.  
  - `*srcstmt`: Source statement numbering for error feedback.  
  - `datedit(*dmy)`: Date edit word for day/month/year.

- **Comment Block**  
  Describes program name (`AT121R`), system, and author/change history.  
  Lists screen indicator assignments (LR, *INxx indicators), including their purposes (like rollup, rolldown, protect, etc.).

---

## 2. File Declarations

- **Display File**  
  ```
  fat121d    cf   e             workstn sfile(b1sfl:w_srrn)
                                        infds(dspfbk)
  ```
  - Declares display file `AT121D` with subfile support (SFL, controlled by `b1sfl` and relative record number `w_srrn`).
  - Feedback area named `dspfbk` for display file status info.

---

## 3. Data Area and Data Structures

- **Local Data Area (LDA)**  
  ```
  d                uds
  d l_user                911    920
  d l_firm                944    946  0
  d l_fnav                951    980
  ```
  - Maps system LDA into user, firm, and name variables.

- **Display Feedback DS**  
  ```
  d dspfbk          ds
  d  d_fcrn               378    379I 0
  ```
  - Feedback DS for display file; `d_fcrn` stores the first record number on the page.

---

## 4. Work Variables

Defines numerous working variables, for example:

- `w_date`, `w_time`: Current date/time.
- `w_fcrn`, `w_stel`, etc.: Subfile control (first record on page, record counters, etc.).
- `b_forn`, `b_oppr`, `b_anul`: Boolean flags controlling logic flow.
- `w_aktuid`, `w_aktfgp`, ...: Fields for AKTIST table records.
- `sqlquery`: Buffer for dynamically built SQL queries.
- `b_open_sok`: Boolean flag indicating if new subfile search is needed.

**Constants:**
- `c_sfil`: Subfile page size (18 rows per page).

---

## 5. Program Flow and Main Loop

### Main Tag/Screen Handler ("B2CTL")

The code is structured around a *main loop* using tags and subroutines (`exsr`), with control handed to various routines based on user input (*IN indicators).

- **Display main command screen (`b2cmd`)**
- **Display subfile:**  
  Calls subroutine `dsp_subfile` to display the subfile.

- **Function Key Handling:**  
  Uses `select/when` for handling PF keys:
  - `*inkc`, `*inkl`: Exit program.
  - `*inkd`: Inquiry function (`spørring`).
  - `*inke`: Refresh (`forny`).
  - `*inkf`: Add new (`xc1bld`).
  - `*in10`: Rollup (load next subfile page).
  - (Other keys and logic commented out)

- **Positioning:**  
  If any positioning field is set, runs the `posisjoner` routine which resets search fields, clears, and recreates the subfile.

- **Subfile Processing:**  
  Calls `subfile` subroutine for row-level logic.

- **Program Exit:**  
  Tag `avslutt` sets `*inlr` to ON and returns, ending the program.

---

## 6. Subroutines (EXSR)

### a) `forny` – Refresh subfile

- Chains to subfile if current record pointer is set.
- Flags that new search is needed, clears and recreates the subfile.

---

### b) `posisjoner` – Position in subfile

- Clears subfile and refills it according to positioning fields set on screen.
- Afterwards, clears the positioning fields (for a fresh search).

---

### c) `subfile` – Subfile Row Processing

- Loops through all records in the subfile, reading changed rows.
- For each, checks if a function (change, copy, delete, display) has been requested via selection fields (`b1valg`):
  - **Change (2):** Prepares update via `xc2bld`
  - **Copy (3):** Prepare new via `xc1bld`
  - **Delete (4):** Confirm and delete via `xd1win`
  - **Display (5):** View row via `xc2bld`
- After subfile actions, if any function has updated the file, refreshes subfile.

---

### d) `xc1bld` – Add Row Screen

- If copying, pre-populates new row fields from selected row.
- Displays the Add screen (`c1bld`).
- If the attempted row already exists, shows a warning message (`xc1msg`).
- If not, passes control to change/view logic via `xc2bld`.
- On completion, refreshes subfile if necessary.

---

### e) `xc1msg` – Row Exists Message

- Shows an informational message if an attempt is made to add a duplicate record.

---

### f) `xd1win` – Delete Confirmation

- Fetches the row to be deleted for display (populating delete screen).
- Shows a confirmation window.
- If confirmed, deletes from AKTIST.

---

### g) `xc2bld` – Change/View Screen

- If updating, loads fields from the add screen into the change screen.
- Otherwise, fetches all AKTIST fields for the selected record.
- Shows the record change/view screen (`c2bld`).
- Handles PF keys for exit.
- If user confirms:
  - Updates the record if in change mode.
  - Otherwise, inserts a new record if adding.
- Checks SQL return code for errors and repeats if necessary.

---

### h) `clr_subfile` – Clear Subfile

- Sets indicator to clear subfile, writes control record, resets key counters.

---

### i) `crt_subfile` – Create/Fill Subfile

- Calculates how many rows to fetch.
- If a new search is requested (positioning or search fields changed), builds a dynamic SQL query, optionally with WHERE clauses based on filter fields.
- Prepares and opens a cursor for the select.
- Loops, fetching rows, and writing them to the subfile until page-size is reached or no more data.
- Updates internal paging counters.

---

### j) `dsp_subfile` – Display Subfile

- If there are records, sets indicator for subfile display, otherwise writes command screen.
- Writes the control record and updates paging state.

---

### k) `spørring` – Inquiry Handling

- Placeholder for inquiry logic (commented out). Could be extended for customer-specific lookups.

---

### l) `*inzsr` – Initial Program Setup

- Loads firm from LDA.
- Initializes date/time.
- Prepares and fills the subfile for the first screen display.

---

## 7. SQL Integration

The program uses embedded SQL for all database operations (select/insert/update/delete) against AKTIST. It makes extensive use of dynamic SQL queries for filtering and cursor-based row retrieval for subfile paging.

---

## 8. Screen/Indicator Conventions

- Uses many *IN indicators* controlling screen logic, subfile display, and input protection.
- Comments are clear about the purpose of each indicator and subfile variable (record numbers, paging, cursor location).

---

## 9. Error Handling

- SQL error codes (`sqlcod`) are checked after nearly all SQL operations.
- Duplicate record attempts are caught and messaged.
- Negative return codes lead to exit or repeat logic as appropriate.

---

## 10. Naming Conventions and Norwegian Terminology

- Many variables, comments, and tags use Norwegian terms:
  - `forny` (renew/refresh), `posisjoner` (position), `spørring` (inquiry), `beskrivelse` (description), etc.
- Field and subroutine naming is consistent, aiding maintainability.

---

## 11. Subfile Paging/Navigation

- The program supports paging through records, positioning, and rolling up/down via indicators and counters (`w_fcrn`, `w_stel`, `w_srrn`, etc.).

---

## Summary

**This program is a classic RPG/400 subfile maintenance application ported to ILE RPG, highly modular, using tags and subroutines for major actions. It leverages embedded SQL for database operations, uses subfiles for interactive record handling, and employs a detailed indicator and paging system to manage the screen flow and user interactions.**

It is well-commented and designed for extensibility and maintainability on IBM i platforms.