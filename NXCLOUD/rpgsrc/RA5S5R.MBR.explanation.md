# RPG Source Code Explanation

This RPG program (RA5S5R) is written for the IBM i (AS/400) platform. It is an interactive program for inquiring about standard VAT codes ("Std. Mva-koder, Spørring") using subfiles on a display. The code is based on RPG/400 with cycle-control (`C` specs) and uses native display (workstn) and database file I/O.

The code is heavily commented in Norwegian; this explanation will clarify the functionality and structure.

---

## High-Level Overview

- **Purpose**: Allows users to view, search, and select VAT codes and descriptions from a subfile interface.
- **Main features**:
    - Supports paging (next/previous) through subfile
    - Allows positioning/jumping by code or description
    - Accepts selection (valg) by marking subfile row
    - Handles various function keys (roll, home, etc.)

---

## File Declarations

```rpg
fras5pf    if   e           k disk    rename(ras5pfr:ras5pfr)
**ras5l2    if   e           k disk    rename(ras5pfr:ras5l2r)
fra5s5d    cf   e             workstn sfile(b1sfl:w_srrn)
```
- **ras5pf**: Main physical file holding VAT code data
- **ras5l2**: (commented out), possibly for alternate access path/index
- **ra5s5d**: Workstation file with subfile (`b1sfl`) and relative record counter `w_srrn`

---

## Data Structures and Fields

### Constants
- `lo`, `up`: Contain lowercase and uppercase letters (for translation/comparison)

### Parameters
- `p_kode`, `p_text`: For passing VAT code and description in/out

### Variables for Subfile Operations
- `w_stel`: Subfile counter
- `w_spge`: Page size (records in a subfile page)
- `w_srrn`: Current relative subfile record number
- `w_sfrn`: First record's number on the current page
- `w_ssrn`: Last record's number on the current page

### Other Useful Variables
- `w_seqe`: Mode/Sequence indicator, e.g., searching by description (`'B'`)
- `b_valg_ok`: Indicates whether the user has made a valid selection
- `scan1/2/3`, `l1/2/3`: Used for dividing a search string into up to three words for scanning
- `i`, `j`: Loop/index variables
- `w_text`: Temporary text, often for comparison/search

### Constants for Subfile
- `c_sfil`: Number of records per page (default `8`)

---

## Indicator Usage

Explain in comments, e.g.:
- `LR`: Last record, ends program
- `10/11`: Roll up/down
- `12`, `13`, `14`, `15`: Various subfile control flags (display, clear, end)
- `21`, `22`: Home/position cursor
- `30+`: Protect fields, errors, work

---

## Main Program Logic

### Main Loop

- Starts with label `b2taga`:
    - Writes command screen (`b2cmd`)
    - Calls subfile display (`dsp_subfile`)
    - Handles page positioning if a current record exists
- Processes function keys (`select/when`):
    - `*inkc or *inkl`: Exit/Cancel
    - `*inke`: Refresh/Forny (rebuild subfile)
    - `*in10`: Roll/Page Down (create next subfile page)
    - `*in11`: Roll/Page Up (previous page)
    - `*in21`: Home key (position cursor)
- Performs Positioning if code or description entered (`posisjoner` subroutine)
- Handles subfile selection (`subfile` subroutine)
- Loops back until exit criteria met

### Program Exit

- Tag `avslutt`: Sets `*inlr` to end the program

---

## Subroutines

### `forny` – Refresh Subfile

- Repositions based on the first record key of the current subfile page
- Clears and recreates the subfile

---

### `posisjoner` – Position by Code or Description

- If code is entered, positions file to that key
- If description is entered, positions using alternate index (for description search)
- Clears and rebuilds subfile starting from positioned row
- Clears input fields

---

### `subfile` – Handle User Selection

- Switches subfile control indicator `*in22` on each entry
- Reads through currently displayed subfile for selection
- If a selection (`b1valg = 1`) is found, stores code and text and sets `b_valg_ok`

---

### `clr_subfile` – Clear Subfile

- Clears subfile indicators and variables
- Writes subfile control record to clear screen

---

### `crt_subfile` – Create/Build Subfile Page

- Increments page and record counters
- **Advanced Search (Lines marked 7.02):**
    - Splits user search phrase into up to 3 words (scan1/2/3)
- Loops, reading records, filtering on search terms:
    - For each record, compares up to 3 search words with the description (`rsetxt`)
    - If all search terms found, writes record to subfile
- Ends when page size reached or no more records

---

### `bck_subfile` – Page Up

- If already at end, positions one page back
- Reads backwards through file to get previous page, then clears/rebuilds subfile starting from there

---

### `dsp_subfile` – Display Subfile

- If there are subfile rows, sets display indicator and shows subfile control record, else writes command panel

---

### `*inzsr` – Initialization

- Loads input parameters
- Sets up positioning keys for the database files
- Initially positions and fills the first subfile page

---

## Additional Notes

- Some logic is commented out (lines with `**` or `7.02`), indicating alternates, enhancements, or version control.
- The code supports both code and free-text search (multi-word), and paged navigation via subfiles.
- All subfile operations are safely isolated in subroutines, making maintenance and understanding easier.
- Error/edge case handling is mostly managed with indicators and checks for EOF.

---

## Workflow Summary

1. **Initialization:** Set up variables, subfiles, and position at first record.
2. **User Interaction Loop:**
    - Display subfile and accept input.
    - Process function keys for paging, new search, home, etc.
    - On selection, set output parameters and exit.
3. **Positioning and Search:**
    - On search input, re-position subfile appropriately.
    - Allows both code-based and description-based searches.
    - Handles free-text, multi-word search by splitting input.
4. **Exit:** Release program resources and return.

---

## Subfile Paging and Navigation

- **Page Down** (`*in10`): Move forward by `c_sfil` records.
- **Page Up** (`*in11`): Move back by `c_sfil` records.
- **Position by Code/Description:** Jumps to matching record(s) using key or alternate index.
- **Selection:** User marks a line for selection; code+text returned.

---

## Key Takeaways for New Developers

- The program is structured for interactive display file handling on IBM i.
- Subfiles are dynamically built, cleared, and navigated in pages.
- Input handling is driven by display file indicators and subroutine calls.
- Search functionality supports both key and flexible free-text matching.
- Reading and updating the display are managed via subfile control indicators (`*in12`/`*in13`/etc.).
- Most logic is encapsulated in subroutines, improving maintainability.

---

**Tip**: For maintainers/enhancers, pay particular attention to the subfile control (paging, positioning), and how multi-word free-text search is implemented for the description field using RPG's string and scan operations.

---

**The program is a classic interactive subfile-driven browser/search utility, typical for IBM i applications using older RPG/400 and display file techniques.**