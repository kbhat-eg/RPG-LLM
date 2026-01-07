# IBM ILE RPG Program JC100R: Explanation

This RPG program ("JC100R") is a comprehensive interactive maintenance tool for managing surcharges ("påslag") and related data, leveraging both native database access and subfiles for user interaction. Its structure, conventions, and functionality are typical of advanced RPG/400 applications in ERP or logistics settings.

---

## 1. **Header and Documentation**

- The **header** section includes important metadata: system, program name, description, version history, and indicator usage guidelines.
- Documentation is in Norwegian—key functions include surcharge maintenance and navigation by "prisgruppe", "varegruppe", "leverandør", etc.

---

## 2. **File Declarations**

- **Physical & logical files** (`fjpfalr`, `fjpfalu`, etc.): These refer to various database tables/physical/logical files, mostly with `R` or `L` suffixes (e.g. `jpfalr`, `jlevl1`), many renamed for program-internal clarity.
- **Workstation file** (`fjc100d`): Used for interactive display (DDS).
- **Sfile(b1sfl:w_srrn)**: Main subfile for listing surcharge records in the display.

---

## 3. **Data Structures and Variables**

- **Local Data Area fields**: For user, firm and file navigation.
- **Information Data Structure**: Captures screen feedback, e.g. cursor row.
- **Key variables** for logical/physical file access by business entity (prisgruppe, gruppe, leverandør, vare, priskode, etc.).

### **Working Variables**
- For managing subfile row numbers, navigation, flags for operation mode (`b_oppr`, `b_forn`, etc.), and field values.

### **Constants**
- E.g. `c_sfil`: subfile page size.

---

## 4. **Screen and Subfile Management**

- **Subfile workflow** is central: the user interacts with a paged list (subfile) of records, with actions ("valg") for change, copy, delete, or view.
- **Subfile Positioning**: The code supports navigation and positioning based on various keys (prisgruppe, overgruppe, modul, vare, etc.), ensuring efficient access and display.
- **Function keys** (e.g. F1, F10, F11, F21) mapped to corresponding subroutines for filter, next/previous page, etc.

---

## 5. **Main Loop and User Interaction**

- **Entry point tags** (`b2taga`, `b2tagb`) begin main loop.
- **Display and function key handler**: The user is presented the subfile, and on key press, the program evaluates the input and branches accordingly.
- **Navigation/Positioning**: If relevant fields are entered (e.g. group or code), the program will position the subfile accordingly.

---

## 6. **Subroutines (Subrs)**

### **forny**
- Refresh subfile after change.
- Uses sequence (`w_seqe`) to determine how to position (e.g. by group, module, etc.).
- Executes `clr_subfile` and `crt_subfile` to clear and refill the subfile.

### **posisjoner**
- Subfile positioning logic, based on which filtering/positioning fields are entered, updating sequence and keys appropriately.
- After positioning, clears and refills the subfile, then resets fields.

### **subfile**
- Main handler for subfile operations:
    - Iterates through visible subfile rows.
    - On "valg" field (action code), calls routines for edit (`xc2bld`), copy (`xk1win`), delete (`xd1win`), view, or display related items.
    - Handles updating of subfile after changes.
    - Integration with subroutines for logistics logic, e.g. for identifying "logistikkvare".

### **xc1bld, xc2bld, xd1win, xk1win**
- Display/modal management for creating, editing, deleting, and copying records, including user validation, feedback, and field population.

### **spørring**
- Handles queries/lookups for various fields (groups, suppliers, modules, item number, etc.), calling external RPG programs for lookup dialogs.

### **sjekk_input**
- Validates entered fields by checking existence/validity in relevant files.
- Ensures relationships (e.g. overgruppe and undergruppe) are consistent.
- Retrieves descriptions for codes and sets error indicators upon validation failure.

### **hent_info**
- Populates description fields (text labels) for the current data context (group name, supplier name, product description, etc.).

### **clr_subfile, crt_subfile**
- Clear subfile and refill with new data (possibly filtered), handling paging and subfile row management.

### **bck_subfile, dsp_subfile**
- Handle paging back (previous page) and displaying subfile, including preservation of the current position.

### **finn_logivar**
- Checks if an item is a logistics item ("logistikkvare") for special pricing logic.
- Uses embedded SQL to retrieve data.

---

## 7. **Initialization**

- **INZSR**
    - Defines key lists for all file accesses (for CHAIN, SETLL, etc.).
    - Loads firm from LDA.
    - Initializes screen for first display and fills the subfile with initial data.

---

## 8. **Error and Message Handling**

- Uses a set of indicators (`*in31`–`*in79`) for field validation errors and screen feedback.
- Certain subroutines display error or info dialogs (`xc1msg`) if, for example, the user tries to create a duplicate row.

---

## 9. **Filter Functionality (F1)**
- A dedicated screen for advanced filtering (subroutine `xf1win`), with multi-field validation and error handling.

---

## 10. **External Program Calls**

- For group, item, code, and supplier lookups, the program calls out to dedicated RPG dialog programs (`VG510R`, `VG511R`, etc.) with the current context.

---

# **Summary for New Developers**

- **JC100R** is a maintenance program for surcharges organized by codes such as group, supplier, module, item, and more.
- It uses RPG’s full-screen display capabilities (subfiles) and supports paging, filtering, and CRUD operations with robust validation.
- There are extensive mechanisms for field validation, context-sensitive lookups, and user feedback.
- Navigation is heavily based on "positioning" to a given group, item, or other code; the user is always working within a subfile.
- The code separates logic into well-named subroutines for maintainability.
- Familiarity with legacy RPG, subfiles, and Scandinavian ERP conventions will help in onboarding.

---

### **Key Takeaways**
- **Subfile-centric interactive maintenance program.**
- **Supports navigation, filtering, and operations on surcharges (“påslag”) by richly structured business keys.**
- **Integrates with other database files and programs for lookups and validation.**
- **Robust validation and user feedback via indicator fields and dialog screens.**
- **Well-commented and version-controlled, but comments and some variable names are in Norwegian.**

---

### **Suggested Onboarding Actions**

1. **Understand the database structure**—study the logical/physical files referenced.
2. **Familiarize with the user interface screens** (DDS layouts, field names).
3. **Review subroutine flows**—especially subfile handling and field validation.
4. **Note integration points**—external programs for lookups, SQL usage, etc.
5. **Learn indicator conventions** for error/warning/messages.
6. **Consult Norwegian resources or team members** for business terms.

---

*This should provide a thorough foundation for understanding, maintaining, or extending this RPG program.*