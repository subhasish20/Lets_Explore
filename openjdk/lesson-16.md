# OpenJDK Internals: Day 16 – Serial & Parallel Garbage Collectors

In Lesson 15, we studied the hyper‑advanced, ultra‑low‑latency Garbage Collectors (G1, ZGC, Shenandoah). They use complex regions and inject heavy barriers into the JIT‑compiled machine code to move objects concurrently.

But what if you're running an overnight batch data science job? You don't care if the application pauses for 2 seconds – as long as the total job finishes in 4 hours instead of 5. Or what if you're deploying a tiny microservice to a Kubernetes pod with exactly 1 CPU core? ZGC's background threads would starve your application.

In this lesson, we look at the **raw throughput kings**: the **Serial** and **Parallel** Garbage Collectors. These collectors implement the classic **Weak Generational Hypothesis** using strict physical memory segregation. They strip away complex concurrent machinery in favour of absolute maximum execution speed for the mutator threads, paying the price with monolithic Stop‑The‑World (STW) pauses.

---

## 1. The Big Picture (Mental Model)

Both Serial and Parallel GC use a strict, physically contiguous generational memory layout.

```
       The Classic Generational Heap Layout
┌────────────────────────────────────────────────────────────────────────┐
│                              Java Heap                                 │
│                                                                        │
│   [        Young Generation (NewGen)       ]   [   Old Generation  ]   │
│   ┌────────────┬────────────┬────────────┐     ┌───────────────────┐   │
│   │    Eden    │ Survivor 0 │ Survivor 1 │     │      Tenured      │   │
│   │            │   (From)   │    (To)    │     │                   │   │
│   └────────────┴────────────┴────────────┘     └───────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘

       Execution Timeline: Parallel GC
┌────────────────────────────────────────────────────────────────────────┐
│ Mutator Threads:  ████████████│          │██████████████│          │██ │
│ Mutator Threads:  ████████████│   STW    │██████████████│   STW    │██ │
│ Mutator Threads:  ████████████│  PAUSE   │██████████████│  PAUSE   │██ │
│                               │          │              │          │   │
│ GC Worker 1:                  │▒▒▒▒▒▒▒▒▒▒│              │▓▓▓▓▓▓▓▓▓▓│   │
│ GC Worker 2:                  │▒▒▒▒▒▒▒▒▒▒│              │▓▓▓▓▓▓▓▓▓▓│   │
│                                (Young GC)                (Old GC)      │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | Spec says... | HotSpot does... |
| :------ | :----------- | :-------------- |
| **Heap layout** | Not specified. | Implements a strictly generational heap (Young + Old). |
| **Collection** | Not defined. | Uses Stop‑The‑World pauses exclusively; no concurrent phases. |
| **Threading** | Not mandated. | Serial = 1 GC thread; Parallel = N GC threads (parallel STW). |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/gc/serial/` | Single‑threaded collector (DefNew, Tenured). |
| `src/hotspot/share/gc/parallel/` | Multi‑threaded throughput collector. |
| **Key files** | `defNewGeneration.cpp`, `tenuredGeneration.cpp`, `parallelScavengeHeap.cpp`, `psScavenge.cpp`. |

---

## 4. Key Concepts You Need to Know

### The Copying Algorithm (Young GC)
Because ~98% of objects in Eden die quickly, it's inefficient to find the dead ones and free them individually. Instead, HotSpot finds the 2% that are *live*, copies them to a Survivor space, and then **instantly reclaims the entire Eden** by simply resetting a single memory pointer to 0.

### Mark‑Sweep‑Compact (Old GC)
You can't use the Copying algorithm for the Old Generation – that would require reserving a huge empty "To" space, wasting 50% of RAM. Instead, Old GC uses **Mark‑Sweep‑Compact**:

1. **Mark** – identifies all live objects.
2. **Sweep** – records the gaps left by dead objects.
3. **Compact** – slides live objects to one end of the space to eliminate fragmentation.

### The Card Table (Write Barrier)
During a Minor GC (only Young generation), how does the GC know if an Old‑generation object points to a Young object? It can't scan the whole Old gen. HotSpot uses a **Card Table** – a byte array where each byte represents ~512 bytes of heap memory. When Java code writes an object reference, the JIT injects a tiny write barrier that marks the corresponding card as "dirty". During Minor GC, only dirty cards in the Old gen are scanned.

---

## 5. Architecture – How It Works

1. **Allocation** – threads allocate into Eden using TLABs (Lesson 13). There is almost zero GC overhead injected into the JIT code (only a simple card‑table write barrier).
2. **Minor GC (Young)** – when Eden fills up, a STW pause occurs. The GC traces from Roots and from dirty cards in the Old gen. Live objects in Eden and Survivor‑From are copied to Survivor‑To. If an object has survived enough Minor GCs (its age reaches the `TenuringThreshold`), it is **promoted** to the Old Generation.
3. **Major GC (Old)** – when the Old Generation fills up, a massive STW pause occurs. The GC performs a full Mark‑Sweep‑Compact on the entire heap.
4. **Threading:**
   - **Serial GC** – uses exactly 1 C++ thread for all copying and compacting.
   - **Parallel GC** – uses N C++ threads (usually equal to CPU cores) to do the work in parallel, slashing pause times.

---

## 6. Important C++ Classes / Structs

### Serial
- `DefNewGeneration` – the young generation implementation.
- `TenuredGeneration` – the old generation implementation.

### Parallel
- `ParallelScavengeHeap` – the overarching heap class.
- `PSYoungGen` / `PSOldGen` – the generational memory spaces.
- `PSScavenge` – coordinates parallel copying of young objects.
- `PSMarkSweep` / `PSParallelCompact` – full heap compaction.

### Shared
- `CardTable` – the byte array tracking cross‑generation references.

---

## 7. Critical Functions

- `DefNewGeneration::collect()` – executes the Serial Young GC.
- `PSScavenge::invoke()` – entry point for a Parallel Young GC pause.
- `PSParallelCompact::invoke()` – entry point for a Parallel Old GC pause.

---

## 8. Important JVM Flags

| Flag | Effect |
| :--- | :--- |
| `-XX:+UseSerialGC` | Enable Serial GC (1 thread, low memory footprint). |
| `-XX:+UseParallelGC` | Enable Parallel GC (multi‑threaded throughput). |
| `-XX:ParallelGCThreads=N` | Set the number of GC threads. |

> **Note:** `UseParallelGC` was the default in Java 8. In modern Java, G1 is the default.

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: Parallel Scavenge (Young GC)
- **Open:** `src/hotspot/share/gc/parallel/psScavenge.cpp`
- **Find:** `PSScavenge::invoke()`. Scroll through to see how it sets up `GCTaskManager` to distribute work to multiple GC threads. Notice the phase where it swaps the Survivor pointers (`From` ↔ `To`).

### Tour 2: Serial Old GC
- **Open:** `src/hotspot/share/gc/serial/tenuredGeneration.cpp`
- **Look for:** `TenuredGeneration::collect()`. It delegates heavily to the shared `GenMarkSweep` utility.

### Tour 3: The Card Table
- **Open:** `src/hotspot/share/gc/shared/cardTable.hpp`
- **Read the comments** – see how a heap pointer is mapped to a byte index in the `_byte_map` array. This is the essence of the write barrier.

---

## 10. Execution Flow – Parallel Minor GC

1. Java thread fills its TLAB. Eden is 100% full.
2. `CollectedHeap::allocate_new_tlab` fails → triggers `PSScavenge::invoke()`.
3. **Safepoint** halts all Java threads.
4. `GCTaskManager` wakes up 8 GC worker threads.
5. Workers divide the GC Roots (Stacks, JNI) and the dirty cards in the Card Table.
6. Workers trace the graph, finding live objects in Eden.
7. Workers **copy** live objects to `Survivor‑To` using atomic CAS to claim allocation buffers.
8. Pointers in Stacks and Old generation are updated to the new `Survivor‑To` addresses.
9. Eden and `Survivor‑From` are declared 100% empty (pointer reset).
10. Safepoint ends – Java threads resume instantly.

---

## 11. Real Java Example – Throughput Matters

```java
public class BatchProcessor {
    public static void main(String[] args) {
        // For batch processing (AI training, CSV parsing, video encoding),
        // you want maximum CPU throughput. Parallel GC gives 100% of CPU
        // to your code – no read barriers, no concurrent overhead.

        // Run with: -XX:+UseParallelGC -XX:ParallelGCThreads=4
        List<Data> dataset = loadMassiveData();

        for (Data data : dataset) {
            // High allocation of short‑lived maths objects
            Result r = performComplexMath(data);
            save(r);
        }
    }
}
```

---

## 12. Why This Design? (The "Why")

**Why are Serial and Parallel still maintained when G1 and ZGC exist?**

- **Serial GC** – perfect for tiny containers (1 CPU core, <512MB RAM). Spinning up background threads for G1/ZGC would be catastrophic overhead. Serial has the lowest memory footprint and simplest code.
- **Parallel GC** – the "Throughput Collector". For Hadoop, Spark, or overnight ETL, a 3‑second STW pause is irrelevant. Parallel GC has no read barriers, so the Java application runs at **absolute maximum CPU speed**.

> **Tradeoff:** Serial/Parallel sacrifice latency for maximum throughput and minimum GC footprint. They are deterministic and predictable.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| "Stop‑The‑World is always bad." | STW is highly efficient – no concurrency locks or barriers on mutator threads. It's a deliberate tradeoff for throughput. |
| "Parallel GC is concurrent." | **Parallel** = multiple GC threads work **while the application is paused**. **Concurrent** = GC threads work **while the application runs**. They are different! |
| "I should always use ZGC." | For batch jobs and latency‑insensitive workloads, Parallel GC gives better raw throughput. |

---

## 14. Summary

- **Serial & Parallel GC** are classical, generational collectors.
- **Heap layout** – Young (Eden + 2 Survivors) + Old (Tenured).
- **Young GC** – copying collector (STW, fast).
- **Old GC** – Mark‑Sweep‑Compact (STW, slower).
- **Serial** – 1 GC thread; **Parallel** – N GC threads (scales with CPU).
- They impose **zero read barriers** – maximum mutator throughput.

---

## 15. Mental Model to Remember

```
Generations: Eden + S0/S1  →  Tenured (Old)
Young GC:  Copy live objects, wipe the rest (STW, fast)
Old GC:    Mark‑Sweep‑Compact (STW, slower)
Serial:    1 thread
Parallel:  N threads (throughput focused)
```

---

## 16. Important Classes / Structs

- `ParallelScavengeHeap`
- `PSYoungGen` / `PSOldGen`
- `PSScavenge`
- `CardTable`

---

## 17. Important Functions / Methods

- `PSScavenge::invoke()`
- `PSParallelCompact::invoke()`
- `DefNewGeneration::collect()`

---

## 18. Important Files

- `src/hotspot/share/gc/parallel/psScavenge.cpp`
- `src/hotspot/share/gc/parallel/parallelScavengeHeap.cpp`
- `src/hotspot/share/gc/serial/defNewGeneration.cpp`
- `src/hotspot/share/gc/shared/cardTable.cpp`

---

## 19. Code‑Reading Exercises

1. **Card Table** – open `src/hotspot/share/gc/shared/cardTable.hpp` and read the comments. See how it maps a heap address to a byte in the dirty card array.

2. **Parallel allocation failure** – open `src/hotspot/share/gc/parallel/parallelScavengeHeap.cpp` and find `mem_allocate`. Notice how it tries Young allocation first, then triggers `PSScavenge`.

3. **Serial Young GC** – open `src/hotspot/share/gc/serial/defNewGeneration.cpp` and look at `DefNewGeneration::collect()`. Notice how straightforward the STW collection is – no background concurrency.

---

## 20. Self‑Check Questions

1. Why does the Young Generation have **two** Survivor spaces (From and To)? Walk through what happens during two consecutive Minor GCs.

2. An application allocates a massive 500 MB `byte[]` array, but Eden is only 200 MB. How does Parallel GC handle this allocation?

3. Why does ZGC (concurrent) require a Load Barrier, but Parallel GC only needs a simple Card Table write barrier? What race condition is Parallel GC immune to?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/services/memTracker.hpp` | How HotSpot tracks native memory usage outside the Java heap. |
| `src/hotspot/share/memory/allocation.hpp` | Base classes for C++ memory allocation inside HotSpot. |

---

## 22. Coming Up Next

**Lesson 17 – Native Memory Management**  
We've covered the Java heap inside and out. But HotSpot also uses a lot of memory outside the heap – Metaspace, Code Cache, thread stacks, and internal C++ structures. Next, we'll look at how HotSpot tracks and manages its native memory footprint.
