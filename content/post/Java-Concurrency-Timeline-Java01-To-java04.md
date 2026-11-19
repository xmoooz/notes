---
title: "Java Thread & Concurrency Timeline: Java1 to Java4"
date: 2026-11-01T23:30:00Z
lastmod: 2026-11-01T23:30:00Z
draft: false
categories: ["java", "concurrency", "timeline"]
tags: ["java", "concurrency", "timeline", "threads", "threadlocal", "volatile", "nio", "java1", "java2", "java3", "java4"]
series: ["java concurrency timeline"]
author: "xmoooz"
description: "history of Java threads and concurrency from JDK 1.0 through JDK 1.4 — why each feature was added, what to still use, what to never use, and practical tips for each version."
url: 2026/11/18/java-concurrency-timeline-java1-to-java4
aliases:
  - /2026/11/18/java-concurrency-timeline-java1-to-java4/
---

> Part 1 of the Java Concurrency Timeline — JDK 1.0 through 1.4. There was never an official "Java 3" release. JDK 1.2 was rebranded "Java 2", then came J2SE 1.3, J2SE 1.4, then Java 5.

> **How to read this post:** Start with the Global Vision. Each version then follows the same pattern: What / Why / Still Use? / Never Use / Code / Best Practice / Tips. Skim the Cheat Sheet at the end.

---

## 0. Global Vision — The Whole Story in 2 Minutes

Java did not add concurrency as an afterthought. It made threads part of the language in 1996, when C/C++ still treated them as an OS library. That early bet created 30 years of evolution in one direction:

**From manual → managed → declarative.**

| Era | Mental model | How you wrote code |
| :--- | :--- | :--- |
| **1.0 - 1.4** | **Manual** | You create the thread, you lock, you wait, you clean up. No pool, no Future, no timeout. |
| **Java 5** | **Managed** | `ExecutorService`, `Future`, `ReentrantLock`, `ConcurrentHashMap`, `CountDownLatch`. You submit tasks, framework manages threads. |
| **Java 8+** | **Declarative** | `CompletableFuture`, parallel streams, reactive. You describe *what*, not *how* threads run. |
| **Java 21+** | **Cheap** | Virtual Threads. `new Thread()` per task finally becomes viable again, because threads are cheap. |

### The 4 problems every version was trying to solve

1.  **Parallelism:** Do more than one thing at once without freezing UI / server.
2.  **Safety:** Stop races, lost updates, stale reads.
3.  **Coordination:** Make threads wait for each other, signal, cancel.
4.  **Scale:** Handle 10k clients without 10k OS threads.

JDK 1.0 solved 1 badly. 1.2 solved 2 partially. 1.3 solved scheduling. 1.4 solved 4 for I/O only. Java 5 finally solved 1-3 properly.

### The one sentence to remember for each version

- **1.0:** Threads exist, but you own all the pain.
- **1.1:** Cancellation finally has a protocol (`interrupt()`), UI gets its own thread (EDT).
- **1.2:** Per-thread isolation (`ThreadLocal`) + thread-safe collections, but both have traps.
- **1.3:** One background scheduler (`Timer`) + graceful shutdown (hooks) + HotSpot makes `synchronized` cheap.
- **1.4:** One thread can watch 10k sockets (`NIO` / Reactor). I/O scales, compute still does not.

### What is still relevant in 2026? What is dead?

**Still use every day (as concepts):** `Thread`, `Runnable`, `synchronized`, `wait/notify`, `volatile`, `interrupt()`, `ThreadLocal` (carefully), NIO/Reactor mental model.

**Use, but via modern wrapper:** `Timer` -> `ScheduledExecutorService`. Synchronized collections -> `ConcurrentHashMap` / `CopyOnWriteArrayList`. Raw `Thread` -> `ExecutorService` / Virtual Threads.

**Never use in new code:** `Thread.stop()`, `suspend()`, `resume()`, `ThreadGroup` as API, priorities for correctness, `notify()` when `notifyAll()` is safer, `ThreadLocal` without `remove()` in a pool.

---

## 1. Java 1.0 (1996) — Birth of Threads

### What
`Thread`, `Runnable`, `synchronized`, `wait()`, `notify()`, `notifyAll()`, `sleep()`, `join()`, `yield()`, `interrupt()`, `ThreadGroup`, Daemon Threads, Priorities (1-10).

### Why
In 1996 everything was single-threaded. A GUI froze while downloading. A server handled one client at a time. Java made threads a language feature, not a library, because it was built for network, UI, and servers.

### Benefit
- One process, many tasks.
- UI stays responsive while background work runs.
- Server handles many clients.

### Downside
- No pool. `new Thread()` per task = ~1MB stack + OS thread cost.
- `run()` returns `void` — no return value, no exception propagation.
- `synchronized` is the only lock — no timeout, no try-lock, no fairness.
- Easy deadlock, race, missed `notify()`.

### Status in 2026
| API | Verdict |
| :--- | :--- |
| `Thread`, `Runnable`, `synchronized`, `wait/notify`, `sleep`, `join`, `interrupt` | **STILL CORE.** You must know them. |
| `Thread.stop()`, `suspend()`, `resume()` | **DEPRECATED / DO NOT USE.** Unsafe. `stop()` releases locks mid-update. `suspend()` can deadlock. Removed / throws in modern JDKs. |
| `ThreadGroup` | **LEGACY.** Do not build APIs on it. |
| Priorities (1-10), `yield()` | **HINT ONLY.** Never rely on them for correctness. |

---

### 1.1 Creating a Thread — `Thread` vs `Runnable`

**Prefer `Runnable`. Always.**

```java
// BAD: wastes inheritance, couples task to thread
class MyThread extends Thread {
    public void run() { System.out.println("Running"); }
}
new MyThread().start();

// GOOD: task is separate from execution
class MyTask implements Runnable {
    public void run() { System.out.println("Running"); }
}
new Thread(new MyTask()).start();

// MODERN (same idea, shorter):
new Thread(() -> System.out.println("Running")).start();
```

**Why `Runnable` wins:**
- Java has no multiple inheritance — extending `Thread` blocks extending anything else.
- Same `Runnable` can be reused by many threads, a pool, or a virtual thread.
- Separation: `Runnable` = *what* to do, `Thread` / `Executor` = *how* to run it.

**Best Practice:** Never call `run()` directly. `run()` = normal method call in current thread. `start()` = ask JVM to create a new thread, then JVM calls `run()` there. Call `start()` only once — second call throws `IllegalThreadStateException`.

```java
// Common trap:
new MyTask().run();              // runs in MAIN thread, no new thread
new Thread(new MyTask()).start(); // real new thread
```

### 1.2 Lifecycle — The 6 States

`Thread.State` enum itself arrived in Java 5, but the states are as old as `Thread`:

```text
NEW --start()--> RUNNABLE --run() returns--> TERMINATED
                    |  ^
        monitor     |  |  lock acquired
        contention  v  |
                  BLOCKED
                    |
     wait()/join()/park()          sleep(ms)/wait(ms)/join(ms)
                    v                          v
                 WAITING                  TIMED_WAITING
```

| State | Meaning | Entered by |
| :--- | :--- | :--- |
| `NEW` | Created, not started | `new Thread(...)` |
| `RUNNABLE` | Executing or ready, includes blocking I/O | `start()` |
| `BLOCKED` | Waiting for a `synchronized` monitor | Another thread holds lock |
| `WAITING` | Waiting forever | `wait()`, `join()`, `LockSupport.park()` |
| `TIMED_WAITING` | Waiting with timeout | `sleep(ms)`, `wait(ms)`, `join(ms)` |
| `TERMINATED` | Done | `run()` returned / threw |

**Tips:**
- `RUNNABLE` does not mean "on CPU now". It includes threads blocked in I/O.
- `BLOCKED` = monitor contention only. A thread in `wait()` is `WAITING`, not `BLOCKED`.
- `getState()` is a snapshot — it can change immediately after you read it.

### 1.3 Race Conditions + `synchronized` — The Core

`count++` is NOT atomic. It is read + add + write. Two threads can interleave and lose an update.

```java
class Counter {
    private int count = 0;
    void increment() { count++; } // RACE
}
```

**Fix with monitor:**

```java
class Counter {
    private int count = 0;
    // Option A: method lock (locks `this`)
    synchronized void increment() { count++; }

    // Option B (BEST PRACTICE): private lock object
    private final Object lock = new Object();
    void incrementSafe() {
        synchronized (lock) { count++; }
    }
    int get() {
        synchronized (lock) { return count; }
    }
}
```

**What `synchronized` actually gives you:**
1.  **Mutual exclusion** — one thread at a time.
2.  **Visibility** — writes before unlock are visible to next locker (happens-before).
3.  **Reentrancy** — same thread can re-enter a lock it already holds.

**Best Practice — How to use `synchronized` correctly:**

- **Lock on a `private final Object`, never on `this`, a `String` literal, or boxed primitive.** Outside code can lock the same object and deadlock / interfere with you.
- **Keep critical section tiny.** No I/O, no network, no callbacks while holding a lock.
- **Lock both read and write.** Synchronizing only `increment()` but reading `count` unsynchronized can still see a stale value.
- **Instance vs static are different monitors.** `synchronized` instance method locks `this`, `static` locks the `Class` object. They do not exclude each other.
- **Synchronize the whole invariant, not one field at a time.** If two fields must stay consistent together (e.g. `balance` + `lastUpdated`, `x` + `y`), guard both with the same lock in every method that reads or writes them.
- **Do not call foreign / overridable code inside a lock.** A callback can call back into your class, try to take the same or another lock, or block forever — and you hold the lock while it does.
- **Do not expose the lock.** If you return the lock object, synchronize on a public field, or lock on a collection that callers can access, callers can block you. Keep the lock private.
- **In modern code:** Prefer `ConcurrentHashMap`, `AtomicInteger`, or `ReentrantLock` (Java 5+) for compound logic. Use `synchronized` for simple, short, single-monitor invariants.
- **One lock per independent state.** Do not use one big lock for unrelated fields — it serializes unrelated work. Split state so each lock guards only the fields it must protect.

**Real sample — safe counter + test:**

```java
public class SafeCounterDemo {
    static class Counter {
        private int count = 0;
        private final Object lock = new Object();
        void increment() { synchronized (lock) { count++; } }
        int get() { synchronized (lock) { return count; } }
    }
    public static void main(String[] args) throws Exception {
        Counter c = new Counter();
        Runnable task = () -> { for (int i = 0; i < 100_000; i++) c.increment(); };
        Thread t1 = new Thread(task), t2 = new Thread(task);
        t1.start(); t2.start(); t1.join(); t2.join();
        System.out.println(c.get()); // always 200000
    }
}
```

**Real sample — `synchronized` best practice checklist in code:**

```java
public class BankAccount {
    // 1. Private final lock, never `this`
    private final Object lock = new Object();
    private int balance;
    private long lastUpdated;

    // 2. Same lock guards the whole invariant (balance + lastUpdated)
    public void deposit(int amount) {
        synchronized (lock) {
            balance += amount;
            lastUpdated = System.currentTimeMillis();
            // 3. No I/O, no callback, no foreign code inside the lock
        }
    }

    public int getBalance() {
        // 4. Read also locked — otherwise stale read
        synchronized (lock) {
            return balance;
        }
    }

    // 5. Do slow work OUTSIDE the lock, then take lock only to update
    public void depositWithLogging(int amount) {
        String logLine = "deposit " + amount; // outside
        synchronized (lock) {
            balance += amount;
            lastUpdated = System.currentTimeMillis();
        }
        System.out.println(logLine); // outside
    }
}
```

**Tips — `synchronized` mistakes to avoid:**
- Locking on `this` in a public class: any caller can do `synchronized (account) { ... }` and block your methods.
- Locking on a `String` literal or `Integer`: literals are interned / cached, so unrelated code can share the same lock.
- Synchronizing the method but reading the field directly elsewhere: the unsynchronized read can see a stale value.
- Holding a lock while calling `listener.onChange()`: the listener may call back and deadlock.

### 1.4 `wait()` / `notifyAll()` — Coordination Rules

`wait()` releases the monitor and parks the thread. `notifyAll()` wakes waiters, but they must reacquire the monitor before returning.

**Best Practice:**
- **Always wait in `while`, never `if`.** Two reasons: spurious wakeups (allowed by JVM), and another thread may consume the condition before you reacquire the lock.
- **Prefer `notifyAll()` over `notify()`.** If waiters wait on different conditions, `notify()` can wake the wrong one, which goes back to sleep while the right one sleeps forever = lost wakeup.
- `wait()/notify()` must be called while holding the monitor, else `IllegalMonitorStateException`.
- `wait()` is `Object` method, `sleep()` is `Thread` method.

**`sleep()` vs `wait()`:**

|  | `sleep()` | `wait()` |
| :--- | :--- | :--- |
| Releases lock? | No | Yes |
| Needs `synchronized`? | No | Yes |
| Wakes by? | Timeout | `notify()` / `notifyAll()` |
| Belongs to? | `Thread` | `Object` |

### 1.5 Deadlock — The Classic Problem

Two threads acquire the same two locks in opposite order.

```java
void transfer(Account from, Account to) {
    synchronized (from.lock) {
        synchronized (to.lock) { /* move money */ }
    }
}
// Thread A: transfer(a, b) -> locks a, then b
// Thread B: transfer(b, a) -> locks b, then a => DEADLOCK
```

**Fixes (in order of preference):**
1.  **Global lock ordering:** Sort by stable ID (e.g. `accountId`) and always lock in that order.
2.  **One lock at a time:** Refactor to avoid nested locks.
3.  **Timed `tryLock`:** Only from Java 5 (`ReentrantLock.tryLock(timeout)`). Not possible with `synchronized`.

**Real sample — fixed ordering:**

```java
void transferSafe(Account a, Account b, int amount) {
    Account first = a.id < b.id ? a : b;
    Account second = a.id < b.id ? b : a;
    synchronized (first.lock) {
        synchronized (second.lock) { /* move money */ }
    }
}
```

**Tips:** Diagnose with `jstack <pid>` — it prints `Found one Java-level deadlock` with both stacks. A `synchronized` deadlock cannot be broken from code in this era — no timeout, no interrupt. Prevention is the only option.

### 1.6 `interrupt()` — Cooperative Cancellation

`interrupt()` does NOT stop a thread. It sets a flag. If the thread is in `sleep/wait/join`, that call throws `InterruptedException` and clears the flag.

```java
// Pattern 1: loop checks flag
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // do one chunk of work
    }
    System.out.println("clean exit");
});
worker.start();
worker.interrupt(); // request stop, not force stop

// Pattern 2: handle InterruptedException correctly
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // RESTORE flag, never swallow
    return; // clean up and exit
}
```

**Best Practice:**
- `Thread.interrupted()` checks **and clears** flag. `isInterrupted()` leaves it set. Know the difference.
- **Never swallow `InterruptedException` with empty catch.** Restore flag or rethrow. Otherwise callers can no longer tell the thread was asked to stop.
- Interrupt cannot stop a CPU-bound loop unless the loop checks the flag.
- This is why `stop()` was deprecated — cooperative cancellation is safe, forced stop is not.

### 1.7 Why `new Thread()` Per Task Does Not Scale

- A platform thread is 1:1 with an OS thread. Default stack ~1MB (`-Xss`). 10k threads = ~10GB stack reservation + heavy context switching.
- No pool, no queue, no return value, no `Future`. All hand-built until Java 5.

**Tips:** In Java 5+ use `ExecutorService`. In Java 21+ use Virtual Threads — `new Thread()` per task finally becomes cheap again, but for a different reason (virtual, not OS threads).

---

## 2. Java 1.1 (1997) — The Quiet Release

### What
No new locks or pools. Added:
- **Working `interrupt()` protocol** — 1.0 had the method, 1.1 made it actually interrupt sleeping threads reliably.
- **JIT compiler** — hot code compiled to native, thread-heavy code got faster.
- **Inner classes** — anonymous `Runnable` becomes idiomatic: `new Thread(new Runnable() { ... })`.
- **AWT Event Dispatch Thread (EDT)** — UI events delivered on one single thread.
- **Reflection, JavaBeans, RMI** — all blocking APIs, pushing devs toward worker threads.

### Why It Mattered
Cancellation finally had a standard signal. The EDT created a new discipline: never block UI thread, do work on background thread, post result back to EDT.

### Downside
Still no pool, no return value, no timeout. Interrupt is cooperative — code that ignores flag cannot be stopped. `invokeAndWait` from EDT deadlocks.

### Status in 2026
- `interrupt()` protocol: **STILL CORE.**
- Inner classes: replaced by lambdas (Java 8), same idea.
- EDT rule: **STILL CORE** for Swing / JavaFX / Android UI thread. Modern form: "Never block the UI thread."

**Tips:**
- EDT rule in one line: UI thread paints and handles events only. Any slow work (I/O, DB, network) goes to a background thread, then the result is posted back to EDT with `SwingUtilities.invokeLater(...)`.
- Never call `invokeAndWait()` from the EDT itself — it waits for itself and deadlocks.
- If someone says "`ThreadLocal` is from 1.1" — they are wrong. It is 1.2 (`@since 1.2`). This is a common trivia trap.

---

## 3. Java 1.2 (1998) — Java 2

### What
- **`ThreadLocal` + `InheritableThreadLocal`** (`@since 1.2`).
- **Synchronized collection wrappers** — `Collections.synchronizedList/Set/Map/...`.
- **`stop()`, `suspend()`, `resume()` deprecated.**
- **`java.lang.ref`** — weak/soft/phantom refs, the machinery behind `ThreadLocalMap` weak keys.

### Why
1.  Per-thread state: servers needed request/user/format state without passing it through every method.
2.  Collections Framework was not thread-safe — wrappers were a quick fix.
3.  Unsafe thread control had to go.

### Benefit / Downside
- `ThreadLocal` gave isolation without locks, but leaks in pools if `remove()` forgotten.
- Wrappers made collections usable, but lock whole collection + iteration still needs external lock + compound actions still race.

---

### 3.1 `ThreadLocal` — Deep Dive (Most Misunderstood API in This Era)

**What:** Not a thread. A container that gives a **different value per thread**.

```text
Thread T1 -> [ ThreadLocalMap: { local1 -> 100 } ]
Thread T2 -> [ ThreadLocalMap: { local1 -> 200 } ]
```

Every `Thread` has a hidden `ThreadLocalMap`. `set()` writes to current thread's map. `get()` reads from current thread's map. No synchronization needed — threads never touch each other's map.

**Real sample — the classic use cases:**

```java
// 1. Non-thread-safe formatter per thread (pre-Java 8 fix for SimpleDateFormat)
static final ThreadLocal<SimpleDateFormat> SDF =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
String s = SDF.get().format(new Date());

// 2. Request / user context in a web server
static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();
// In filter: CURRENT_USER.set(user);
// Anywhere in request: CURRENT_USER.get()
// In finally: CURRENT_USER.remove();

// 3. Per-thread counter
static final ThreadLocal<Integer> COUNTER = ThreadLocal.withInitial(() -> 0);
```

**`InheritableThreadLocal`:** Child thread inherits a copy of parent's value at creation time. Normal `ThreadLocal` does NOT.

```java
static InheritableThreadLocal<String> local = new InheritableThreadLocal<>();
local.set("ParentValue");
new Thread(() -> System.out.println(local.get())).start(); // "ParentValue"
```

**Best Practice — How to use `ThreadLocal` safely:**

1.  **Always `remove()` in `finally` when using a pool / web server.** Threads are reused, they never die. Forgetting `remove()` = next request sees previous user's data + memory leak.
2.  **Prefer `ThreadLocal.withInitial(...)` over anonymous subclass.** Cleaner, avoids subclassing.
3.  **Use `static final` field.** `ThreadLocal` itself is meant to be shared as the key; the value is per-thread.
4.  **Do NOT use `ThreadLocal` to share data.** If T1 sets "Hello", T2 `get()` returns `null`, not "Hello". For sharing, use `volatile` / `ConcurrentHashMap`.
5.  **In modern code:** Prefer passing context explicitly (method param, request object) or using scoped values / virtual-thread-friendly context (Java 21+ `ScopedValue`). Use `ThreadLocal` only for truly per-thread, framework-level state (e.g. transaction, formatter).

**Real sample — leak vs fix:**

```java
// BAD - in ExecutorService / Tomcat
executor.execute(() -> {
    CURRENT_USER.set(user);
    // forget remove -> next request on same pooled thread sees OLD user
});

// GOOD
executor.execute(() -> {
    try {
        CURRENT_USER.set(user);
        // do work
    } finally {
        CURRENT_USER.remove(); // MUST clean
    }
});
```

**Why does it leak?** Thread holds strong ref to `ThreadLocalMap`, map holds value strongly. Key is `WeakReference`, so key can be GC'd, but value stays. In a pool, thread never dies, value never GC'd.

**Tips:** "What happens if you forget `remove()` in a pool?" -> Answer: stale data leak (security bug) + memory leak.

### 3.2 `volatile` vs `ThreadLocal` — Opposite Problems

> `volatile` = **ONE value, shared by ALL threads, everyone sees latest.**
> `ThreadLocal` = **MANY values, one per thread, nobody sees other's.**

|  | `volatile` | `ThreadLocal` |
| :--- | :--- | :--- |
| What? | Keyword for a variable | Class, container |
| Copies? | 1 copy for all | 1 copy per thread |
| Visibility? | Change visible to all | Change invisible to others |
| Atomicity? | **NO** — `count++` still races | No race, because no sharing |
| Use for? | Flag, status, stop signal | Per-thread data — user, formatter, connection |
| Leak? | No | Yes, if forget `remove()` in pool |

**Real sample — see the difference:**

```java
static volatile int sharedCount = 0;
static ThreadLocal<Integer> localCount = ThreadLocal.withInitial(() -> 0);

Runnable task = () -> {
    sharedCount++; // race! but shared
    localCount.set(localCount.get() + 1); // isolated
    System.out.println(Thread.currentThread().getName() +
        " | volatile=" + sharedCount + " | threadLocal=" + localCount.get());
};
new Thread(task, "T1").start(); // local=1
new Thread(task, "T2").start(); // local=1 again, NOT 2
```

**When to use which:**
- `volatile`: `volatile boolean running = true;` // stop flag. One writer, many readers, simple read/write. Never `volatile count++`.
- `ThreadLocal`: `ThreadLocal<SimpleDateFormat>`, `ThreadLocal<User>`. Data must NOT be shared.

**Mistakes to never make:**
- `volatile int count; count++;` — WRONG. Still 3 steps. Fix: `AtomicInteger` or `synchronized`.
- `ThreadLocal` to share data — WRONG. Use `volatile` / `ConcurrentHashMap`.

### 3.3 `volatile` — What It Really Guarantees

Keyword exists since 1.0. What changed: Before Java 5 (JSR-133), JMM did not guarantee visibility/ordering well enough to make double-checked locking safe. Java 5 redefined `volatile` with happens-before semantics.

```java
class StopFlag {
    private volatile boolean running = true;
    void stop() { running = false; } // visible immediately
    void work() { while (running) { /* chunk */ } }
}
```

Guarantees: **Visibility** + **Ordering** (no reordering across volatile access) + **Atomicity of single read/write**. Does NOT give mutual exclusion. Does NOT make compound actions atomic.

**Best Practice:** Use `volatile` for one independent flag / state field, not for invariants spanning multiple fields.

### 3.4 Synchronized Collections — And Their Traps

Before 1.2, `Vector` / `Hashtable` synchronized every method. 1.2 replaced them with unsynchronized `ArrayList` / `HashMap` + optional wrappers.

```java
List<String> list = Collections.synchronizedList(new ArrayList<>());
list.add("A");

// Iteration MUST hold same lock — wrapper does NOT do it for you
synchronized (list) {
    for (String s : list) System.out.println(s);
}
```

**Traps:**
- Iteration without `synchronized(list)` -> `ConcurrentModificationException`.
- Compound actions still race: `if (!list.contains("A")) list.add("A");`
- Every op takes whole collection lock — poor throughput.

**Modern replacement:** `ConcurrentHashMap` (reads don't block), `CopyOnWriteArrayList` (read-heavy), `BlockingQueue` (for queueing work). `synchronizedList` is legacy in new code.

---

## 4. Java 1.3 (2000) — HotSpot, Timers, Shutdown Hooks

### What
- **HotSpot becomes default JVM** — adaptive optimization, faster `synchronized` / thread creation.
- **`java.util.Timer` + `TimerTask`** — built-in scheduler on one background thread.
- **`Runtime.addShutdownHook(Thread)`** — cleanup on JVM shutdown.
- Still no pool, no `Future`, no `java.util.concurrent`.

### Why
Periodic work before 1.3 = hand-written thread + `sleep` loop. Shutdown needed a portable way to flush buffers on Ctrl+C / `System.exit`.

### Downside
`Timer` uses **one** thread: one slow task delays all others, one uncaught exception kills timer thread and cancels whole schedule. Schedules drift after stall. Shutdown hooks run concurrently, in unspecified order, must be fast.

### Status in 2026
| API | Verdict |
| :--- | :--- |
| `Timer` / `TimerTask` | **LEGACY.** Use `ScheduledExecutorService` (Java 5). |
| `addShutdownHook` | **STILL USEFUL.** For flushing, closing resources. Keep it fast, idempotent, no dependency on other services. |
| HotSpot | Default, invisible, still core. |

**Real sample — old vs modern:**

```java
// LEGACY - 1.3 way (do not use in new code)
Timer timer = new Timer(true); // daemon
timer.scheduleAtFixedRate(new TimerTask() {
    public void run() { System.out.println("tick"); }
}, 0, 1000);

// MODERN - Java 5+ way (use this)
ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
scheduler.scheduleAtFixedRate(() -> System.out.println("tick"), 0, 1, TimeUnit.SECONDS);

// Shutdown hook - still valid
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("flushing buffers...");
    scheduler.shutdown();
}));
```

**Tips:** "Why is `Timer` legacy?" -> One thread, no exception isolation, no pool. One failing task cancels schedule. `ScheduledExecutorService` fixes all three.

---

## 5. Java 1.4 (2002) — NIO + Non-Blocking I/O

### What
`java.nio`, `Channel`, `Buffer`, `Selector`, `ServerSocketChannel`.

### Why
Old I/O was blocking: 1 thread = 1 client. 10k clients = 10k threads = OOM + context-switch storm. This is the C10K problem. NIO lets ONE thread watch MANY connections and only process ready ones.

### Benefit
Scales to thousands of connections with few threads. Less memory, less thread cost. Foundation for Netty, Tomcat NIO.

### Downside
Very hard API — `Selector`, `SelectionKey`, `ByteBuffer` are low-level and error-prone. Still need to handle threading yourself for actual work. Did not solve general concurrency, only I/O.

### Status in 2026
- Raw NIO: **LOW-LEVEL, RARELY WRITE DIRECTLY.** Use Netty / frameworks.
- Reactor mental model: **STILL CORE.** Every modern server (Netty, Node.js, Tomcat NIO) is Reactor + Worker Pool.
- `java.nio` APIs: still underlying, but prefer `java.nio.file` / async channels / frameworks.

---

### 5.1 NIO — How ONE Thread Watches MANY

**Old problem — Blocking I/O:**

```java
Socket client = serverSocket.accept(); // BLOCKS
String line = in.readLine(); // BLOCKS
// Client connects but says nothing -> thread stuck, eating 1MB stack, doing nothing
```

**NIO trick — Don't wait, just check:**

> Old: "Wait for THIS ONE client to send data"
> NIO: "Hey OS, tell me WHICH of my 10,000 clients is ready"

That is what `Selector` does, via `epoll` (Linux) / `kqueue` (Mac).

**3 parts:**

| Part | Code | Job |
| :--- | :--- | :--- |
| **Channel** | `ServerSocketChannel`, `SocketChannel` | Non-blocking socket |
| **Selector** | `Selector.open()` | The watcher — one thread owns one |
| **SelectionKey** | `channel.register(selector, OP_ACCEPT / OP_READ)` | "Watch this channel for this event" |

**The loop:**

```java
while (true) {
    selector.select(); // block until AT LEAST ONE channel is ready
    Set<SelectionKey> ready = selector.selectedKeys(); // only ready ones
    for (SelectionKey key : ready) {
        if (key.isAcceptable()) { /* new client */ }
        if (key.isReadable()) { /* read now, won't block */ }
    }
    ready.clear();
}
```

```text
OLD: [T1 -> C1 stuck] [T2 -> C2 stuck] ... 1000 threads
NIO: [1 Thread + Selector] -> watches -> [C1 idle, C2 READY, C3 idle, C4 READY]
                                 Only processes C2, C4
```

### 5.2 Reactor Pattern — The Idea Behind NIO

**NIO = Reactor Pattern:**

| Reactor Role | NIO Code | Job |
| :--- | :--- | :--- |
| **Reactor** | `Selector` + Event Loop | Waits, dispatches |
| **Event** | `SelectionKey` (`OP_ACCEPT`, `OP_READ`, `OP_WRITE`) | Something happened |
| **Handler** | Your `if (key.isReadable())` code | What to do |

Flow: Client event -> OS / epoll -> `Selector.select()` wakes -> Dispatcher -> Handler -> Back to waiting.

**What is Reactor Pattern, in one paragraph:**
One thread waits for events from the OS via `Selector`, then dispatches each event to its handler. No thread per connection. The Reactor thread only does fast I/O — accept, read, write. It never does slow work.

**Full minimal Reactor:**

```java
public class ReactorDemo {
    public static void main(String[] args) throws Exception {
        Selector selector = Selector.open();
        ServerSocketChannel server = ServerSocketChannel.open();
        server.bind(new InetSocketAddress(8080));
        server.configureBlocking(false);
        server.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            selector.select();
            for (SelectionKey key : selector.selectedKeys()) {
                if (key.isAcceptable()) {
                    SocketChannel client = ((ServerSocketChannel) key.channel()).accept();
                    client.configureBlocking(false);
                    client.register(selector, SelectionKey.OP_READ);
                    System.out.println("New client");
                } else if (key.isReadable()) {
                    SocketChannel client = (SocketChannel) key.channel();
                    ByteBuffer buf = ByteBuffer.allocate(1024);
                    int read = client.read(buf);
                    if (read == -1) client.close();
                    else System.out.println("Got: " + new String(buf.array(), 0, read));
                }
            }
            selector.selectedKeys().clear();
        }
    }
}
```

**Best Practice — Reactor Must Never Do Slow Work:**

```java
if (key.isReadable()) {
    // BAD: Reactor stuck, other 9999 clients wait!
    database.query(); // DON'T
}
```

**Real-world fix — Reactor + Worker Pool:**
- Reactor thread (1) only does: accept + read + write — fast I/O.
- Worker threads (`ExecutorService`) do: DB, business logic, heavy work.
- This is what Netty, Tomcat NIO, Node.js do.

**Tips:** "What is Reactor Pattern?" -> One thread waits for events from OS via Selector, then dispatches each event to its handler. No thread per connection. Production = Reactor + Worker Pool.

**Reactor vs Thread-Per-Connection:**

|  | Thread-Per-Connection | Reactor |
| :--- | :--- | :--- |
| Threads | 1 per client | 1 Reactor + small worker pool |
| Idle client cost | Full thread + stack, blocked | Almost zero, just a registered key |
| Scales to | Hundreds | Thousands / tens of thousands |
| Code | Simple, blocking | Harder, event-driven |
| Risk | OOM + context switching | Reactor blocked by slow work |

---

## 6. Best Practices — The Cheat List

### Threads & Locks
- Use `Runnable` / `ExecutorService`, not `extends Thread`.
- Lock on `private final Object lock`, never `this` / `String` / boxed primitive.
- Keep critical section tiny, no I/O inside lock.
- Synchronize both reads and writes of shared state.
- Synchronize the whole invariant, not one field at a time.
- Do not call foreign / overridable code while holding a lock.
- Prefer `ConcurrentHashMap`, `AtomicInteger`, `ReentrantLock` over raw `synchronized` for compound logic.

### Coordination
- Wait in `while`, not `if`. Prefer `notifyAll()` over `notify()`.
- Never swallow `InterruptedException` — restore flag or exit.
- Use global lock ordering to prevent deadlock. Diagnose with `jstack`.

### `volatile` / `ThreadLocal`
- `volatile` for flags only. Never `volatile count++`.
- `ThreadLocal` for isolation only. Always `remove()` in `finally` in pools.
- Never use `ThreadLocal` to share data. Never use `volatile` for atomicity.

### Scheduling & I/O
- Never use `Timer`. Use `ScheduledExecutorService`.
- Never do slow work on Reactor / Selector thread. Offload to worker pool.
- Shutdown hooks: fast, idempotent, no dependency on other services.

### Deprecated / Legacy — Never in New Code
- `Thread.stop()`, `suspend()`, `resume()` — unsafe, deprecated.
- `ThreadGroup` as API — legacy.
- Priorities / `yield()` for correctness — hint only.
- `Collections.synchronizedList` for new code — use concurrent collections.
- Raw `new Thread()` per request — use pool / virtual threads.

---

## 7. Tips — 30-Second Answers

Use these as short, practical answers. Each proves you have seen production bugs, not just docs.

1.  **"Why is `count++` not thread-safe even with `volatile`?"**
    > `volatile` gives visibility, not atomicity. `count++` is read + add + write. Two threads can read same value and lose update. Need `synchronized` or `AtomicInteger`.

2.  **"Why `while`, not `if`, for `wait()`?"**
    > Spurious wakeups are allowed, and another thread may consume condition before I reacquire lock. Always re-check.

3.  **"Why `notifyAll()` over `notify()`?"**
    > If waiters wait on different conditions, `notify()` can wake wrong waiter, which goes back to sleep while right waiter sleeps forever. Lost wakeup.

4.  **"`ThreadLocal` memory leak?"**
    > In a pool, thread never dies, so value never GC'd. Key is weak, value is strong. Next request sees old user's data. Fix: `remove()` in `finally`.

5.  **"Why was `stop()` deprecated?"**
    > It releases monitors while object is half-updated, leaving corrupted state. Cooperative `interrupt()` + flag is safe.

6.  **"Why is `Timer` legacy?"**
    > One thread, no exception isolation. One failing task kills schedule and delays all others. `ScheduledExecutorService` fixes it.

7.  **"What is Reactor?"**
    > One thread + Selector waits for OS events, dispatches to handlers. No thread per connection. Production adds Worker Pool for slow work — Reactor only does fast I/O.

8.  **"Why did 1.0 not scale?"**
    > 1:1 OS thread, ~1MB stack, no pool, no Future. 10k threads = OOM. NIO solved I/O scale, Java 5 solved task scale with pools, Java 21 solves it again with virtual threads.

9.  **"`synchronized` vs `volatile`?"**
    > `synchronized` = mutual exclusion + visibility. `volatile` = visibility + ordering only, no exclusion. Use `synchronized` for compound, `volatile` for single flag.

10. **"When would you still use `synchronized` in 2026?"**
    > For simple, short, single-monitor invariants. For anything compound, timeout, or fairness — `ReentrantLock` / concurrent collections.

---

## 8. Cheat Sheet — Java 1.0 to 1.4

### Timeline

| Version | Year | Headline | Still Use? |
| :--- | :--- | :--- | :--- |
| **1.0** | 1996 | `Thread`, `Runnable`, `synchronized`, `wait/notify` | **Yes — core** |
| **1.1** | 1997 | Working `interrupt()`, JIT, EDT | **Yes — interrupt + EDT rule** |
| **1.2** | 1998 | `ThreadLocal`, synchronized collections, deprecate `stop()` | **Yes — ThreadLocal carefully** |
| **1.3** | 2000 | HotSpot, `Timer`, shutdown hooks | **Hooks yes, Timer no** |
| **1.4** | 2002 | NIO, `Selector`, Reactor | **Model yes, raw NIO rarely** |

### API Quick Reference

| API | What it does | Verdict |
| :--- | :--- | :--- |
| `Thread` / `Runnable` | Task + execution | Use `Runnable`, prefer pool |
| `synchronized` | Monitor lock + visibility | Core, keep small, private lock |
| `wait()` / `notifyAll()` | Coordination | Core, `while` + `notifyAll()` |
| `sleep()` / `join()` | Pause / wait for thread | Core |
| `interrupt()` | Cooperative cancel | Core, restore flag |
| `volatile` | Visibility + ordering | Flag only, not atomic |
| `ThreadLocal` | Per-thread isolation | Use + `remove()` in `finally` |
| `InheritableThreadLocal` | Parent -> child copy | Rare, context propagation |
| `Collections.synchronized*` | Wrapped collections | Legacy, use concurrent |
| `Timer` / `TimerTask` | Scheduler | Legacy, use `ScheduledExecutorService` |
| `addShutdownHook` | Cleanup on exit | Still useful |
| `Selector` / `Channel` | NIO Reactor | Model core, use framework |
| `Thread.stop/suspend/resume` | Force stop | **NEVER — deprecated** |
| `ThreadGroup` / Priorities | Grouping / hint | Legacy / hint only |

### Decision Tree

```text
Need to share data?
  ├─ One flag, one writer? -> volatile
  ├─ Compound / multi-field? -> synchronized (simple) / ReentrantLock / Atomic*
  └─ High contention collection? -> ConcurrentHashMap / CopyOnWriteArrayList

Need per-thread data?
  └─ ThreadLocal + remove() in finally

Need to wait / signal?
  └─ wait() in while + notifyAll()

Need to cancel?
  └─ interrupt() + flag check, never stop()

Need periodic task?
  └─ ScheduledExecutorService, never Timer

Need 10k connections?
  └─ Reactor (Selector) + Worker Pool, never thread-per-connection
```
