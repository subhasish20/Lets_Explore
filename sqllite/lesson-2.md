# SQLite Source Code - Lesson 2: Source Tree Overview

## Lesson Overview

In this lesson, we will map the SQLite source code repository. As an experienced C programmer, you know that understanding a codebase starts with understanding its physical layout and dependency graph.

SQLite has a unique development model. What the world downloads and links against (`sqlite3.c`) is **not** what the SQLite developers actually write. We are going to explore the canonical source tree, distinguishing between handwritten code, test code, and code generators.

**Prerequisites:** Familiarity with the `lex`/`yacc` or `flex`/`bison` paradigm (though SQLite uses its own tool), and basic Makefile architecture.

---

## Mental Model

Here is the conceptual layout of the canonical SQLite source tree:

```text
                  SQLite Canonical Repository
                               │
       ┌───────────────┬───────┴───────┬───────────────┐
       │               │               │               │
     src/            tool/           test/           ext/
       │               │               │               │
 Handwritten      Generators       Tcl-based     Official Add-ons
  C files,        (Lemon,          Test Suite      (FTS5, JSON,
  Headers,        mkkeywordhash)                 R-Tree, etc.)
  parse.y
       │               │
       └───────┬───────┘
               │
          Build System
          (Make / Tcl)
               │
               ▼
           sqlite3.c
         (Amalgamation)
```

---

## Database Concept

**Source of Truth vs. Generated Artifacts**  
Database engines require extremely fast parsing and execution. Writing a high-performance SQL parser or a perfect keyword‑hash lookup by hand in C is tedious and error‑prone. Database engineers use **code generators** at build time to translate high‑level definitions (like a SQL grammar file) into highly optimized, unreadable C state machines. The grammar file is the "source of truth"; the C code is just a build artifact.

---

## Architecture

The canonical source tree physically mirrors the architectural layers we discussed in Lesson 1:

| Architectural Layer | Corresponding Source Files |
|---------------------|----------------------------|
| **SQL Engine**      | `tokenize.c`, `parse.y` (→ `parse.c`), `where.c`, `select.c` |
| **VDBE**            | `vdbe.c`, `vdbeaux.c`, `vdbeapi.c` |
| **Storage (B-tree & Pager)** | `btree.c`, `pager.c`, `wal.c` |
| **VFS & OS**        | `os.c`, `os_unix.c`, `os_win.c` |

---

## Repository Location

- **Repository area:** The root directory of a cloned SQLite repository (from Fossil or the official GitHub mirror).
- **Important directories:**
  - `src/` – Handwritten canonical source.
  - `tool/` – Build scripts and C‑based code generators.
  - `test/` – Over 100,000 lines of Tcl test scripts.
- **Important files:** `Makefile.in`, `configure`.
- **Related subsystems:** The build system and amalgamation generator.

---

## Concepts Required Before Reading

### Tcl (Tool Command Language)
You do not need to be a Tcl expert, but you must know that SQLite relies heavily on Tcl. Richard Hipp originally wrote SQLite as a Tcl extension. Today, Tcl is used to:
- Generate the amalgamation
- Run the massive test suite
- Generate C code (e.g., `mkkeywordhash`)

If you see `.tcl` files in the repository, they are critical infrastructure.

### Lemon
SQLite does **not** use Bison/Yacc. It uses a custom LALR(1) parser generator called **Lemon** (located in `tool/lemon.c`). Lemon is safer and re‑entrant by design.

---

## Important Structures (Directory Layout)

Since we are looking at a source tree rather than C structs, our "structures" are the directories:

| Directory | Purpose |
|-----------|---------|
| `src/` | Owned by the core developers. **You will spend 99% of your time here.** |
| `tool/` | Owned by the build system. Programs here are compiled on the host machine, executed to generate C code, and then discarded. |
| `bld/` (or similar) | Owned by the compiler. Where `sqlite3.c` is ultimately born. |

---

## Important Functions (Generators)

Instead of runtime functions, let's look at the build‑time "functions" (standalone C programs):

| Generator | Purpose |
|-----------|---------|
| `lemon` | Compiled from `tool/lemon.c`. Reads `src/parse.y` and outputs `parse.c`. |
| `mkkeywordhash` | Compiled from `tool/mkkeywordhash.c`. Generates a **perfect hash function** in C for blazing‑fast SQL keyword recognition (e.g., distinguishing `SELECT` from `SET` instantly). |
| `mksqlite3c.tcl` | The Tcl script that concatenates all the individual C files, resolves internal `#include`s, and produces the `sqlite3.c` amalgamation. |

---

## Important Macros / Utilities

- **`SQLITE_OMIT_*` and `SQLITE_ENABLE_*`**: Found throughout the `src/` files. SQLite is heavily customizable at compile time. These macros conditionally compile entire subsystems (e.g., omitting WAL or enabling JSON).  
  Example:  
  ```c
  #ifdef SQLITE_ENABLE_JSON1
  # include "json1.c"
  #endif
  ```

---

## Source Code Exploration

Let's explore the contents of `src/`:

- **`sqliteInt.h`** – The master internal header. It defines the core structs (`sqlite3`, `Vdbe`, `Btree`, `Pager`). Every `.c` file in `src/` includes this.
- **`parse.y`** – The Lemon grammar file defining the SQL language syntax.
- **`main.c`** – The entry point for the public API.
- **`vdbe.c`** – The core of the virtual machine.
- **`btree.c` / `pager.c` / `wal.c`** – The backend storage engine.

**What to look for:** Notice how cleanly the files are named after their architectural components. There is no mixing of parser logic inside `btree.c`.

---

## Control Flow (Build Process)

The flow of the *source code* through the build process:

```
src/parse.y  →  lemon  →  parse.c
tool/mkkeywordhash.c  →  mkkeywordhash executable  →  keywordhash.h
(src/*.c + parse.c + keywordhash.h)  →  mksqlite3c.tcl  →  sqlite3.c (The Amalgamation)
```

---

## Real SQL Example (in the Source Tree)

If we trace `SELECT name FROM users;` in the *source tree*:

1. The string hits `src/tokenize.c` (which uses `keywordhash.h` generated by `tool/mkkeywordhash.c`).
2. Tokens go to `parse.c` (generated by Lemon from `src/parse.y`).
3. The AST goes to `src/select.c` and `src/where.c` to generate the query plan.
4. Execution happens in `src/vdbe.c`.

---

## Design Reasoning

**Why use a custom parser generator (Lemon)?**  
Bison/Yacc use global variables and are traditionally not thread‑safe (or require awkward workarounds to be re‑entrant). Lemon was written to be thread‑safe by design, passing the parser state explicitly as a pointer. Furthermore, Lemon handles memory leaks elegantly: if the parser fails halfway through a SQL statement, Lemon's design ensures the AST nodes built so far are properly freed.

**Example snippet from `parse.y` (simplified):**
```yacc
%type select { Select * }
select(A) ::= select(S) where_opt(W) orderby_opt(O) limit_opt(L) {
   A = sqlite3SelectNew(S, W, O, L);
}
```

---

## Error Handling

In the context of the source tree, build errors in `tool/` (like a syntax error in `parse.y`) will stop the build process before the amalgamation is ever created. Lemon enforces strict LALR(1) grammar rules and will throw a shift/reduce conflict error if the SQL grammar is ambiguous.

---

## Common Beginner Mistakes

- ❌ **Mistake:** Making changes to `sqlite3.c` or `parse.c` and wondering why they are overwritten.  
  ✅ **Correction:** Always edit `src/*.c` or `src/parse.y`. The amalgamation and parser are build artifacts.

- ❌ **Mistake:** Trying to compile the `src/*.c` files directly using `gcc src/*.c`.  
  ✅ **Correction:** SQLite's internal `#include` structures and generated headers require the Makefile/build system to stage the code properly.

---

## Summary

The canonical SQLite repository is elegantly divided into handwritten source (`src/`), build tools (`tool/`), and tests (`test/`). The core engineering philosophy relies heavily on build‑time code generation to write perfect, highly optimized C code (like state machines and hash tables) so the developer doesn't have to.

---

## Mental Model to Remember

```text
Canonical Source (src/) + Generators (tool/)  →  Amalgamation (sqlite3.c)
```

---

## Key Structures (Directories) to Remember

- `src/`
- `tool/`
- `test/`

## Key Generators to Remember

- `lemon`
- `mkkeywordhash`
- `mksqlite3c.tcl`

## Key Files to Remember

- `src/parse.y` (The SQL Grammar)
- `src/sqliteInt.h` (The Internal Brain)
- `tool/lemon.c` (The Parser Generator)

---

## Code-Reading Exercises

1. Open `tool/lemon.c` and read the massive block comment at the top. It explains exactly why Lemon exists and how it differs from Yacc/Bison.

2. Open `src/parse.y`. Scroll past the initial C code blocks to find the grammar rules (e.g., look for `cmd ::= select.`). Notice how clean the grammar definitions are.

3. Open `tool/mkkeywordhash.c`. Look at the `Keyword` struct and the `aKeywordTable` array. This is the hardcoded list of every SQL keyword SQLite understands.

---

## Understanding Questions

1. If you want to add a new SQL keyword (e.g., `TRUNCATE`), which file in the repository must you modify first, and what build tool will process that change?

2. Why is the Tcl language a hard dependency for building SQLite from the canonical repository, even though SQLite itself is written in pure C?

3. As a C programmer, what are the advantages of shipping an "amalgamation" (`sqlite3.c`) to end‑users rather than distributing the `src/` directory and a `Makefile`?

---

## Suggested Next Files

- `Makefile.in` – To see how the build targets orchestrate Lemon and the amalgamation.
- `configure` – To see how the build environment is detected.

---

## Suggested Next Lesson

**Lesson 3 – Building SQLite from Source**
