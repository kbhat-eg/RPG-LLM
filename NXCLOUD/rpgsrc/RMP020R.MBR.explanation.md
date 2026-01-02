# RPG Program rmp020R - Maintenance for MapProff Transfers

---

## High-Level Overview

This program is a traditional ILE RPG program with indicators, subfile processing, and database interaction, used for maintaining "MapProff-transfers" (likely transactional or interface records). It interacts with multiple database physical files (rmpflr, rmpfl1, rmpflu) and displays records using DSPF (display file) with subfile support.

The program supports:
- Displaying a paginated list of records (`subfile` construction and navigation)
- Editing and viewing individual records
- Adding, updating, and deleting records (possibly bonuses or transactions)
- Uses indicator-based UI and device feedback

---

## File Declarations

```rpg
frmpflr    if   e           k disk    rename(rmpfpfr:rmpflrr)
frmpfl1    if   e           k disk    rename(rmpfpfr:rmpfl1r)
frmpflu    uf a e           k disk    rename(rmpfpfr:rmpflur)

frmp020d   cf   e             workstn sfile(b1sfl:w_srrn)
f                                     infds(dspfbk)
```
- **rmpflr**, **rmpfl1**, **rmpflu**: Database files, all renamed for internal usage.
- **rmp020d**: Display file with subfile support (`b1sfl:w_srrn`), and INFDS (feedback data structure) for state management.

---

## Data Structures & Variables

- **Local Data Area** (`uds`): Reads user, date, text, and firm info from LDA.
- **Display Feedback** (`dspfbk`): For reading the screen cursor record number (`d_fcrn`).
- **Key Variables**: Used to build keys for chained/random access to files.
- **Various Work Variables**: `w_fcrn`, `w_spge`, `w_srrn`, etc, supporting subfile navigation, cursor placement, etc.
- **Flags (e.g., b_forn, b_oppr, etc)**: Used to indicate update/operation state and as error or status "switches".
- **Constants**: `c_sfil` sets the subfile page size (normally 13 records per page).

---

## Indicator Usage

The header comment explains indicator assignments (10–99 for different UI and logic states).

- *10*: Page up
- *11*: Page down
- *12*: Subfile display
- *13*: Subfile display control
- *14*: Subfile clear
- *15*: Subfile end
- *21*: Home key pressed
- *22*: Cursor position
- *30*: Field protection

And others for warnings, errors, and work indicators.

---

## Program Flow

### 1. Main Loop

- **Initial Tag:** `b2taga`  
  - Write command line (B2CMD record) to screen.
  - Goto data display.

- **Data Display Tag:** `b2tagb`  
  - Subfile display (`exsr dsp_subfile`)
  - Handle function keys with `select`:
    - *Page Up/Down*: Build new subfile page.
    - *Exit keys*: End program.
    - *Home*: Set cursor positioning.
    - *Other keys*: Refresh or update subfile.

  - Subfile maintenance via `exsr subfile`.

### 2. Subfile Maintenance

- Loops through subfile entries (`do w_ssrn`)
- For each changed record (`readc b1sfl`):
  - If *edit* chosen (`b1valg = 2`): Enter edit/maintain subroutine.
  - If *delete* (`b1valg = 4`): Enter delete subroutine.
  - If *display* (`b1valg = 5`): Enter view/maintain subroutine.

- Refreshes subfile if any changes were made.

### 3. Subfile Operations

#### `clr_subfile` - Clear Subfile
- Sets indicators to clear subfile region on the display.
- Resets subfile counters.

#### `crt_subfile` - Create Subfile Page
- Calculates page range.
- Reads records from file, populates the subfile, writes to screen until page is full or EOF.

#### `bck_subfile` - Page Backward
- Navigates to previous page by reading backwards (`readp`) until a full page is prepared.
- Clears and repopulates subfile with previous records.

#### `dsp_subfile` - Display Subfile
- Handles display logic, manages indicators for window/control.

---

## Record Maintenance

### `xc2bld` - Edit/View Record

- Chain fetches full record from rmpfl1 by key.
- Unpacks a semi-colon delimited string into multiple fields (GLN, date, time, line, EAN, amount, order no., department, customer no, customer name), field by field.
- Displays info in C2BLD-format screen until exit or update.
- On update, rebuilds the semi-colon delimited record string, writes the result back to rmpflu (update).
- Field validation is stubbed (commented out).

### `xd1win` - Delete Record

- Displays confirmation dialog.
- On confirm, chains and deletes the matching record from rmpflu.

---

## Initialization

### `*inzsr` - Program Initialization

- Defines key lists for all three files.
- Gets firm code from LDA.
- Sets up time/date.
- Clears display variables.
- Positions database to start and fills subfile with first page of records.

---

## Additional Notes

- There are some stubs/comments for unimplemented field validations or calls (e.g., customer inquiry/program call).
- The program is traditional, indicator-driven RPG IV, with Scandinavian comments and variable names.
- The display logic is tightly controlled with indicators and subfile feedback (INFDS, screen cursor, etc.).
- Subfile paging is manually managed, with `w_srrn`, `w_ssrn`, etc.
- No *procedure* or /free, just classic fixed-form RPG IV with subroutines (`begsr`/`endsr`).

---

## Conclusion

This program is a classic ILE RPG "work with" application, used for maintenance of MapProff interface/bonus records, with subfile/paging and record CRUD capability, relying heavily on RPG indicators, file-level operations, and DSPF subfiles.

Key skills to work with or maintain this code:
- Understanding of RPG indicators and subfile processing
- Familiarity with chained/random database file access
- Comfort with DSPF/INFDS device feedback
- Ability to reason about multi-record (subfile) UI logic
- Reading Norwegian variable and comment conventions.