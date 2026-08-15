# OpenJDK Internals: Day 17 – Native Memory Management

Up until now, we have focused almost entirely on the **Java Heap** – the memory managed by the Garbage Collector where Java objects (`oop`s) live. But HotSpot is a massive C++ application with its own internal data structures: JIT compiler abstract syntax trees, thread management blocks, garbage collector card tables, and class metadata.

If HotSpot C++ developers just used standard `malloc()` and `new` everywhere, two problems would arise:

1. The JVM would suffer severe memory fragmentation.
2. When the JVM crashed with an Out‑Of‑Memory (OOM) error from the OS, developers would have no idea whether the Java Heap was full or a C++ memory leak in the JIT compiler consumed all the RAM.

In this lesson, we step outside the Java Heap. We'll learn how HotSpot manages its own native C++ memory using categorized base classes (`CHeapObj`, `ResourceObj`), high‑speed thread‑local bump allocators (`Arena`), and Native Memory Tracking (NMT).

---

## 1. The Big Picture (Mental Model)

The OS sees the JVM as a single process with a massive virtual memory footprint. Inside, HotSpot rigorously segregates memory based on lifecycle and ownership.

```
       OS Process Memory (The JVM)
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│  [ Java Heap ] (Managed by GC)                                             │
│   - Java objects, arrays.                                                  │
│                                                                            │
│ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│                                                                            │
│  [ Native Memory ] (Managed by HotSpot C++ code)                           │
│                                                                            │
│   ┌─────────────────────────┐   ┌───────────────────────────────────────┐   │
│   │ C-Heap (malloc)         │   │ Arenas / ResourceAreas (Bump ptr)     │   │
│   │ - Long-lived C++        │   │ - Short-lived C++ objects             │   │
│   │   objects.              │   │ - E.g., JIT Compiler IR nodes.        │   │
│   │ - GC Card Tables.       │   │ - Cleared instantly via ResourceMark. │   │
│   └─────────────────────────┘   └───────────────────────────────────────┘   │
│                                                                            │
│   ┌─────────────────────────┐   ┌───────────────────────────────────────┐   │
│   │ Code Cache              │   │ Metaspace                             │   │
│   │ - JIT Machine Code      │   │ - InstanceKlass (Class Metadata)      │   │
│   │ - Generated Stubs       │   │ - Constant Pools                      │   │
│   └─────────────────────────┘   └───────────────────────────────────────┘   │
│                                                                            │
│   ┌────────────────────────────────────────────────────────────────────┐   │
│   │ Thread Native Stacks (OS allocated via pthread_create)             │   │
│   └────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | Spec says... | HotSpot does... |
| :------ | :----------- | :-------------- |
| **Internal memory management** | Completely ignored. | Introduces a strict C++ inheritance hierarchy for all native allocations. |
| **Method Area** | Exists. | Implemented as Metaspace (native memory). |
| **Native stacks** | Exists. | Uses OS‑allocated stacks via `pthread_create`. |
| **Allocation discipline** | Not specified. | All C++ objects must inherit from a specific base class (e.g., `CHeapObj` or `ResourceObj`). |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/memory/` | Core C++ allocation abstractions and Arenas. |
| `src/hotspot/share/services/` | Native Memory Tracking (NMT). |
| **Key files** | `allocation.hpp`, `arena.hpp`, `memTracker.hpp`. |

---

## 4. Key Concepts You Need to Know

### NMT (Native Memory Tracking)
A diagnostic feature (enable with `-XX:NativeMemoryTracking=summary`). It intercepts every `malloc`, `free`, `mmap`, and `munmap` to categorise them. If the JVM consumes 4GB of RAM but the Java Heap is only 1GB, NMT tells you exactly which subsystem (Code Cache, GC, Thread Stacks, Metaspace) is using the other 3GB.

### RAII (Resource Acquisition Is Initialization)
A C++ paradigm where resource management is tied to object lifetime. HotSpot uses this heavily via `ResourceMark`. When a `ResourceMark` is created on the C++ stack, it marks the current position in a thread‑local `Arena`. When the C++ function returns and the `ResourceMark` is destroyed, it instantly resets the `Arena` pointer, "freeing" all memory allocated in that scope in **O(1)** time.

---

## 5. Architecture – How HotSpot Enforces C++ Memory Discipline

1. **Global `new` operator** – HotSpot globally overrides the standard C++ `new`/`delete` and makes them throw a fatal error. You cannot use them by accident.

2. **Base Classes** – Every C++ class must inherit from a specific base class defined in `allocation.hpp`:

| Base Class | Where it lives | Lifecycle |
| :--------- | :------------- | :-------- |
| `CHeapObj<mtX>` | OS `malloc`/`free` (C‑Heap) | Manual `delete`; tracked by NMT with a specific tag. |
| `ResourceObj` | Thread‑local `Arena` | Bump‑pointer allocation; cleared by `ResourceMark`. |
| `StackObj` | C++ stack only | Compiler‑enforced; cannot be heap‑allocated. |
| `ValueObj` | Embedded inside other objects | No special allocation. |

3. **OS Abstraction** – `CHeapObj` doesn't call `malloc` directly. It calls `os::malloc`, which wraps the OS syscall with NMT tracking.

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `AllocatedObj` | `allocation.hpp` | Absolute root of the allocation hierarchy. |
| `CHeapObj<MEMFLAGS F>` | `allocation.hpp` | Base for objects in the C‑Heap; template param is the NMT category. |
| `ResourceObj` | `allocation.hpp` | Base for temporary objects allocated in an `Arena`. |
| `Arena` | `arena.hpp` | Fast bump‑pointer memory allocator. |
| `ResourceMark` | `arena.hpp` | RAII scope guard that rolls back an `Arena`'s bump pointer. |
| `MemTracker` | `memTracker.hpp` | Static class intercepting all native memory operations. |

---

## 7. Critical Functions

- `os::malloc(size_t size, MEMFLAGS flags)` – HotSpot's wrapper for `malloc`, with NMT tracking.
- `os::free(void* mem)` – HotSpot's wrapper for `free`.
- `Arena::Amalloc(size_t x)` – bump‑pointer allocation inside an Arena.
- `MemTracker::record_malloc()` – internal NMT hook.

---

## 8. Important Macros / Utilities

| Macro | Purpose |
| :---- | :------ |
| `NEW_C_HEAP_ARRAY(type, size, memflags)` | Allocate a C‑heap array with NMT tracking. |
| `FREE_C_HEAP_ARRAY(type, old_array)` | Free a C‑heap array. |
| `RESOURCE_MARK_(thread)` | Instantiate a `ResourceMark` for a specific thread's Arena. |
| `MEMFLAGS` enum | Categorises every native allocation (`mtClass`, `mtCode`, `mtGC`, `mtThreadStack`, etc.). |

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: The Holy Grail – allocation.hpp
- **Open:** `src/hotspot/share/memory/allocation.hpp`
- **Read the block comment** at the top – it explains the entire HotSpot C++ design philosophy.
- **Look for:** `class CHeapObj` – notice it overrides `operator new` to take a `MEMFLAGS` argument and calls `AllocateHeap`.
- **Look for:** `class StackObj` – notice `operator new` is explicitly `private` so the C++ compiler prevents heap allocation.

### Tour 2: The Arena and ResourceMark
- **Open:** `src/hotspot/share/memory/arena.hpp`
- **Look for:** `class ResourceMark`. Examine its constructor (records the Arena's current `_chunk` and `_hwm`) and its destructor (restores them).

### Tour 3: Native Memory Tracking
- **Open:** `src/hotspot/share/services/memTracker.hpp`
- **Look at:** the different `MEMFLAGS` tags (defined in `memory/memTag.hpp`). You'll see `mtJavaHeap`, `mtClass`, `mtThreadStack`, `mtCode`, etc.

---

## 10. Execution Flow – JIT Compilation Allocating AST Nodes

1. `CompilerThread` picks up a task to compile `MyClass.foo()`.
2. The compilation entry function creates a `ResourceMark rm;` on the C++ stack.
3. The `ResourceMark` snapshots the thread's current `ResourceArea` (Arena) state.
4. The C1 compiler parses bytecodes and creates an Intermediate Representation (IR).
5. It calls `new (ResourceObj::C_HEAP) IRNode()`.
6. Because `IRNode` inherits from `ResourceObj`, the overloaded `operator new` bypasses `malloc` and simply bumps a pointer in the thread's `Arena`. Allocation takes **~2 nanoseconds**.
7. The compiler generates machine code and copies it to the Code Cache.
8. The compilation function returns.
9. The `ResourceMark rm` goes out of scope – its destructor runs.
10. The destructor resets the `Arena` bump pointer to its original state. **Millions of `IRNode` objects are "freed" instantly in O(1) time** – without a single call to `free()`.

### Code Snippet: ResourceMark Usage
```cpp
// In HotSpot C++ code (simplified)
void compile_method(Method* method) {
    ResourceMark rm;  // Snapshot the current Arena state

    // Allocate IR nodes – all come from the Arena
    IRNode* node1 = new (ResourceObj::C_HEAP) IRNode();
    IRNode* node2 = new (ResourceObj::C_HEAP) IRNode();

    // ... do compilation ...

} // rm destructor runs here – all IR nodes are automatically freed
```

---

## 11. Real Java Example – Impact on Native Memory

```java
import java.nio.ByteBuffer;

public class NativeMemoryTest {
    public static void main(String[] args) {
        // This does NOT allocate 1GB on the Java Heap.
        // The JVM creates a tiny Java object (DirectByteBuffer) on the Heap,
        // but calls native code (Unsafe.allocateMemory) to allocate
        // 1GB of raw OS memory outside the Java Heap.
        ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024 * 1024);

        // This memory is tracked by NMT under 'mtInternal' or 'mtNio'.
        // It is freed only when the DirectByteBuffer object is GC'd.
    }
}
```

---

## 12. Why This Design? (The "Why")

### Why use Arenas/ResourceMarks instead of C++ smart pointers?
The JIT compiler creates millions of tiny, interconnected graph nodes (AST/IR) during a compilation. If managed by `std::shared_ptr` or `std::unique_ptr`, the overhead of reference counting and individual destructor calls would drastically slow down the compiler. With Arenas, the JVM doesn't care about individual object lifetimes – it just resets the memory pointer at the end of compilation, annihilating the entire graph **instantly**.

### Why intercept every `malloc` with NMT?
Java applications run in strict cloud environments (Kubernetes/Docker) with hard memory limits. If the JVM exceeds its container limit, the OS OOM‑Kills it instantly. If the Java Heap is perfectly sized but the JVM still gets OOM‑Killed, engineers need to know whether the C++ Metaspace, Code Cache, or JNI allocations are the culprit. NMT provides this exact visibility.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| Confusing the Java Heap with the C‑Heap. | The Java Heap (`-Xmx`) holds Java objects. The C‑Heap holds HotSpot's internal C++ structures, managed by `os::malloc` and NMT. |
| Forgetting a `ResourceMark` in C++ HotSpot development. | If you allocate `ResourceObj` instances but omit `ResourceMark`, the thread's Arena grows infinitely until a native OOM crash. |
| Assuming `-XX:MaxDirectMemorySize` limits all native memory. | It only limits `ByteBuffer.allocateDirect()`. It does **not** limit Metaspace, Code Cache, thread stacks, or GC structures. |

---

## 14. Summary

- HotSpot **does not** rely on generic C++ memory management.
- It enforces a strict inheritance hierarchy (`allocation.hpp`) requiring all C++ objects to declare their memory domain.
- **`CHeapObj`** – long‑lived structures; uses `os::malloc` with NMT tracking; manually `delete`d.
- **`ResourceObj`** – short‑lived, high‑volume allocations (JIT graphs); uses thread‑local `Arena` bump‑pointers; swept instantly by `ResourceMark` RAII guards.
- **NMT** – the omniscient watcher of all C++ `os::malloc`/`mmap` calls, enabling production observability.

---

## 15. Mental Model to Remember

| Base Class | Allocation | Lifecycle | NMT Tracking |
| :--------- | :--------- | :-------- | :----------- |
| `CHeapObj<mtX>` | `os::malloc` | Manual `delete` | Yes (tagged) |
| `ResourceObj` | Arena bump‑pointer | Auto‑freed by `ResourceMark` | No (internal) |
| `StackObj` | C++ stack | Function scope | No |
| `ValueObj` | Embedded | Contained object's lifetime | No |

---

## 16. Important Classes / Structs

- `CHeapObj`
- `ResourceObj`
- `StackObj`
- `Arena`
- `ResourceMark`
- `MemTracker`

---

## 17. Important Functions / Methods

- `os::malloc()`
- `os::free()`
- `Arena::Amalloc()`

---

## 18. Important Files

- `src/hotspot/share/memory/allocation.hpp`
- `src/hotspot/share/memory/arena.cpp`
- `src/hotspot/share/services/memTracker.cpp`

---

## 19. Code‑Reading Exercises

1. **StackObj enforcement** – open `src/hotspot/share/memory/allocation.hpp` and read the `StackObj` class definition. Notice how `operator new` is declared `private` to prevent heap allocation.

2. **NMT in os.hpp** – open `src/hotspot/share/runtime/os.hpp` and search for `malloc`. Notice it requires a `MEMFLAGS` argument – this forces developers to categorise allocations for NMT.

3. **MEMFLAGS enum** – open `src/hotspot/share/memory/memTag.hpp` (or `memTracker.hpp` depending on the JDK version). Browse the `MEMFLAGS` enum. Identify the tag for GC structures vs. JNI allocations.

---

## 20. Self‑Check Questions

1. A Java application heavily uses `ByteBuffer.allocateDirect(10MB)`. Over time, the JVM consumes 4GB of RAM, but the Java Heap reports only 500MB used. Where is the remaining memory, and which C++ subsystem manages it?

2. In the JIT compiler, why is allocating thousands of AST nodes derived from `ResourceObj` fundamentally faster than using standard C++ `std::shared_ptr<ASTNode>`?

3. You add a new subsystem to HotSpot (e.g., a custom AI profiler) and inherit your configuration objects from `CHeapObj<mtInternal>`. How will an operations engineer monitor its memory footprint?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/runtime/javaThread.hpp` | See the native thread structures that hold these Arenas. |
| `src/hotspot/share/runtime/osThread.hpp` | The OS‑level thread mapping. |

---

## 22. Coming Up Next

**Lesson 18 – Threads, Thread Stacks & Stack Frames**  
We've covered memory for objects and C++ structures. Now we need to understand the execution context – how HotSpot represents Java threads internally, how their native OS stacks are structured, and how the interpreter and JIT walk these stacks to inspect frames, local variables, and call stacks.
