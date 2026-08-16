1. 🏗️ SQLite Architecture & Repository Structure
2. 🌳 SQLite Source Tree Overview
3. 🔨 Building SQLite from Source
4. 📦 SQLite Amalgamation (`sqlite3.c`)
5. 📚 SQLite Public API (`sqlite3.h`)
6. 🔐 SQLite Internal Header (`sqliteInt.h`)
7. 🔌 SQLite Connection Object (`sqlite3`)
8. 🔄 Database Connection Lifecycle
9. 🧠 SQLite Memory Management
10. 💾 Virtual File System (VFS) Architecture
11. 🔤 SQL Tokenizer (`tokenize.c`)
12. 🧩 SQLite Grammar (`parse.y`)
13. ⚙️ Lemon Parser Generator
14. 🏭 Generated Parser (`parse.c`)
15. 🌲 SQL Parse Structures
16. 🌳 Expression Trees (`Expr`)
17. 🔎 Name Resolution
18. 🧠 SQL Semantic Analysis
19. 🧭 Query Planner Architecture
20. 🔍 WHERE Clause Processing
21. 🎯 WHERE Terms & Constraints
22. 📌 Index Selection
23. 📊 Table Scan vs Index Scan
24. 🔗 Join Planning & Join Order
25. 💰 Query Planner Cost Estimation
26. 📈 `ANALYZE` & SQLite Statistics
27. ⚡ Query Planner Optimizations
28. 🔄 Subquery Flattening & Query Transformations
29. ⚙️ VDBE Architecture (`vdbe.c`)
30. 🧱 VDBE Program & Opcode Architecture
31. 🚀 `sqlite3VdbeExec()` — Execution Loop
32. 🗃️ VDBE Registers & `Mem` Values
33. 🏷️ SQLite Data Types & Type Affinity
34. 🎯 VDBE Cursors
35. 🧬 Expression → VDBE Bytecode Generation
36. 📖 `SELECT` → VDBE Bytecode
37. ✏️ `INSERT` / `UPDATE` / `DELETE` → VDBE Bytecode
38. 💽 SQLite Database File Format
39. 🌳 B-tree Architecture (`btree.c`)
40. 📄 Database Pages, Cells & Page Layout
41. 🗂️ Table B-trees vs Index B-trees
42. 🔍 B-tree Search & Cursor Movement
43. ➕ B-tree Insert & Delete
44. ⚖️ B-tree Balancing & Page Splitting
45. 🧾 SQLite Record Format & Serial Types
46. 📦 Pager Architecture (`pager.c`)
47. 🔄 Rollback Journal & Atomic Commit
48. 🔒 Transactions, Locks & Concurrency
49. ⚡ Write-Ahead Logging (WAL — `wal.c`)
50. 🚀 End-to-End SQLite Execution: SQL → Parser → Planner → VDBE → B-tree → Pager → VFS → Disk
