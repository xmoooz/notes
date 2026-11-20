---
title: "Java Thread & Concurrency Timeline: Java5"
date: 2026-11-02T23:30:00Z
lastmod: 2026-11-03T23:30:00Z
draft: false
categories: ["java", "concurrency", "timeline"]
tags: ["java", "concurrency", "timeline", "threads", "executors", "locks", "atomics", "java5"]
series: ["java concurrency timeline"]
author: "xmoooz"
description: "How Java 5 rebuilt concurrency: Executors, Callable/Future, ReentrantLock, atomics and CAS, concurrent collections, and synchronizers."
url: 2026/11/19/java-concurrency-timeline-java5
---

> This is the Doug Lea release. Modern Java concurrency starts here.

---

## 0. Global Vision — Java 5 in 2 Minutes

This file covers only Java 5. Nothing after that.

The story inside Java 5 is simple:

**Before Java 5, every company wrote its own thread pool, lock, queue, and latch — all buggy. Java 5 standardized all of it.**

| Problem in 1.0-1.4 | What Java 5 gives you |
| :--- | :--- |
| `new Thread()` per task, no reuse, no control | **Executor Framework** — pool, queue, shutdown |
| `Runnable` returns `void`, no result, no checked exception | **Callable + Future** — return value, exception, timeout, cancel |
| `synchronized` only: no timeout, no try-lock, no interrupt while waiting, one wait-set | **Locks** — `ReentrantLock`, `ReadWriteLock`, `Condition` |
| `synchronized` for a simple counter is heavy | **Atomics + CAS** — lock-free single-variable updates |
| `Hashtable` / `synchronizedMap` lock the whole map | **Concurrent Collections** — segment locking, `BlockingQueue` |
| Hand-written `wait/notify` for every coordination | **Synchronizers** — `CountDownLatch`, `CyclicBarrier`, `Semaphore`, `BlockingQueue` |

### The mental shift in Java 5

| Before (1.0-1.4) | In Java 5 |
| :--- | :--- |
| You manage threads | You submit tasks |
| You manage locks | You choose a lock with the control you need |
| You manage wait/notify | You use a synchronizer / queue that does it for you |
| You protect data by blocking everyone | You pick: block, try-lock, CAS, or segment lock |

### The one sentence to remember for each part

- **Executors:** Stop creating threads manually. Submit tasks to a pool.
- **Callable + Future:** A task can now return a value and throw a checked exception.
- **Locks:** `synchronized` with control — timeout, try-lock, interrupt, fairness, multiple conditions.
- **Atomics + CAS:** For one variable, don't block — try, and retry if someone changed it.
- **Concurrent Collections:** Don't lock the whole map for one bucket.
- **Synchronizers:** Don't hand-write coordination. Use a latch, barrier, semaphore, or queue.

### What to take away from Java 5

If you only learn the APIs, you miss the point. The point is choosing the right tool for the coordination you need:

- Need a result? `Callable`, not a shared variable + `join()`.
- Need to avoid deadlock? `tryLock(timeout)`, not blocking forever.
- Need to coordinate? Latch / Barrier / Semaphore / Queue, not manual `wait/notify`.
- Need speed for one counter? CAS, not a full lock.
- Need to pass work? `BlockingQueue`, not a hand-built queue.


---

## 1. Executor Framework — Stop Creating Threads Manually

### What
`Executor`, `ExecutorService`, `Executors` factory, `ThreadPoolExecutor`.

### Why
`new Thread()` per task = expensive (1 MB stack per thread), no reuse, no queue, no control. If 1000 requests arrive, 1000 threads crash the JVM.

### Benefit
- Thread reuse via pool.
- Queue + rejection policy + shutdown control.
- Separates *what to run* from *how to run*.

### Downside
- If you forget `shutdown()`, JVM never exits.
- Unbounded queue + `newCachedThreadPool()` can still OOM if tasks pile up.
- Need to understand `RejectedExecutionException`.

### Status in Java 5
| API | Verdict |
| :--- | :--- |
| `ExecutorService`, `newFixedThreadPool`, `newSingleThreadExecutor`, `newScheduledThreadPool` | **CORE. Use these.** |
| `newCachedThreadPool()` | **USE WITH CARE.** Unbounded growth. Only for short-lived, fast tasks with known load. |
| Raw `new Thread()` per request | **DO NOT USE in server code.** No reuse, no backpressure. |

### Real Sample Code

```java
import java.util.concurrent.*;

public class ExecutorDemo {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        executor.execute(() -> System.out.println("Task by " + Thread.currentThread().getName()));

        executor.shutdown(); // MUST shutdown, else JVM never exits
        // executor.awaitTermination(5, TimeUnit.SECONDS); // optional: wait for finish
    }
}
```

```java
// The 4 factories in Java 5 — know when to pick each:
Executors.newFixedThreadPool(5);       // fixed size, bounded concurrency
Executors.newCachedThreadPool();        // grows as needed, dangerous if unbounded load
Executors.newSingleThreadExecutor();    // one thread, ordered execution
Executors.newScheduledThreadPool(2);    // delayed / periodic tasks, replaces Timer
```

**Best Practice:**
- **Always `shutdown()` in `finally` / shutdown hook.** A pool keeps non-daemon threads alive.
- **Prefer `newFixedThreadPool` / `newSingleThreadExecutor` / `newScheduledThreadPool` for server code.** They bound concurrency.
- **Give the pool a bounded queue in production** (via `ThreadPoolExecutor` constructor directly) — unbounded queue hides overload until OOM.
- **Handle rejection:** Know what happens when queue is full — `AbortPolicy` (default, throws), `CallerRunsPolicy`, `DiscardPolicy`, `DiscardOldestPolicy`.
- **Don't call `Future.get()` immediately after `submit()`** if you wanted parallelism — you just serialized it.

**Tip:** "Why Executor over `new Thread()`?" -> Reuse, control queue, control max threads, graceful shutdown, separates task from execution. `new Thread()` per request = OOM under load.

---

## 2. Callable + Future — Runnable That Returns a Value

### What
`Callable<V>`, `Future<V>`, `ExecutorService.submit()`.

### Why
`Runnable.run()` returns `void` and cannot throw checked exceptions. How to get a result from a thread? Old hack: shared variable + `join()` + manual synchronization. Ugly and error-prone.

### Benefit
- Returns a value.
- Can throw checked exception.
- `Future.get()` blocks, `get(timeout)` times out, `isDone()` checks, `cancel()` cancels.

### Downside
- `Future.get()` blocks — if you call it immediately, you lose parallelism.
- No callback in Java 5 — you must poll or block.
- Exception is wrapped in `ExecutionException` — you must unwrap `getCause()`.

### Status in Java 5
| API | Verdict |
| :--- | :--- |
| `Callable`, `Future`, `submit()` | **CORE.** |
| `Runnable` + shared variable + `join()` to get a result | **DO NOT USE.** That is what `Callable` replaced. |

### Real Sample Code — Basic

```java
import java.util.concurrent.*;

public class CallableDemo {
    public static void main(String[] args) throws Exception {
        Callable<Integer> task = () -> {
            Thread.sleep(500);
            return 10 + 20;
        };

        ExecutorService ex = Executors.newSingleThreadExecutor();
        try {
            Future<Integer> future = ex.submit(task);
            System.out.println("Doing other work...");
            System.out.println("Result: " + future.get()); // 30, blocks till done
        } finally {
            ex.shutdown();
        }
    }
}
```

### Runnable vs Callable

| Feature | Runnable | Callable |
| :--- | :--- | :--- |
| Return | void | Value (generic) |
| Exception | No checked | Yes |
| Method | `run()` | `call()` |
| Use with | `execute()` | `submit()` -> `Future` |

### Deep Dive: No Callback in Java 5 — You Must Poll or Block

In Java 5, `Future` cannot call you back. You must go get the result.

**Option 1: Block — `get()`**
```java
Future<Integer> future = executor.submit(() -> { Thread.sleep(2000); return 100; });
Integer result = future.get(); // BLOCKS HERE! Main stuck 2 sec, doing nothing.
```
> **Block = You stop and wait. Like waiting at a shop counter until food is ready. You can't do anything else.**

**Option 2: Poll — `isDone()` in a loop**
```java
Future<Integer> future = executor.submit(() -> { Thread.sleep(2000); return 100; });
while (!future.isDone()) {
    System.out.println("Not done yet...");
    Thread.sleep(500); // Keep asking again and again
}
System.out.println("Done: " + future.get());
```
> **Poll = You keep asking "Are you done? Are you done?" Wastes CPU, ugly code.**

**Option 3 (Best Practice in Java 5): Submit many, then collect**
```java
List<Future<Integer>> futures = new ArrayList<>();
for (int i = 0; i < 5; i++) {
    futures.add(executor.submit(() -> { Thread.sleep(500); return 1; }));
}
// Do other work here, THEN collect — don't get() one by one immediately
int sum = 0;
for (Future<Integer> f : futures) {
    sum += f.get(); // blocks, but all tasks already ran in parallel
}
```

**Best Practice:**
- **Submit all, then `get()` all.** That preserves parallelism.
- **Use `get(timeout)` in production.** `get()` with no timeout can hang forever if task hangs. `get(5, TimeUnit.SECONDS)` + handle `TimeoutException`.
- **Unwrap exceptions:** `Future.get()` throws `ExecutionException` — real cause is `e.getCause()`. Log the cause, not just the wrapper.
- **Cancel properly:** `future.cancel(true)` interrupts if running. `cancel(false)` only cancels if not started. Check `isCancelled()`.

**Tip:** "What is the limitation of Java 5 `Future`?" -> No callback. You must block with `get()` or poll with `isDone()`. That is why you submit many tasks first, do other work, then collect results.

---

## 3. Locks — Better Than `synchronized` 

### What
`Lock`, `ReentrantLock`, `ReadWriteLock` / `ReentrantReadWriteLock`, `Condition`, `lockInterruptibly()`, `tryLock()`.

### Why
`synchronized` cannot: try to lock with timeout, interrupt while waiting for lock, try lock without blocking, or have fair ordering. It also has only one wait-set.

### Benefit
- `tryLock()`, `tryLock(timeout)`, `lockInterruptibly()`.
- `ReentrantLock` — same thread can lock again.
- `ReadWriteLock` — many readers, one writer.
- `Condition` — multiple wait/notify queues.

### Downside
- Must manually `unlock()` in `finally` — forget it = deadlock.
- More verbose than `synchronized`.
- Easier to misuse if you don't follow pattern.

### Status in Java 5
| API | Verdict |
| :--- | :--- |
| `ReentrantLock`, `ReadWriteLock`, `Condition` | **CORE when you need control.** |
| `synchronized` | **STILL CORE for simple, short sections.** Don't replace it everywhere. |

### Real Sample Code — Basic Lock

```java
import java.util.concurrent.locks.*;

class CounterWithLock {
    private int count = 0;
    private final Lock lock = new ReentrantLock();

    void increment() {
        lock.lock();
        try { count++; } finally { lock.unlock(); } // MUST be in finally
    }

    int get() {
        lock.lock();
        try { return count; } finally { lock.unlock(); }
    }
}
```

### Deep Dive: 5 Problems of `synchronized` + How Lock Fixes Them

#### Problem 1: Cannot Try Lock Without Blocking

**`synchronized` — Blocks forever:**
```java
synchronized (lock) {
    // If another thread holds lock, this thread WAITS forever
    // It has no choice. It cannot say "if busy, do something else"
}
```

**Lock — `tryLock()` — Try, if busy, go do other work:**
```java
if (lock.tryLock()) { // Try once, don't wait
    try { /* Got lock */ } finally { lock.unlock(); }
} else {
    System.out.println("Busy, will try later");
}
```
> Real use: Printer busy, don't freeze UI, show "Printer busy, try later".

#### Problem 2: Cannot Lock With Timeout

**`synchronized` — No timeout:**
```java
synchronized (lock) {
    // If other thread deadlocks, this thread waits FOREVER
}
```

**Lock — `tryLock(timeout)`:**
```java
if (lock.tryLock(2, TimeUnit.SECONDS)) {
    try { /* Got lock */ } finally { lock.unlock(); }
} else {
    System.out.println("Timeout, giving up to avoid deadlock");
}
```
> Real use: Bank transfer. If you can't lock account in 2 sec, abort instead of hanging forever. This prevents deadlock.

#### Problem 3: Cannot Interrupt While Waiting For Lock

**`synchronized` — Cannot interrupt:**
```java
// Thread B waits for lock...
// Main tries threadB.interrupt(); // USELESS! B still waits, ignores interrupt
```

**Lock — `lockInterruptibly()`:**
```java
try {
    lock.lockInterruptibly(); // CAN be interrupted while waiting
    try { /* work */ } finally { lock.unlock(); }
} catch (InterruptedException e) {
    System.out.println("I was interrupted while waiting for lock, I'm out!");
    Thread.currentThread().interrupt(); // restore flag
}
```
> Real use: User clicks Cancel. With `synchronized`, Cancel does nothing. With `lockInterruptibly()`, thread wakes up immediately.

#### Problem 4: No Fair Ordering

**`synchronized` — Unfair, JVM decides:**
```java
// T1, T2, T3 all waiting, who gets it next? Random. T3 might starve.
```

**Lock — Fair Lock:**
```java
Lock fairLock = new ReentrantLock(true); // true = Fair, FIFO queue
```
> Real use: Ticket booking queue. Fair ensures first person gets ticket first. Downside: Fair is slower. Default is unfair = faster.

#### Problem 5: Only One Wait-Set — `wait()` / `notify()`

**`synchronized` — Only ONE waiting room:**
```java
class Restaurant {
    synchronized void waitForPizza() throws Exception { wait(); }
    synchronized void waitForBurger() throws Exception { wait(); }
    synchronized void pizzaReady() { notifyAll(); } // Wakes EVERYONE! Wasteful.
    // Cannot wake only Pizza waiters.
}
```

**Lock + Condition — Multiple Waiting Rooms:**
```java
class Restaurant {
    Lock lock = new ReentrantLock();
    Condition pizzaWaiters = lock.newCondition(); // Room 1
    Condition burgerWaiters = lock.newCondition(); // Room 2

    void waitForPizza() throws Exception {
        lock.lock();
        try { pizzaWaiters.await(); } finally { lock.unlock(); }
    }
    void pizzaReady() {
        lock.lock();
        try { pizzaWaiters.signalAll(); } // Wake ONLY pizza waiters!
        finally { lock.unlock(); }
    }
}
```
> `await()` = `wait()`, `signal()` = `notify()`, `signalAll()` = `notifyAll()`.

### ReentrantLock — Same Thread Can Lock Again

```java
void methodA() {
    lock.lock();
    try { methodB(); } finally { lock.unlock(); }
}
void methodB() {
    lock.lock(); // SAME thread locks again - ALLOWED, count 2
    try { /* work */ } finally { lock.unlock(); } // 2 -> 1
}
```
If NOT reentrant, `methodB` would deadlock on itself. Both `synchronized` and `ReentrantLock` are reentrant.

### ReadWriteLock — Many Readers, One Writer

**`synchronized` problem:** Even if 10 threads only want to READ, they block each other. Reading is safe.

```java
ReadWriteLock rw = new ReentrantReadWriteLock();

String read() {
    rw.readLock().lock(); // Many threads can hold readLock TOGETHER
    try { return data; } finally { rw.readLock().unlock(); }
}
void write(String v) {
    rw.writeLock().lock(); // Only ONE thread, and NO readers allowed
    try { data = v; } finally { rw.writeLock().unlock(); }
}
```
> Real use: Cache, config file. 99% reads, 1% writes. ReadWriteLock makes reads fast.

**Best Practice:**
- **Always `unlock()` in `finally`.** No exception. This is the #1 Lock bug.
- **Use `tryLock(timeout)` for any lock ordering that could deadlock** (e.g. transfer between two accounts).
- **Don't use fair lock by default.** Fair = slower. Use fair only when starvation / ordering actually matters.
- **Don't hold a `ReadWriteLock` write lock while doing I/O.** Same rule as `synchronized`: keep critical section tiny.

### Summary Table — `synchronized` vs `Lock`

| Feature | `synchronized` | `Lock` |
| :--- | :--- | :--- |
| Try without blocking | No | `tryLock()` |
| Timeout | No | `tryLock(2, SECONDS)` |
| Interrupt while waiting | No | `lockInterruptibly()` |
| Fair ordering | No, random | `new ReentrantLock(true)` |
| Multiple wait sets | 1 set only | Many `Condition`s |
| Need unlock() in finally? | No, auto | Yes, must do manually |
| Simple to use? | Very simple | More verbose |

### When To Use Which?
**Use `synchronized` when:** Simple, short critical section, don't need timeout/interrupt/fairness, want less code.
**Use `Lock` when:** Need timeout to avoid deadlock, need to interrupt waiting thread, need fairness, need multiple Conditions, need ReadWriteLock.

> `synchronized` = Automatic, simple, but no control. `Lock` = Manual, more code, but full control.

---

## 4. Atomics + CAS — Lock-Free 

### What
`AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference`, `AtomicStampedReference`, CAS (Compare-And-Swap).

### Why
For a simple counter, `synchronized` is heavy — OS lock, context switch. Atomics use CPU CAS instruction — no lock, much faster.

### Benefit
- Lock-free, very fast for counters, flags.
- No deadlock.

### Downside
- Only for single variable. Cannot atomically update two variables together.
- Under high contention, CAS retries can spin and waste CPU.

### Status in Java 5
| API | Verdict |
| :--- | :--- |
| `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference` | **CORE for single-variable counters / flags / references.** |
| `synchronized` for a single counter | **DO NOT USE if `Atomic*` fits.** Heavier, slower. |

### Real Sample Code

```java
import java.util.concurrent.atomic.*;

public class AtomicDemo {
    static AtomicInteger count = new AtomicInteger(0);

    public static void main(String[] args) throws Exception {
        Runnable task = () -> { for (int i = 0; i < 1000; i++) count.incrementAndGet(); };
        Thread t1 = new Thread(task); Thread t2 = new Thread(task);
        t1.start(); t2.start(); t1.join(); t2.join();
        System.out.println(count.get()); // always 2000
    }
}
```

### What Is CAS? — Compare-And-Swap

> **CAS = "Check if value is still what I saw, if yes, change it. If no, try again."**

It's a **CPU instruction**, not Java code. CPU does check + change in **one atomic step**.

### Problem CAS Solves

`count++` is NOT one step. It's 3 steps:
```java
// 1. READ count (10)
// 2. ADD 1 (11)
// 3. WRITE count (11)
```
Race:
```text
T1 reads 10
T2 reads 10  <- Both read same old value!
T1 writes 11
T2 writes 11  <- One increment LOST! Should be 12, but is 11
```
**Lock fixes it** by blocking T2. But blocking is slow.
**CAS fixes it** without blocking — by retrying.

### How CAS Works

CAS has 3 inputs: Expected value, New value, Memory location.
```text
if (currentValue == expectedValue) {
    currentValue = newValue;
    return true; // Success!
} else {
    return false; // Fail, someone changed it, try again
}
```

Example — `AtomicInteger.incrementAndGet()`:
```text
// T1: CAS(expected=10, new=11) -> current is 10? YES -> change to 11 -> SUCCESS
// T2: CAS(expected=10, new=11) -> current is 11? NO -> FAIL
// T2: Re-read current=11, try CAS(expected=11, new=12) -> SUCCESS
```

What `AtomicInteger` does internally (simplified):
```java
public int incrementAndGet() {
    while (true) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next)) return next;
        // Fail -> loop again, re-read and retry
    }
}
```
This loop is called **Spin / Busy Retry**. This is **Optimistic Locking** — "Assume no conflict, if conflict, retry."

### CAS vs Lock — Full Comparison

|  | **CAS (Lock-Free)** | **Lock (`synchronized` / `ReentrantLock`)** |
| :--- | :--- | :--- |
| How it works? | CPU atomic instruction, retry if fail | Block other thread, OS suspends it |
| Blocking? | **No**, never blocks | **Yes**, other threads wait / sleep |
| Context Switch? | No | Yes, expensive |
| Deadlock? | **Impossible**, no lock to hold | **Possible** |
| Speed - Low Contention | **Very Fast** - no OS call | Slower - OS lock |
| Speed - High Contention | **Slow** - 100 threads retrying = CPU spinning | **Better** - threads sleep, don't waste CPU |
| What it can do? | Only **ONE variable** at a time | Can protect **many variables / whole block** |
| Example | `AtomicInteger`, `AtomicReference` | `synchronized`, `ReentrantLock` |
| Type | Optimistic - "Try, if fail, retry" | Pessimistic - "Lock first, then work" |

```text
LOCK:
T1 [LOCK] -> working -> [UNLOCK]
T2 [WAIT........] -> [LOCK] -> working -> [UNLOCK]
     ^ T2 sleeps, wastes no CPU, but slow to wake up

CAS:
T1: Try -> Success
T2: Try -> Fail -> Try again -> Success
     ^ T2 keeps spinning, wastes CPU, but no OS wake-up delay
```

### When To Use Which?
**Use CAS / `Atomic*` when:** Single variable, low contention (2-3 threads), want max speed, no deadlock.
```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
AtomicReference<User> user = new AtomicReference<>();
user.compareAndSet(oldUser, newUser);
```

**Use Lock when:** Multiple variables must change together, high contention (100 threads), complex logic.
```java
// Need to move money: debit A + credit B must be TOGETHER
// CAS can't do 2 variables at once!
synchronized (lock) {
    accountA -= 100;
    accountB += 100;
}
```

### Dark Side — ABA Problem 

CAS only checks **value**, not **history**.
```text
1. T1 reads A
2. T2 changes A -> B -> A (back to A)
3. T1 does CAS(expected=A, new=C)
4. CPU sees current is A, says SUCCESS
5. But T1 didn't know it changed to B and back! History was lost.
```
Fix in Java 5: `AtomicStampedReference` — stores value + version stamp. CAS checks both.
```java
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
// T2 changes A->B stamp 0->1, B->A stamp 1->2
// T1 tries CAS(expected=A, stamp=0) -> FAIL because stamp is now 2
```

**Best Practice:**
- **Use `Atomic*` for counters, flags, single references.** Don't hand-roll CAS loops unless you need custom logic.
- **Don't use CAS for multi-variable invariants.** One CAS = one variable. Two variables need a lock.
- **Watch contention:** If 100 threads hammer one `AtomicInteger`, spinning wastes CPU. Consider sharding / batching, or a lock.

> **Lock = "Block others, I'll do it alone."**
> **CAS = "Let everyone try, if someone changed it, I'll try again."**
> **Low contention -> CAS wins. High contention / multi-variable -> Lock wins.**

### Q: `volatile` vs `AtomicInteger`?
> `volatile int` = visibility only, `count++` still race. `AtomicInteger` = visibility + atomicity via CAS, `incrementAndGet()` is safe.

---

## 5. Concurrent Collections

### What
`ConcurrentHashMap`, `CopyOnWriteArrayList`, `CopyOnWriteArraySet`, `BlockingQueue` implementations, `ConcurrentLinkedQueue`.

### Why
`Hashtable` and `synchronizedMap` lock the WHOLE map for every operation. 10 threads = 10 threads waiting. `ConcurrentHashMap` locks only a segment/bucket.

### Benefit
- High concurrency, no full lock.
- `BlockingQueue` — built-in producer-consumer, no manual wait/notify.

### Downside
- `size()` / iterators are weakly consistent — may not see latest writes.
- `CopyOnWriteArrayList` copies whole array on every write — terrible for write-heavy use.

### Status in Java 5
| API | Verdict |
| :--- | :--- |
| `ConcurrentHashMap` | **CORE for shared maps.** Default choice in Java 5. |
| `CopyOnWriteArrayList` | **USE ONLY for read-heavy, write-rare lists** (listeners, config). |
| `Hashtable`, `synchronizedMap` | **DO NOT USE in new Java 5 code.** Whole-map lock. |
| `Vector` | **DO NOT USE in new code.** Same reason. |

### Real Sample Code

```java
import java.util.concurrent.*;

public class ConcurrentDemo {
    public static void main(String[] args) throws Exception {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        map.put("A", 1);
        map.putIfAbsent("A", 2); // atomic check-and-put, no race — map still 1

        BlockingQueue<String> queue = new LinkedBlockingQueue<>();
        queue.put("data"); // blocks if full
        String data = queue.take(); // blocks if empty
        System.out.println(data);
    }
}
```

### Q: How does `ConcurrentHashMap` work in Java 5?
> It uses **Segment Locking** — map divided into 16 segments, each segment has its own lock. Thread locking Segment 1 doesn't block thread in Segment 2. `get()` is lock-free (volatile read). That is why it is much faster than `Hashtable` / `synchronizedMap`, which lock the whole map.

### Q: `ConcurrentHashMap` vs `synchronizedMap` vs `Hashtable`?
|  | `Hashtable` | `synchronizedMap` | `ConcurrentHashMap` |
| :--- | :--- | :--- | :--- |
| Locks | Whole map | Whole map | Segment only |
| Null key/value? | No | Depends | No null key/value |
| Speed | Slow | Slow | Fast |
| Iterator | Fail-fast (throws) | Fail-fast | Weakly consistent (no throw) |

**Best Practice:**
- **Use `putIfAbsent`, `remove(key, value)`, `replace()`** — they are atomic. Don't do `if (!map.containsKey(k)) map.put(k, v)` — that is still a race.
- **Don't rely on `size()` for exactness** in concurrent code — it is weakly consistent. Use it for monitoring, not for correctness.
- **Don't use `CopyOnWriteArrayList` for write-heavy lists.** Every write copies the whole array.
- **Never put `null` key/value in `ConcurrentHashMap`.** It throws `NullPointerException` — by design, to avoid ambiguity between "missing" and "null".

---

## 6. Synchronizers — CountDownLatch, CyclicBarrier, Semaphore, BlockingQueue 

These 4 are called **Synchronizers** because they don't protect data like `Lock` does. They **coordinate threads** — they control *when* threads run, not *what* data they touch.

> `Lock` = "Don't touch this data at same time"
> `Synchronizer` = "Wait here until everyone is ready" / "Only 2 allowed inside"

---

### 6.1 CountDownLatch — "Wait For Them To Finish"

**What?** One-time gate. Set count e.g. 3. Workers call `countDown()` when done. Main calls `await()` and waits until count hits 0.

**Why?** Before Java 5, you had to `join()` threads. But what if workers are in a pool? You don't have Thread objects. What if you want to wait for *services* to start, not threads? Latch lets you wait for *events*, not threads.

**How?**
```java
CountDownLatch latch = new CountDownLatch(3);
latch.countDown(); // 3 -> 2 -> 1 -> 0, in worker
latch.await(); // Blocks until count = 0, in main
```

**Real Sample Code — 2 Real Uses:**

**Use 1: Main waits for workers**
```java
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        System.out.println("Worker done");
        latch.countDown();
    }).start();
}
latch.await();
System.out.println("All done, main continues");
```

**Use 2: All threads start TOGETHER — Load Test**
```java
CountDownLatch startSignal = new CountDownLatch(1);
for (int i = 0; i < 100; i++) {
    new Thread(() -> {
        try { startSignal.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        System.out.println("GO!");
    }).start();
}
Thread.sleep(1000);
startSignal.countDown(); // Gate opens, all 100 start at EXACT same time
```

**Best Practice:**
- **Always `countDown()` in `finally`.** If worker crashes without counting down, `await()` waits forever.
- **Always use timeout in production:** `await(5, TimeUnit.SECONDS)` + check return value. Never bare `await()` for a service startup gate.
- **One-time only, cannot reset.** Once 0, stays 0 forever.

```java
try { /* work */ } finally { latch.countDown(); }
boolean ok = latch.await(5, TimeUnit.SECONDS);
if (!ok) System.out.println("Timeout!");
```

---

### 6.2 CyclicBarrier — "Wait For Each Other, Then Go Together"

**What?** Meeting point. Set parties e.g. 3. All 3 threads must arrive at `await()`. When last one arrives, barrier opens and **all 3 continue together**. And it **resets** automatically, so you can use it again.

**Why?** Latch is one thread waiting for others. Barrier is **all threads waiting for each other**. Example: 3 friends go on trip. "Meet at gas station, then we go together." No one leaves until all 3 are there.

**How?**
```java
CyclicBarrier barrier = new CyclicBarrier(3);
// In EACH thread:
barrier.await(); // Wait until all 3 arrive, then all wake up
```

**Real Sample Code — Game / Multi-Phase Task**
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println(">>> All 3 reached barrier, starting next level!");
});
for (int i = 1; i <= 3; i++) {
    final int id = i;
    new Thread(() -> {
        try {
            System.out.println("Player " + id + " loading...");
            Thread.sleep(id * 1000);
            System.out.println("Player " + id + " ready, waiting");
            barrier.await(); // Wait for others
            System.out.println("Player " + id + " playing level 2");
            barrier.await(); // Can reuse! Wait again for level 3
            System.out.println("Player " + id + " playing level 3");
        } catch (Exception e) { Thread.currentThread().interrupt(); }
    }).start();
}
```

**Best Practice:**
- **Handle `BrokenBarrierException`.** If ONE thread fails / times out / interrupts, barrier **breaks** for everyone. All others get `BrokenBarrierException`. Treat it as a signal to abort the phase, not retry blindly.
- **Use for multi-phase tasks** (Level 1 -> Level 2 -> Level 3). Has optional barrier action — code that runs once when all arrive.

**Latch vs Barrier:**
|  | `CountDownLatch` | `CyclicBarrier` |
| :--- | :--- | :--- |
| Who waits? | 1 thread waits for others | All threads wait for each other |
| Reusable? | No, one-time | Yes, resets automatically |
| Who counts down? | Workers `countDown()`, main `await()` | All threads `await()` |
| Example | Main waits for workers | 3 friends meet, then go together |

---

### 6.3 Semaphore — "Only N Allowed Inside"

**What?** Permit counter. Set permits e.g. 2. Thread must `acquire()` a permit to enter. If no permits left, it waits. When done, it `release()`s permit.

**Why?** Sometimes you don't want to block everyone, you want to **limit** concurrency. DB allows only 5 connections. Printer allows only 1 job. API allows only 10 requests at a time.

**How?**
```java
Semaphore sem = new Semaphore(2); // Only 2 permits
sem.acquire(); // Take 1, if none left, WAIT
try { /* Only 2 threads can be here */ } finally { sem.release(); }
```

**Real Sample Code — DB Connection Limit**
```java
Semaphore dbPermits = new Semaphore(2);
for (int i = 1; i <= 5; i++) {
    final int id = i;
    new Thread(() -> {
        try {
            System.out.println("User " + id + " waiting for DB...");
            dbPermits.acquire();
            System.out.println("User " + id + " GOT DB, querying...");
            Thread.sleep(2000);
        } catch (InterruptedException e) { Thread.currentThread().interrupt();
        } finally { dbPermits.release(); } // MUST release!
    }).start();
}
// Only 2 inside at a time, others queue.
```

**Fair vs Unfair:**
```java
new Semaphore(2, true);  // Fair - FIFO queue
new Semaphore(2, false); // Unfair - random, faster, default
```

**Best Practice:**
- **Always `release()` in `finally`.** If you forget, permit leaks. After few leaks, no permits left, all wait forever = deadlock.
- **Use `tryAcquire(timeout)` for rate limiting** where waiting forever is worse than failing fast.
- **Don't use `Semaphore(1)` as a lock replacement** unless you have a reason — `ReentrantLock` gives ownership, fairness, and `Condition`. Semaphore's power is N > 1.

**Semaphore vs Lock:**
|  | `Semaphore(1)` | `ReentrantLock` |
| :--- | :--- | :--- |
| Permits | Any thread can release | Only owner can unlock |
| Use for | Limit N threads | Protect 1 critical section |
| Example | 5 DB connections | 1 shared variable |

---

### 6.4 BlockingQueue — The 4th Synchronizer That Killed `wait/notify` 

**What?** A queue that blocks you automatically. If empty, `take()` waits. If full, `put()` waits. You write ZERO `wait()` / `notify()` code. It's the real Producer-Consumer solution.

**Why?** Before Java 5, Producer-Consumer was painful. You had to write manual `wait/notify`, remember `while` not `if`, remember `notifyAll()`, remember `synchronized`. One mistake = deadlock. `BlockingQueue` does all inside, tested by JDK.

**Old Way (1.0-1.4):**
```java
synchronized void put(String s) throws Exception {
    while (list.size() == MAX) wait();
    list.add(s);
    notifyAll();
}
synchronized String take() throws Exception {
    while (list.isEmpty()) wait();
    String s = list.remove(0);
    notifyAll();
    return s;
}
```

**New Way (Java 5):**
```java
BlockingQueue<String> queue = new LinkedBlockingQueue<>(5);
queue.put("data"); // If full, BLOCKS until consumer takes one
String data = queue.take(); // If empty, BLOCKS until producer puts one
// No synchronized, no wait(), no notify()
```

**Real Sample Code — Producer-Consumer in 20 Lines:**
```java
BlockingQueue<String> queue = new LinkedBlockingQueue<>(3);
new Thread(() -> {
    try {
        for (int i = 1; i <= 5; i++) {
            System.out.println("Producing " + i);
            queue.put("Item-" + i); // Blocks if full
            Thread.sleep(500);
        }
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
}).start();
new Thread(() -> {
    try {
        for (int i = 1; i <= 5; i++) {
            String item = queue.take(); // Blocks if empty
            System.out.println("Consumed " + item);
            Thread.sleep(1000);
        }
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
}).start();
```

**4 Types of Methods — Note:**
|  | Throws Exception | Returns Special Value | Blocks Forever | Blocks With Timeout |
| :--- | :--- | :--- | :--- | :--- |
| **Add (Producer)** | `add(e)` - throws if full | `offer(e)` - returns false if full | `put(e)` - waits until space | `offer(e, 2, SECONDS)` |
| **Remove (Consumer)** | `remove()` - throws if empty | `poll()` - returns null if empty | `take()` - waits until item | `poll(2, SECONDS)` |

- Use `put()` / `take()` — 90% cases, you want to wait.
- Use `offer()` / `poll()` — don't want to wait, just try.
- Use `offer(timeout)` / `poll(timeout)` — wait, but not forever, prevents deadlock.

**Which BlockingQueue To Use?**
| Queue | How it works | Use When |
| :--- | :--- | :--- |
| `ArrayBlockingQueue` | Fixed size array, e.g. `new ArrayBlockingQueue<>(100)` | Know max size, want bounded, fast |
| `LinkedBlockingQueue` | Linked list, optional bound, default almost unbounded | Default choice, general Producer-Consumer |
| `SynchronousQueue` | Size 0! No storage. Producer must hand directly to Consumer | Handoff, `newCachedThreadPool()` uses this |
| `PriorityBlockingQueue` | Unbounded, orders by priority, not FIFO | Tasks with priority: VIP first |
| `DelayQueue` | Item available only after delay | Scheduled tasks, expiring cache |

**Best Practice:**
- **Always give a bound in production:** `new LinkedBlockingQueue<>(1000)`. Unbounded queue can grow to millions, OOM if producers faster than consumers.
- **Use timeout or `interrupt()` for `put()` / `take()`** — they can block forever if other side dies.
- **Poison Pill for graceful shutdown:**
```java
queue.put("POISON"); // Producer
// Consumer:
while (true) {
    String item = queue.take();
    if (item.equals("POISON")) break;
    process(item);
}
```

**Real-World Uses:**
1. ThreadPool's Work Queue — `ExecutorService` uses BlockingQueue inside to hold waiting tasks.
2. Log Processing — Web threads put logs -> Queue -> One writer thread takes and writes to file. Web threads never block on slow disk.

**BlockingQueue vs Latch / Barrier / Semaphore:**
|  | `BlockingQueue` | `CountDownLatch` | `CyclicBarrier` | `Semaphore` |
| :--- | :--- | :--- | :--- | :--- |
| Passes Data? | **Yes**, passes objects | No, only signals | No, only signals | No, only permits |
| Use for | Producer-Consumer, pass work | Wait for finish | Wait for each other | Limit N inside |
| Reusable? | Yes, forever | No, one-time | Yes | Yes |

> **Before: You wrote wait/notify to build a queue. Now: You use a queue that already does wait/notify for you.**

### Summary — Which Synchronizer To Use? 

| Need | Use |
| :--- | :--- |
| Main waits for workers to finish | `CountDownLatch` |
| All threads wait for each other, then go together, repeat | `CyclicBarrier` |
| Only N threads allowed at same time | `Semaphore` |
| Wait for services to start before accepting requests | `CountDownLatch` |
| Game levels, multi-phase calculation | `CyclicBarrier` |
| Limit DB connections / API rate limit | `Semaphore` |
| Pass work from producer to consumer | `BlockingQueue` |

> **Latch = "Wait for them to finish, then I go."**
> **Barrier = "Wait for each other, then we all go together."**
> **Semaphore = "Only 2 allowed inside, rest wait outside."**
> **BlockingQueue = "Put here, take from there, queue handles waiting."**

---

## 7. Best Practices — The Cheat List (Only Java 5)

### Executors & Future
- Submit tasks, don't create threads manually.
- Always `shutdown()` pool. Use bounded queue.
- Submit all, then `get()` all. Use `get(timeout)`, unwrap `ExecutionException`.

### Locks
- Always `unlock()` in `finally`.
- Use `tryLock(timeout)` when deadlock is possible.
- Use fair lock only when ordering matters. Default unfair is faster.
- Keep critical section tiny, no I/O inside.

### Atomics
- Single variable, low contention -> `Atomic*` + CAS.
- Multi-variable or high contention -> lock.
- Never hand-roll CAS for a simple counter — use `incrementAndGet()`.

### Collections & Synchronizers
- Shared map -> `ConcurrentHashMap`, use atomic `putIfAbsent` / `replace`.
- Read-heavy list -> `CopyOnWriteArrayList` only if writes are rare.
- Pass work -> `BlockingQueue` with bound + timeout.
- Coordinate -> Latch / Barrier / Semaphore, always `countDown()` / `release()` in `finally`, always use timeout.

### Never in New Java 5 Code
- Raw `new Thread()` per request.
- `Runnable` + shared variable + `join()` to get a result — use `Callable`.
- Hand-written `wait/notify` queue — use `BlockingQueue`.
- `Hashtable` / `synchronizedMap` / `Vector` for new shared collections.
- `synchronized` where you need timeout / try-lock / interrupt — use `Lock`.

---

## 8. Tips — 30-Second Answers (Only Java 5)

1.  **"Why Executor over `new Thread()`?"**
    > Reuse, queue, max threads, shutdown, separates task from execution. `new Thread()` per request = OOM.

2.  **"What is the limitation of Java 5 `Future`?"**
    > No callback. You must block with `get()` or poll with `isDone()`. So submit many, do other work, then collect.

3.  **"`synchronized` vs `Lock`?"**
    > `synchronized` = auto, simple, no timeout / try-lock / interrupt / fairness / multiple conditions. `Lock` = manual, verbose, full control. Use `Lock` when you need any of those.

4.  **"Why `unlock()` in `finally`?"**
    > If code throws before `unlock()`, lock stays held forever = deadlock. `finally` guarantees release.

5.  **"CAS vs Lock?"**
    > CAS = optimistic, try + retry, no blocking, no deadlock, fast at low contention, spins at high contention, one variable only. Lock = pessimistic, blocks, can deadlock, better at high contention / multi-variable.

6.  **"What is ABA problem?"**
    > CAS checks value, not history. A -> B -> A looks unchanged to CAS, but history was lost. Fix in Java 5: `AtomicStampedReference` with version stamp.

7.  **"`volatile` vs `AtomicInteger`?"**
    > `volatile` = visibility only, `count++` still races. `AtomicInteger` = visibility + atomicity via CAS.

8.  **"How does `ConcurrentHashMap` work in Java 5?"**
    > Segment locking — 16 segments, each with own lock. `get()` lock-free. Much faster than `Hashtable` which locks whole map.

9.  **"Latch vs Barrier vs Semaphore?"**
    > Latch = one waits for others, one-time. Barrier = all wait for each other, reusable. Semaphore = only N inside, permits.

10. **"Why `BlockingQueue` over manual `wait/notify`?"**
    > It does `wait/notify`, bounds, and blocking correctly inside JDK. Manual version has one-mistake = deadlock. Plus built-in backpressure — producer auto-slows when consumer is slow.

---

## 9. Cheat Sheet — Java 5 Only

### Timeline

| Part | Headline | Verdict in Java 5 |
| :--- | :--- | :--- |
| **Executors** | Pool, queue, shutdown | **Core** |
| **Callable + Future** | Return value + exception | **Core** |
| **Locks** | `ReentrantLock`, `ReadWriteLock`, `Condition` | **Core when you need control** |
| **Atomics + CAS** | Lock-free single variable | **Core for counters / flags** |
| **Concurrent Collections** | Segment lock, `BlockingQueue` | **Core** |
| **Synchronizers** | Latch, Barrier, Semaphore, Queue | **Core** |

### API Quick Reference

| API | What it does | Verdict |
| :--- | :--- | :--- |
| `ExecutorService` | Pool, submit, shutdown | Core |
| `Callable` / `Future` | Task with result | Core, `get(timeout)` |
| `ReentrantLock` | Lock with timeout / try-lock / interrupt | Core when need control |
| `ReadWriteLock` | Many readers, one writer | Core for read-heavy |
| `Condition` | Multiple wait-sets | Core for selective wakeup |
| `AtomicInteger` / `AtomicReference` | CAS, lock-free | Core for single variable |
| `AtomicStampedReference` | CAS + version, fixes ABA | Core for ABA-sensitive refs |
| `ConcurrentHashMap` | Segment-locked map | Core, no null, atomic `putIfAbsent` |
| `CopyOnWriteArrayList` | Copy on write | Read-heavy only |
| `CountDownLatch` | One-time gate | Core, `countDown()` in `finally` |
| `CyclicBarrier` | Reusable meeting point | Core, handle `BrokenBarrierException` |
| `Semaphore` | N permits | Core, `release()` in `finally` |
| `BlockingQueue` | Blocking producer-consumer | Core, bounded + timeout |

### Decision Tree (Only Java 5)

```text
Need to run tasks?
  └─ ExecutorService, not new Thread()

Need a result from a task?
  └─ Callable + Future, submit all then get all with timeout

Need a lock?
  ├─ Simple, short? -> synchronized
  ├─ Timeout / try-lock / interrupt / fairness / multiple conditions? -> ReentrantLock
  └─ Many readers, one writer? -> ReadWriteLock

Need to update one variable?
  ├─ Low contention? -> Atomic* + CAS
  └─ Multi-variable / high contention? -> Lock

Need to share a map?
  └─ ConcurrentHashMap, use putIfAbsent / replace

Need to coordinate?
  ├─ Main waits for workers? -> CountDownLatch
  ├─ All wait for each other, repeat? -> CyclicBarrier
  ├─ Only N inside? -> Semaphore
  └─ Pass work? -> BlockingQueue (bounded)
```

---
