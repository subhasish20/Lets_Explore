# OpenJDK Internals: Day 20 – Synchronization, Locks & Object Monitors

You have written `synchronized (obj)` in Java. You know it prevents two threads from executing a block of code simultaneously. But how is this actually implemented in a JVM that manages millions of objects and thousands of threads?

If HotSpot allocated an OS-level POSIX mutex (`pthread_mutex_t`) for every Java object upon creation, the JVM would consume gigabytes of native memory for locks that might never be used. Furthermore, calling the OS to acquire a mutex takes thousands of CPU cycles.

In this lesson, we'll study **Lock Inflation**. You'll learn how HotSpot uses the object's `markWord` to implement lock-free, stack-allocated **Lightweight Locking** for uncontended fast-paths, and how it dynamically inflates to a heavyweight C++ `ObjectMonitor` only when multiple threads actually contend for the same object.

---

## 1. The Big Picture (Mental Model)

A Java object starts unlocked. As threads interact with it, the lock "inflates" (escalates) depending on contention.

```
       Phase 1: Unlocked Object
┌────────────────────────────────────────────────────────────────────────┐
│  oop (Java Object)                                                     │
│   └── markWord: [ HashCode | GC Age | 01 (Unlocked bits) ]             │
└────────────────────────────────────────────────────────────────────────┘
            │
            ▼  (Thread 1 locks it without contention)
       Phase 2: Lightweight Lock (Stack-Allocated)
┌────────────────────────────────────────────────────────────────────────┐
│  Thread 1 Stack                         oop (Java Object)              │
│  [ BasicLock (Lock Record) ]  ◀───────   └── markWord: [ Pointer | 00 ]│
│  (Holds the displaced header)                                          │
└────────────────────────────────────────────────────────────────────────┘
            │
            ▼  (Thread 2 tries to lock it, CAS fails, contention!)
       Phase 3: Heavyweight Lock (Inflated)
┌────────────────────────────────────────────────────────────────────────┐
│  oop (Java Object)                                                     │
│   └── markWord: [ Pointer to ObjectMonitor | 10 (Heavyweight bits) ]   │
│                                │                                       │
│                                ▼                                       │
│  [ ObjectMonitor ] (Allocated in Native C-Heap)                        │
│   ├─ _owner: Thread 1                                                  │
│   ├─ _cxq: [ Thread 2, Thread 3 ] (Contention queue, blocking on OS)   │
│   └─ _WaitSet: [ Thread 4 ] (Called obj.wait(), resting)               │
└────────────────────────────────────────────────────────────────────────┘
```

> **Historical Note:** Biased locking was an additional phase before Lightweight Locking, but it was deprecated in JDK 15 (JEP 398) and removed in modern OpenJDK because modern CPU atomic instructions are fast enough that the complex maintenance of biased locks was no longer worth the C++ code complexity.

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | What the Spec says | HotSpot does |
| :------ | :----------------- | :----------- |
| **Monitor** | Every object has an associated monitor. | Does **not** allocate a monitor for every object – uses inflation. |
| **Bytecodes** | Defines `monitorenter` and `monitorexit`. | Implements them with fast assembly paths and slow C++ fallbacks. |
| **wait/notify** | Requires support for `wait()`, `notify()`, `notifyAll()`. | Implemented via `ObjectMonitor::wait()` and `ObjectMonitor::notify()`. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/runtime/` | The core synchronization engine. |
| **Key files** | `synchronizer.hpp` / `.cpp` (inflation orchestration), `objectMonitor.hpp` / `.cpp` (heavyweight monitor), `basicLock.hpp` (lightweight lock record). |

---

## 4. Key Concepts You Need to Know

### CAS (Compare-And-Swap)
The fundamental hardware instruction (e.g., `LOCK CMPXCHG` on x86) used to build all lock-free synchronization. It atomically performs: *"Look at memory address X. If its value is Y, change it to Z."* HotSpot relies heavily on CAS to avoid OS locks on the fast path.

### Displaced Mark Word
When a thread successfully lightweight-locks an object, it overwrites the object's `markWord` with a pointer to its own stack. But what happens to the object's HashCode and GC age? Before overwriting, the thread copies the original `markWord` into its stack-allocated `BasicLock`. This copy is the **displaced mark word**. When unlocking, it copies it back.

---

## 5. Architecture – How Locking Works

### Fast Path (Assembly)
The Interpreter or JIT hits `monitorenter`. The assembly stub:
1. Reads the `markWord`.
2. If it's unlocked, the thread allocates a `BasicLock` on its native stack.
3. Copies the `markWord` into the `BasicLock` (displaced header).
4. Executes a CAS to replace the object's `markWord` with a pointer to the `BasicLock`.
5. If successful → **lightweight lock acquired** – all done in userspace!

### Slow Path (C++)
If the CAS fails (another thread owns it, or it's already inflated), the assembly jumps to `InterpreterRuntime::monitorenter`, calling into C++ `ObjectSynchronizer::enter`.

### Inflation
If the lock is contended, HotSpot:
1. Allocates an `ObjectMonitor` in native memory.
2. Copies the displaced `markWord` into the monitor.
3. Updates the object's `markWord` to point to the `ObjectMonitor` (setting the lock bits to `10`).

### Blocking
The calling thread enters `ObjectMonitor::enter`:
1. Tries to **spin** (busy-wait) for a few cycles.
2. If it fails, it puts itself on the `_cxq` (contention queue).
3. Yields to the OS (`os::PlatformEvent`), physically suspending the OS thread.

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `BasicObjectLock` / `BasicLock` | `basicLock.hpp` | Stack-allocated lock record containing the displaced `markWord`. |
| `ObjectSynchronizer` | `synchronizer.hpp` | Static class that coordinates transitions between unlocked, lightweight, and heavyweight states. |
| `ObjectMonitor` | `objectMonitor.hpp` | C++ class representing an inflated lock – contains `_owner`, `_cxq`, `_EntryList`, `_WaitSet`. |
| `markWord` | `markWord.hpp` | The 64-bit header; bits encode lock state, hashcode, GC age. |

---

## 7. Critical Functions

- `ObjectSynchronizer::enter()` / `exit()` – C++ entry points for locking/unlocking.
- `ObjectSynchronizer::inflate()` – complex logic to safely transition from lightweight to heavyweight.
- `ObjectMonitor::enter()` – logic for acquiring an inflated lock (spin + block).
- `ObjectMonitor::wait()` – implementation of `Object.wait()` (releases lock, parks thread).

---

## 8. Important Macros / Utilities

- **Cache line padding** – `ObjectMonitor` structures are strictly padded to CPU cache line boundaries to prevent **false sharing** (when multiple monitors on the same cache line cause unnecessary L1 invalidations).

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: The Synchronizer
- **Open:** `src/hotspot/share/runtime/synchronizer.cpp`
- **Find:** `ObjectSynchronizer::enter`. Observe how it handles the fast path (lightweight via CAS) and delegates to `inflate()` on contention.
- **Find:** `ObjectSynchronizer::inflate`. This is a massive state machine – it has to handle race conditions where multiple threads try to inflate the same object simultaneously!

### Tour 2: The Heavyweight Monitor
- **Open:** `src/hotspot/share/runtime/objectMonitor.hpp`
- **Look for:** fields `_owner` (current holder), `_EntryList` (threads waiting to acquire), and `_WaitSet` (threads that called `wait()`).
- **Look for:** `_recursions` – Java locks are re-entrant; this counts how many times the same thread has locked the monitor.

### Tour 3: Monitor Enter Logic
- **Open:** `src/hotspot/share/runtime/objectMonitor.cpp`
- **Find:** `ObjectMonitor::enter`. Notice the **adaptive spinning** – before putting the thread to sleep (expensive), it spins in a tight loop hoping the owner releases the lock soon.

---

## 10. Execution Flow – Contended `synchronized(obj)`

1. **Thread 1** locks `obj` – fast-path CAS succeeds. `markWord` points to Thread 1's stack.
2. **Thread 2** attempts `synchronized(obj)` – CAS fails. Traps to C++.
3. **Thread 2** enters `ObjectSynchronizer::inflate()`.
4. **Thread 2** allocates an `ObjectMonitor`. It copies Thread 1's displaced `markWord` into the monitor. It CAS's `obj`'s `markWord` to point to the monitor. Lock is now inflated.
5. **Thread 2** calls `ObjectMonitor::enter()`. It sees Thread 1 is the `_owner`.
6. **Thread 2** spins briefly. Fails.
7. **Thread 2** adds itself to `_cxq` and calls OS blocking primitives to sleep.
8. **Thread 1** finishes its block. It tries a fast-path unlock but sees the `markWord` is inflated. It traps to C++ `ObjectMonitor::exit()`.
9. **Thread 1** clears `_owner`, sees Thread 2 in the queue, and wakes Thread 2 via an OS signal.
10. **Thread 2** wakes up, claims `_owner`, and executes the Java code.

---

## 11. Real Java Example – Producer/Consumer

```java
public class ProducerConsumer {
    private final Object lock = new Object();
    private boolean hasData = false;

    public void produce() throws InterruptedException {
        synchronized (lock) {           // Thread A: monitorenter (Fast path - lightweight)
            hasData = true;
            lock.notify();              // Moves Thread B from _WaitSet to _EntryList
        }                               // monitorexit
    }

    public void consume() throws InterruptedException {
        synchronized (lock) {           // Thread B: monitorenter (Contention -> Inflates!)
            while (!hasData) {
                // Thread B releases _owner, placed in _WaitSet
                lock.wait();
            }
            hasData = false;
        }
    }
}
```

---

## 12. Why This Design? (The "Why")

### Why stack-allocate Lightweight Locks?
If 90% of `synchronized` blocks are only ever executed by a single thread at a time (e.g., `StringBuffer` or old `Vector` classes), allocating C-Heap structures and making OS syscalls would be catastrophic. Stack allocation is O(1) – it just moves the stack pointer. The CAS instruction takes a few nanoseconds. The C++ runtime is never invoked unless absolutely necessary.

### Why does `ObjectMonitor::enter` spin before sleeping?
Context switching (putting a thread to sleep and waking it via the kernel) takes tens of thousands of CPU cycles. If the lock holder is going to release it in 500 cycles, it's cheaper for the waiting thread to just burn 500 cycles in a useless `while` loop (spinning) than to go to sleep. HotSpot **adaptively** measures lock hold times to decide how long to spin.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| `synchronized` always pauses the thread in the OS. | **No.** Uncontended locks complete entirely in userspace using atomic CAS instructions – the OS kernel never knows a lock was acquired. |
| `Thread.sleep()` and `Object.wait()` are similar. | `sleep()` keeps ownership of the monitor. `wait()` releases `_owner`, places the thread in `_WaitSet`, and allows other threads to enter. |
| Every Java object needs a mutex. | **No.** HotSpot inflates locks only on contention; most objects never get a monitor allocated. |

---

## 14. Summary

Java's `synchronized` keyword relies on **Lock Inflation**:

- **Uncontended** – ultra-fast, stack-allocated `BasicLock` via CAS on the `markWord`.
- **Contended** – inflates to a native `ObjectMonitor` with `_owner`, `_cxq`, `_EntryList`, and `_WaitSet`.
- **wait/notify** – managed by `ObjectMonitor::wait()` and `ObjectMonitor::notify()`.

This design gives the performance of lightweight locks for the common case, while providing robust blocking semantics when contention actually occurs.

---

## 15. Mental Model to Remember

```
Unlocked  →  Lightweight (CAS markWord to point to Stack BasicLock)
          →  Heavyweight (Inflate to C-Heap ObjectMonitor, block via OS)
```

| Operation | Effect |
| :-------- | :----- |
| `wait()` | Thread goes to `_WaitSet`, releases lock. |
| `notify()` | Moves a thread from `_WaitSet` to `_EntryList`. |
| `notifyAll()` | Moves all threads from `_WaitSet` to `_EntryList`. |

---

## 16. Important Classes / Structs

- `ObjectSynchronizer`
- `BasicLock`
- `ObjectMonitor`
- `markWord`

---

## 17. Important Functions / Methods

- `ObjectSynchronizer::enter()`
- `ObjectSynchronizer::inflate()`
- `ObjectMonitor::enter()`
- `ObjectMonitor::wait()`

---

## 18. Important Files

- `src/hotspot/share/runtime/synchronizer.cpp`
- `src/hotspot/share/runtime/objectMonitor.cpp`
- `src/hotspot/share/runtime/basicLock.hpp`

---

## 19. Code‑Reading Exercises

1. **Inflation logic** – open `src/hotspot/share/runtime/synchronizer.cpp` and find the `inflate` method. Read the `for (;;)` loop – notice how it repeatedly checks the `markWord` to handle races where another thread wins the inflation race.

2. **ObjectMonitor fields** – open `src/hotspot/share/runtime/objectMonitor.hpp` and find `_owner`, `_WaitSet`, and `_EntryList`. This maps exactly to the JVM Specification for wait/notify semantics.

3. **wait implementation** – open `src/hotspot/share/runtime/objectMonitor.cpp` and search for `ObjectMonitor::wait`. Trace the logic: adds the thread to `_WaitSet`, calls `exit()` to drop the lock, then calls `os::PlatformEvent::park()` to suspend via the OS.

---

## 20. Self‑Check Questions

1. Thread A has a lightweight lock on `obj` (`markWord` points to Thread A's stack). Thread B tries to lock `obj`, forcing inflation. How does Thread B safely overwrite the `markWord` to point to an `ObjectMonitor` without corrupting Thread A's execution?

2. Why does calling `obj.wait()` from outside a `synchronized(obj)` block throw `IllegalMonitorStateException`? (Hint: think about what C++ object must exist and who must be `_owner`.)

3. `ObjectMonitor` structures are dynamically allocated on the C-Heap. How and when are they cleaned up (deflated) so the JVM doesn't leak native memory?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/runtime/orderAccess.hpp` | Memory ordering and CPU memory barriers. |
| `src/hotspot/share/utilities/globalDefinitions.hpp` | Global HotSpot types and memory barrier definitions. |

---

## 22. Coming Up Next

**Lesson 21 – Java Memory Model**  
We've locked objects to prevent simultaneous execution. But what about variables accessed *without* locks? If Thread A writes `x = 5`, when is Thread B guaranteed to see `5` instead of `0`? Next, we descend into CPU caches, compiler reordering, and the C++ atomic operations that enforce the Java Memory Model.
