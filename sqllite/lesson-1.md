# SQLite Source Code - Lesson 1: High-Level Architecture

## Lesson Overview

Welcome to the SQLite source code! In this first lesson, we'll establish the big picture view of SQLite - what it is, how its internal pieces fit together, and how the source code is organized.

As an experienced C programmer, you might want to jump straight into `.c` files and start reading. **Don't do that yet.** SQLite is a complex state machine that handles concurrency and data safety. Before reading code, you need to understand the boundaries that keep this complexity manageable.

**Prerequisites:** You should be familiar with C build systems (Makefiles) and standard C library functions.

---

## Mental Model

This diagram is your most important reference. Every file we read will fit into one of these boxes:

```text
                 SQLite Library Boundary
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   SQL Engine         VDBE           Storage Engine
 (Front-End)       (Bridge)          (Back-End)
        │                 │                 │
     Parser           Opcodes            B-tree
        │                 │                 │
     Planner ─────────────┘                 │
                                           Pager
                                             │
                                          WAL/Journal
                                             │
─────────────────────────────────────────────┼───────────── OS Boundary
                                             │
                                      VFS (Virtual File System)
                                             │
                                   Operating System (POSIX/Win32)
                                             │
                                           Disk
```

---

## Database Concept

### The Embedded Database Model

Most databases (PostgreSQL, MySQL) use a **client-server** model. Your app talks to a separate server process over a network connection. The server manages its own memory, concurrency, and crash recovery separately.

SQLite is different - it's an **embedded** database. It's just a C library (`libsqlite3.a` or `.so`). It runs inside **your application's process**:
- Shares your application's memory
- Runs on your application's threads
- Uses your application's file descriptors

If your app crashes, SQLite crashes too - there's no background daemon to save the day. This is why SQLite is **extremely careful** about crash recovery through its Pager and WAL (Write-Ahead Logging) subsystems.

---

## Architecture Overview

SQLite has **four major layers**, each carefully separated from the others:

### 1. Core / API Layer
Public C functions that your application calls:
- `sqlite3_open()`
- `sqlite3_prepare_v2()`
- `sqlite3_step()`
- `sqlite3_close()`

### 2. SQL Compiler (Front-End)
Takes your SQL string and turns it into bytecode:
- **Tokenizer:** Breaks SQL into tokens
- **Parser:** Checks syntax and builds a parse tree
- **Planner:** Figures out the most efficient way to get your data (optimization)

### 3. VDBE (Virtual Database Engine)
A virtual machine that executes database operations. It's the strict boundary between:
- *What* SQL was requested (Front-End)
- *How* bytes are stored on disk (Back-End)

### 4. Storage Engine (Back-End)
Manages how data is physically stored:
- **B-tree:** Maintains the table and index structures
- **Pager/WAL:** Handles caching, transactions, locking, and safe writing to disk

### 5. VFS (Virtual File System)
An OS abstraction layer - SQLite never calls `open()`, `read()`, or `write()` directly. Instead, it uses VFS function pointers that work on Unix, Windows, and other platforms.

---

## Repository Location

SQLite's source code is managed using Fossil (created by SQLite's author, Richard Hipp), but it's officially mirrored to GitHub.

**Important:** Don't confuse the `sqlite3.c` amalgamation file (a single huge file) with the actual source repository.

### Important Directories

| Directory | Contents |
|-----------|----------|
| `src/` | All handwritten canonical C source files |
| `tool/` | Build scripts, code generators, testing utilities |
| `ext/` | Official extensions (FTS5 full-text search, JSON support) |
| `test/` | Massive TCL-based test suite |

---

## The Amalgamation (Important Concept!)

You won't find `sqlite3.c` in the `src/` directory. Why?

SQLite is built from **over 100 separate `.c` files** in `src/`. During the build process, a TCL script (`tool/mksqlite3c.tcl`) stitches all these files together (along with generated parser code) into a single massive C file called `sqlite3.c` - the **amalgamation**.

**Rule of thumb:**
- Study the architecture in `src/`
- Compile and deploy the `sqlite3.c` amalgamation

---

## Important Structures

### The Master Structure: `sqlite3`

This is the database connection object. Everything starts here.

| Aspect | Details |
|--------|---------|
| **Purpose** | Represents a single connection to a database |
| **Created** | When you call `sqlite3_open()` |
| **Destroyed** | When you call `sqlite3_close()` |
| **Contains** | VFS pointer, attached B-trees, memory allocator state, active VDBE virtual machines (prepared statements) |

---

## Important Functions (The Public API)

| Function | Purpose |
|----------|---------|
| `sqlite3_open()` | Initializes the connection, attaches to VFS and Pager |
| `sqlite3_prepare_v2()` | **Front-End entry point** - compiles SQL to VDBE bytecode |
| `sqlite3_step()` | **VDBE/Back-End entry point** - executes the bytecode program |
| `sqlite3_close()` | Destroys connection, flushes caches through Pager |

---

## Important Macros / Utilities

| Macro | Purpose |
|-------|---------|
| `SQLITE_PRIVATE` | Marks internal functions as static in the amalgamation, hiding them from the linker |
| `SQLITE_API` | Marks public functions exposed in `sqlite3.h` |

---

## Source Code Exploration

Start by looking inside `src/` at these key files:

| File | Subsystem |
|------|-----------|
| `main.c` | Core public API implementations (`sqlite3_open`, `sqlite3_close`) |
| `vdbe.c` | Virtual Database Engine execution loop |
| `btree.c` | B-tree implementation |
| `pager.c` | Transaction and caching layer |
| `os.c` / `os_unix.c` / `os_win.c` | VFS layer |

**Don't read them line-by-line yet.** Just open them to see how they're organized into discrete subsystems.

---

## Control Flow

Execution flows hierarchically through the subsystems:

### SQL Compilation Flow
```
Application
    ↓
sqlite3_prepare_v2()
    ↓
Parser (check syntax)
    ↓
Planner (optimize query)
    ↓
VDBE Bytecode (generated)
```

### Query Execution Flow
```
Application
    ↓
sqlite3_step()
    ↓
VDBE Execution Loop (run bytecode)
    ↓
B-tree Cursor Seek (find data)
    ↓
Pager Cache Fetch (get pages)
    ↓
VFS Read (OS disk read)
```

---

## Real SQL Example

**Query:** `SELECT * FROM users WHERE id = 1;`

1. **Tokenizer/Parser** (`tokenize.c`, `parse.y`): Verifies the SQL syntax is correct
2. **Planner** (`where.c`): Sees `id = 1` and decides to use an Index B-tree instead of scanning the entire Table B-tree
3. **VDBE** (`vdbe.c`): Generates opcodes like `OpenRead`, `SeekGE`, `Column`, `ResultRow`
4. **B-tree** (`btree.c`): Interprets the `SeekGE` opcode to traverse tree nodes
5. **Pager** (`pager.c`): Ensures memory pages requested by B-tree are loaded from disk

---

## Design Reasoning

### Why compile SQL to VDBE bytecode?

**Question:** Why not just have the parser call B-tree functions directly?

**Answer:** **Separation of concerns.** By compiling SQL to bytecode:
- SQLite separates complex query planning (which indexes to use, join ordering) from data retrieval mechanics
- You can prepare a statement once (`sqlite3_prepare_v2`) and execute it millions of times (`sqlite3_step`) without paying the parsing/planning cost again

---

## Error Handling

Since SQLite is embedded, it **cannot** use `abort()`, `exit()`, or unhandled signals - doing so would kill your application.

Instead:
- Every subsystem returns explicit integer codes: `SQLITE_OK`, `SQLITE_IOERR`, `SQLITE_CORRUPT`
- Memory allocations are strictly checked at every layer
- Errors propagate back up the call stack

---

## Common Beginner Mistakes

### ❌ Mistake 1: Reading the Amalgamation
Trying to learn SQLite by reading the generated `sqlite3.c` file.

✅ **Correction:** Always read the canonical `src/*.c` files - they're logically separated.

### ❌ Mistake 2: Assuming SQLite Writes Directly
Thinking SQLite writes directly to disk files.

✅ **Correction:** SQLite writes to the **Pager**, which writes to the **WAL/Journal**, which calls the **VFS**, which calls OS `write()`.

---

## Summary

SQLite isn't one big program - it's a **pipeline of distinct state machines** running inside your application's process:

```
Front-End (SQL/Planner) → Bridge (VDBE) → Back-End (B-tree/Pager/OS)
```

- **SQL Engine** decides *what* to do
- **VDBE** dictates *how* to do it step-by-step
- **B-tree** manages logical structure
- **Pager** handles durability and safety
- **VFS** bridges to the operating system

---

## Key Structures to Remember

| Structure | Purpose |
|-----------|---------|
| `sqlite3` | Database Connection |
| `Vdbe` | Prepared Statement / Virtual Machine |
| `Btree` | Logical Storage |
| `Pager` | Physical Storage & Transactions |

## Key Functions to Remember

- `sqlite3_open()`
- `sqlite3_prepare_v2()`
- `sqlite3_step()`

## Key Files to Remember

- `src/main.c`
- `src/vdbe.c`
- `src/btree.c`
- `src/pager.c`

---

## Code-Reading Exercises

1. **Navigate to `src/main.c`** and find `sqlite3_open()`. Notice how minimal it is - it immediately delegates to an internal function (usually `sqlite3_open_v2` or `openDatabase`).

2. **Navigate to `src/vdbe.c`** and search for `sqlite3VdbeExec()`. This is the giant switch-statement loop that runs the virtual machine. Just glance at its size and structure.

3. **Navigate to `src/os.h`** and find the `sqlite3_vfs` struct. Look at the function pointers: `xOpen`, `xRead`, `xWrite`. See how SQLite abstracts POSIX and Win32 APIs.

---

## Understanding Questions

1. **If your application receives a `SIGKILL` while SQLite is writing to the database, why is a daemon-based database (like PostgreSQL) potentially safer in memory, but SQLite's Pager/WAL architecture essential for on-disk recovery?**

2. **Why does `sqlite3_prepare_v2()` not touch the Pager or B-tree for actual row data?**

3. **Looking at the architecture diagram, if you wanted to implement an in-memory-only database, which subsystem (VDBE, B-tree, Pager, or VFS) is the highest layer you would need to replace or modify?**

---

## Suggested Next Files

- `src/sqliteInt.h` - The master internal header that wires all these subsystems together
- `Makefile.in` / `tool/mksqlite3c.tcl` - To see how the amalgamation is generated

---

## Suggested Next Lesson

**Lesson 2 — SQLite Source Tree Overview**
