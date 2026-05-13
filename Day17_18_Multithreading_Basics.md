# Day 17–18 — Multithreading Basics
**Level:** Intermediate | **Java Version:** 8+

---

## 🌟 What is a Thread?

A **Thread** is the smallest unit of execution within a process. A single Java application (Process) can have multiple threads running concurrently, sharing the same memory space (Heap), but maintaining their own execution stack.

**Why use multithreading?**
- Better CPU utilization (especially on multi-core processors).
- Keep UI responsive (background tasks don't freeze the main thread).
- Handle multiple clients simultaneously (e.g., web servers).

---

## 🏗️ Creating Threads

There are two primary ways to create a thread in Java:

### 1. Implement `Runnable` (Preferred)
```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Thread is running: " + Thread.currentThread().getName());
    }
}

// Usage
Thread t1 = new Thread(new MyRunnable(), "Worker-1");
t1.start(); // NEVER call run() directly! start() creates the new OS thread.
```
*Why preferred?* Because Java only allows single inheritance. If you implement `Runnable`, your class can still extend another class.

### 2. Extend `Thread`
```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread is running: " + getName());
    }
}

// Usage
MyThread t2 = new MyThread();
t2.start();
```

### 3. Using Lambdas (Modern Approach)
Since `Runnable` is a Functional Interface, you can use lambdas!
```java
Thread t3 = new Thread(() -> {
    System.out.println("Lambda thread running!");
});
t3.start();
```

---

## 🚦 Thread Lifecycle State

A thread goes through several states during its lifetime:
1. **NEW:** Created but `start()` hasn't been called.
2. **RUNNABLE:** Ready to run, waiting for CPU time.
3. **RUNNING:** Currently executing code (often grouped with RUNNABLE in Java).
4. **BLOCKED:** Waiting to acquire a monitor lock to enter a `synchronized` block.
5. **WAITING:** Waiting indefinitely for another thread to perform an action (e.g., calling `Object.wait()`).
6. **TIMED_WAITING:** Waiting for a specified time (e.g., `Thread.sleep(1000)`).
7. **TERMINATED:** Finished execution.

---

## 🛑 The Problem: Race Conditions

When multiple threads access and modify shared data simultaneously, unpredictable results occur.

```java
class Counter {
    private int count = 0;

    // Not thread-safe!
    // count++ is actually 3 operations: read, increment, write
    public void increment() {
        count++; 
    }

    public int getCount() { return count; }
}
```
If two threads call `increment()` at the exact same time, they might both read `count=5`, increment it to `6`, and write `6`. The count should be `7`, but an update was lost.

---

## 🔒 Synchronization

To prevent race conditions, we use **Synchronization** (locking). Only one thread can hold a lock at a time.

### 1. Synchronized Methods
Locks the entire instance (`this`) of the object.
```java
class SafeCounter {
    private int count = 0;

    // Only one thread can execute this method at a time on this object instance
    public synchronized void increment() {
        count++;
    }
}
```

### 2. Synchronized Blocks
Locks only a specific section of code, allowing better performance (finer granularity).
```java
class BlockCounter {
    private int count = 0;
    private final Object lock = new Object(); // Custom lock object

    public void increment() {
        // Other non-critical work can happen here concurrently

        synchronized (lock) { // Thread must acquire the 'lock' object
            count++;          // Critical section
        }                     // Lock is released here
    }
}
```

---

## 💤 Thread Methods

- **`Thread.sleep(ms)`**: Pauses current thread for a specific time. Does NOT release locks.
- **`t1.join()`**: Causes current thread to wait until thread `t1` finishes.
- **`Thread.yield()`**: Current thread hints to the scheduler it is willing to yield its current use of the CPU (rarely used).

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between calling `start()` and `run()` on a Thread?**
> Calling `start()` tells the JVM to allocate a new, separate call stack and OS thread, and then the JVM invokes `run()` on that new thread asynchronously. If you call `run()` directly, it just executes as a normal method call synchronously on the *current* thread. No new thread is created.

**Q2. What is a Deadlock?**
> A Deadlock occurs when two or more threads are blocked forever, waiting for each other to release locks. 
> Example: Thread A holds Lock 1 and waits for Lock 2. Thread B holds Lock 2 and waits for Lock 1. Neither can proceed.
> *Prevention:* Always acquire multiple locks in the exact same order across all threads.

**Q3. What does the `volatile` keyword do?**
> `volatile` guarantees **visibility** of changes to variables across threads. Without it, a thread might cache a variable's value in its CPU register and not see updates made by other threads in main memory. Marking a variable `volatile` forces all reads and writes to go directly to main memory. *Note:* `volatile` does NOT guarantee atomicity (it doesn't fix the `count++` race condition).

**Q4. What is the difference between `wait()` and `sleep()`?**
> `sleep(ms)` is a static method on `Thread` that pauses execution but **keeps all locks** it currently holds. `wait()` is an instance method on `Object` used for inter-thread communication. Calling `wait()` **releases the lock** on that object and puts the thread in a WAITING state until another thread calls `notify()` or `notifyAll()` on that exact same object. `wait()` must be called from within a `synchronized` block.

**Q5. Why is `stop()` deprecated?**
> `Thread.stop()` is deprecated because it is inherently unsafe. It forcibly stops the thread, causing it to immediately release all locks it holds. This can leave shared data structures in an inconsistent or corrupted state. To stop a thread safely, use a volatile boolean flag or the `interrupt()` mechanism.

---

## ✅ Best Practices

1. **Implement `Runnable`** or use lambdas instead of extending `Thread`.
2. **Minimize the scope of `synchronized` blocks.** Don't synchronize entire methods if only a few lines need protection. This reduces contention.
3. **Use final objects for custom locks** (`private final Object lock = new Object();`). Never lock on Strings or boxed primitives because they might be cached/interned and shared elsewhere in the JVM.
4. **Avoid creating Raw Threads.** Managing `new Thread().start()` manually in production code is bad practice. Use the `ExecutorService` (Concurrency Utilities) instead.

---

## 🛠️ Hands-on Practice

1. Write a program where Thread A prints "Ping" and Thread B prints "Pong". Make them run concurrently. Add `Thread.sleep(500)` to slow them down.
2. Create a class with a shared `ArrayList`. Start two threads that each add 1000 items to the list concurrently. Observe the `ConcurrentModificationException` or missing items. Fix it by synchronizing the add method.
3. Write a program demonstrating a Deadlock deliberately using two locks and two threads.
4. Implement a safe thread stop mechanism using a `private volatile boolean running = true;` flag inside a while loop, and a method `stopThread()` that sets it to false.
