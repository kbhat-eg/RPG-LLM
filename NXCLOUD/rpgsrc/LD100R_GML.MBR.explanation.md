# RPG Source Code Explanation: LD100R (Bestilling-linje, Registration of Purchase Orders)

---

## General Overview

This program, `LD100R`, is a critical part of the purchasing module, handling the creation, maintenance, and synchronization of purchase order lines ("bestilling-linje"). It's designed for IBM i systems and written in ILE RPG IV, with enhancements and bugfixes across many years (as indicated by the detailed change log). 

The program allows users to:

- Register, update, and delete purchase order lines.
- Perform various lookups (items, order types, suppliers, etc.).
- Handle subfile-driven display/editing of order lines.
- Synchronize purchase orders with sales orders when appropriate.
- Log changes and notify relevant users (such as salespeople) if related data is modified.

---

## Structure

### 1. File Definitions

The F-specs define all the database files and display files the program uses, including purchase order header/line files, product master files, synchronization registers, and display files for subfiles.

- Most files use the `k` (keyed), `e` (externally-described), and `disk` or `workstn` (workstation) keywords.
- Some files are renamed at open-time to distinguish between different logical views or usage in the program.
- The display file (`fld100d`) uses subfiles (SFILE) and information data structures (INFDS).

### 2. Data Definitions

- **Local Data Area (LDA):** Used to store information across CL and RPG.
- **Data Structures:** For EAN128 barcodes, stock update structures, and display feedback.
- **Work Variables:** For form fields, calculations, state flags, and keys for file access.
- **Constants:** For common string values (alphabet, digits), initialized for use in processing.

### 3. Main Logic and Subroutines

The program is structured in classical RPG style, with:

- **Initialization Section (`*inzsr`)**
- **Mainline Processing (`d1tag`)**
- **Various Subroutines** for tasks such as file management, subfile handling, record validation, order synchronization, and logging.

---

## Main Workflow

### Initialization (`*inzsr`)

- Receives parameters (status, company, order number, suffix, etc.).
- Reads user data, status codes, and sets flags for cost price access, EAN display, etc.
- Gets synchronization info and configures program behavior based on company/customer configuration.
- Performs any one-time lookups or settings needed before entering the main user loop.

### Display and Entry (`d1tag`, `d1taga`, etc.)

- **Subfile Handling:** Reads and displays the current set of order lines using subfile logic, with support for paging, adding, editing, deleting, and viewing lines.
- **Function Key Handling:**
  - **F1:** Toggle totals view.
  - **F2:** Insert text line.
  - **F3:** Commit changes and optionally print/send orders or synchronize with sales orders.
  - **F4:** Item lookup.
  - **F6:** Retrieve campaign items.
  - **F9/F10/F11:** Item, supplier, and fixed information lookup/maintenance.
  - **F14, F20, F21, F23:** Specialized vendor/information toggles and advanced functions.
  - **Special values (e.g., `b2valg`)**: Provide 'quick-action' options for cloud/client efficiency.

### Adding or Modifying Lines

- **Validation:** Checks that line type and item are provided and correct, with detailed validation for item numbers (alphanumeric, EAN, supplier item number, etc.).
- **Warehouse Check:** Ensures that the item is a legitimate inventory item.
- **Registration:** Calls the appropriate external programs for registering lines or text, updating state flags as necessary.

### Subfile Operations

- **Add, Edit, Delete, View:** Each operation is handled in its own subroutine, updating/inserting/deleting from the purchase lines file and, when necessary, the inventory and synchronization files.
- **Synchronized Updates:** When the purchase order is linked to a sales order, changes can be synchronized automatically, with checks for user rights, locking, and data consistency.

### Order/Inventory Synchronization

- **Order-Sync Eligibility:** Complex logic determines whether lines can be synchronized with corresponding sales order lines, respecting business rules around locking, suffix logic, internal/external order types, etc.
- **Update/Notify:** If changes are made and the user has access, the sales order is updated; if not, the sales rep is notified via email/log entry.

### Logging

- **Change Logging:** When changes affect linked sales orders, or when synchronization fails due to locking or other errors, logs are written for future auditing and user notification.
- **Email Notification:** When appropriate, notifications are sent to responsible parties.

### Miscellaneous

- **Weight/Volume Calculation:** For logistics and pricing, many routines calculate the total weight and volume for the order or individual lines.
- **Input Field Cleaning:** Handles special cases, e.g., cleaning up display fields for compatibility with special UI tools (like NewLook).

---

## Key "Interesting" Technical Points

- **Flexible Item Identification:** Recognizes items by multiple numbering schemes, including supplier number, EAN-128, padded item numbers, etc.
- **Advanced Synchronization:** Carefully manages the complex logic around when a purchase order and a sales order can/should be kept in sync, including user permissions, order status, and different types of changes (quantity, pricing, etc.).
- **Subfile-Driven UI:** Classic RPG subfile pattern supports efficient data entry, review, and editing, with support for keyboard navigation and function key shortcuts common in IBM i applications.
- **Robust Logging/Notification:** Aggressively logs and notifies when business rules are violated, access is restricted, or synchronization fails.

---

## Change Log / Maintenance Notes

The source contains a **very thorough change log**, indicating the maturity and criticality of this code. Each change is labeled with a version marker and short summary (often with a developer’s initials), making it easier to trace the evolution and reason about legacy logic.

Many subroutines and code blocks are marked with these versioning comments (e.g., `6.21`, `7.04`; sometimes with full blocks added for new features, bug fixes, or customer-specific adaptations).

---

## Best Practices & Onboarding Tips

- **Understand the Data Models:** The logic is tightly coupled to detailed, multi-file data relationships (orders, order lines, items, status codes, users, etc). Make sure you understand the data dictionary and key fields.
- **Follow the Comments/Changelog:** There are many special-case routines and customer-specific tweaks. Use the change log and inline comments to avoid reintroducing old bugs or breaking compatibility.
- **Use Provided Keys for File Access:** All keys are defined at the end of the source, making code updates safer and more readable.
- **Leverage Subroutines:** Most logic is encapsulated in named subroutines, making it easier to read, reuse, and test.
- **External Program Calls:** Watch for side-effects and required external programs that must be present/compatible (e.g., `LD105R`, `LO730R`, `VL710R`, etc.).
- **Testing:** Given how central this code is, strong regression testing, especially around order synchronization and user permission checks, is critical.

---

## Summary

**LD100R** is a robust, mature program for managing purchase order lines on IBM i, with deep integration into item, supplier, and sales order management. It supports advanced workflows including real-time synchronization, detailed validation, and extensive audit logging, all operated through a subfile-based interactive interface. Its complexity comes both from business rules and technical debt accumulated over years of real-world use, but it is well-commented and logically structured.

New developers should start by tracing the typical workflow (initialization → subfile interaction → order update/sync → logging), and then dive into subroutines for details. Always verify changes against the documented change log and test against edge-case business scenarios.

---
**If you want more detailed explanations of specific subroutines, file layouts, or business rules, please request that specifically!**