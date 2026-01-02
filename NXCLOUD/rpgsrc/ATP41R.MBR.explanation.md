# RPG Program ATP41R – Explanation

This RPG III-style source code describes a display file-driven program for IBM i (AS/400), supporting "Avtale" (Agreement) subfile selection. The code features robust use of subfiles and function key handling for user interaction, with a Scandinavian/Norwegian context.

Below is a structured explanation of the source code to help new developers understand the key components and program flow.

---

## 1. Program Description & Structure

- **Title:** ATP41R – Subfile-driven lookup/selection for agreement types.
- **Function:** Allows a user to browse and select from a predefined list of agreement types using a display file subfile with paging and function-key navigation.

### Key Features

- Uses subfiles as display lists for selection.
- Supports function keys for operations: Roll Up/Down, Home, field protection, cursor placement, and messaging.
- Returns selected code and description via parameter passing.

---

## 2. File and Data Structures

### File Declarations

```rpg
fatp41d    cf   e             workstn sfile(b1sfl:w_srrn)
```
- **File:** `atp41d` – Display device file using subfile `b1sfl`, relative record number `w_srrn`.
- **Keywords:** `cf` (display), `e` (external), `workstn` (workstation), `sfile()`.
- **INFDS:** Uses `dspfbk` as Information Data Structure for display feedback.

### Parameter Definitions

```rpg
d p_tkod          s              3
d p_tbes          s             20
```
- **p_tkod:** 3-character code (agreement type).
- **p_tbes:** 20-character description.

### Data Structures

```rpg
d dspfbk          ds
d  d_fcrn               378    379I 0
```
- `dspfbk` for display feedback, retrieves current record number from the workstation file.

### Working Variables

```rpg
d w_fcrn          s              4  0  // First record on page
d w_stel          s              4  0  // Subfile counter
d w_spge          s              4  0  // Subfile page size
d w_srrn          s              4  0  // Relative record number in subfile
d w_sfrn          s                   like(w_srrn) // First record on last page
d w_ssrn          s                   like(w_srrn) // Last record on last page

d w_seqe          s              1
d b_valg_ok       s              1    inz(*off)    // Selection OK
d b_kreq          s              1    inz(*off)    // Request flag
d w_tkod          s              3    // Working agreement code
d w_tbes          s             20    // Working agreement description

d c_sfil          s              2  0 inz(8)       // Subfile page size constant
```

#### Subfile Control Variables
- **w_fcrn:** First record number on page
- **w_stel:** Subfile loop counter
- **w_spge:** Page size in subfile
- **w_srrn:** Relative record number in subfile
- **w_sfrn:** First record number of last page
- **w_ssrn:** Last record number
- **b_valg_ok:** Boolean flag for a successful selection
- **b_kreq:** Boolean flag for request mode

---

## 3. Mainline Flow Overview

### Program Start

- If `b_kreq = *on`, branch to `avslutt` label to terminate the program.

### Main Loop

- Program consists of a tag/label loop using `goto` (e.g., `b2taga` and `b2tagb`)
- Loop alternates between displaying command keys and the subfile list.
- Handles function keys for navigation and selection.
- If a selection is made, program exits.

### Function Key Processing

- `*INKC`, `*INKL`: End/Cancel – Branch to exit.
- `*INKE`: Refresh (perhaps Enter) – Calls `forny` subroutine (currently empty).
- `*IN10`: Roll up – Calls `crt_subfile` (adds more entries to the subfile).
- `*IN11`: Roll down – Calls `bck_subfile` to page back in the subfile.
- `*IN21`: Home-key – Sets cursor placement flag.
- All other keys fall through.

---

## 4. Subroutines

### `forny` – Refresh Subfile
- Empty, placeholder for refilling the subfile.

---

### `subfile` – Handle Selection in Subfile

- Toggles the cursor position indicator `*IN22`.
- Loops through records in the visible subfile page:
    - Reads each record with `READC` (read changed).
    - If end of subfile (`%EOF`), exit loop.
    - If the select field (`b1valg`) is set to '1':
        - Copies code and description from selected line into parameters.
        - Sets flag to indicate successful selection.
        - Exits subroutine.

---

### `clr_subfile` – Clear Subfile

- Sets indicator *IN14 (clear subfile) and writes subfile control record (`b2ctl`).
- Resets all counters to zero, indicator *IN15 (subfile end) to *off.

---

### `crt_subfile` – Populate Subfile (Paging Up)

- Increments the page size (`w_spge`) by 8 (constant), then loops to fill the subfile:
    - For each new record:
        - Sets agreement type code (`b1tkod`) and description (`b1tbes`)
        - Up to 11 possible agreement types are hardcoded.
        - When exceeding 10, sets subfile-end indicator.
    - Writes entries to the subfile.
- Handles record counters for paging.

---

### `bck_subfile` – Page Back Subfile

- Rewinds subfile page counters.
- Calls `clr_subfile` and `crt_subfile` to refresh the display window.

---

### `dsp_subfile` – Display Subfile

- If there are subfile records, sets indicator *IN12 (for display).
- Always sets indicator *IN13 (subfile control). Performs `EXFMT b2ctl` to show the screen and get input.
- Updates the current page/record tracking.

---

### `sett_beskr` – Set Agreement Type Description

- Sets variable `w_tbes` based on given `w_tkod` using a `SELECT` statement.
- For an unknown code, blanks out description.

---

### `*inzsr` – Initialization Subroutine

- Uses *ENTRY to receive parameters.
- Initializes display and flags.
- If a code (`p_tkod`) is passed in, sets request flag and description.
- Otherwise, populates the subfile initially.

---

## 5. Subfile Mechanics and User Interaction

- The subfile (`b1sfl`) displays agreement types for selection.
- User can scroll/paginate (using Page Up/Down keys), select an agreement type (by marking selection field with '1'), or exit.
- The main loop alternates between writing the command key screen and the subfile list, depending on the user's actions.
- On selection, the chosen code and description are output via the program parameters.

---

## 6. Key Indicators

- `LR`: Last record, ends program.
- `10`: Roll Up (Page Down)
- `11`: Rolldown (Page Up)
- `12-15`: Subfile display/clear indicators.
- `21-22`: Home Key/Cursor management.
- `31-79`: Error/warning indicators.
- `80-99`: Work indicators.

---

## 7. Summary of User Flow

1. **Program Starts:** Initializes screen based on parameters.
2. **Display List:** Shows subfile of agreement types.
3. **User Action:**
   - Selects record (marks '1') and presses ENTER.
   - Pages up/down to see other agreement types.
   - Exits with function key.
4. **On Selection:** Program returns selected code and description to the caller.
5. **On Exit:** Program ends and returns control/parameters.

---

## 8. Hardcoded Agreement Types

These are the types available for selection:

| Code | Description             |
|------|-------------------------|
| 001  | Salg (Sale)             |
| 002  | Innkjøp (Purchase)      |
| 003  | Vedlikehold (Maintenance)|
| 004  | Samarbeid (Cooperation) |
| 005  | Kjøp/Salg av eiend. (Prop. Buy/Sell)|
| 006  | Finansiering (Financing)|
| 007  | Prosjekt (Project)      |
| 008  | Lisens (License)        |
| 009  | Leie (Lease)            |
| 010  | Utleie (Renting out)    |
| 011  | Forsikringer (Insurance)|

---

## 9. Additional Notes

- This is legacy, fixed-format RPG (pre-RPG IV / free format).
- Uses `GOTO` and labeled tags instead of structured loops.
- All field names, comments, and labels are mostly in Norwegian.
- Could be refactored to modern free-format RPG for maintainability.

---

### Onboarding Tips

- Focus on understanding subfile mechanics (paging, selection, display indicators).
- The `b1sfl` subfile record and its selection field `b1valg` are central to user interaction.
- The program does no database reads; all rows are hardcoded in the source.
- To extend: Add more agreement types or change navigation logic as needed.
- For debugging: Use the indicators to follow step-by-step user actions and state changes.