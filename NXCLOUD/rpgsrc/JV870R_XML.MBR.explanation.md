# IBM ILE RPG Source Code Explanation  
## Program: JV86AR_XML  
**Description:**  
This program updates or creates supplier records (leverandør) linked to products (varer) in the NOBB 80 (XML format) system. It handles both product-supplier and supplier media information, logging operations, and error handling.

---

## High-Level Structure

### 1. **Header & Documentation**
- The `h` spec: Sets options for debug and date formatting.
- The extensive header documentation: Change log, description, and notes (in Norwegian).

---

### 2. **File Specifications (`F` Spec)**
- Files represent different tables:
  - `jvlelu`, `jvlel1`: Product-Supplier tables (main, logical view).
  - `jvlmlu`, `jvlml1`: Supplier Media tables.
  - `jstsl1`: Status lookup.
  - All files have error subroutine `FileErr` for error handling and custom rename for clarity.
- Files are defined as update (uf), input (if), keyed (k), external (e), etc.

---

### 3. **Data and Variable Declarations (`D` Spec)**
- **Message Table:**  
  - `a_meld`: Array for messages (could be for logs or user feedback).
- **Parameters:**  
  - `p_lrec`: Input product-supplier record (max 200 chars).
  - `p_lmrec`: Array of 25 media records (for linked media).
  - `p_job_log`, `p_inlr`: Job logging and RPG’s "last record" indicator.
- **Job Log Structure:**  
  - `d_job_log`: 150-character data structure, with overlays for various counters and fields, e.g., product numbers, supplier IDs, counters for written/updated records.
- **Supplier Record Data Structure:**  
  - `d_lrec`: 200-character overlay structure for unpacking different data fields (supplier number, name, owner flag, product numbers, expiry date, markings).
- **Supplier Media Data Structure:**  
  - `d_lmrec`: 120-character overlay for media type and URL.
- **Key Variables:**  
  - Used for file operations (read/update).
- **User/Firm:**  
  - Local data area variables for the operator and firm code.
- **Exception Data Structure:**  
  - `PgmError`: System Data Structure (SDS) for runtime error information (line number, file, job info, etc.).

---

### 4. **Main Program Logic (`C` spec)**
#### **Initialization**
- Loads the job log and job ID from parameters.
- If `p_inlr = *on` (probably meaning "process"), moves input record `p_lrec` into local structure and processes it if it has a valid NOBB number.

#### **Main Product-Supplier Update (`skr_leve` subroutine)**
- **Checks Error Flag:**  
  - If set, exits immediately.
- **Clears previous output structure.**
- **Loads keys (product number, supplier number) from input.**
- **Checks for Existing Record:**  
  - Reads logical file for product-supplier (update mode).
- **Populates output record fields from input structure.**
- **Handles Update or Create:**  
  - If found, update it and increment updated counter.
  - If not found, create a new record and increment created counter.
- **Handles Media Links:**  
  - Calls subroutine to delete existing media links, then loops through possible media records and writes valid ones via `skr_lemed`.

---

#### **Supplier Media Subroutines**
- **`skr_lemed`:**  
  - Populates media record fields and writes them to media table.
- **`del_lemed`:**  
  - Reads and deletes all media links for the current product-supplier.

---

#### **File Error Handler (`FileErr` subroutine)**
- Sets status and constructs an informative error message (file name, exception data, job info).
- Calls an external program `'AB705R'` to log/handle the error and then exits the program.

---

#### **Program Initialization Subroutine (`*inzsr`)**
- Sets up parameter list.
- Defines the keyed lists (klist) for all file operations.
- Initializes firm code from local data area.

---

## **Supplemental Notes**

### **Parameter Passing**
- The program expects to be called with:
  - A product-supplier record (string)
  - An array of supplier media records
  - A job log string
  - An INLR flag for last record/processing

### **Error Handling**
- Exception/Errors are centralized in `FileErr`, which constructs a detailed error message and calls another program for handling.

### **Record Handling**
- All I/O (read, update, write, delete) uses RPG’s built-in file/data record operations using key lists for file access.

### **Comments and Versioning**
- Extensive inline comments in Norwegian.
- Code tagged for specific revisions and incremental changes (e.g., `6.30` and `6.31`).

---

## **Typical Program Flow**
1. **Entry:** Receives a product-supplier record and associated media.
2. **Data Unpack:** Loads/parses fields from the input structures.
3. **Check/Update/Create:**  
   - If the record exists: Update and count.
   - If not: Create and count.
4. **Media References:**  
   - Deletes old media for the product-supplier.
   - Loops through all supplied media records and writes new ones.
5. **Error/Job Logging:**  
   - Errors routed to handler.
   - Job log fields updated for reporting.
6. **End/Exit:**  
   - Sets *INLR and exits.

---

## **Key Takeaways**

- **Modular design:** Uses subroutines for logical operations (main update, media handling, deletion, errors).
- **Field overlay structures:** Makes parsing inbound variable-length records efficient.
- **Comprehensive error & job logging** for traceability.
- **Easily adaptable** due to parameter-driven processing and table-driven I/O.

---

## **Who Should Read This?**
- Developers supporting NOBB supplier/product integration.
- RPG programmers onboarding this interface/program module.
- Operators troubleshooting supplier update jobs.

---