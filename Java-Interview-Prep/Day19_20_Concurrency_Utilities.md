# Day 19–20 — Concurrency Utilities
**Level:** Intermediate → Advanced | **Java Version:** 5+ (mostly `java.util.concurrent`)

---

## 🌟 Why `java.util.concurrent`?

Creating raw threads (`new Thread().start()`) in production code is a bad practice. It's expensive (OS resources), hard to manage, and doesn't scale. Java 5 introduced the `java.util.concurrent` package to solve these issues, providing Thread Pools, concurrent collections, and synchronization tools.

---

## 🏊 Thread Pools (`ExecutorService`)

An **ExecutorService** manages a pool of worker threads. You submit tasks to it, and it assigns them to available threads. This reuses threads, avoiding the overhead of creating/destroying them constantly.

```java
import java.util.concurrent.*;

// 1. Create a Thread Pool with 3 fixed threads
ExecutorService executor = Executors.newFixedThreadPool(3);

// 2. Submit Runnable tasks (fire and forget)
for (int i = 1; i <= 5; i++) {
    int taskId = i;
    executor.submit(() -> {
        System.out.println("Executing Task " + taskId + " via " + Thread.currentThread().getName());
        try { Thread.sleep(1000); } catch (Exception e) {}
    });
}

// 3. Shutdown the executor (rejects new tasks, finishes existing ones)
executor.shutdown();
```

### Types of Thread Pools
1. `Executors.newFixedThreadPool(n)`: Exactly `n` threads. Good for limiting resource usage.
2. `Executors.newCachedThreadPool()`: Creates new threads as needed, reuses idle ones. Good for many short-lived tasks.
3. `Executors.newSingleThreadExecutor()`: Just 1 thread. Guarantees tasks execute sequentially.
4. `Executors.newScheduledThreadPool(n)`: For scheduling tasks to run after a delay or periodically.

---

## 🎯 `Callable` and `Future`

While `Runnable` returns `void`, **`Callable<T>`** returns a value of type `T` and can throw checked exceptions.

When you submit a `Callable` to an `ExecutorService`, it immediately returns a **`Future<T>`**. A `Future` represents the pending result of the asynchronous computation.

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

// Callable task
Callable<Integer> heavyMathTask = () -> {
    System.out.println("Calculating...");
    Thread.sleep(2000); // Simulate heavy work
    return 42;
};

// Submit returns a Future immediately
Future<Integer> futureResult = executor.submit(heavyMathTask);

System.out.println("Doing other things on main thread...");

try {
    // get() BLOCKS the current thread until the result is ready!
    Integer result = futureResult.get(); 
    System.out.println("Result is: " + result);
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}

executor.shutdown();
```

---

## 🔐 Synchronizers

High-level locking mechanisms that coordinate threads.

### 1. CountDownLatch
Used when one or more threads must wait for a set of operations in other threads to complete. It cannot be reset.

```java
CountDownLatch latch = new CountDownLatch(3); // Wait for 3 events

Runnable task = () -> {
    // Do work...
    System.out.println("Service initialized.");
    latch.countDown(); // Decrement the count
};

executor.submit(task);
executor.submit(task);
executor.submit(task);

// Main thread waits until count reaches 0
latch.await(); 
System.out.println("All services up. Starting main app.");
```

### 2. CyclicBarrier
Similar to CountDownLatch, but threads wait for *each other* to reach a barrier point. Once all arrive, they all proceed. It CAN be reset/reused. (e.g., Multiplayer game waiting for all players to load).

### 3. Semaphore
Controls access to a shared resource through a system of permits. (e.g., Limiting DB connections to 10 max).

```java
Semaphore semaphore = new Semaphore(2); // Only 2 threads allowed at a time

Runnable limitedTask = () -> {
    try {
        semaphore.acquire(); // Request a permit (blocks if 0 available)
        System.out.println(Thread.currentThread().getName() + " got permit.");
        Thread.sleep(1000);
    } catch (Exception e) {} finally {
        System.out.println(Thread.currentThread().getName() + " releasing permit.");
        semaphore.release(); // Return permit
    }
};
```

---

## ⚛️ Atomic Variables

Located in `java.util.concurrent.atomic`. They use low-level hardware instructions (Compare-And-Swap / CAS) to update variables **thread-safely without locks**. Very fast!

```java
import java.util.concurrent.atomic.AtomicInteger;

class Counter {
    // Replaces: private int count = 0; + synchronized increment()
    private AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet(); // Thread-safe atomic operation
    }

    public int getCount() {
        return count.get();
    }
}
```

---

## 🚀 CompletableFuture (Java 8+)

The modern way to handle asynchronous programming. It represents a Future that can be explicitly completed, and allows you to chain multiple async operations non-blocking (like Promises in JavaScript).

```java
// Run async without blocking main thread
CompletableFuture.supplyAsync(() -> {
    System.out.println("Fetching user details...");
    return "Aman"; // Returns string
})
.thenApply(name -> {
    System.out.println("Modifying user...");
    return name.toUpperCase(); // Transforms result
})
.thenAccept(finalName -> {
    System.out.println("Final Output: " + finalName); // Consumes result
});

// Wait slightly to let async threads finish before JVM exits (for demo purposes)
Thread.sleep(1000); 
```

---

## ❓ Interview Questions & Answers

**Q1. What is the difference between `Runnable` and `Callable`?**
> `Runnable` has a `void run()` method that cannot return a result and cannot throw checked exceptions. `Callable<T>` has a `T call()` method that returns a value of type `T` and can throw checked exceptions.

**Q2. What is the difference between `submit()` and `execute()` in an ExecutorService?**
> `execute(Runnable)` takes a `Runnable` and returns `void`. You don't get a handle to the task's result or status. `submit()` takes a `Runnable` or `Callable` and returns a `Future` object, allowing you to check if the task is done, cancel it, or get its return value (using `get()`). Also, exceptions thrown inside `execute()` will crash the thread and print to console, while exceptions in `submit()` are swallowed until you call `Future.get()`.

**Q3. What is the difference between `CountDownLatch` and `CyclicBarrier`?**
> `CountDownLatch` maintains a count. Threads call `countDown()` to decrement it, and other threads call `await()` to block until the count reaches zero. Once it hits zero, it cannot be reset. `CyclicBarrier` is used when multiple threads must all wait for *each other* to reach a common barrier point. Once the specified number of threads arrive, the barrier trips and all proceed. It can be reused (cyclic).

**Q4. How do Atomic variables (like `AtomicInteger`) achieve thread-safety without `synchronized`?**
> They use an algorithm called **Compare-And-Swap (CAS)**. It's a low-level CPU instruction. It essentially says: "Update this variable to the new value, BUT ONLY IF the current value matches the expected old value." If another thread changed the value in the meantime, CAS fails, and the Atomic class automatically retries the operation in a rapid loop until it succeeds. This lock-free approach avoids thread suspension overhead.

**Q5. Why is `CompletableFuture` better than a standard `Future`?**
> Standard `Future` is very limited. To get the result, you must call `.get()`, which BLOCKS the current thread. You cannot chain operations. `CompletableFuture` allows you to register callbacks (`thenApply`, `thenAccept`) that execute automatically when the result is ready, without blocking. It also allows combining multiple futures (`allOf`, `anyOf`) easily.

---

## ✅ Best Practices

1. **Always shutdown ExecutorServices.** If you don't, the non-daemon worker threads will keep the JVM alive forever, causing a memory leak. Use a `finally` block or application shutdown hooks.
2. **Never swallow `InterruptedException`.** If you catch it, either rethrow it or at least restore the interrupt status via `Thread.currentThread().interrupt();` so higher-level code knows the thread was interrupted.
3. **Prefer `CompletableFuture` over `Future`** for complex asynchronous workflows to avoid blocking threads.
4. **Prefer `AtomicInteger` over `synchronized`** for simple counters or state flags; it's much faster.

---

## 🛠️ Hands-on Practice

1. Create a `FixedThreadPool` of 2. Submit 5 `Callable` tasks that return random integers. Store the returned `Future`s in a List. Loop through the list and print the results using `.get()`.
2. Write a program using `CountDownLatch(3)`. Start 3 "Worker" threads that sleep for random times, then call `countDown()`. The Main thread should call `await()` and print "All workers finished" only when all 3 are done.
3. Replace a `synchronized` counter class with one using `AtomicLong` and verify it produces accurate results under concurrent load.
4. Use `CompletableFuture.supplyAsync` to fetch a "UserId", chain `thenApplyAsync` to fetch their "Email" based on the ID, and `thenAccept` to print the email.
