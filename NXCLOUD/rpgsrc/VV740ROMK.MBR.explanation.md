# RPG Program: VV740ROMK – Update Item Number (“Varenummer”) Across Multiple Tables

## Purpose
This RPG program is designed to **update item numbers (varenummer)** in all relevant tables except the main item master. It is used in scenarios where an item number needs to be replaced everywhere in the system (due to merges, corrections, etc).

The comments at the top highlight that this program is **manually tailored for each run**: developers comment/uncomment code for only the tables to be updated. It is a copy-based maintenance utility and **should be used with care**.  
**It does NOT update the main item master file.**

---

## Key Features

- **Manual Table Selection:** To update only specific tables, developers must toggle comments in the code for the relevant blocks.
- **Item Number Replacement:** For each selected table, every occurrence of the old item number is replaced with the new one.
- **Audit Trail:** There are updates to modification date, time, and user fields during the update when available in the table (fields like `veedat`, `veeusr`, etc.).
- **Handles Associations:** Handles not only direct item number references but also relations like alternative item, replacement item, notes, etc.
- **Flexible Primary Key:** Uses company/branch and group/item key fields for record selection.

---

## High-Level Structure

### Header/History

- Program options: `*nodebugio` (faster, disables debug I/O), `datedit(*dmy)`
- Extensive header with change logs, describing the evolution and supported tables.

### File Declarations

- **Many physical files** (`F` specs) with **rename** clauses, e.g.
  ```
  fvvenlu uf e k disk rename(vvenpfr:vvenlur)
  ```
  - `uf`: update file  
  - `e`: externally described  
  - `k`: keyed  
  - `rename`: use a record format name
- **Files are commented by version**, enabling easy tracking of table support additions.

### Data Definitions

- **Program variables for parameters and working fields**
- **Key lists** for table searches
- **User field** for audit updates

### Main Logic

#### Initialization

- **Parameter assignment**: assigns procedure parameters to local working fields.
- **Loads associated data** as needed (e.g., retrieves module number for an item for certain tables).

#### Table Update Loops

For each table (file), the code block typically follows this pattern:

```rpg
key01       setll     vvenlu      // Set LL to the item using key list
key01       reade     vvenlur 90  // Read first occurrence, set indicator 90 on EOF
dow         *in90 = *off          // While not EOF
  eval      vevare = p_vare       // Replace item number field with new value
  time                  veedat    // Update modification date (if available)
  time                  veetim    // Update modification time
  eval      veeusr = l_user       // Update modification user
  update    vvenlur               // Write record back to file
  key01     reade     vvenlur 90  // Read next occurrence
enddo
```

- **Most table updates are commented out**; uncomment those you want to process.

#### Special Handling

- **Note and relation tables**: extra keys and logic (e.g., `anotlu` table uses system id and group/item).
- **Module number**: Some tables require updating related module numbers if non-zero (e.g., sortiment and price group).
- **Audit/Tracking**: Always updates last-changed fields where available.

#### End and Cleanup

- **Check `p_inlr` parameter**: If not set, sets *INLR (last record indicator) to signal program completion/termination.
- **Return from program**.

---

## Subroutines

### *INZSR (Initialization)
- Handles **entry parameter list** and **builds key lists** for file access.

---

## Parameters

When called, the program receives:

| Name    | Type      | Purpose                               |
|---------|-----------|---------------------------------------|
| p_firm  | 3p0       | Firm/Company code                     |
| p_gvar  | 15A       | Group or item group identifier        |
| p_vare  | 15A       | New item number to assign             |
| p_inlr  | 1A        | Flag: Should program set *INLR = *on? |

---

## Typical Workflow

1. **Set parameters** (company, group, new item number, *INLR flag).
2. For **each selected table**:
    - Locate records with old item number.
    - Replace with new item number (and update mod/audit fields).
    - Write back the record.
3. Special tables (like alternate items, notes, etc.) are handled with their own specific key lists.
4. **Program ends** by optionally setting *INLR according to input.

---

## For New Developers

- **Do NOT run this program as-is**: uncomment the code block(s) for the table(s) you want to update, and (re-)comment out the rest.
- Review and adjust key lists as needed for your data.
- Always test on a copy first – this code indiscriminately changes **all occurrences** in the selected tables for the given key(s).
- **BACKUP your data** before using.
- The program should be run by someone who understands the data relationships in the application.
- After run, verify that required changes are made and that no duplicate keys or referential issues have been introduced.

---

## Summary Table Block Example

```rpg
c*     key01         setll     vvenlu
c*     key01         reade     vvenlur                                90
c*                   dow       *in90 = *off
c*                   eval      vevare = p_vare
6.10 c*                   time                    veedat
6.10 c*                   time                    veetim
6.10 c*                   eval      veeusr = l_user
c*                   update    vvenlur
c*     key01         reade     vvenlur                                90
c*                   enddo
```
- Replace item number field (`vevare`) in each occurrence for that table.

---

## Caution

- **Highly destructive if misused**: Only run after careful selection of relevant table(s) and thorough understanding of impact.
- **Does not update primary item master file**; it assumes the item master has been handled already or separately.

---

## Final Words

This program is a last-resort, batch-processing maintenance tool, built for legacy environments and requiring direct intervention. Use only if you know *exactly* what you are doing, and never in production without full backup and approval.