## Overview

This RPG program acts as a bridge between IBM i (AS/400) and a Java-based export service, specifically for exporting or deleting order data. It receives input parameters (order info, connection details, etc.), instantiates a Java object to perform the export/delete action, and logs the outcome using a custom messaging program (`XP900C`). Exception handling is robust, with integration to IBM i APIs for message retrieval.

---

## Structure and Key Components

### 1. **Program Attributes**

- `DFTACTGRP(*NO) ACTGRP(*NEW)`: Runs in its own activation group; required for Java integration.
- `option(*nodebugio)`: No debug for file I/O.
- `datedit(*dmy) decedit(',')`: Locale-specific date/decimal handling.

---

### 2. **Parameters and Variables**

#### **Input Parameters**
- `p_filg` (10a): File group or identifier.
- `p_firm` (3a): Company code.
- `p_numm` (8a): Order number.
- `p_suff` (2a): Order suffix.
- `p_type` (1a): Action type (`'D'` = delete, otherwise export).
- `p_addr` (30a): Host address for the Java service.
- `p_port` (5a): Port number.

#### **Error Handling**
- `w_errrun` (1s 0): Prevents recursive error handling.
- `PgmError` (SDS): Standard program error data structure, including job, routine, and message context.

#### **Message API Variables**
- For interfacing with IBM i API `QMHRCVPM` to retrieve last message details in case of errors.

#### **Java Integration Variables**
- Java object references for interacting with the Java export service (`no.asp.as.nexsteptrigger.ExportDocument`).
- Constructor and method prototypes for Java object creation and method calls.
- Helper variables for string conversion between RPG and Java.

#### **Other Variables**
- `w_head`, `w_text`, `w_ttyp`: For composing and categorizing messages sent to the messaging program.

---

### 3. **Java Integration**

#### **Key Java Classes/Methods**
- `ExportDocument.getInstance()`: Singleton/factory for service interaction.
- `ExportDocument.export(...)`: Main business method for exporting or deleting orders.
- `java.lang.String`: Used for parameter passing and string conversion.

#### **Parameter Preparation**
- All RPG parameters are trimmed and converted to Java `String` objects using `mkstr`.
- Static values for user (`'jms'`) and password (`'qwerty'`).
- Operation type (`DELETE` for deletion, blank otherwise).
- Data type is always `'ordre'`.

---

### 4. **Main Program Flow**

1. **Java Object Creation**
    - Instantiate the export service object and prepare all parameters as Java strings.

2. **Determine Action**
    - If `p_type = 'D'`, set the command type to `'DELETE'` (for deletion); otherwise, it is left blank (for export).

3. **Call Java Export Method**
    - Calls the Java method `export` with all relevant parameters.

4. **Success Handling**
    - On successful call, compose a success message and call `XP900C` to log the event.

5. **(Commented Out) Manual HTTP Handling**
    - There is legacy/commented-out code for constructing and sending HTTP requests directly. This is superseded by the Java service integration.

---

### 5. **Error Handling**

- **Custom Error Routine (`*pssr`)**
    - If an unhandled exception occurs, the program:
        - Calls the subroutine `seterrtxt` to retrieve the last message from the job log using `QMHRCVPM`.
        - Formats error details and sends an error message via `XP900C`.
        - Sets `w_errrun` to prevent recursion.
    - Does not halt on error (no `GOTO avslutt`), but logs and continues to return.

- **`seterrtxt` Subroutine**
    - Calls `QMHRCVPM` API to retrieve the last message for diagnostic purposes.
    - Extracts and trims the message, storing it in `w_text`.

---

### 6. **Initialization (`*inzsr`)**

- Maps the program parameters to the working variables at program start.

---

### 7. **Messaging and Logging**

- **`XP900C` Program**
    - All significant events (success or error) are logged by calling this external program.
    - Parameters: message header, message text, message type (`'E'` for error, blank for info).

---

## Business and Domain-Specific Logic

- **Order Export/Deletion**
    - The program is used to trigger export or deletion of order data by invoking a Java-based service.
    - The action is determined by the `p_type` parameter.
    - Parameters are passed through as-is from the RPG caller, with minimal validation or transformation.

- **Integration Patterns**
    - Java integration is preferred over direct HTTP handling, likely for maintainability and encapsulation.
    - All communication with the export service is abstracted behind the Java interface.

- **Error Diagnosis**
    - Uses IBM i system APIs to capture detailed error context for support and troubleshooting.
    - Error handling is defensive to avoid infinite error loops.

---

## Design Patterns and Conventions

- **Java Service Wrapper**
    - All business logic related to order export is delegated to a Java service, keeping the RPG layer thin and focused on orchestration.

- **Parameter Passing**
    - All parameters are explicitly trimmed and converted before passing to Java, ensuring data cleanliness.

- **Message Logging**
    - Centralized logging via `XP900C` ensures all events are consistently recorded.

- **Commented Legacy Code**
    - Retains old HTTP handling logic for reference or rollback, but is not active in the current design.

- **Exception Handling**
    - Standard RPG error handling using `*pssr` and system data structures, with additional API calls for message retrieval.

---

## Interactions with Other Modules

- **Java Package: `no.asp.as.nexsteptrigger.ExportDocument`**
    - Main integration point for export operations.

- **IBM i API: `QMHRCVPM`**
    - Used for retrieving job messages in error situations.

- **Program: `XP900C`**
    - Handles all message logging and notification.

---

## Summary

This program is a tightly focused integration layer between IBM i order processing and an external Java-based export/deletion service. It is designed for robustness, with strong error handling and standardized logging. All business logic is delegated to the Java layer, and the RPG code is primarily responsible for parameter preparation, orchestration, and error reporting. Legacy code for direct HTTP handling is retained but inactive, reflecting the evolution of the integration strategy.