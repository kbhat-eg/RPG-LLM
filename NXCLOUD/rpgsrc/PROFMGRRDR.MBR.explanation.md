# Explanation of the RPG Source Code (PROFMGRRDR)

## Purpose

This ILE RPG program (`PROFMGRRDR`) is designed to process messages from a message queue (described as "Kafka"), handling messages related to profiles, articles, and rules. It extracts relevant data from JSON messages and updates or inserts records into corresponding DB2 tables.

---

## Structure Overview

### Compilation

- **Compile1 and Compile2**: Comments at the top show how to compile the code into a module and link it into a program using standard IBM i commands.
- **/copy QRPGLESRC,MSGRDRPR**: Includes external definitions such as data structures and prototypes (especially `QueuedMessage`).

---

## Main Procedure

### Control Options

```rpg
ctl-opt PGMINFO(*NO);
```
- Disables inclusion of program info in compiled object, for smaller/faster objects.

### Variable and Data Structure Declarations

- Many `dcl-s` and `dcl-ds` statements declare variables and data structures for holding message and extracted data.
- Examples: `msg` (incoming message data structure), `w_messagedata` (raw JSON from the message), and various "w_" variables for extracted fields.

---

### Library List Management

The program initially **resets and builds the library list** to ensure the correct environment for SQL/database access.

1. Remove all libraries from the user portion of the library list:
    ```rpg
    command = 'CHGLIBL LIBL(*NONE)';
    exec sql CALL QSYS2.QCMDEXC(:command) ;
    ```
2. Sequentially add required libraries: `QGPL`, `QTEMP`, `ADB`, `NXKORR`, `NXCLOUD`, `RWUTIL`.
3. Each change is committed if successful.

---

### Message Processing

#### Subscription to Message Channel

```rpg
consumer = 'Profile_Manager_Consumer';
SubscribeMessages('profiles-output-channel':consumer:myProcPtr);
commit;
return;
```
- Subscribes to the `profiles-output-channel` as `Profile_Manager_Consumer` using a callback-style function (`MyProc`).

---

## Internal Procedure: `MyProc`

This is the main **callback** called for each message received.

### Parameters

- `message` (LIKEDS(QueuedMessage)): Contains key information (`messageKey`, `messageData`, status info).

### Processing Steps

1. **Error Handling**:  
   If `message.status = 500`, the procedure exits immediately (error from upstream).

2. **Pre-processing**:
   - `w_messagedata` is assigned the message's JSON data.
   - Null values in JSON are replaced with `"x"` (to simplify downstream processing).

3. **Extract File Group**:
   - If `messageKey` contains a dash (`-`), the 6 characters after the dash are used as a file group (`w_filgrp`).
   - The library list is updated to use this file group as the current library.

---

### Data Extraction and Database Processing

Depending on the key (`messageKey`), the handler processes profiles, articles, or rules.

#### 1. **Profiles**

- If `messageKey` contains `'profile'`:
    - Uses **SQL JSON_TABLE** support to extract fields from the JSON message, mapping them to RPG variables.
    - Attempts to **update** a record in `prmpst` table.
    - If **update does not find a row** (`sqlcod = 100`), attempts an **insert**.
    - Every successful database write is followed by a `commit`.

#### 2. **Articles**

- If `messageKey` contains `'article'`:
    - Extracts article info from JSON.
    - Updates or inserts into the `prmast` table, as above.

#### 3. **Rules**

- If `messageKey` contains `'rule'`:
    - Extracts rule info from JSON.
    - Updates or inserts into the `prmrst` table.

#### In all cases:
- On any SQL error (non-zero `sqlcod`, except for expected 100/Not found), the procedure exits early.

---

## Key Concepts Illustrated

- **Dynamic SQL** and **library management** at runtime.
- **Use of JSON_TABLE** to parse JSON directly within SQL (DB2 for i feature).
- **Upsert behavior**: update if found, insert if not.
- **Commit control**: Ensures all updates/inserts are committed immediately.
- **Message-driven programming**: Each incoming message triggers a callback (`MyProc`), which processes and persists data.
- **Callback model**: `SubscribeMessages` is assumed (from included prototypes) to work with a callback pointer.

---

## Key Variables

- `w_firm`, `w_filgrp`: Used for partitioning data by firm and file group.
- `w_prof_id`, `w_code`, etc.: Profile attributes extracted from JSON.
- `w_externalNumber`, `w_name`, etc.: Article attributes.
- `w_rule_id`, `w_rule_name`, etc.: Rule attributes.
- `sqlcod`: Standard SQL status code, checked after every DB operation.
- `messageKey`, `messageData`: Metadata and content from the incoming Kafka/message queue.

---

## Error Handling

- Exits the procedure on:
  - SQL errors (non-zero and non-100 result)
  - Upstream error in the message (`status = 500`).
- Only commits when an operation has succeeded.

---

## Typical Flow

1. **Set up environment (library list)**
2. **Subscribe to messages**
3. For each message:
   - Determine message type (profile, article, rule)
   - Parse JSON
   - Update or insert into the right table
   - Commit changes
   - Early exit on errors or handled conditions

---

## Extensibility

- New message types (i.e., new branches under `if %scan(...)`) can be added easily.
- Table structure and JSON field mappings can be extended as needed.

---

## Comments & Documentation

- The code is well-documented with headers and change logs, suitable for team development.
- Comments indicate customer-specific changes and describe the business logic.

---

**Summary:**  
This RPGLE program is a robust, message-driven integration tool that processes JSON-formatted events from a message queue, parses them using DB2 for i JSON support, and synchronizes them with the local database, supporting upsert logic and dynamic environment setup. Ideal for onboarding developers involved with IBM i event-driven architectures.