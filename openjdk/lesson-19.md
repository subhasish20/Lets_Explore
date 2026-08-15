# OpenJDK Internals: Day 19 – Thread Scheduling, Safepoints & Handshakes

We have threads executing at full speed across millions of instructions per second. But imagine the Garbage Collector needs to relocate objects in memory. If Java Thread A reads an object reference while the GC is actively moving it to a new address, Thread A will corrupt memory or crash the JVM with a segmentation fault.

How do we stop the world safely? We cannot simply send a violent OS signal (like `SIGSTOP`) to a `pthread` because the thread might be in the middle of a critical native OS call or holding an internal VM lock. If we freeze it there, the entire JVM deadlocks.

Instead, HotSpot uses a brilliant cooperative coordination mechanism called **Safepoints** and **Handshakes**. In this lesson, we'll deconstruct how the JVM brings thousands of concurrent threads to a synchronized halt, how safepoint polling is embedded directly into JIT‑compiled machine code, and how modern HotSpot uses handshakes to pause *individual* threads without stopping the entire world.

---

## 1. The Big Picture (Mental Model)

A **Safepoint** is a state where a Java thread's execution context is fully known to the JVM, and no object references are "in transit" (e.g., partially written or in CPU registers without metadata).

```
       Safepoint Coordination
┌────────────────────────────────────────────────────────────────────────┐
│ [ VMThread ] (The Coordinator)                                         │
│ Sets global flag: _safepoint_state = _sync                             │
└──────────────────────────┬─────────────────────────────────────────────┘
                           │
    ┌──────────────────────┴──────────────────────┐
    ▼                                             ▼
┌───────────────────────────┐                 ┌───────────────────────────┐
│ Java Thread 1 (Interpreter)│               │ Java Thread 2 (JIT Code)  │
│ - Hits bytecode dispatch. │                 │ - Executes mov rax, [addr]│
│ - Checks safepoint flag.  │                 │ - Hits Safepoint Poll     │
│ - Sees flag is SET.      │                 │   (Reads unmapped page)   │
│ - Yields at safepoint.    │                 │ - Traps to OS signal      │
│                           │                 │   handler (SIGSEGV).      │
│                           │                 │ - Suspends thread.        │
└───────────────────────────┴─────────────────└───────────────────────────┘
    │                                             │
    └──────────────────────┬──────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────────────────────┐
│ All Threads Reached Safepoint!                                         │
│ VMThread executes global task (e.g., GC Mark/Evacuate).                │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | What the Spec says | HotSpot does |
| :------ | :----------------- | :----------- |
| **Thread suspension** | Not specified – doesn't mention how to pause threads. | Implements **cooperative** suspension via Safepoints and Handshakes. |
| **Safepoints** | Not mentioned. | Core mechanism – all Java threads must reach a known safe state before certain VM operations. |
| **Handshakes** | Not mentioned. | Point‑to‑point thread coordination without stopping the world. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/runtime/` | Core Safepoint and Handshake logic. |
| **Key files** | `safepoint.hpp` / `safepoint.cpp`, `handshake.hpp` / `handshake.cpp`, `safepointMechanism.hpp`. |

---

## 4. Key Concepts You Need to Know

### The Safepoint Poll (The "Red Zone" Trick)

How does a JIT‑compiled method check if it needs to pause, **without** adding an expensive `if (safepoint_requested)` branch to every single loop instruction?

HotSpot uses a **memory protection trick**:

1. At compile time, the JIT inserts a **Safepoint Poll** at the end of loops and method returns.
2. This poll simply **reads a byte** from a designated page in memory (the polling page).
3. When the VM wants a safepoint, it uses `mprotect` to change the permissions of that page from Read/Write to **Read‑Protected (None)**.
4. The very next time a thread executes the poll instruction, the OS throws a `SIGSEGV` (Segmentation Fault).
5. HotSpot's custom signal handler intercepts this signal, quietly suspends the thread, and handles the safepoint.

**Result:** Zero branch overhead during normal execution – it's a hardware‑level trap only triggered when a safepoint is actually requested.

### Handshakes

Stopping the entire world just to inspect a single thread's stack (e.g., for a thread dump or biased locking revocation) is overkill. A **Handshake** allows the VM to execute a callback on a *specific* target Java thread while that thread is in a safe state, **without pausing any other threads**.

---

## 5. Architecture – How a Safepoint Works

1. **Request** – The `VMThread` (or any thread requiring global coordination) calls `SafepointSynchronize::begin()`.
2. **Global Flag** – The VM sets the global safepoint state and changes the memory protection of the polling page.
3. **Cooperative Polling**:
   - **Interpreted code** – checks the safepoint flag between bytecode dispatches.
   - **JIT code** – hits the protected poll page, triggers a `SIGSEGV`, and traps into the HotSpot signal handler.
   - **Native code** – when a Java thread calls a JNI native method, its state changes to `_thread_in_native`. The JVM doesn't wait for it to poll; it simply marks it as safe, provided the native code doesn't touch Java objects.
4. **Execution** – once all threads reach a safepoint, the `VMThread` executes its mission (e.g., GC).
5. **Resumption** – the VM restores the polling page permissions, clears the global flag, and threads resume execution.

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `SafepointSynchronize` | `safepoint.hpp` | Static class that manages starting, running, and ending global safepoints. |
| `SafepointMechanism` | `safepointMechanism.hpp` | Abstracts how threads poll (page polling vs. explicit state flags). |
| `Handshake` | `handshake.hpp` | Coordinates point‑to‑point operations on individual threads. |
| `ThreadSafepointState` | `safepoint.hpp` | C++ helper object attached to every thread tracking its progress to a safepoint. |

---

## 7. Critical Functions

- `SafepointSynchronize::begin()` – brings the world to a halt.
- `SafepointSynchronize::end()` – releases the threads.
- `SafepointMechanism::block_if_requested(JavaThread*)` – the function a thread executes when it hits a safepoint poll to suspend itself.
- `Handshake::execute(HandshakeClosure*)` – executes a task on a target thread safely.

---

## 8. Source Code Exploration (Guided Tour)

### Tour 1: The Safepoint Engine
- **Open:** `src/hotspot/share/runtime/safepoint.cpp`
- **Find:** `SafepointSynchronize::begin()`. Notice how it loops through all threads, updates their state, polls the page, and waits until every thread has transitioned to a safe state.

### Tour 2: The Polling Mechanism
- **Open:** `src/hotspot/share/runtime/safepointMechanism.hpp`
- **Look for:** how it handles page‑polling vs. global word polling, and how the OS signal handler intercepts safepoint traps.

### Tour 3: Thread Handshakes
- **Open:** `src/hotspot/share/runtime/handshake.cpp`
- **Find:** `HandshakeState::process_self_requests()`. This shows how a thread checks if another subsystem has queued a handshake request for it to execute safely.

---

## 9. Execution Flow – A Safepoint Triggered by GC

1. Parallel GC needs to clear weak references. It calls `SafepointSynchronize::begin()`.
2. The `VMThread` changes the global polling page to `PROT_NONE`.
3. **Thread 1** is executing a tight JIT loop. It hits the instruction `test 0x0, [poll_address]`. Because the page is protected, the CPU traps and fires `SIGSEGV`.
4. HotSpot's registered C++ signal handler (`os::is_safepoint_playable()`) catches the signal. It recognises the instruction address belongs to a Safepoint Poll.
5. The signal handler redirects execution to `SafepointMechanism::block_if_requested()`.
6. Thread 1 updates its internal state to `_thread_blocked` and waits on a condition variable.
7. Once **all** threads reach this state, `SafepointSynchronize::begin()` returns control to the `VMThread`.
8. The GC executes its work safely.
9. `SafepointSynchronize::end()` resets the polling page to `PROT_READ` and unblocks all threads.

---

## 10. Real Java Example – The Tight Loop

```java
public class SafepointTest {
    public static void main(String[] args) {
        // A tight loop with NO method calls and NO object allocations.
        // In older JVM versions, a tight loop like this could cause a
        // "Safepoint Blocker" because the JIT compiler forgot to insert
        // a poll instruction! Modern JIT compilers automatically insert
        // a safepoint poll at every loop backedge.
        long sum = 0;
        while (true) {
            sum++;
        }
    }
}
```

---

## 11. Why This Design? (The "Why")

**Why use a protected memory page instead of an explicit `if` check in JIT code?**

An explicit `if (SafepointSynchronize::_state != _not_synced)` executed millions of times per second inside a hot loop would add branch prediction overhead and degrade performance.

The memory protection trick (`PROT_NONE`) converts the check into a **hardware‑level feature**. When there is no safepoint, execution speed is 100% native with **zero branch instructions**. The penalty is paid *only* when a safepoint is actively requested – and at that point, the JVM is already pausing, so the overhead is irrelevant.

---

## 12. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| Threads are killed or suspended by the OS kernel during a Safepoint. | Threads are **cooperative**. The JVM sets a trap or flag, and threads **voluntarily** suspend themselves when they reach a safe instruction boundary. |
| Every line of Java code is a safepoint. | **No.** Safepoints only occur at specific locations: method returns, loop back‑edges, and allocation sites where the JIT/interpreter has explicitly inserted a poll. |
| `System.gc()` triggers an immediate safepoint. | It requests a GC, which then requests a safepoint – but the actual pause happens cooperatively when threads next poll. |

---

## 13. Summary

Safepoints provide a cooperative mechanism to bring all threads to a synchronized halt. By combining:

- **Memory protection page‑fault traps** (`SIGSEGV`) for JIT code.
- **Periodic flag checks** for the interpreter.

HotSpot achieves **zero‑overhead safety checks** during normal execution. For localized actions, **Thread Handshakes** bypass global Stop‑The‑World pauses entirely, allowing point‑to‑point thread interaction.

---

## 14. Mental Model to Remember

| Mechanism | Scope | How it works |
| :-------- | :---- | :----------- |
| **Safepoint Poll (JIT)** | Global | JIT reads protected page → OS throws `SIGSEGV` → trap catches it → thread suspends voluntarily. |
| **Safepoint Poll (Interpreter)** | Global | Interpreter checks a global flag between bytecodes. |
| **Handshake** | Single thread | Targeted callback executed on a specific thread in a safe state – no global pause. |

---

## 15. Important Classes / Structs

- `SafepointSynchronize`
- `SafepointMechanism`
- `Handshake`
- `ThreadSafepointState`

---

## 16. Important Functions / Methods

- `SafepointSynchronize::begin()`
- `SafepointSynchronize::end()`
- `SafepointMechanism::block_if_requested()`
- `Handshake::execute()`

---

## 17. Important Files

- `src/hotspot/share/runtime/safepoint.cpp`
- `src/hotspot/share/runtime/safepointMechanism.hpp`
- `src/hotspot/share/runtime/handshake.cpp`

---

## 18. Code‑Reading Exercises

1. **Safepoint begin** – open `src/hotspot/share/runtime/safepoint.cpp` and find `SafepointSynchronize::begin()`. Trace the loop that waits for all active Java threads to change their state.

2. **Polling page** – open `src/hotspot/share/runtime/safepointMechanism.hpp` and read how it switches the polling page configuration between `dl_read` and `dl_none`.

3. **Handshake execution** – open `src/hotspot/share/runtime/handshake.cpp` and find the entry point for executing a handshake on a target thread. Observe how it checks if the thread is alive.

---

## 19. Self‑Check Questions

1. Why must a thread be in a "safe state" before the Garbage Collector can scan its stack frames? What would happen if the GC scanned a stack frame while a JIT‑compiled method was actively shuffling registers?

2. Explain why a tight `while(true)` loop without method calls or object allocations could theoretically lock up a JVM in older versions, and how modern JIT compilers prevent this "Safepoint Bug."

3. How does a Thread Handshake differ conceptually from a global Stop‑The‑World Safepoint, and when would the JVM prefer to use a handshake?

---

## 20. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/runtime/objectMonitor.hpp` | How threads coordinate when they encounter synchronization locks. |
| `src/hotspot/share/runtime/synchronizer.hpp` | The modern locking subsystem (biased locking, thin locks, etc.). |

---

## 21. Coming Up Next

**Lesson 20 – Synchronization, Locks & Object Monitors**  
We now know how threads are scheduled and paused. But how do they coordinate when accessing shared data? Next, we'll dive into Java's locking primitives – `synchronized`, `wait()`, `notify()` – and explore HotSpot's implementation of Object Monitors, biased locking, and the lock coarsening optimisations performed by the JIT compiler.
