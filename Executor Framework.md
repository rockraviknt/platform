# Executor Framework & ExecutorService

The **`ExecutorService`** is a built-in framework in Java (located in the `java.util.concurrent` package) that simplifies running tasks asynchronously.

Instead of manually creating, starting, and cleaning up a `new Thread()` every time you need to execute code in the background, you submit tasks to an `ExecutorService`, which manages a pool of reusable worker threads for you.

---

## 🧠 Why Do We Need It? (The Problem with `new Thread()`)

Manually creating threads introduces major performance flaws:

1. **High Overhead**: Creating and destroying a Java thread is an expensive operating system operation.
2. **Resource Exhaustion**: If a web server spawns a new thread for every incoming request, a sudden spike in traffic can crash the server due to out-of-memory errors.
3. **No Central Control**: You cannot easily manage execution limits, queue configurations, or track when background tasks finish.

The `ExecutorService` acts as a buffer layer. It decouples task **submission** (what you want to do) from task **execution** (how and when a thread runs it).

---

## ⚙️ How It Works — Architecture

An `ExecutorService` relies on a structural pipeline containing three components:

1. **Task Submission**: You send your tasks (packaged as `Runnable` or `Callable`) to the service.
2. **Blocking Queue**: If all threads in the pool are currently busy, new incoming tasks wait in an internal First-In-First-Out (FIFO) queue.
3. **Thread Pool**: A set of pre-allocated worker threads that constantly pull tasks out of the queue, execute them, and return to the pool to wait for the next task.

---

## 🛠️ The 4 Common Types of Thread Pools

You can instantiate an `ExecutorService` using the factory methods provided by the `Executors` utility class:

| Factory Method | Description |
|---|---|
| `Executors.newFixedThreadPool(int n)` | Allocates a fixed number of threads. Extra tasks wait in the queue. Ideal for guarding resources and limiting system load. |
| `Executors.newCachedThreadPool()` | Spawns new threads as needed; reuses idle ones. Idle threads are destroyed after 60 seconds. Ideal for high volumes of short-lived tasks. |
| `Executors.newSingleThreadExecutor()` | Uses exactly one background thread. Tasks execute strictly one after another. Ideal for ordered event logs or background queues where ordering matters. |
| `Executors.newScheduledThreadPool(int n)` | Schedules tasks to run after a delay or repeatedly at fixed intervals. Replaces the older `java.util.Timer`. |

---

## 💻 Practical Code Example

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class ExecutorServiceExample {
    public static void main(String[] args) {
        // 1. Create a pool with 3 reusable threads
        ExecutorService executor = Executors.newFixedThreadPool(3);

        // 2. Submit a fire-and-forget task (Runnable)
        executor.execute(() -> {
            System.out.println("Task 1 running in: " + Thread.currentThread().getName());
        });

        // 3. Submit a task that returns a value (Callable)
        Future<String> futureResult = executor.submit(() -> {
            Thread.sleep(1000); // Simulate network latency
            return "Task 2 Data Fetched!";
        });

        try {
            // .get() blocks the main thread until Task 2 finishes and returns data
            String result = futureResult.get();
            System.out.println("Result received: " + result);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // 4. CRUCIAL STEP: Always shut down your executor to stop the JVM threads cleanly
            executor.shutdown();
        }
    }
}
```

---

## ⚖️ Submitting Tasks: `Runnable` vs `Callable`

When pushing code to an `ExecutorService`, you can use two formats:

| Property | `Runnable` | `Callable<V>` |
|---|---|---|
| **Execution Method** | `execute(Runnable)` or `submit(Runnable)` | `submit(Callable)` |
| **Return Value** | `void` (fire-and-forget) | Returns a value of type `V` |
| **Exception Handling** | Cannot throw checked exceptions | Can natively throw checked exceptions |
| **Tracking Result** | Returns a `Future<?>` that resolves to `null` | Returns a `Future<V>` containing the computed value |

---

## ⚠️ The Golden Rule: Always Shutdown

An `ExecutorService` creates **non-daemon threads** by default. If you don't explicitly call `executor.shutdown()`, your Java application **will never stop running** because those idle pool threads stay alive in memory forever.

| Method | Behavior |
|---|---|
| `shutdown()` | Allows currently executing and queued tasks to finish before stopping. |
| `shutdownNow()` | Attempts to aggressively stop all active tasks instantly and discards queued tasks. |

---

## ⏱️ How to Stop a Running Thread After 10 Minutes

You cannot safely kill a running thread forcefully in Java. The correct approach is cooperative shutdown: send an interruption signal and let the worker exit cleanly.

### Option 1: `Future#get()` Timeout (Recommended)

Submit the task to an `ExecutorService`, then call `future.get(10, TimeUnit.MINUTES)`.

- If the task completes in time, continue normally.
- If it exceeds the limit, `TimeoutException` is thrown.
- Catch the timeout and call `future.cancel(true)` to interrupt the running task.

### Option 2: `ScheduledExecutorService` (Fire-and-Forget)

Start the worker task, then schedule a cancellation command exactly 10 minutes later:

- `scheduler.schedule(() -> future.cancel(true), 10, TimeUnit.MINUTES)`

This pattern is useful when the caller should not block waiting on `future.get(...)`.

### ⚠️ The Golden Rule

Cancellation works only if your worker code is interruption-aware. Your task should either:

- Periodically check `Thread.currentThread().isInterrupted()`, or
- Use interruptible operations that throw `InterruptedException` (such as `Thread.sleep(...)`, blocking queue calls, and many I/O waits).

```java
while (!Thread.currentThread().isInterrupted()) {
    // Perform small chunks of work
}
```



