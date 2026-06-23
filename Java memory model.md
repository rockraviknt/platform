# Java Memory Model (JMM)

The **Java Memory Model (JMM)** is a specification/rules that defines how the Java Virtual Machine (JVM) interacts with physical memory (RAM and CPU caches).

Its primary purpose is to guarantee how and when changes made by one thread become visible to other threads, so multithreaded programs behave predictably across different hardware.
JMM rules must be implemented by all JVM's

---

## The Core Problem the JMM Solves

Modern CPUs do not read directly from RAM for every operation because RAM is slower than CPU caches. Each core uses fast local caches and registers.

Without JMM rules, multithreaded Java code would commonly face:

1. **Visibility issues**: Thread A updates a value, but Thread B still reads an old cached value.
2. **Instruction reordering issues**: Compiler/JVM/CPU reorder instructions for optimization, which can break cross-thread correctness.

---

## Conceptual Structure of the JMM

The JMM is usually explained with two logical memory areas:

- **Main Memory**: Shared memory accessible by all threads. Objects and instance fields conceptually live here.
- **Thread-Local Working Memory (stack/cache view)**: Each thread works with local copies of shared variables during execution.

When a thread reads or writes shared data, JMM rules define when those values must be synchronized with main memory.

---

## Core Guarantees of the JMM

The JMM provides three key guarantees.

### 1. Visibility (`volatile`)

If a variable is declared `volatile`:

- Writes to that variable are made visible to other threads promptly.
- Reads fetch the latest value according to JMM visibility rules.

This helps prevent stale reads for that variable.

### 2. Ordering (Happens-Before)

The JMM defines **happens-before** relationships.

If action A happens-before action B, then:

- All effects of A are visible to B.
- Reordering cannot violate that visibility guarantee.

Important happens-before rules:

- **Program Order Rule**: Within one thread, actions appear to execute in program order.
- **Volatile Rule**: A write to a `volatile` variable happens-before every subsequent read of that same variable.
- **Lock Rule**: An unlock on a monitor (`synchronized`) happens-before every subsequent lock on that same monitor.

### 3. Atomicity (`synchronized` and atomic classes)

The JMM guarantees atomic read/write for most single variable operations (with historical caveats for non-volatile `long`/`double` on older 32-bit contexts).

For compound operations such as `count++`, check-then-act, or read-modify-write sequences, use:

- `synchronized` blocks/methods, or
- classes from `java.util.concurrent.atomic`

to guarantee atomicity and correctness.

---

## Quick Summary

- Use **`volatile`** for visibility and ordering of a single shared variable.
- Use **`synchronized`** or **atomic classes** for compound atomic operations.
- Use **happens-before** rules to reason about cross-thread correctness.
