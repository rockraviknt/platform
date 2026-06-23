# Async Processing in Java

Asynchronous programming in Java allows you to execute non-blocking operations. It lets your application spin off a long-running task (like a database query or an API call) to a background thread, while the calling thread continues executing other work immediately.

---

## Java 8: The `CompletableFuture` Revolution

Java 8 introduced [`CompletableFuture`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html), which modernized the old blocking `Future#get()` style. It brings promise-like composition to Java, so you can chain async transformations, handle failures, and combine multiple async paths using functional APIs.

### Golden Cheat Sheet for Java 8 Async Pipelines

| Method Category | Method Name | Behavior |
|---|---|---|
| **Initiate** | `supplyAsync(Supplier)` | Starts an async task that returns a value. |
| **Transform** | `thenApply(Function)` | Transforms the output value when ready (similar to `map`). |
| **Compose** | `thenCompose(Function)` | Flattens nested `CompletableFuture` chains (similar to `flatMap`). |
| **Consume** | `thenAccept(Consumer)` | Consumes the final result as a terminal operation. |
| **Fallback** | `exceptionally(Function)` | Catches exceptions from upstream stages in the pipeline. |

### Java 8 Production-Ready Blueprint

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Java8AsyncDemo {
	public static void main(String[] args) {
		// Use an explicit pool for I/O work to avoid overloading commonPool.
		ExecutorService ioExecutor = Executors.newFixedThreadPool(10);

		CompletableFuture.supplyAsync(() -> fetchRawOrder(42), ioExecutor)
			.thenApply(raw -> calculateTax(raw))
			.thenAccept(finalPrice -> System.out.println("Order processed: " + finalPrice))
			.exceptionally(ex -> {
				System.err.println("Pipeline failed: " + ex.getMessage());
				return null;
			});

		ioExecutor.shutdown();
	}

	private static double fetchRawOrder(int id) {
		return 250.0;
	}

	private static double calculateTax(double price) {
		return price * 1.2;
	}
}
```

---

## Java 9: Flow API
- `Flow` API: Java 9 also introduced `java.util.concurrent.Flow` (`Publisher`, `Subscriber`, and related types), which forms the core reactive-streams foundation used by libraries such as Reactor.

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class TimeoutExample {
	public static void main(String[] args) {
		CompletableFuture.supplyAsync(() -> fetchUnreliableExternalApi())
			.orTimeout(2, TimeUnit.SECONDS)
			.exceptionally(ex -> "Fallback cached data")
			.thenAccept(System.out::println);
	}

	private static String fetchUnreliableExternalApi() {
		return "Live API response";
	}
}
```

---

## Java 21+: Virtual Threads

Java 21 introduced Virtual Threads (Project Loom), which dramatically reduces async boilerplate for many I/O-heavy use cases.

Virtual threads are lightweight threads managed by the JVM. When a virtual thread hits a blocking call, the runtime can unmount it from an OS carrier thread, let other work run, and remount it when the operation is ready.

The result is simple, synchronous-looking code with high concurrency and strong scalability.

```java
import java.util.concurrent.Executors;

public class VirtualThreadDemo {
	public static void main(String[] args) {
		try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
			executor.submit(() -> {
				String user = fetchUserBlocking();
				String orders = fetchOrdersBlocking();
				return combineData(user, orders);
			});
		}
	}

	private static String fetchUserBlocking() {
		return "user";
	}

	private static String fetchOrdersBlocking() {
		return "orders";
	}

	private static String combineData(String user, String orders) {
		return user + " + " + orders;
	}
}
```

---

## Reactive Programming in Java

Reactive programming is a declarative model for processing asynchronous, non-blocking streams of data with built-in backpressure.

- **Data as streams**: Treat data as events over time (0, 1, or many elements).
- **Non-blocking execution**: Threads are not held idle waiting for slow I/O.
- **Backpressure**: Consumers can signal producers to slow down and avoid overload.
- **Core standards**: Commonly built on Java `Flow` and libraries such as Project Reactor or RxJava.

Key Reactor types:

- `Flux<T>`: Stream of 0 to N elements.
- `Mono<T>`: Stream of 0 or 1 element.

---

## What Is Spring WebFlux?

Spring WebFlux is Spring's non-blocking reactive web stack (introduced in Spring Framework 5) and an alternative to classic Spring MVC for specific workloads.

- **Architecture shift**: MVC commonly uses a thread-per-request model, while WebFlux uses an event-loop-driven model.
- **Runtime model**: Frequently deployed on Netty for asynchronous, event-driven networking.
- **Best fit**: High-concurrency, I/O-bound systems such as API gateways, streaming systems, chat services, and heavily interconnected microservice environments.
