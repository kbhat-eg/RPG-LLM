```markdown
# RPG Program Explanation: ASSHOP, Program BOHNTR

This RPG (ILE RPG, likely RPG/400 or RPG IV / RPGLE) program is designed for the **ASSHOP** system, specifically for the `BOHNTR` program. Its main functionality is to fetch data from other systems, such as order and shop receipt (bong) records, in a shop/retail environment. The code uses workstation IO and interacts with other programs for data retrieval.

---

## Overview of Program Structure

- **File Specification**
    ```rpg
    fbohntd    cf   e             workstn
    ```
    - Defines a workstation file (likely a display file called `BOHNTD`) for user interaction.

- **Data Definitions**

### Parameters (used for passing values between programs):
```rpg
d p_retu          s                   like(w_retu)
d p_firm          s              3  0
d p_aarr          s                   like(w_aarr)
d p_wsid          s                   like(w_wsid)
```
- `p_retu`: return code/status indicator
- `p_firm`: firm/company number (3-digit)
- `p_aarr`: year (format matches w_aarr)
- `p_wsid`: workstation ID (format matches w_wsid)

### Working Variables:
```rpg
d w_retu          s              1
d w_wsid          s             10
d w_aarr          s              4  0
d w_numm          s              6  0
d w_suff          s              2  0
```
- Used for holding intermediate data for orders/receipts.

---

## Main Logic Flow

### **Main selection loop (`a1tag` and `a1tagb`):**
#### Tags are used to control flow, classic in RPG.

1. **Display Menu Window:**
    ```rpg
    c                   exfmt     a1win
    ```
    - Shows a menu/selection screen to the user.

2. **Function Key Handling:**
    ```rpg
    c                   if        *inkc = *on or
    c                             *inkl = *on
    c                   eval      p_retu = '9'
    c                   goto      avslutt
    c                   endif
    ```
    - If the user presses function keys (probably Exit/Cancel), set `p_retu` to '9' and exit the program.

3. **User Selection Processing:**
    - If user selects 1: call the subroutine to fetch order data (`hent_ordre`).
    - If user selects 2: call the subroutine to fetch shop receipt/bong data (`hent_bong`).
    - Otherwise, re-display the menu.

### **End Program (`avslutt` tag):**
```rpg
c                   eval      *inlr = *on
c                   return
```
- Sets last record indicator (end of program) and returns.

---

## Subroutines

### 1. **hent_ordre** — Fetch Order Data
- Calls program `BOHHOR` to query for an order (user enters order info).
- If not cancelled (`w_retu <> '9'`), calls program `BOHNOR` to fetch the actual order data.

### 2. **hent_bong** — Fetch Shop Receipt ("bong") Data
- Calls program `BOHHBR` to query for shop receipt info (user enters receipt info).
- If not cancelled (`w_retu = *blank`), calls program `BOHNBR` to fetch the receipt data.

### 3. **Initialization Routine (`*inzsr`)**
- Handles program entry and parameter reception (`*entry plist`).
- Receives program parameters for the session: `p_retu`, `p_firm`, `p_aarr`, `p_wsid`.

---

## Call/Parm Details

- `call 'ProgramName'`
    - Calls an external program.
    - `parm` statements specify parameters passed to these programs.

---

## Summary of Data Flow

- The user is presented with a menu to choose between fetching by **Order** or **Shop Receipt/Bong**.
- Depending on their selection, they are prompted for relevant data.
- The program calls external modules both to ask the user for data (query) and to retrieve the actual data.
- Exit is possible at any time via function key.

---

## Key Points for Onboarding

- **User Interface:** Handled via a display file (`fbohntd`), with a selection window and function key handling.
- **Inter-program Communication:** Uses `CALL` and `PARM` for invoking and passing data to external programs (BOHHOR, BOHNOR, BOHHBR, BOHNBR).
- **Subroutine Usage:** Business logic (fetching order/receipt) is encapsulated in subroutines (`hent_ordre`, `hent_bong`).
- **Exit Handling:** Uses function keys, sets `*inlr = *on` for orderly program termination.
- **Parameter Handling:** Program is likely called from another program, passing in key context data.

---

## Common Norwegian Terms Used:
- **Henter inn fra andre systemer**: "Fetches from other systems"
- **Butikk-bong**: Shop register receipt
- **Ekspedisjon fra håndterminal**: Processing from handheld terminal

---

## Next Steps

- For maintenance, focus on adjusting subroutine logic or external program calls.
- For enhancements, modify menu options or expand functionality in the subroutines.
```
