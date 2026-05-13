# Day 22–23 — JVM Architecture & Garbage Collection
**Level:** Advanced | **Java Version:** 8+

---

## 🌟 The Java Virtual Machine (JVM)

Java's motto is "Write Once, Run Anywhere".
1. **Compiler (javac):** Converts `.java` code into `.class` files (Bytecode).
2. **JVM:** An abstract machine that reads Bytecode and executes it on the specific underlying hardware/OS.

### The 3 Main Components of the JVM

1. **Class Loader Subsystem:** Loads, links, and initializes `.class` files into RAM.
2. **Runtime Data Areas (Memory):** Where data is stored during execution.
3. **Execution Engine:** Interprets bytecode, compiles hotspots to native code (JIT), and runs Garbage Collection.

---

## 🧠 Runtime Data Areas (JVM Memory)

Understanding memory is crucial for fixing `OutOfMemoryError` and optimizing performance.

### 1. Method Area (Metaspace since Java 8)
- **What it stores:** Class-level data, static variables, method code, and the constant pool.
- **Scope:** Shared across ALL threads.
- **Note:** Pre-Java 8 it was called "PermGen" and had a fixed size. Now it's "Metaspace" and dynamically scales using native OS memory.

### 2. Heap Area
- **What it stores:** **ALL Objects** (`new Keyword`) and their instance variables (arrays are also objects).
- **Scope:** Shared across ALL threads.
- **Note:** This is where **Garbage Collection (GC)** happens. If this fills up, you get `java.lang.OutOfMemoryError: Java heap space`.

### 3. Stack Area
- **What it stores:** Local variables (primitives and object *references*), method calls, and partial results.
- **Scope:** **One Stack per Thread.** Not shared! (This is why local variables are inherently thread-safe).
- **Note:** If recursion goes too deep, the stack fills up, throwing `java.lang.StackOverflowError`.

### 4. PC Register (Program Counter)
- **What it stores:** The memory address of the JVM instruction currently being executed.
- **Scope:** One per Thread.

### 5. Native Method Stack
- **What it stores:** State for native code (methods written in C/C++ accessed via JNI).
- **Scope:** One per Thread.

---

## 🗑️ Garbage Collection (GC)

In C/C++, you must manually `free()` memory. If you forget, you get a memory leak. If you free it twice, the app crashes.
In Java, a daemon thread called the **Garbage Collector** automatically destroys objects that are no longer reachable by the application.

### How does GC determine what to destroy?
**Reachability Analysis:** GC starts from "GC Roots" (local variables in active threads, static variables). It traces a graph of all object references. Any object on the heap that CANNOT be reached from a GC Root is marked as "garbage" and deleted.

---

## 🏗️ Heap Architecture (Generational GC)

Objects have different lifespans. Empirical analysis shows that **most objects die young** (e.g., local variables in a loop). The JVM Heap is split into "Generations" to optimize collection.

### 1. Young Generation (Where objects are born)
Split into 3 parts:
- **Eden Space:** All new objects are allocated here.
- **Survivor 0 (S0) & Survivor 1 (S1):** Objects that survive a GC in Eden move here.
*Process:* When Eden is full, a **Minor GC** runs. It's very fast. Surviving objects move to a Survivor space. An object's "age" increments.

### 2. Old (Tenured) Generation (Long-lived objects)
- Objects that survive multiple Minor GCs (usually 15 cycles) are promoted to the Old Gen.
- Large objects may be allocated here directly.
*Process:* When Old Gen is full, a **Major GC (or Full GC)** runs. This is slow and triggers a **"Stop-The-World" (STW)** pause, freezing your application threads entirely while it cleans up.

---

## ⚙️ Popular Garbage Collectors

Different apps have different needs (Low latency vs High throughput). You can choose the GC via JVM flags.

1. **Serial GC:** Single-threaded. Pauses the app completely. Good for small apps/client machines. (`-XX:+UseSerialGC`)
2. **Parallel GC:** Uses multiple threads for Minor GC. Optimizes for **Throughput** (raw processing power). Was default in Java 8. (`-XX:+UseParallelGC`)
3. **G1 GC (Garbage First):** The default since Java 9. Divides the heap into small regions. Optimizes for **Predictable Low Latency**. It tries to meet a pause-time target (e.g., max 200ms pause). (`-XX:+UseG1GC`)
4. **ZGC (Z Garbage Collector):** Java 11+. Designed for massive heaps (Terabytes). Max pause time of < 1ms! Almost entirely concurrent. (`-XX:+UseZGC`)

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between Heap and Stack memory?**
> **Heap** is used for dynamic memory allocation of Java objects (`new` keyword). It is shared across all threads and managed by the Garbage Collector. **Stack** is used for thread execution, storing local variables (primitives) and references to heap objects. Each thread has its own stack. Memory is automatically cleared when a method returns (LIFO order).

**Q2. Can a Java application have memory leaks?**
> Yes! While Java handles unreferenced objects automatically, a leak occurs when an object is no longer logically needed by the program, but a reference to it is still held (e.g., caching objects in a static `HashMap` and forgetting to remove them). Since the GC can still "reach" the object from a static root, it will never be deleted, eventually causing an `OutOfMemoryError`.

**Q3. What is a "Stop-The-World" (STW) event?**
> It's a phase during Garbage Collection where all application threads are completely paused (suspended) so the GC can safely move objects around and reclaim memory without the app altering references simultaneously. Modern GCs (like G1 and ZGC) try to minimize STW pauses to sub-milliseconds.

**Q4. What is the difference between Minor GC and Major GC?**
> **Minor GC** cleans the Young Generation (Eden + Survivor spaces). It runs frequently, is very fast, and usually causes unnoticeable pauses. **Major GC (Full GC)** cleans the Old (Tenured) Generation. It is triggered when Old Gen is full. It is much slower and causes significant Stop-The-World pauses, which can degrade application latency.

**Q5. How does the JVM handle Strings in memory?**
> String literals are stored in the **String Constant Pool**, which is a special area inside the Heap memory. If you use `new String("text")`, it bypasses the pool and creates a standard object in the heap. The pool helps save memory by reusing immutable string instances.

**Q6. What happens if you call `System.gc()`?**
> It acts as a *suggestion* to the JVM that now is a good time to run Garbage Collection. However, the JVM is free to ignore this request. Relying on `System.gc()` in production code is considered a severe anti-pattern because it usually triggers a Full GC, killing application performance.

---

## ✅ Best Practices / Tuning Flags

- **`-Xms` and `-Xmx`**: Set the initial and maximum heap size. Best practice is to set them to the *same value* in production (e.g., `-Xms4G -Xmx4G`) to prevent the OS from constantly resizing the JVM memory, which causes stuttering.
- **Avoid object churn:** Continuously creating and destroying millions of short-lived objects forces the GC to work overtime. Reuse objects where possible (e.g., use `StringBuilder` instead of `String` concatenation in loops).
- **Use Profilers:** Use tools like VisualVM or JProfiler to track memory usage and identify memory leaks before they crash the app.
- **Generate Heap Dumps on OOM:** Always run production apps with `-XX:+HeapDumpOnOutOfMemoryError`. If the app crashes, it writes a `.hprof` file to disk containing the exact memory state at the time of the crash, which you can analyze later.

---

## 🛠️ Hands-on Practice (Mental Exercises)

1. **Stack or Heap?** Where does `int x = 5;` declared inside a method go? Where does `Object obj = new Object();` go? Where does the *reference* `obj` go?
2. Write a small program that intentionally causes a `java.lang.StackOverflowError` (Hint: use an infinite recursive method).
3. Write a small program that intentionally causes a `java.lang.OutOfMemoryError: Java heap space` (Hint: keep adding arrays or large objects to a static List in an infinite loop).
4. Run your OOM program from the terminal using `java -Xmx10m YourClass` to force it to crash faster.
