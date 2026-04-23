# COBOL-SEQ3000 – Sequential & Indexed File Maintenance

**Course:** COBOL Programming – Sequential & Indexed File Assignment
**Author:** [Gabe Dilley](https://github.com/gawdilley)
**GitHub:** [COBOL-SEQ3000](https://github.com/gawdilley/COBOL-SEQ3000)

---

## Description

This repository contains three COBOL programs that together demonstrate two approaches to maintaining an employee master file — sequential file maintenance and indexed file maintenance. The programs handle real-world file update operations: adding new records, changing existing ones, and deleting records that are no longer needed, all driven by a transaction file. An error transaction file captures any invalid or unmatched updates.

---

## Programs Overview

| Program | Purpose |
|---------|---------|
| `SEQ3000.cbl` | Sequential file maintenance — merges a transaction file against a sequential old master to produce an updated new master |
| `EMPIND01.cbl` | Indexed file creation — converts a sequential employee file into an VSAM-style indexed file |
| `EMPIND02.cbl` | Indexed file maintenance — applies transactions directly to an indexed file using random access |

---

## What Each Program Does

### SEQ3000 — Sequential File Maintenance

SEQ3000 implements the classic sequential file update pattern. It reads two sorted input files — a transaction file (`EMPTRAN`) and an old master file (`OLDEMP`) — in parallel and merges them into a new updated master file (`NEWEMP`), writing any invalid transactions to an error file (`ERRTRAN`).

**Input files:**
- `EMPTRAN` — transaction records, each containing a transaction code (`A`=Add, `C`=Change, `D`=Delete) and employee data fields
- `OLDEMP` — the existing sequential employee master file, sorted by employee ID

**Processing:**
1. **Opens** all four files (two input, two output).
2. **Reads both files in parallel**, using independent need-to-read switches (`NEED-TRANSACTION-SWITCH`, `NEED-MASTER-SWITCH`) to control when each file advances — a key technique in sequential merge processing.
3. **Compares** the current transaction and master employee IDs and branches into one of three cases:
   - **Master ID > Transaction ID** (`350-PROCESS-HI-MASTER`) — the transaction has no matching master. If it is an Add, the new record is written; otherwise it is an error.
   - **Master ID < Transaction ID** (`360-PROCESS-LO-MASTER`) — no transaction exists for this master, so the master is written unchanged to the new file and the master is advanced.
   - **Master ID = Transaction ID** (`370-PROCESS-MAST-TRAN-EQUAL`) — a match is found. A Delete skips the master; a Change updates specific fields; anything else is an error.
4. **Uses `HIGH-VALUE`** as a sentinel when either file reaches end-of-file, ensuring the remaining records of the other file are still processed correctly.
5. **Writes error transactions** to `ERRTRAN` for any unmatched or invalid transaction codes, with file status checking and `DISPLAY` diagnostics on write failure.
6. **Closes** all files and stops.

**Output files:**
- `NEWEMP` — the updated employee master file
- `ERRTRAN` — rejected transactions that could not be applied

---

### EMPIND01 — Indexed File Creation

EMPIND01 is a utility program that converts the existing sequential employee file (`OLDEMP`) into an indexed file (`EMPMASTI`), using the employee ID as the record key. This prepares the master file for the random-access maintenance performed by EMPIND02.

**Processing:**
1. **Opens** `OLDEMP` for input and `EMPMASTI` for output with `ORGANIZATION IS INDEXED`.
2. **Reads** each sequential record using `READ ... INTO` to populate a working-storage employee master record.
3. **Writes** each record to the indexed file using `WRITE ... INVALID KEY`, displaying a diagnostic message and halting if a duplicate key is detected.
4. **Closes** both files and stops.

---

### EMPIND02 — Indexed File Maintenance

EMPIND02 maintains the indexed employee master file (`EMPMASTI`) in place using random access, applying Add, Change, and Delete transactions from `EMPTRAN`. Unlike SEQ3000, there is no old/new master pair — updates are applied directly to the indexed file opened in `I-O` mode.

**Processing:**
1. **Opens** `EMPTRAN` for input, `EMPMASTI` for I-O (read/write), and `ERRTRAN` for output.
2. **Reads** each transaction and immediately attempts a **random read** of the indexed file by moving the transaction's employee ID to the record key and issuing `READ ... INVALID KEY / NOT INVALID KEY` to set the `MASTER-FOUND` switch.
3. **Branches** on the transaction code and master-found status:
   - **Delete + master found** → `DELETE EMPMASTI` removes the record by key.
   - **Delete + no master** → error transaction written.
   - **Add + master found** → duplicate key, error transaction written.
   - **Add + no master** → new record built from transaction fields and written with `WRITE ... INVALID KEY`.
   - **Change + master found** → only non-blank / non-zero fields from the transaction overwrite the master, then `REWRITE` updates the record in place.
   - **Change + no master** → error transaction written.
4. **Writes error transactions** to `ERRTRAN` with file status checking and `DISPLAY` diagnostics.
5. **Closes** all files and stops.

---

## Example: SEQ3000 Transaction Processing

```
EMPTRAN (Transaction File)        OLDEMP (Old Master File)
-----------------------------     ----------------------------
A 10045 JAMES BROWN  SALES01...   10032 EMILY CLARK  HR001...
C 10032              ACCT01 ...   10067 MARCUS LEE   IT002...
D 10067                           10089 SARA JONES   SALES01...

Processing result:

  Master 10032 < Tran 10032 → EQUAL  → Change applied → written to NEWEMP
  Master 10067 < Tran 10045 → LO     → Master 10032 written unchanged... (handled above)
  Master HIGH   > Tran 10045 → HI    → Add applied → new record written to NEWEMP
  Master 10067 = Tran 10067  → EQUAL → Delete applied → record skipped
  Master 10089 > EOF          → LO   → Master 10089 written unchanged to NEWEMP

NEWEMP (New Master File)          ERRTRAN (Error File)
-----------------------------     ----------------------------
10032 EMILY CLARK  ACCT01...      (empty — all transactions valid)
10045 JAMES BROWN  SALES01...
10089 SARA JONES   SALES01...
```

---

## New Concepts Used

- **Sequential file maintenance pattern** — reading two sorted files in parallel using independent read-control switches (`NEED-TRANSACTION`, `NEED-MASTER`) and a write-control switch (`WRITE-MASTER`), rather than reading both files on every loop iteration
- **`HIGH-VALUE` sentinel** — assigning `HIGH-VALUE` to the employee ID when either file hits end-of-file, so the merge comparison logic continues processing the remaining records of the other file without special EOF handling
- **Indexed file organization** — declaring a file with `ORGANIZATION IS INDEXED` and `RECORD KEY IS` in the `FILE-CONTROL` section to enable keyed access
- **Sequential vs. random access modes** — using `ACCESS IS SEQUENTIAL` in EMPIND01 for the initial load, and `ACCESS IS RANDOM` in EMPIND02 for direct record lookups by key
- **`I-O` file open mode** — opening the indexed master file with `OPEN I-O` in EMPIND02 to allow both reading and writing (rewriting/deleting) within the same program run
- **`READ ... INVALID KEY / NOT INVALID KEY`** — performing a random read on an indexed file and branching on whether the key was found, using both the `INVALID KEY` and `NOT INVALID KEY` phrases in a single `READ` statement
- **`DELETE` statement** — removing a record from an indexed file by its current key without needing to move data, distinct from the sequential approach of simply omitting a record from the new master
- **`REWRITE` statement** — updating an existing record in place on the indexed file after modifying fields in working storage, only valid after a successful random read of that record
- **`FILE STATUS IS`** — declaring a two-character file status field on `NEWEMP` and `ERRTRAN` and using level-88 condition names (`NEWEMP-SUCCESSFUL`, `ERRTRAN-SUCCESSFUL`) to check the outcome of write operations
- **`DISPLAY` for runtime diagnostics** — using `DISPLAY` to output error messages and file status codes to the console when a write or rewrite operation fails, providing visibility into file I/O errors at runtime
- **`READ ... INTO`** — reading a file record directly into a working-storage structure in a single statement, rather than reading into the FD record area and then moving it

---

## Author

| Name | Profile |
|------|---------|
| Gabe Dilley | [GitHub](https://github.com/gawdilley) |
