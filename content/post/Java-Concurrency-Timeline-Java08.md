---
title: "Java Thread & Concurrency Timeline: Java 8"
date: 2026-11-04T23:30:00Z
lastmod: 2026-11-04T23:30:00Z
draft: false
categories: ["java", "concurrency", "timeline"]
tags: ["java", "concurrency", "timeline", "threads", "completablefuture", "streams", "stampedlock", "java8"]
series: ["java concurrency timeline"]
author: "xmoooz"
description: "Java 8 concurrency: lambdas, parallel streams, CompletableFuture, StampedLock, LongAdder, and ConcurrentHashMap improvements — what to still use, what to never use, and practical tips."
url: 2026/11/04/java-concurrency-timeline-java8
---

## 0. Global Vision — Java 8 in 2 Minutes

This file covers only Java 8. Nothing before, nothing after.

The story inside Java 8 is simple:

**Java 5 gave you tasks and locks. Java 8 gave you a way to describe what should happen when tasks finish, without blocking a thread to wait for it.**

| Problem in Java 5-7 | What Java 8 gives you |
| :--- | :--- |
| `Future.get()` blocks, no callback, no chaining | **`CompletableFuture`** — chain, combine, handle errors, run async |
| `ForkJoinPool` is powerful but verbose for simple parallel work | **Parallel streams** — `list.parallelStream()` for data-parallel work |
| `ReentrantReadWriteLock` is correct but slow under write contention | **`StampedLock`** — optimistic read, faster for read-heavy work |
| `AtomicLong` spins and contends under many writers | **`LongAdder` / `LongAccumulator`** — striped counters, no contention |
| `ConcurrentHashMap` in Java 5 is good but has no atomic bulk operations | **`ConcurrentHashMap` in Java 8** — `compute`, `merge`, `computeIfAbsent`, parallel bulk operations |
| Anonymous `Runnable` / `Callable` everywhere | **Lambdas + method references** — less noise, same idea |

### The mental shift in Java 8

| In Java 5-7 | In Java 8 |
| :--- | :--- |
| You block to get a result | You describe what happens when the result arrives |
| You write a loop to process data | You describe a pipeline: filter, map, reduce |
| You share one counter and contend on it | You stripe the counter, merge at the end |
| You lock for every read | You try optimistic read first, lock only if it failed |

### The one sentence to remember for each part

- **Lambdas:** Less noise for `Runnable` / `Callable`. Same thread model, cleaner code.
- **Parallel streams:** Easy data parallelism, but you don't control the pool or the splitting.
- **`CompletableFuture`:** `Future` you can compose. Chain, combine, handle errors, without blocking.
- **`StampedLock`:** Optimistic read for read-heavy data. Fast, but easy to misuse.
- **`LongAdder`:** Counter that scales. Use it when many threads increment the same counter.
- **`ConcurrentHashMap` (Java 8):** Atomic `compute` / `merge` / `computeIfAbsent`. Stop doing check-then-act by hand.

### What to take away from Java 8

- Don't block if you can chain. `thenApply`, `thenCompose`, `thenCombine` are the core of Java 8 concurrency.
- Don't use parallel streams for everything. They are for CPU-bound, divisible, stateless work on large data.
- Don't replace every lock with `StampedLock`. It is faster only when reads dominate and writes are short.

---

## 1. Lambdas + Method References — Less Noise, Same Thread Model

### What
Lambda expressions, method references, functional interfaces (`Runnable`, `Callable`, `Supplier`, `Consumer`, `Function`).

### Why
Before Java 8, every task was 5 lines of anonymous class noise. Lambdas did not change the thread model — they changed how much code you need to express a task.

### Benefit
- `new Thread(() -> ...)` instead of an anonymous class.
- `executor.submit(() -> ...)` reads like the task itself.
- Method references make existing methods usable as tasks: `executor.submit(this::loadData)`.

### Downside
- Lambdas hide the type. A long lambda inside `submit()` is hard to read and hard to test.
- Checked exceptions still need handling — a lambda that throws a checked exception does not fit `Runnable`.

### Status in Java 8
| API | Verdict |
| :--- | :--- |
| Lambdas, method references | **CORE.** Use them for tasks. |
| Long lambdas inside `submit()` / `supplyAsync()` | **AVOID.** Extract to a named method. |

### Real Sample Code

```java
import java.util.concurrent.*;

public class LambdaDemo {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        try {
            // Before: new Runnable() { public void run() { ... } }
            // Java 8:
            executor.execute(() -> System.out.println("Task on " + Thread.currentThread().getName()));
            Future<Integer> f = executor.submit(() -> 10 + 20);
            System.out.println(f.get()); // 30
        } catch (Exception e) {
            Thread.currentThread().interrupt();
        } finally {
            executor.shutdown();
        }
    }
}
```

**Best Practice:**
- Keep task lambdas short. If it needs more than a few lines, extract a method and use a method reference.
- Don't capture mutable shared state in a lambda and mutate it from many threads. That is the same race as before, with less visible code.
- Don't use a lambda to hide a blocking call inside a parallel stream. The blocking is still there.

---

## 2. Parallel Streams — Easy Data Parallelism, With Traps

### What
`Collection.parallelStream()`, `Stream.parallel()`, `ForkJoinPool.commonPool()`.

### Why
Splitting a list across cores by hand with `ForkJoinPool` is verbose. Parallel streams give you the same idea in one method call: split, process in parallel, merge.

### Benefit
- One line to parallelize a data pipeline.
- Uses the common `ForkJoinPool` by default, no pool to manage.
- Good for CPU-bound, stateless, divisible work on large collections.

### Downside
- You don't control the pool. Parallel streams share the common pool.
- Blocking I/O inside a parallel stream blocks common-pool workers and starves other parallel streams.
- Stateful lambdas, shared mutable state, and order-dependent code break silently.
- Small data sets are slower in parallel — splitting costs more than it saves.

### Status in Java 8
| API | Verdict |
| :--- | :--- |
| `parallelStream()` for CPU-bound, large, stateless work | **CORE when it fits.** |
| `parallelStream()` for I/O, small data, shared mutable state | **DO NOT USE.** |

### Real Sample Code

```java
import java.util.*;
import java.util.stream.*;

public class ParallelStreamDemo {
    public static void main(String[] args) {
        List<Integer> numbers = IntStream.range(0, 1_000_000).boxed().toList();

        // Sequential: one thread
        long seq = numbers.stream().filter(n -> n % 2 == 0).count();

        // Parallel: split across cores, merge at the end
        // GOOD: stateless, CPU-bound, large data, no shared mutable state
        long par = numbers.parallelStream().filter(n -> n % 2 == 0).count();
        System.out.println(seq + " / " + par);
    }
}
```

```java
// BAD: shared mutable list + parallel stream = race, lost updates, wrong order
List<String> out = new ArrayList<>();
numbers.parallelStream().map(n -> "v" + n).forEach(out::add); // DON'T

// GOOD: let the stream collect, it handles merging
List<String> good = numbers.parallelStream()
    .map(n -> "v" + n)
    .toList(); // or .collect(Collectors.toList())
```

**Best Practice:**
- Use parallel streams only when: data is large, work is CPU-bound, lambda is stateless, and you don't need order.
- Never do I/O, blocking, or `sleep()` inside a parallel stream.
- Never mutate shared state inside `forEach` in a parallel stream. Use `collect`, `reduce`, or `map` + `toList`.
- Don't use parallel streams for a handful of items. Measure — sequential is often faster below a few thousand items.
- If you need a custom pool, don't force it through parallel streams in Java 8. Use an `ExecutorService` / `CompletableFuture` with your own pool instead.

---

## 3. CompletableFuture — The Core of Java 8 Concurrency

### What
`CompletableFuture`, `supplyAsync`, `runAsync`, `thenApply`, `thenCompose`, `thenCombine`, `allOf`, `anyOf`, `exceptionally`, `handle`, `whenComplete`.

### Why
Java 5 `Future` has no callback. You must block with `get()` or poll with `isDone()`. `CompletableFuture` lets you say: "when this finishes, do that — on another thread, without blocking the caller."

### Benefit
- Chain: `supplyAsync(...).thenApply(...).thenAccept(...)`.
- Combine: two futures into one with `thenCombine` / `allOf` / `anyOf`.
- Errors: `exceptionally` / `handle` instead of unwrapping `ExecutionException` everywhere.
- No blocking: the caller thread keeps working while the chain runs.

### Downside
- Default pool is the common `ForkJoinPool`. Blocking work inside a chain starves it, same as parallel streams.
- Chaining without thinking creates long, hard-to-debug chains. Exceptions can get swallowed if you forget `exceptionally` / `handle`.
- `get()` still blocks — calling it immediately after `supplyAsync()` throws away the benefit.

### Status in Java 8
| API | Verdict |
| :--- | :--- |
| `CompletableFuture`, `supplyAsync`, `thenApply`, `thenCompose`, `thenCombine`, `allOf`, `anyOf`, `exceptionally`, `handle` | **CORE. This is the Java 8 way to write async code.** |
| `Future.get()` immediately after `submit()` / `supplyAsync()` | **DO NOT USE as a pattern.** You just serialized async work. |

### Real Sample Code — Basic Chain

```java
import java.util.concurrent.*;

public class CompletableFutureDemo {
    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture
            .supplyAsync(() -> loadUser(42))        // runs on common pool
            .thenApply(user -> "Hello, " + user)    // runs when loadUser finishes
            .thenApply(String::toUpperCase)
            .exceptionally(ex -> "Fallback user");  // runs if anything above throws

        // Do other work here, don't block immediately
        System.out.println("Doing other work...");
        System.out.println(future.join()); // blocks here, at the end
    }
    static String loadUser(int id) {
        try { Thread.sleep(300); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return "user-" + id;
    }
}
```

### Real Sample Code — Combine Two Futures

```java
CompletableFuture<Integer> price = CompletableFuture.supplyAsync(() -> 100);
CompletableFuture<Integer> tax = CompletableFuture.supplyAsync(() -> 20);

// Both run in parallel. thenCombine runs when BOTH finish.
CompletableFuture<Integer> total = price.thenCombine(tax, Integer::sum);
System.out.println(total.join()); // 120
```

### Real Sample Code — Custom Pool (Important)

```java
import java.util.concurrent.*;

// BAD in Java 8 for blocking work: default common pool
// CompletableFuture.supplyAsync(() -> blockingHttpCall());

// GOOD: pass your own pool for blocking / I/O work
ExecutorService ioPool = Executors.newFixedThreadPool(10);
try {
    CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> blockingHttpCall(), ioPool);
    System.out.println(f.join());
} finally {
    ioPool.shutdown();
}
static String blockingHttpCall() { return "ok"; }
```

### `thenApply` vs `thenCompose` vs `thenCombine`

| Method | What it does | Use when |
| :--- | :--- | :--- |
| `thenApply(fn)` | Transform result, same chain | `User` -> `String`, no new async work |
| `thenCompose(fn)` | Return a *new* `CompletableFuture`, flatten it | First call returns an id, second call needs that id |
| `thenCombine(other, fn)` | Combine two independent futures | Price + tax, user + orders |

```java
// thenCompose: second async call depends on first result
CompletableFuture<String> profile = CompletableFuture
    .supplyAsync(() -> 42)
    .thenCompose(id -> CompletableFuture.supplyAsync(() -> loadUser(id)));

// thenApply would give CompletableFuture<CompletableFuture<String>> — wrong shape
```

### Errors — `exceptionally` vs `handle` vs `whenComplete`

| Method | What it does |
| :--- | :--- |
| `exceptionally(ex -> fallback)` | On error, replace result with fallback. Chain continues as success. |
| `handle((result, ex) -> ...)` | On success or error, produce a new result. You see both. |
| `whenComplete((result, ex) -> ...)` | Side effect only (log). Does not change the result. |

```java
CompletableFuture<String> f = CompletableFuture
    .supplyAsync(() -> { if (true) throw new RuntimeException("boom"); return "ok"; })
    .handle((result, ex) -> ex != null ? "fallback: " + ex.getMessage() : result);
System.out.println(f.join());
```

**Best Practice:**
- Use `thenCompose` when the next step is async. Use `thenApply` when it is a plain transform.
- Always handle errors at the end of a chain. An unhandled exception in a `CompletableFuture` chain fails silently until someone calls `join()` / `get()`.
- Use `join()` in application code, `get()` when you need checked exceptions. `join()` wraps errors in `CompletionException` (unchecked).
- Use a custom pool for blocking / I/O work. Never block the common pool.
- Use `allOf` when you need to wait for many futures, then collect results yourself. `allOf` returns `Void` — it does not give you the list of results.
- Use `anyOf` for fastest-wins / timeout-style logic.

```java
// allOf pattern: start all, then collect
List<CompletableFuture<Integer>> futures = List.of(
    CompletableFuture.supplyAsync(() -> 1),
    CompletableFuture.supplyAsync(() -> 2),
    CompletableFuture.supplyAsync(() -> 3));
CompletableFuture<Void> all = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
all.join();
List<Integer> results = futures.stream().map(CompletableFuture::join).toList();
```

---

## 4. StampedLock — Optimistic Read for Read-Heavy Data

### What
`StampedLock`, `tryOptimisticRead()`, `validate(stamp)`, `readLock()`, `writeLock()`.

### Why
`ReentrantReadWriteLock` is correct, but every read still takes a lock and contends. For read-heavy data where writes are rare and short, an optimistic read — "read without locking, then check if anyone wrote" — is much faster.

### Benefit
- Optimistic read has no lock, no contention, very fast when writes are rare.
- Falls back to a real read lock if validation fails.
- Write lock is exclusive, like before.

### Downside
- Easy to misuse. If you forget `validate()`, you can read torn / inconsistent data.
- Not reentrant. Same thread cannot re-enter a `StampedLock` it already holds — it deadlocks.
- `StampedLock` is not fair by default and does not support `Condition` in the same way.

### Status in Java 8
| API | Verdict |
| :--- | :--- |
| `StampedLock` for read-heavy, short writes, no reentrancy needed | **CORE when it fits.** |
| `StampedLock` for general locking, reentrant code, `Condition` | **DO NOT USE.** Use `ReentrantLock` / `ReentrantReadWriteLock`. |

### Real Sample Code

```java
import java.util.concurrent.locks.StampedLock;

public class StampedLockDemo {
    private final StampedLock lock = new StampedLock();
    private double x, y;

    // Optimistic read: try without lock, validate, fallback if needed
    double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead(); // no lock taken
        double curX = x, curY = y;
        if (!lock.validate(stamp)) { // someone wrote while we read
            stamp = lock.readLock(); // fallback: real read lock
            try {
                curX = x; curY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.hypot(curX, curY);
    }

    void move(double dx, double dy) {
        long stamp = lock.writeLock();
        try { x += dx; y += dy; } finally { lock.unlockWrite(stamp); }
    }
}
```

**Best Practice:**
- Always `validate()` after an optimistic read. No validation = no guarantee.
- Keep the optimistic section short and copy fields to locals first, then validate.
- Never use `StampedLock` in reentrant code. If a method that holds the lock calls another method that takes the same lock, it deadlocks.
- Prefer `ReentrantReadWriteLock` when code is complex or reentrancy matters. Use `StampedLock` only when reads clearly dominate and you measured the win.

---

## 5. LongAdder + LongAccumulator — Counters That Scale

### What
`LongAdder`, `LongAccumulator`, `DoubleAdder`, `DoubleAccumulator`.

### Why
`AtomicLong` uses one CAS slot. With 100 threads hammering `incrementAndGet()`, they all contend on that one slot and spin. `LongAdder` stripes the counter — each thread updates its own cell, `sum()` merges at the end.

### Benefit
- Much faster than `AtomicLong` under high contention.
- Same idea as sharding, built into the JDK.

### Downside
- `sum()` is not atomic with updates. It is a snapshot — another thread can increment while you sum.
- More memory than `AtomicLong` (cells).
- No CAS on the total. If you need `compareAndSet` on the total, `LongAdder` is the wrong tool.

### Status in Java 8
| API | Verdict |
| :--- | :--- |
| `LongAdder` for high-contention counters (metrics, request counts) | **CORE.** |
| `AtomicLong` for low contention or when you need CAS on the total | **STILL CORE.** |

### Real Sample Code

```java
import java.util.concurrent.atomic.*;

public class LongAdderDemo {
    // BAD under high contention: one CAS slot
    static AtomicLong atomicCount = new AtomicLong();

    // GOOD under high contention: striped cells
    static LongAdder adderCount = new LongAdder();

    public static void main(String[] args) throws Exception {
        Runnable task = () -> { for (int i = 0; i < 100_000; i++) adderCount.increment(); };
        Thread t1 = new Thread(task), t2 = new Thread(task);
        t1.start(); t2.start(); t1.join(); t2.join();
        System.out.println(adderCount.sum()); // 200000
    }
}

// LongAccumulator: custom merge function, e.g. max
LongAccumulator max = new LongAccumulator(Long::max, 0);
max.accumulate(10);
max.accumulate(25);
System.out.println(max.get()); // 25
```

**Best Practice:**
- Use `LongAdder` for counters, metrics, and stats that many threads update.
- Use `AtomicLong` when you need `compareAndSet`, or when contention is low and you need an exact, atomic read.
- Don't use `LongAdder.sum()` for correctness-critical decisions that need an atomic snapshot. It is for monitoring / counting, not for invariants.

---

## 6. ConcurrentHashMap in Java 8 — Atomic Bulk Operations

### What
Java 8 `ConcurrentHashMap`: `compute`, `computeIfAbsent`, `computeIfPresent`, `merge`, `forEach`, `search`, `reduce`, plus internal striped locking instead of Java 5 segments.

### Why
In Java 5, `putIfAbsent` helped, but `if (!map.containsKey(k)) map.put(k, compute(k))` was still a race, and `compute` had to be hand-written with locks. Java 8 makes those operations atomic inside the map.

### Benefit
- `computeIfAbsent` is atomic: the mapping function runs at most once per key under contention (in practice, don't rely on exactly-once for side effects — keep it pure).
- `merge` makes counting / grouping atomic without external locks.
- Parallel bulk operations (`forEach`, `search`, `reduce`) split work across cores.

### Downside
- Mapping functions for `compute` / `computeIfAbsent` must be short, pure, and non-blocking. Blocking or recursive map access inside them can deadlock.
- `computeIfAbsent` with a slow mapping function blocks other threads on the same bin.

### Status in Java 8
| API | Verdict |
| :--- | :--- |
| `compute`, `computeIfAbsent`, `merge`, `putIfAbsent`, `replace` | **CORE. Use these instead of check-then-act.** |
| Check-then-act by hand (`containsKey` + `put`) | **DO NOT USE.** It is still a race. |

### Real Sample Code

```java
import java.util.concurrent.*;

public class ConcurrentMapDemo {
    public static void main(String[] args) {
        ConcurrentHashMap<String, LongAdder> counts = new ConcurrentHashMap<>();

        // GOOD: atomic merge — no external lock, no race
        Runnable task = () -> {
            for (int i = 0; i < 1000; i++) {
                counts.computeIfAbsent("hits", k -> new LongAdder()).increment();
            }
        };
        Thread t1 = new Thread(task), t2 = new Thread(task);
        try { t1.start(); t2.start(); t1.join(); t2.join(); } catch (Exception e) { Thread.currentThread().interrupt(); }
        System.out.println(counts.get("hits").sum()); // 2000

        // merge: counting words, atomic
        ConcurrentHashMap<String, Integer> wordCounts = new ConcurrentHashMap<>();
        wordCounts.merge("java", 1, Integer::sum);
        wordCounts.merge("java", 1, Integer::sum);
        System.out.println(wordCounts.get("java")); // 2
    }
}
```

**Best Practice:**
- Use `computeIfAbsent` for caches, `merge` for counters / grouping.
- Keep mapping functions pure, short, and non-blocking. No I/O, no locks, no recursive map access inside them.
- Don't rely on `size()` for exactness in concurrent code — same as Java 5, it is weakly consistent.

---

## 7. Best Practices — The Cheat List (Only Java 8)

### CompletableFuture
- Chain, don't block. Use `thenApply` / `thenCompose` / `thenCombine`.
- Handle errors at the end of every chain.
- Use a custom pool for blocking / I/O work. Never block the common pool.
- `join()` in app code, `get(timeout)` when you need a timeout.

### Streams
- Parallel streams only for CPU-bound, large, stateless, order-free work.
- Never mutate shared state in a parallel stream. Use `collect` / `reduce`.
- Never do I/O or blocking inside a parallel stream.

### Locks & Counters
- `StampedLock`: optimistic read + `validate()`, fallback to read lock. No reentrancy.
- `LongAdder` for high-contention counters. `AtomicLong` when you need CAS.
- `ConcurrentHashMap`: `compute` / `merge` / `computeIfAbsent`, keep mapping functions pure.

### Never in New Java 8 Code
- `Future.get()` immediately after `submit()` / `supplyAsync()`.
- Parallel streams for I/O, small data, or shared mutable state.
- Check-then-act on a concurrent map by hand.
- Long lambdas inside async calls — extract a named method.
- `StampedLock` in reentrant code.

---

## 8. Tips — 30-Second Answers (Only Java 8)

1.  **"Why `CompletableFuture` over `Future`?"**
    > `Future` has no callback — you must block or poll. `CompletableFuture` chains, combines, and handles errors without blocking.

2.  **"`thenApply` vs `thenCompose`?"**
    > `thenApply` transforms a result. `thenCompose` returns a new future and flattens it — use it when the next step is async.

3.  **"Why not parallel streams for I/O?"**
    > They share the common pool. Blocking I/O blocks common-pool workers and starves every other parallel stream.

4.  **"Why `LongAdder` faster than `AtomicLong` under contention?"**
    > `AtomicLong` has one CAS slot everyone fights over. `LongAdder` stripes into cells, each thread updates its own cell, `sum()` merges.

5.  **"Why `StampedLock` optimistic read needs `validate()`?"**
    > Optimistic read takes no lock. Without `validate()`, a concurrent write can leave you with torn / inconsistent data.

6.  **"Why `computeIfAbsent` mapping must be pure?"**
    > It runs while the bin is locked. Blocking, I/O, or recursive map access inside it can deadlock or stall other threads.

7.  **"When to use `CompletableFuture` custom pool?"**
    > For any blocking / I/O work. Default common pool is for CPU-bound work only.

8.  **"`allOf` returns what?"**
    > `Void`. It waits for all futures, but does not collect results. Collect them yourself after `join()`.

9.  **"Why not mutate shared state in a parallel stream?"**
    > Many threads run the lambda at once. Shared mutation is a race, even if it "works" in a test.

10. **"When is `StampedLock` the wrong choice?"**
    > When code is reentrant, complex, or needs `Condition`. Use `ReentrantReadWriteLock` there.

---

## 9. Cheat Sheet — Java 8 Only

### Timeline

| Part | Headline | Verdict in Java 8 |
| :--- | :--- | :--- |
| **Lambdas** | Less noise for tasks | **Core** |
| **Parallel streams** | Data parallelism via common pool | **Core when it fits** |
| **`CompletableFuture`** | Composable async | **Core** |
| **`StampedLock`** | Optimistic read | **Core for read-heavy** |
| **`LongAdder`** | Striped counter | **Core for high contention** |
| **`ConcurrentHashMap` (Java 8)** | Atomic `compute` / `merge` | **Core** |

### API Quick Reference

| API | What it does | Verdict |
| :--- | :--- | :--- |
| `CompletableFuture` | Chain, combine, handle errors | Core, custom pool for I/O |
| `supplyAsync` / `runAsync` | Start async work | Core |
| `thenApply` / `thenCompose` / `thenCombine` | Transform / flatten / combine | Core |
| `allOf` / `anyOf` | Wait for all / first | Core, `allOf` returns `Void` |
| `exceptionally` / `handle` / `whenComplete` | Error handling | Core, handle at end of chain |
| `parallelStream()` | Data parallelism | CPU-bound, large, stateless only |
| `StampedLock` | Optimistic read + write lock | Read-heavy only, no reentrancy |
| `LongAdder` / `LongAccumulator` | Striped counter | High contention counters |
| `ConcurrentHashMap.compute` / `merge` / `computeIfAbsent` | Atomic map ops | Core, pure mapping functions |

### Decision Tree (Only Java 8)

```text
Need async work?
  ├─ Need result + chaining? -> CompletableFuture, custom pool for I/O
  └─ Need to wait for many? -> allOf, then collect results yourself

Need data parallelism?
  ├─ CPU-bound, large, stateless, order-free? -> parallelStream()
  └─ Else? -> sequential stream / ExecutorService

Need a counter?
  ├─ High contention, no CAS needed? -> LongAdder
  └─ Low contention / need CAS? -> AtomicLong

Need a lock?
  ├─ Read-heavy, short writes, no reentrancy? -> StampedLock (optimistic + validate)
  └─ Else? -> ReentrantLock / ReentrantReadWriteLock

Need to update a map atomically?
  └─ compute / computeIfAbsent / merge, pure mapping function
```
