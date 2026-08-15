# OpenJDK Internals: Day 18 – Threads, Thread Stacks & Stack Frames

Every piece of Java code you have ever written executed on a thread. But what *is* a Java thread under the hood?

If you create 1,000 Java threads, does the JVM simulate them, or does the operating system manage them? Where do local variables actually go when a method is called?

In this lesson, we'll peel back the abstraction of `java.lang.Thread`. We'll explore the **1:1 mapping** between Java threads and OS threads (Platform Threads). We'll dive into the C++ classes that represent these threads, and most importantly, we'll dissect the **Thread Stack** – the raw OS memory where the JVM interleaves C++ execution frames with Java execution frames.

---

## 1. The Big Picture (Mental Model)

A Java thread is a thin wrapper over a heavy OS thread. The execution stack is a single, contiguous block of native memory provided by the OS.

```
       The Thread Object Topology
┌────────────────────────────────────────────────────────┐
│  Java Heap                                             │
│  [ java.lang.Thread ] (Java Object)                    │
│           │ (eetop pointer)                            │
└───────────┼────────────────────────────────────────────┘
            ▼
┌────────────────────────────────────────────────────────┐
│  Native C-Heap                                         │
│  [ JavaThread ] (C++ Object, inherits from Thread)     │
│           │ _osthread                                  │
│           ▼                                            │
│  [ OSThread ] (C++ Object, Platform-specific)          │
│           │ _thread_id                                 │
└───────────┼────────────────────────────────────────────┘
            ▼
┌────────────────────────────────────────────────────────┐
│  Operating System (e.g., Linux Kernel)                 │
│  [ pthread ] (Actual OS Thread Structure)              │
│           │                                            │
│           ▼ (Owns 1MB of Virtual Memory for the Stack) │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Native OS Stack (Grows Downward)                 │  │
│  │                                                  │  │
│  │ [ C++ Frame: JVM_StartThread ]                   │  │
│  │ [ C++ Frame: JavaThread::run ]                   │  │
│  │ [ Java Frame: Interpreter (MyClass.main) ]       │  │
│  │ [ Java Frame: Compiled C2 (MyClass.calculate) ]  │  │
│  │ [ C++ Frame: JNI Method (System.nanoTime) ]      │  │
│  │                                                  │  │
│  │ <--- Stack Pointer (RSP)                         │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | What the Spec says | HotSpot does |
| :------ | :----------------- | :----------- |
| **Java Virtual Machine Stack** | Separate from Native Method Stack. | **Merges them** – Java and C++ frames are interleaved on the same OS stack. |
| **Thread mapping** | Not specified. | Uses a **1:1 mapping** (Platform Threads) – one Java thread = one OS thread. |
| **Stack inspection** | Not specified. | Uses the `frame` C++ struct to parse raw stack memory. |

> **Note:** Project Loom introduced **Virtual Threads** (M:N scheduling), but that's a separate topic (covered in later lessons). This lesson covers the traditional Platform Thread model.

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/runtime/` | Platform‑independent thread and frame logic. |
| `src/hotspot/os/linux/` | Platform‑specific OS thread creation (`pthread_create`). |
| **Key files** | `thread.hpp`, `javaThread.hpp`, `osThread.hpp`, `frame.hpp`. |

---

## 4. Key Concepts You Need to Know

### The Stack Frame
When the CPU executes a function, it pushes a **frame** onto the stack. This frame contains:
- The **return address** – where to go when the function finishes.
- Space for **local variables**.

Two CPU registers manage this:
- **Stack Pointer (SP/RSP)** – points to the top of the current frame.
- **Frame Pointer (FP/RBP)** – points to the fixed start of the current frame, allowing local variables to be accessed at predictable offsets (e.g., `FP - 8`).

### Thread Local Storage (TLS)
With 100 threads running the same C++ code, how does the code know *which* thread is executing it? The OS provides TLS – a special CPU register (like `FS` or `GS` on x86) that points to data unique to the current thread.

---

## 5. Architecture – How a Java Thread is Created

1. **Java code** calls `new Thread().start()`.
2. This invokes the native method `start0()`.
3. HotSpot allocates a `JavaThread` C++ object in the C‑Heap.
4. The `JavaThread` creates an `OSThread` and calls the OS (e.g., `pthread_create`).
5. The OS allocates a ~1MB stack and starts a new OS‑level thread.
6. The new thread begins executing the C++ entry point `JavaThread::thread_main_inner()`.
7. The C++ code prepares the thread state and calls into the Interpreter via `JavaCalls::call`, executing the `run()` bytecode.
8. **Stack walking** – whenever the JVM needs to inspect the stack (for GC, exception unwinding, or deoptimization), it creates a `frame` C++ object. This reads the CPU's FP/SP registers and parses the raw stack memory to find `oop`s and return addresses.

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `Thread` | `thread.hpp` | The ultimate C++ base class for all HotSpot threads. |
| `JavaThread` | `javaThread.hpp` | Represents a Java application thread (mutator). |
| `VMThread` | `vmThread.hpp` | A singleton internal thread that executes STW operations (like GC). |
| `CompilerThread` | `compilerThread.hpp` | Background JIT compiler threads. |
| `OSThread` | `osThread.hpp` | Platform‑specific thread wrapper (holds `pthread_t` or Windows `HANDLE`). |
| `frame` | `frame.hpp` | A "magnifying glass" – points to raw stack memory and provides methods to parse it. |

---

## 7. Critical Functions

- `JVM_StartThread()` – the JNI entry point called by `java.lang.Thread.start0()`.
- `JavaThread::run()` – the C++ lifecycle method for the new thread.
- `frame::sender()` – given a stack frame, calculates and returns the *caller's* frame, enabling stack walking.

---

## 8. Important Macros / Utilities

- **`THREAD` / `TRAPS`** – in HotSpot C++ code, you'll constantly see functions ending with `(..., Thread* THREAD)`. Fetching the current thread from OS TLS is slow, so HotSpot passes the `Thread*` pointer as an explicit argument to almost every function. `TRAPS` is a macro that expands to this argument.

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: The Thread Class Hierarchy
- **Open:** `src/hotspot/share/runtime/thread.hpp`
- **Look at:** the inheritance tree – `JavaThread`, `NonJavaThread`, and `VMThread` branch off from `Thread`.
- **Notice:** fields for TLABs and memory allocation.

### Tour 2: The JavaThread
- **Open:** `src/hotspot/share/runtime/javaThread.hpp`
- **Look for:**
  - `_threadObj` – points back to the Java `java.lang.Thread` on the heap.
  - `_osthread` – points to the OS wrapper.
  - `_thread_state` – values like `_thread_in_Java`, `_thread_in_native`.

### Tour 3: The Stack Frame Parser
- **Open:** `src/hotspot/share/runtime/frame.hpp`
- **Look at:** `interpreter_frame_local_at(int index)`. This shows how HotSpot calculates the exact memory address of a Java local variable using pointer math relative to the Frame Pointer.

---

## 10. Execution Flow – Creating a Thread

Trace: `Thread t = new Thread(myRunnable); t.start();`

1. Java `Thread.start()` calls native `start0()`.
2. Execution enters `JVM_StartThread` (in `jvm.cpp`).
3. HotSpot allocates `new JavaThread(&thread_entry, sz)`.
4. `JavaThread` constructor allocates `new OSThread()`.
5. HotSpot calls `os::create_thread()`.
6. On Linux, `pthread_create` is invoked, allocating a ~1MB stack and spawning a kernel thread.
7. The OS schedules the thread; it starts executing `thread_native_entry` (in `os_linux.cpp`).
8. The thread transitions to `JavaThread::run()`.
9. The thread state changes to `_thread_in_Java`.
10. `JavaCalls::call_virtual` invokes `myRunnable.run()`.
11. The Java thread is now alive and executing bytecodes.

---

## 11. Real Java Example – Stack Trace Walk

```java
public class StackTraceExample {
    public static void main(String[] args) {
        a();
    }
    static void a() { b(); }
    static void b() { c(); }
    static void c() {
        // When this exception is thrown, the JVM creates a C++ 'frame'
        // starting at the CPU's current Frame Pointer.
        // It calls frame::sender() repeatedly to walk up the raw memory
        // of the OS stack, translating instruction addresses into Java
        // method names using the Code Cache metadata.
        throw new RuntimeException("Stack walk!");
    }
}
```

---

## 12. Why This Design? (The "Why")

### Why pass `Thread* THREAD` to every C++ function?
If a C++ function needs to throw a Java exception or allocate in a TLAB, it needs the current `JavaThread`. It could call `Thread::current()` (which queries OS TLS), but that costs several CPU cycles. Passing the pointer explicitly keeps it in a fast CPU register, maximising performance.

### Why merge Java and C++ stacks?
If Java and C++ had separate stacks, calling JNI methods would require copying arguments from the Java stack to the C stack and managing two stack pointers. By interleaving them, a compiled Java method can call a C++ JNI function using the same standard CPU `call` instruction – just like regular C++ code.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| Local variables live on the Java Heap. | **No.** They live on the Thread Stack (raw OS memory). When a method returns, the stack pointer is decremented and memory is instantly reclaimed – the GC doesn't touch it. |
| Java Threads are lightweight. | Platform Threads map **1:1** to OS threads. OS threads require ~1MB of stack memory and heavy kernel context switches. Creating 100,000 threads will crash the JVM. |
| `Thread.stop()` is just a state change. | **No.** `Thread.stop()` asynchronously throws `ThreadDeath` at an arbitrary bytecode, potentially corrupting objects and locks. It is **deprecated and dangerous**. |

---

## 14. Summary

A Java Platform Thread is a triad of objects:
- `java.lang.Thread` – on the Java Heap.
- `JavaThread` – in the C‑Heap.
- `pthread` – in the OS kernel.

The thread executes on a single, contiguous block of native memory (the Thread Stack), where C++ frames, JNI frames, interpreted Java frames, and JIT‑compiled Java frames are seamlessly interleaved. HotSpot parses this raw memory at runtime using the `frame` C++ struct to perform stack traces, GC root scanning, and deoptimization.

---

## 15. Mental Model to Remember

| Layer | Where it lives | Role |
| :---- | :------------- | :--- |
| `java.lang.Thread` | Java Heap | Java‑visible thread object. |
| `JavaThread` | C‑Heap (Native) | HotSpot's internal representation. |
| `OSThread` | C‑Heap (Native) | Platform‑specific OS wrapper. |
| `pthread` / kernel thread | OS Kernel | Actual OS scheduling entity. |
| Thread Stack | OS Memory (1MB) | Contains interleaved Java and C++ frames. |

---

## 16. Important Classes / Structs

- `JavaThread`
- `OSThread`
- `VMThread`
- `frame`

---

## 17. Important Functions / Methods

- `JVM_StartThread()`
- `JavaThread::run()`
- `frame::sender()`

---

## 18. Important Files

- `src/hotspot/share/runtime/javaThread.hpp`
- `src/hotspot/share/runtime/frame.hpp`
- `src/hotspot/os/linux/os_linux.cpp`
- `src/java.base/share/native/libjava/Thread.c`

---

## 19. Code‑Reading Exercises

1. **Thread states** – open `src/hotspot/share/runtime/javaThread.hpp` and look at the `JavaThreadState` enum. Find `_thread_in_native` and `_thread_in_Java`. These states are critical for the JVM to know if a thread can be safely paused for GC.

2. **Frame structure** – open `src/hotspot/share/runtime/frame.hpp` and look at the `frame` constructor. Notice it takes `sp` (stack pointer), `pc` (program counter), and `fp` (frame pointer).

3. **JNI binding** – open `src/java.base/share/native/libjava/Thread.c` and find `Java_java_lang_Thread_start0`. This binds the Java `start0` native method to `JVM_StartThread`.

---

## 20. Self‑Check Questions

1. A Java method is caught in an infinite recursive loop (`void loop() { loop(); }`). Why does it throw `StackOverflowError` instead of `OutOfMemoryError`? What physical memory boundary did it cross?

2. During a GC pause, the GC must scan thread stacks for GC Roots. If the stack is just raw bytes and CPU frames, how does HotSpot know which 8‑byte values are `oop`s (pointers) and which are integers? (Hint: think about what the Interpreter and JIT must generate to help the GC.)

3. Why is a context switch between two Java threads significantly more expensive than a simple method call within the same thread?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/runtime/safepoint.hpp` | How the JVM stops all threads. |
| `src/hotspot/share/runtime/handshake.hpp` | How the JVM talks to individual threads. |

---

## 22. Coming Up Next

**Lesson 19 – Thread Scheduling, Safepoints & Handshakes**  
We now have thousands of threads running on native OS stacks at full speed. But what happens when the Garbage Collector says, *"Stop! I need to move objects"*? We can't freeze OS threads randomly – they might hold C++ locks or be writing to memory. We need a coordinated, cooperative system to bring the entire JVM to a safe halt. Next, we learn about **Safepoints**.
