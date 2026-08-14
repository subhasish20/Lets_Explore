# OpenJDK Internals: Day 15 – G1, ZGC & Shenandoah

In Lesson 14, we established the theory: trace from Roots, mark live objects, reclaim the dead, and move objects to prevent fragmentation.

Historically, moving objects meant stopping the entire application (Stop‑The‑World). With a 100GB heap, that could freeze your application for 5–10 seconds. In modern finance or web services, a 10‑second pause is indistinguishable from a server crash.

To solve this, HotSpot engineered **Region‑based** and **Concurrent** Garbage Collectors. In this lesson, we'll study three distinct architectural marvels:

1. **G1 (Garbage‑First)** – the default, balanced collector. Uses regions and background marking, but still relies on short STW pauses to move objects.
2. **ZGC** – an ultra‑low‑latency collector. Uses *colored pointers* to move objects **while the Java application is actively running and reading them**.
3. **Shenandoah** – another ultra‑low‑latency collector. Uses *forwarding pointers* and *load barriers* to achieve concurrent movement.

---

## 1. The Big Picture (Mental Model)

All three modern GCs abandon the idea of a single massive contiguous block. They divide the Java Heap into a grid of **Regions** (usually 1MB–32MB each).

```
       The Region‑Based Heap
┌────────────────────────────────────────────────────────┐
│  [ Eden ] [ Survr] [ Old  ] [ Old  ] [ Free ] [ Eden ] │
│  [ Old  ] [ Free ] [ Humongous (Large Obj)  ] [ Survr] │
│  [ Free ] [ Eden ] [ Old  ] [ Old  ] [ Free ] [ Eden ] │
└────────────────────────────────────────────────────────┘
```

Because the heap is split into regions, the GC doesn't have to clean the whole heap at once. It calculates which regions contain the most garbage and collects **only those** (hence, "Garbage‑First").

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | Spec says... | HotSpot does... |
| :------ | :----------- | :-------------- |
| **Heap layout** | Not specified. | G1, ZGC, and Shenandoah each implement a region‑based heap. |
| **Barriers** | Not mentioned. | The JIT Compiler (C1/C2) and Interpreter are modified to inject GC barrier logic directly into Java machine code. |
| **Concurrent collection** | Not required. | ZGC and Shenandoah achieve concurrent evacuation via load barriers and colored/forwarding pointers. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/gc/g1/` | Garbage‑First implementation. |
| `src/hotspot/share/gc/z/` | ZGC implementation. |
| `src/hotspot/share/gc/shenandoah/` | Shenandoah implementation. |
| `src/hotspot/share/gc/shared/c2/` | Where GCs inject barrier logic into the C2 JIT compiler. |

---

## 4. Key Concepts You Need to Know

### GC Barriers (Crucial!)
**Not** CPU memory barriers (like `mfence`). A GC Barrier is a tiny snippet of assembly code injected by the JIT *before or after* a Java thread reads or writes an object field.

- **Write Barrier** – injected when you do `obj.field = newObj`. Used by G1 to track pointers crossing between regions.
- **Load Barrier (Read Barrier)** – injected when you do `Object x = obj.field`. Used by ZGC and Shenandoah to intercept a thread trying to read an object that the GC is currently moving.

### Remembered Sets (RSets) & Card Tables
If G1 collects Region 5, how does it know if an object in Region 84 points to it? Scanning the whole heap would be too slow. Each region has an `RSet` (a hash table) tracking exactly which other regions point *into* it. JIT Write Barriers update these RSets dynamically.

---

## 5. Architecture – Separated by Collector

### A. G1 (Garbage‑First)

- **Heap** – divided into 1MB–32MB regions, logically typed as Eden, Survivor, or Old.
- **Allocation** – threads allocate into Eden regions using TLABs.
- **Marking** – concurrent global marking determines which Old regions have the most dead space.
- **Evacuation (STW)** – G1 pauses the application, picks a set of high‑garbage regions (the "Collection Set"), copies live objects to free regions, updates all pointers, and resumes.
- **Barriers** – uses Pre‑Write and Post‑Write barriers to maintain RSets and SATB (Snapshot‑At‑The‑Beginning) marking queues.

### B. ZGC (The Z Garbage Collector)

- **Colored Pointers** – on 64‑bit systems, pointers only use 44–48 bits for the actual address. ZGC hijacks 4 of the unused high‑order bits for GC metadata (Marked0, Marked1, Remapped).
- **Evacuation (Concurrent)** – ZGC does **not** stop the world to move objects. It copies an object to a new region and leaves an entry in a global Forwarding Table.
- **Load Barrier** – when a Java thread reads a pointer (`x = obj.field`), the JIT assembly checks the pointer's color bits. If the "Remapped" bit is not set, the thread jumps to a slow‑path, looks up the new address in the Forwarding Table, updates the pointer in memory to the new address (Self‑Healing), and continues.

### C. Shenandoah

- **Architecture** – similar goals to ZGC, but does **not** use colored pointers (making it easier to port to 32‑bit or compressed‑oop environments). It uses bits in the `markWord` and a forwarding data structure.
- **Evacuation (Concurrent)** – moves objects concurrently.
- **Load Reference Barrier (LRB)** – upon loading a reference, it checks if the object is in a region currently being evacuated (the Collection Set). If so, it forwards the pointer.

---

## 6. Important C++ Classes / Structs

### G1
- `G1CollectedHeap` – the main entry point.
- `HeapRegion` – a single grid square of the G1 heap.
- `G1RemSet` – the Remembered Set tracking cross‑region pointers.

### ZGC
- `ZCollectedHeap` – the ZGC entry point.
- `ZPage` – ZGC's term for a region.
- `ZAddress` – C++ utility to manipulate the colored bits of a pointer.

### Shenandoah
- `ShenandoahHeap` – the Shenandoah entry point.
- `ShenandoahBarrierSet` – dictates when to inject LRBs.

---

## 7. Critical Functions

- `G1CollectedHeap::evacuate_collection_set()` – the STW phase where G1 copies live objects.
- `ZBarrier::load_barrier_on_oop_field()` – the C++ slow‑path called when the ZGC JIT fast‑path detects an incorrectly colored pointer.
- `ShenandoahHeap::evacuate_object()` – the C++ logic handling concurrent object copying.

---

## 8. Important Macros / Utilities

- **`BarrierSet`** – the abstract C++ base class (`src/hotspot/share/gc/shared/barrierSet.hpp`). Every GC implements this. The Interpreter and JIT compilers query the `BarrierSet` to know exactly what assembly to inject for memory accesses.

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: G1's Region Structure
- **Open:** `src/hotspot/share/gc/g1/heapRegion.hpp`
- **Look for:** the `_rem_set` field – this is the Remembered Set for this region.

### Tour 2: ZGC's Colored Pointers
- **Open:** `src/hotspot/share/gc/z/zAddress.hpp`
- **Look for:** functions like `is_good()`, `is_remapped()`, and `offset()`. Notice the bitwise AND (`&`) and shift (`>>`) operators used to extract the real address from the colored bits.

### Tour 3: GC/JIT Integration
- **Open:** `src/hotspot/share/gc/z/c2/zBarrierSetC2.cpp`
- **Notice:** how ZGC intercepts C2's IR generation to inject Load Barriers.

---

## 10. Execution Flow – Concurrent Relocation in ZGC

1. ZGC decides to compact Region A. It allocates space in Region B.
2. ZGC starts copying objects from A to B in a background C++ thread.
3. **Concurrently**, the Java application executes `User u = cache.getUser();`
4. The JIT‑compiled code contains a **Load Barrier** – it checks the color bits of the pointer.
5. The CPU detects the pointer is "bad" (the object is in Region A, currently being evacuated).
6. The Java thread jumps to the ZGC slow‑path C++ stub.
7. The stub checks the Forwarding Table:
   - **If already copied** → reads the new address (Region B), updates the pointer in `cache` (Self‑Healing), and continues.
   - **If not yet copied** → the Java thread itself copies the object to Region B, updates the Forwarding Table, updates the pointer, and continues.
8. The Java thread resumes – **it never experienced a STW pause!**

---

## 11. Real Java Example – Write & Read Barriers

```java
public class Cache {
    private Object[] elements = new Object[1000];

    public void updateCache(int index, Object newObj) {
        // G1 Write Barrier triggers here!
        // JIT executes assembly to log that the region holding 'elements'
        // now contains a pointer to the region holding 'newObj'.
        elements[index] = newObj;
    }

    public Object readCache(int index) {
        // ZGC / Shenandoah Load Barrier triggers here!
        // JIT executes assembly to check if the pointer returned
        // needs to be remapped before handing it to Java logic.
        return elements[index];
    }
}
```

---

## 12. Why This Design? (The "Why")

**Why do ZGC and Shenandoah guarantee sub‑millisecond pauses regardless of heap size?**

In G1, with 50GB of live data to move, the application pauses while the CPU copies 50GB. Memory bandwidth is finite – this math can't be beaten.

ZGC and Shenandoah **shift the cost**. The GC copies objects in the background. If the application thread needs an object *right now*, it helps copy it (via the Load Barrier). The only STW pause required is a tiny microsecond pause to scan the GC Roots (Thread Stacks). **Pause time is tied to the number of threads, not the heap size.**

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| "Concurrent GC means zero pauses." | All modern GCs have STW pauses. ZGC and Shenandoah keep them <1ms by limiting them to root scanning, not object copying. |
| "ZGC/Shenandoah are always better than G1." | They use load barriers on **every** object read, adding 5–15% CPU overhead. For maximum throughput (e.g., batch processing), G1 or Parallel GC is superior. |

---

## 14. Summary

- **G1** – regions + Write Barriers + RSets + STW evacuation. Balanced throughput/latency.
- **ZGC** – regions + **Colored Pointers** + Load Barriers + concurrent evacuation. Ultra‑low latency.
- **Shenandoah** – regions + forwarding pointers + Load Barriers + concurrent evacuation. Ultra‑low latency.

ZGC and Shenandoah achieve sub‑millisecond pauses by letting Java threads help move objects via Load Barriers – the cost is paid during application execution, not during GC pauses.

---

## 15. Mental Model to Remember

| Collector | Key Feature | Barrier Type | Pause Size |
| :-------- | :---------- | :----------- | :--------- |
| **G1** | Write Barriers + RSets | Write | 10–200 ms |
| **ZGC** | Colored Pointers | Load | <1 ms |
| **Shenandoah** | Forwarding Pointers | Load | <1 ms |

---

## 16. Important Classes / Structs

- `HeapRegion` (G1)
- `G1RemSet` (G1)
- `ZPage` (ZGC)
- `ZAddress` (ZGC)

---

## 17. Important Functions / Methods

- `G1CollectedHeap::evacuate_collection_set()`
- `ZBarrier::load_barrier_on_oop_field()`

---

## 18. Important Files

- `src/hotspot/share/gc/g1/g1CollectedHeap.cpp`
- `src/hotspot/share/gc/z/zAddress.hpp`
- `src/hotspot/share/gc/shared/barrierSet.hpp`

---

## 19. Code‑Reading Exercises

1. **BarrierSet interface** – open `src/hotspot/share/gc/shared/barrierSet.hpp`. Look at the abstract virtual methods like `write_ref_field_pre` and `load_at`. This is the boundary between the Execution Engine and the GC.

2. **ZGC colored pointers** – open `src/hotspot/share/gc/z/zAddress.hpp` and read the inline methods. See how heavily they rely on 64‑bit bitwise operations.

3. **G1 evacuation** – open `src/hotspot/share/gc/g1/g1CollectedHeap.cpp` and search for `evacuate_collection_set`. Notice it's called from within a Safepoint (STW pause).

---

## 20. Self‑Check Questions

1. When a ZGC Load Barrier detects that a Java thread is trying to read an object being copied, why is it safe for the thread to copy it itself? How does ZGC handle the race if the background GC thread is copying it at the exact same time?

2. G1 uses Write Barriers to maintain RSets. Why don't ZGC or Shenandoah need RSets for cross‑region pointers during concurrent evacuation?

3. Why is ZGC fundamentally impossible to implement on a 32‑bit CPU architecture using its current colored pointer design?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/gc/serial/serialHeap.cpp` | See how simple GC used to be. |
| `src/hotspot/share/gc/parallel/parallelScavengeHeap.cpp` | The throughput king. |

---

## 22. Coming Up Next

**Lesson 16 – Serial & Parallel Garbage Collectors**  
We looked at the hyper‑advanced, concurrent modern GCs first because they dominate today's deployments. But to truly appreciate them – and to know what to use for batch jobs or tiny 1‑core containers – we need to look back at the throughput champions: the Serial and Parallel collectors.
