# Locking in Java: `ReentrantLock` and `ReentrantReadWriteLock`

A `ReentrantLock` is an advanced mutual exclusion (mutex) lock in the `java.util.concurrent.locks` package. It is a more flexible and explicit alternative to the `synchronized` keyword.

The term reentrant means the same thread can safely acquire the same lock multiple times without deadlocking itself.

Flexible (can lock in method A, unlock in method B).

---

## How Reentrancy Works

If a thread already holds a `ReentrantLock` and reaches another section protected by that same lock, it can re-enter.

Internally, the lock tracks an acquisition count:

- Each successful `lock()` increments the count.
- Each `unlock()` decrements the count.
- The lock is fully released only when the count returns to `0`.

---

## Non-Blocking Lock Attempts with `tryLock()`

Instead of waiting indefinitely for a lock, use `tryLock()`.

- If the lock is available, `tryLock()` returns `true`.
- If another thread holds the lock, it returns `false` immediately.
- This lets you run fallback logic rather than blocking the thread.

### When to Use `static` for a Lock

- If the data you are protecting belongs to a single object instance, your lock variable should not be static. Examples include individual bank accounts, separate user sessions, or isolated cache instances.
- If the data you are protecting is shared across all instances of a class, your lock must be static. Examples include a global counter, a shared connection pool, or a static configuration map.
- Regardless of whether your lock is static or instance-level, always use `private final`.

```java
import java.util.concurrent.locks.ReentrantLock;

public class TryLockDemo {
	private static final ReentrantLock lock = new ReentrantLock();

	public static void main(String[] args) {
		if (lock.tryLock()) {
			try {
				// Manipulate shared state safely.
				System.out.println("Lock acquired. Processing critical section...");
			} finally {
				lock.unlock();
			}
		} else {
			System.out.println("Lock busy. Performing fallback tasks instead...");
		}
	}
}
```

---

## `ReentrantReadWriteLock`

A `ReentrantReadWriteLock` is another lock utility in `java.util.concurrent.locks` that improves throughput in read-heavy workloads by separating read and write access.

Core rule:

- Many threads can read simultaneously.
- Only one thread can write at a time.

---

## Core Lock Rules

1. Read lock (shared): Multiple readers are allowed if no writer is active.
2. Write lock (exclusive): Only one writer is allowed, and it blocks both readers and other writers.

A `ReentrantReadWriteLock` exposes two lock handles:

- `readLock()` for read-only operations
- `writeLock()` for updates

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ThreadSafeCache {
	private String sharedData = "Initial Value";

	private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
	private final Lock readLock = rwLock.readLock();
	private final Lock writeLock = rwLock.writeLock();

	// Multiple threads can read concurrently.
	public String readData() {
		readLock.lock();
		try {
			return sharedData;
		} finally {
			readLock.unlock();
		}
	}

	// Only one thread can write at a time.
	public void writeData(String newValue) {
		writeLock.lock();
		try {
			sharedData = newValue;
		} finally {
			writeLock.unlock();
		}
	}
}
```

---

## When Should You Use `ReentrantReadWriteLock`?

Do not use it blindly for every shared object. Managing separate read/write locks adds overhead compared to a simple `synchronized` block or a single lock.

Use it when your system has a high read-to-write ratio, such as:

- Caches
- Routing tables
- Configuration maps
- User profile lookup stores

---

## `Semaphore`

A `Semaphore` is a thread synchronization utility in the `java.util.concurrent` package that controls access to a physical or logical resource by maintaining a set of permits.

Unlike a standard lock, which usually allows only one thread at a time, a semaphore lets you define the maximum number of threads that can access a resource concurrently.

### How It Works

- Each call to `acquire()` takes one permit.
- If no permits are available, the thread waits until one is released.
- Each call to `release()` returns a permit back to the semaphore.
- The total number of available permits defines the concurrency limit.

### When to Use It

Use a semaphore when you want to limit concurrent access to a shared resource, such as:

- A fixed-size database connection pool
- A limited number of API requests
- A printing service with a small number of printers
- Any resource that can safely handle only a bounded number of concurrent users

```java
import java.util.concurrent.Semaphore;

public class SemaphoreDemo {
	private static final Semaphore semaphore = new Semaphore(3);

	public static void main(String[] args) {
		Runnable task = () -> {
			try {
				semaphore.acquire();
				System.out.println(Thread.currentThread().getName() + " entered the critical section");
				Thread.sleep(1000);
			} catch (InterruptedException e) {
				Thread.currentThread().interrupt();
			} finally {
				semaphore.release();
			}
		};

		for (int i = 0; i < 6; i++) {
			new Thread(task, "Worker-" + i).start();
		}
	}
}
```


---

## How to Detect and Resolve Deadlocks in Java

### 1. How to Detect

- Manually (production): Find your process ID with `jps -l`, then run `jstack <PID>`. In the output, the JVM will explicitly report deadlocks (for example, "Found one Java-level deadlock").
- Automatically (in code): Use `ThreadMXBean` to scan for deadlocked threads at runtime.

```java
long[] ids = ManagementFactory.getThreadMXBean().findDeadlockedThreads();
```

### 2. How to Resolve and Prevent

- Enforce lock ordering: Ensure every thread acquires shared locks in the same sequence (for example, sort resources by ID before locking).
- Use `tryLock()` with timeout: Prefer `ReentrantLock` with `lock.tryLock(timeout, TimeUnit)` so a thread can back off and retry instead of waiting forever.