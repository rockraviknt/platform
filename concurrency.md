https://www.youtube.com/watch?v=_n7GOopfvWA&list=PLhfHPmPYPPRk6yMrcbfafFGSbE2EPK_A6&index=23

# The `volatile` Keyword in Java

## Overview

Use the `volatile` keyword when a variable is shared across multiple threads and you need to guarantee its visibility without the heavy performance overhead of full synchronization. Marking a variable as `volatile` forces the JVM and CPU to read and write its value directly from main memory rather than caching it in CPU registers or thread-local caches.

---

## 💡 Ideal Scenarios to Use `volatile`

### Status Flags / Cancellation Signals
The most common use case is a `boolean` flag used to gracefully stop or pause a background thread.

### One Writer, Multiple Readers
Use it when exactly one thread modifies the variable and all other threads only read it.

### Double-Checked Locking (Singleton Pattern)
Use it on the `instance` variable in a lazy-initialization Singleton pattern. It prevents the compiler and CPU from reordering instructions, ensuring that threads do not access a partially initialized object.

---

## ⚠️ When `volatile` is NOT Enough

- You **cannot** use `volatile` if the new value of a variable depends on its previous value (such as a counter).
- The `volatile` keyword only guarantees **visibility**, not **atomicity**.

---

# Guaranteeing Atomicity in Java

To guarantee atomicity in Java, you must ensure that an operation or block of code executes as a **single, indivisible unit** that cannot be interrupted by other threads. Because the `volatile` keyword only guarantees visibility, you must look to alternative mechanisms for atomicity.

---

## 1. Use Atomic Classes

For single variables, the easiest and most performant solution is to use the lock-free atomic utility classes in the `java.util.concurrent.atomic` package. These classes rely on low-level CPU hardware instructions called **Compare-And-Swap (CAS)** to update values without locking the thread.

### Common Atomic Types

- **`AtomicInteger` / `AtomicLong`**: Ideal for counters, sequences, and basic math operations.
- **`AtomicBoolean`**: Ideal for state flags requiring atomic check-and-act operations.
- **`AtomicReference`**: Ideal for atomically updating complex object states. used for building new caching in background

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet(); // ++i Atomically increments by 1
        //compareAndSet(int expect, int update)	Sets to update only if current value == expect; returns true/false
        //getAndDecrement()	i-- — returns old value, then decrements
        //decrementAndGet() --i — decrements, returns new value
        //getAndIncrement()	i++ — returns old value, then increments
    }

    public int getCount() {
        return count.get();
    }
}
```

---

## 2. Use Synchronized Blocks or Methods

When your atomic operation spans **multiple variables** or requires complex business logic, lock-free operations are not enough. Use the `synchronized` keyword to build an explicit **mutual exclusion lock (mutex)** around the code block.

- Only **one thread** can execute the synchronized block at any given time.
- All other threads attempting to enter must **wait** until the lock holder exits.

```java
public class SharedMetrics {
    private int requests = 0;
    private int errors = 0;

    // Mutex lock ensures both variables are updated together as one atomic unit
    public synchronized void logError() {
        requests++;
        errors++;
    }
}
```

---

## 3. Use Explicit Lock Classes

For advanced thread coordination, use the explicit locking mechanisms in the `java.util.concurrent.locks` package. They offer higher flexibility than standard `synchronized` blocks, including timed lock polling, interruptible locks, and fairness settings.

### Lock Types

- **`ReentrantLock`**: A traditional mutual exclusion lock requiring an explicit `lock()` and `unlock()`.
- **`ReentrantReadWriteLock`**: Optimises throughput by allowing multiple concurrent reader threads, but granting exclusive access to a single writer thread.

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class InventoryManager {
    private int stockCount = 100;
    private final Lock lock = new ReentrantLock();

    public void purchase(int quantity) {
        lock.lock(); // Explicitly acquire the lock
        try {
            if (stockCount >= quantity) {
                stockCount -= quantity; // Atomic check-and-act block
            }
        } finally {
            lock.unlock(); // Always release inside a finally block to prevent deadlocks
        }
    }
}
```

---

## 📝 Decision Framework

Use this quick guideline to pick the right strategy for your use case:

| Scenario | Strategy |
|---|---|
| Single primitive or reference variable | **Atomic Classes** — fastest, lock-free |
| Simple operations over a few methods | **`synchronized` keyword** — clean, built-in |
| Need `tryLock()`, timeouts, or advanced queues | **`ReentrantLock`** — maximum flexibility |

---

# ThreadLocal: Thread-Scoped Data Isolation

## Overview

The role of the **`ThreadLocal`** class in Java is to provide **thread-local variables**. It enables you to create data that is **isolated to a specific thread**, meaning each thread that reads or writes to the variable has its own independently initialized copy.

By ensuring data is confined to a single thread, `ThreadLocal` achieves **thread safety without synchronization**, locks, or performance overhead.

---

## 🏛️ Common Use Cases

### User Session and Security Contexts
Enterprise frameworks like Spring use `ThreadLocal` under the hood (e.g., `SecurityContextHolder`) to store the logged-in user's details, permissions, or OAuth tokens for the duration of an HTTP request thread.

### Database Transaction Management
Spring and Hibernate use it to bind a specific database `Connection` to the current executing thread. This ensures that every SQL query executed during a single transaction uses the exact same database connection.

### Thread-Unsafe Utility Isolation
Objects like `SimpleDateFormat` are notoriously thread-unsafe. Instead of creating a new instance on every method call or using heavy synchronization, a single instance is cached per thread using `ThreadLocal`.

### Contextual Logging (MDC)
Logging frameworks (such as Logback or Log4j) use Mapped Diagnostic Context (MDC), which relies on `ThreadLocal` to stamp log statements with a unique `transactionId` or `correlationId` tracking a request's path across systems.

---

## 🛠️ Basic Code Example

Here is how you can use `ThreadLocal` to isolate an instance of an object per thread:

```java
import java.text.SimpleDateFormat;
import java.util.Date;

public class DateFormatterProvider {

    // ThreadLocal allocates one SimpleDateFormat instance per thread
    private static final ThreadLocal<SimpleDateFormat> formatterThreadLocal = 
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static String format(Date date) {
        // Retrieves the isolated formatter instance belonging to the calling thread
        return formatterThreadLocal.get().format(date);
    }

    public static void clear() {
        // Crucial step: Prevent memory leaks
        formatterThreadLocal.remove(); 
    }
}
```

---

# Java Concurrency vs Parallelism

In Java, the distinction between concurrency and parallelism comes down to **how threads are managed by your application** versus **how those threads are executed by CPU hardware**.

- **Java Concurrency** is an **architectural style**. It is about writing a program that manages multiple independent threads of execution, such as a web server handling many requests.
- **Java Parallelism** is an **execution style**. It is about taking a single heavy task, splitting it into smaller pieces, and running them at the same time across **multiple CPU cores** to finish faster.
tasks are mostly independent, so each core works without waiting.
Real case: many parallel algorithms still synchronize at some points.
Example: Fork/Join splits work in parallel, then joins partial results at the end.
So parallelism is not “no coordination.” It is “maximize simultaneous work, minimize coordination cost.”

---

## Quick Comparison Matrix

| Metric | Java Concurrency | Java Parallelism |
|---|---|---|
| Primary Tool | `Thread`, `VirtualThread`, `Runnable` | `ParallelStream`, `ForkJoinPool` |
| Problem Type | **I/O-bound** (waiting on DB/network) | **CPU-bound** (compute-heavy work) |
| Core Goal | **Responsiveness and resource efficiency** | **Speed and throughput** |
| Hardware Need | Works on a **single core** | Requires **multiple cores** |

---

## 1. Java Concurrency: Dealing with Multiple Tasks

Java concurrency focuses on thread scheduling, state preservation, and synchronization. It allows tasks to start, run, and overlap in time.

Even with one CPU core, Java can achieve concurrency using **time-slicing**, where the OS scheduler rapidly switches between runnable threads.(interleaving)

### Typical Code Pattern: `CompletableFuture`

Concurrency is useful when you want background work without blocking the main application thread.

```java
import java.util.concurrent.CompletableFuture;

public class ConcurrentFetchDemo {
    public static void main(String[] args) {
        CompletableFuture<String> userDetails =
            CompletableFuture.supplyAsync(() -> fetchUserFromDatabase());

        CompletableFuture<String> userOrders =
            CompletableFuture.supplyAsync(() -> fetchOrdersFromApi());

        // Wait for both tasks, which can overlap in time
        CompletableFuture.allOf(userDetails, userOrders).join();

        System.out.println(userDetails.join());
        System.out.println(userOrders.join());
    }

    private static String fetchUserFromDatabase() {
        return "user-details";
    }

    private static String fetchOrdersFromApi() {
        return "user-orders";
    }
}
```

### Modern Java Concurrency: Virtual Threads (Java 21+)

Java 21 introduced production-ready **virtual threads**. They are lightweight threads managed by the JVM, enabling very high concurrency for I/O-heavy workloads without the cost of one OS thread per task.

---

## 2. Java Parallelism: Doing Tasks Simultaneously

Java parallelism assumes multiple CPU cores and uses them to solve compute-heavy tasks faster. It often relies on the **Fork/Join framework**, which splits work into sub-tasks (fork) and combines results (join).

### Typical Code Pattern: Parallel Streams

Parallelism is easy to apply with streams by switching to `parallelStream()`.

```java
import java.util.Arrays;
import java.util.List;

public class ParallelStreamDemo {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8);

        // Parallel: work is distributed across multiple cores when available
        numbers.parallelStream()
               .map(ParallelStreamDemo::performHeavyCalculation)
               .forEach(System.out::println);
    }

    private static int performHeavyCalculation(int n) {
        return n * n;
    }
}
```

