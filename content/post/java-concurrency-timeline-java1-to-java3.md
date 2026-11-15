---
title: "Java Thread & Concurrency Timeline: Java 1 to Java 3"
date: 2026-11-14T20:30:00Z
draft: false
categories: ["java", "concurrency"]
tags: ["java", "concurrency", "threads", "threadlocal", "volatile", "java1", "java3"]
series: ["java concurrency timeline"]
author: "xmoooz"
description: "Complete history of Java threads and concurrency from JDK 1.0 through JDK 1.3 — why each feature was added, benefits, downsides."
url: 2026/11/14/java-concurrency-timeline-java1-to-java3
---

> Complete History — Why it was added, Benefits, Downsides, Deep-Dives.
> Part 1 of the Java concurrency timeline — JDK 1.0 through 1.3 (there was no official "Java 3" release)

## How To Use This File

For each version:
- **What:** What was introduced
- **Why:** What problem existed before
- **Benefit:** What it solved
- **Downside:** What was still bad / interview trap
- **Code:** Minimal working sample
- **Interview:** What interviewers actually ask

> Note on code: samples use modern syntax (lambdas, diamond operator) for brevity even when describing old APIs. The API and its semantics are what the version section is about.

---

## Java 1.0 (1996) — Birth of Threads

### What
`Thread`, `Runnable`, `synchronized`, `wait()`, `notify()`, `notifyAll()`, `sleep()`, `join()`, `yield()`, `interrupt()`, `ThreadGroup`, Daemon Threads, Priorities (1-10).

### Why It Was Added
In 1996 programs were single-threaded. A GUI froze while downloading a file. A server could handle only one client at a time. Java wanted to be different — built for network, UI, and servers from day 1. So threads were made a core language feature, not a library add-on like in C/C++.

### Benefit
- One process, many tasks in parallel.
- UI stays responsive while background work runs.
- Server can handle many clients.

<!--more-->

### Downside / Limitation
- Very low-level. Developer had to manage everything manually.
- No thread pool. Creating `new Thread()` for every task was expensive (1 MB stack per thread).
- No return value from a thread. `run()` returns `void`.
- `synchronized` was the only lock — no timeout, no try-lock, no fairness.
- Easy to create deadlock, race condition, missed `notify()` bugs.
- `Thread.stop()`, `suspend()`, `resume()` existed but were unsafe — later deprecated.

---

### 1.0.1 Creating a Thread — Extend Thread vs Implement Runnable [INTERVIEW CORE]

#### Base Is Thread

You must extend `Thread` class and write `run()`, then use constructor to init it and then call `start()`.

```java
class MyThread extends Thread {
    public void run() {
        System.out.println("Running");
    }
}
new MyThread().start();
```

But this is not a good way because:
- Java does not support multiple inheritance, so you lose flexibility if the class needs to extend another class.

#### Better Way Is To Implement Runnable

Implement `Runnable` class, override `run()`, then construct new `Runnable` and pass this object to `Thread` constructor.

```java
class MyTask implements Runnable {
    public void run() {
        System.out.println("Running");
    }
}
new Thread(new MyTask()).start();
```

Why it's better:
- Encourages separation of concerns (`Runnable` only defines the task, `Thread` executes it).
- You can use that same `Runnable` object in other places too, like Executor Services.
- You can still extend another class.
- One `Runnable` can be shared by many threads.

#### What Happens When You Call start()?

When you call `thread.start()`, the JVM creates a new thread of execution.
The `run()` method is executed in this new thread.

Once `start()` is called, the thread runs in the background, meaning it executes independently of the thread that created it.

Threads run asynchronously, meaning their execution order is unpredictable.

```java
// NEVER do this:
new MyTask().run(); // This runs in MAIN thread, NOT new thread!

// ALWAYS do this:
new Thread(new MyTask()).start(); // This creates new thread
```

#### Interview Q: start() vs run()?
> `run()` = normal method call, runs in current thread, no new thread.
> `start()` = asks JVM to create new thread, then JVM calls `run()` in that new thread. You can call `start()` only once per Thread object, second call throws `IllegalThreadStateException`.

#### Sample Code — Lifecycle + Management

```java
class PrintTask implements Runnable {
    private String name;
    public PrintTask(String name) { this.name = name; }
    public void run() {
        for (int i = 1; i <= 3; i++) {
            System.out.println(name + " : " + i);
            try { Thread.sleep(500); } catch (InterruptedException e) {
                System.out.println(name + " interrupted");
            }
        }
    }
}

public class ThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new PrintTask("Thread-1"));
        Thread t2 = new Thread(new PrintTask("Thread-2"));
        t1.start();
        t2.start();
        t1.join(); // wait t1 to finish
        t2.join(); // wait t2 to finish
        System.out.println("All threads finished!");
    }
}
```

#### Thread Management Methods

| Name | One-line summary |
| :--- | :--- |
| `Thread` | A thread of execution; `start()` launches it and the JVM calls its `run()`. |
| `Runnable` | The task to run; a `Thread` executes it, and it can be shared or reused. |
| `synchronized` | Monitor lock on a method or block; gives mutual exclusion plus visibility. |
| `wait()` | Releases the monitor and waits until notified or interrupted. |
| `notify()` | Wakes one arbitrary thread waiting on this object's monitor. |
| `notifyAll()` | Wakes every thread waiting on this object's monitor. |
| `sleep(ms)` | Pauses the current thread for a timeout without releasing locks. |
| `join()` | Waits for another thread to finish before continuing. |
| `yield()` | Hints that the current thread is willing to give up the CPU. |
| `interrupt()` | Requests cancellation by setting the flag or breaking `sleep`/`wait`/`join`. |
| `ThreadGroup` | Groups threads for bulk operations and uncaught-exception handling. |
| `start()` | Starts new thread, calls `run()` | Can be called only once |
| `run()` | Code to execute | Don't call directly |
| `setPriority(1-10)` | `MIN(1)`, `NORM(5)`, `MAX(10)` | Hint only, OS may ignore |
| `setDaemon(true)` | Background thread, JVM exits without waiting | Must set before `start()` |
| `isAlive()` | Check if running |  |
| Daemon threads | Background threads the JVM does not wait for on shutdown. |
| Priorities (1–10) | Scheduling hint only; the OS is free to ignore it. |

```java
// Daemon vs User thread
Thread t = new Thread(() -> { /* work */ });
t.setDaemon(true); // JVM will exit even if t is still running
t.start();
```

### 1.0.2 Thread Lifecycle and States

A Java thread is always in one of six states. The `Thread.State` enum itself arrived in Java 5, but the states are as old as `Thread`:

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
| `RUNNABLE` | Executing or ready to run | `start()`; also blocking I/O on most JVMs |
| `BLOCKED` | Waiting to enter a `synchronized` block | Monitor held by another thread |
| `WAITING` | Waiting indefinitely for another thread | `wait()`, `join()`, `park()` |
| `TIMED_WAITING` | Waiting with a timeout | `sleep(ms)`, `wait(ms)`, `join(ms)` |
| `TERMINATED` | `run()` returned or threw | Normal exit |

Interview traps:
- `RUNNABLE` does not mean "on a CPU right now"; it includes threads blocked in I/O.
- `BLOCKED` is monitor contention only. A thread inside `wait()` is `WAITING`, not `BLOCKED`.
- A terminated thread cannot be restarted; a second `start()` throws `IllegalThreadStateException`.
- `getState()` is a snapshot; the thread can change state immediately after you read it.

### 1.0.3 Race Conditions and `synchronized`

`count++` is three steps: read, add, write. Two threads can interleave them and lose an update.

```java
class Counter {
    private int count = 0;

    void increment() {
        count++; // read + add + write: RACE
    }
}
```

The fix is a monitor lock:

```java
class Counter {
    private int count = 0;

    synchronized void increment() { // locks `this`
        count++;
    }
}

class Counter2 {
    private final Object lock = new Object();

    void increment() {
        synchronized (lock) { // explicit monitor object
            count++;
        }
    }
}
```

What the monitor gives you:
1. **Mutual exclusion** — one thread at a time inside the block.
2. **Visibility** — writes before the unlock are visible to the thread that locks next (happens-before).
3. **Reentrancy** — the same thread can re-enter a block guarded by a monitor it already holds.

Interview notes:
- A `synchronized` instance method locks `this`; a `static` method locks the `Class` object. They are different monitors.
- Two threads synchronizing on different objects do not exclude each other.
- Never lock on `this`, a `String` literal, or a boxed primitive: outside code can lock the same object and interfere.
- `synchronized` fixes visibility too, but only for code that uses the same monitor. A plain field read outside the lock can still see a stale value.
- Locking is not free; keep the critical section small and never do I/O or callbacks while holding a lock.

### 1.0.4 `wait()` / `notifyAll()` — Producer–Consumer Base

`wait()` releases the monitor and parks the thread in the object's wait set. `notify()`/`notifyAll()` wake it, but it must reacquire the monitor before returning from `wait()`.

```java
class OneSlotBox {
    private int value;
    private boolean full = false;

    synchronized void put(int v) throws InterruptedException {
        while (full) {
            wait();          // release lock and wait for a consumer
        }
        value = v;
        full = true;
        notifyAll();         // wake any waiting consumer
    }

    synchronized int take() throws InterruptedException {
        while (!full) {
            wait();          // release lock and wait for a producer
        }
        full = false;
        notifyAll();         // wake any waiting producer
        return value;
    }
}
```

Rules interviewers look for:
- `wait()`/`notify()` must be called while holding the object's monitor, otherwise `IllegalMonitorStateException`.
- Always wait in a `while` loop, never an `if`: spurious wakeups are allowed, and another thread may consume the condition first.
- Prefer `notifyAll()` when waiters wait on different conditions. `notify()` may wake the "wrong" waiter, which goes back to waiting while the real waiter sleeps forever.
- The condition (`full`) must be guarded by the same monitor as the data.
- `wait()` is an `Object` method; `sleep()` is a `Thread` method.

#### Interview Q: Why while, not if, for wait()?
> Because of Spurious Wakeup — thread can wake up without notify(). And because after waking, condition may have been changed by another thread. Always re-check in `while`.

#### Interview Q: sleep() vs wait()?
| | `sleep()` | `wait()` |
| :--- | :--- | :--- |
| Releases lock? | No | Yes |
| Needs synchronized? | No | Yes, must be in synchronized block |
| Wakes by? | Timeout | `notify()` / `notifyAll()` |
| Belongs to? | `Thread` class | `Object` class |

### 1.0.5 Deadlock

Classic cause: two threads acquire the same two monitors in opposite order.

```java
void transfer(Account from, Account to) {
    synchronized (from.lock) {
        synchronized (to.lock) {
            // move money
        }
    }
}
// Thread A: transfer(a, b) locks a, then b
// Thread B: transfer(b, a) locks b, then a  => deadlock
```

Fixes:
1. **Global lock ordering** — sort accounts by a stable id and always lock in that order.
2. **Reduce nested locks** — hold at most one lock at a time.
3. **Use timed `tryLock`** — only available from Java 5 (`ReentrantLock`), not with `synchronized`.
4. **Diagnose** with `jstack <pid>`; it prints "Found one Java-level deadlock" with the two thread stacks.

Interview note: a `synchronized` deadlock cannot be broken from Java code in this era — there is no lock timeout and `Thread.stop()` is unsafe. Prevention is the only option.

### 1.0.6 `interrupt()` — Cooperative Cancellation

`interrupt()` does not stop a thread. It sets the interrupt flag; if the thread is inside `sleep`, `wait`, or `join`, that call throws `InterruptedException` and clears the flag.

```java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        // do a chunk of work
    }
    System.out.println("clean exit");
});
worker.start();
worker.interrupt(); // request stop
```

```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore the flag, don't swallow it
    return;                             // or clean up and exit
}
```

Notes:
- `Thread.interrupted()` checks **and clears** the flag; `isInterrupted()` leaves it set.
- Swallowing `InterruptedException` without restoring the flag is a classic bug: callers up the stack can no longer tell the thread was asked to stop.
- Interrupt cannot stop a CPU-bound loop unless the loop checks the flag.
- `Thread.stop()` was the "force stop" API and is unsafe: it releases monitors while an object may be half-updated. Deprecated in 1.2.

### 1.0.7 `ThreadGroup`, Priorities, and Other Dead Ends

- `ThreadGroup` groups threads for `interrupt()`/`enumerate()` and uncaught-exception handling. It was designed as a security boundary, but `SecurityManager` is gone in modern Java; `ThreadGroup` is effectively legacy. Do not build APIs around it.
- Priorities (1–10) are only a hint. The OS scheduler may ignore them, and they can cause starvation. Never rely on them for correctness.
- `Thread.yield()` is only a hint; it is not a synchronization mechanism.
- `Thread.suspend()`, `resume()`, and `stop()` are unsafe and deprecated since 1.2 (degraded to `UnsupportedOperationException` in modern JDKs).

### 1.0.8 Why `new Thread()` per Task Does Not Scale

- A platform thread is a 1:1 wrapper over an OS thread; the JVM reserves stack space for it (commonly around 1 MB with the default `-Xss`, though it varies by platform and JVM).
- Creating and destroying threads costs system calls and scheduler work; context switches cost cache locality.
- Each thread needs its own stack, so 10,000 threads means roughly 10 GB of stack reservation and heavy scheduling overhead.
- There is no pool, no work queue, no return value, no cancellation, and no `Future` — all of that had to be hand-built until Java 5's `java.util.concurrent`.

---

## Java 1.1 (1997) — The Quiet Release

### What
No new locks, pools, or synchronization primitives. Java 1.1 added:
- **A working interrupt mechanism** — `interrupt()` existed in 1.0 but did not reliably interrupt a sleeping thread; 1.1 implemented the interrupt flag + `InterruptedException` protocol.
- **JIT compiler** — hot bytecode is compiled to native code at runtime, making thread-heavy code faster.
- **Inner classes** — anonymous `Runnable` becomes idiomatic.
- **AWT event model** — UI events are delivered on a single Event Dispatch Thread (EDT).
- **Reflection, JavaBeans, RMI** — APIs that block and therefore push developers toward worker threads.

### Why It Mattered
- Cancellation finally had a standard signal, even though it stayed cooperative.
- The JIT narrowed the performance gap with native threads.
- Inner classes made a task a value you can pass around (`new Thread(new Runnable() { ... })`).
- The EDT rule created a new discipline: never block the UI thread, hand long work to a background thread, and hand results back to the EDT.

### Benefit
- Code could request shutdown instead of calling `stop()`.
- Cheaper execution made thread-per-request servers practical.
- Clear separation between the UI thread and worker threads.

### Downside / Limitation
- Still no pool, no return value, and no timeout on `synchronized`.
- Interrupt is cooperative: code that ignores the flag cannot be stopped.
- The EDT introduced new failure modes — a blocking call freezes the UI, and calling `invokeAndWait` from the EDT deadlocks.

> Common misconception: `ThreadLocal` is often listed under Java 1.1, but it shipped in **Java 1.2** (`@since 1.2`), together with `InheritableThreadLocal`. Its deep dive is in the Java 1.2 section below.

---

## Java 1.2 (1998) — Java 2

### What
- **`ThreadLocal` and `InheritableThreadLocal`** (`@since 1.2`).
- **Synchronized collection wrappers** — `Collections.synchronizedList/Set/Map/SortedMap/SortedSet`.
- **`Thread.stop()`, `suspend()`, `resume()` deprecated** — unsafe and deadlock-prone.
- **`java.lang.ref`** weak/soft/phantom references — the machinery behind `ThreadLocalMap`'s weak keys.

### Why It Was Added
1. **Per-thread state**: servers needed request, user, and format state without passing it through every method.
2. **Thread-safe collections**: the new Collections Framework was not thread-safe, and rewriting every collection was not an option.
3. **Unsafe thread control**: `stop()` could leave objects corrupted and `suspend()` could deadlock. Deprecation was the first step.

### Benefit
- `ThreadLocal` gave isolation without locks.
- One-line wrappers made existing collections usable from multiple threads.
- Deprecating `stop/suspend/resume` pushed code toward flags and interrupts.

### Downside / Limitation
- `ThreadLocal` leaks in pooled threads if `remove()` is forgotten.
- Wrappers lock the whole collection for every read and write, and iteration still needs external synchronization.
- Wrappers do not make compound actions atomic: `if (!map.containsKey(k)) map.put(k, v)` is still a race.

---

### 1.2.1 ThreadLocal — Deep Dive [INTERVIEW CORE]

#### What Is It?
`ThreadLocal` is a variable where **each thread has its own separate copy**.

```java
static int count = 0; // ONE copy, shared by all threads -> race condition

static ThreadLocal<Integer> count = new ThreadLocal<>();
// T1 has its own count, T2 has its own count, T3 has its own count
// They never see each other's value
```

It's not a thread. It's a **container that gives a different value per thread**.

#### Why Was It Needed? — 2 Bad Options Before

**Option A — Shared static variable:** Race condition.
```java
static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
// SimpleDateFormat is NOT thread-safe -> 2 threads format at same time = wrong date / crash
```

**Option B — Pass object through every method:** Ugly.
```java
void methodA(User user) { methodB(user); }
void methodB(User user) { methodC(user); }
void methodC(User user) { /* finally use user */ }
```

`ThreadLocal` solved both. Store once, access anywhere in that thread, without passing and without sharing.

#### How Does It Work Internally?

```text
Thread T1 -> [ ThreadLocalMap: { local1 -> 100, local2 -> "A" } ]
Thread T2 -> [ ThreadLocalMap: { local1 -> 200, local2 -> "B" } ]
```

- Every `Thread` object has a hidden `ThreadLocalMap` inside it.
- When you do `local.set(100)`, it stores `100` in **current thread's map**.
- When you do `local.get()`, it looks in **current thread's map**.
- So no synchronization is needed. Threads never touch each other's map.

#### Basic Code

```java
public class ThreadLocalDemo {
    static ThreadLocal<Integer> counter = new ThreadLocal<Integer>() {
        @Override protected Integer initialValue() { return 0; }
    };

    public static void main(String[] args) {
        Runnable task = () -> {
            counter.set(counter.get() + 1);
            System.out.println(Thread.currentThread().getName() + " = " + counter.get());
        };
        new Thread(task, "T1").start(); // T1 = 1
        new Thread(task, "T2").start(); // T2 = 1, NOT 2
    }
}
```

#### Real-World Uses

**1. SimpleDateFormat per thread:**
```java
static ThreadLocal<SimpleDateFormat> sdf = new ThreadLocal<SimpleDateFormat>() {
    @Override protected SimpleDateFormat initialValue() {
        return new SimpleDateFormat("yyyy-MM-dd");
    }
};
String s = sdf.get().format(new Date());
```

**2. User / Request Context in Web Server:**
```java
// In Filter: ThreadLocal<User> currentUser
// Anywhere in that request: currentUser.get() -> logged-in user
// No need to pass User through 10 layers of service -> dao
```

**3. DB Connection / Transaction per thread:**
```java
static ThreadLocal<Connection> conn = new ThreadLocal<>();
// One connection per request/thread
```

**4. Random per thread:** This is why Java 7 later made `ThreadLocalRandom`.

#### InheritableThreadLocal — Child Inherits From Parent

Normal `ThreadLocal` is NOT visible to child thread.

```java
static InheritableThreadLocal<String> local = new InheritableThreadLocal<>();

local.set("ParentValue");
new Thread(() -> {
    System.out.println(local.get()); // "ParentValue" - inherited!
}).start();
```

Use it when you spawn a child thread and want it to see parent's context.

#### Biggest Danger — Memory Leak in Thread Pools [INTERVIEW #1]

In normal threads, when thread dies, its map dies too. No leak.

But in **ExecutorService / Tomcat**, threads are **reused**, they never die.

```java
// BAD - In a web server:
executor.execute(() -> {
    local.set(user); // sets value in pooled thread
    // ... forget to remove
});
// Next request reuses SAME thread -> sees OLD user's data!
// Also old object never GC'd -> Memory Leak
```

**Fix — Always remove():**
```java
try {
    local.set(user);
    // do work
} finally {
    local.remove(); // MUST clean
}
```

#### Interview Q: Why does ThreadLocal cause memory leak?
> Because Thread holds strong reference to ThreadLocalMap, and Map holds value. In a pool, Thread never dies, so value never GC'd. And ThreadLocal key is WeakReference, so key can be GC'd but value stays -> leak. Fix: `remove()` in `finally`.

---

### 1.2.2 volatile vs ThreadLocal [INTERVIEW CORE]

They solve **opposite** problems.

> `volatile` = **ONE value, shared by ALL threads, everyone sees latest value.**
> `ThreadLocal` = **MANY values, one per thread, nobody sees other's value.**

#### Table

| | `volatile` | `ThreadLocal` |
| :--- | :--- | :--- |
| **What is it?** | Keyword for a variable | Class, a container |
| **How many copies?** | **1 copy** for all threads | **1 copy per thread** |
| **Sharing?** | Yes, shared | No, isolated |
| **Visibility?** | Change by one thread is visible to all | Change by one thread is **invisible** to others |
| **Atomicity?** | **NO** - `count++` is still race | No race, because no sharing |
| **Need synchronized?** | Still need for compound actions like `++` | No need, each thread has own copy |
| **Use for?** | Flag, status, stop signal | Per-thread data - user, date format, connection |
| **Memory Leak?** | No | Yes, if you forget `remove()` in thread pool |

#### Code — See Difference Live

```java
public class VsDemo {
    static volatile int sharedCount = 0;
    static ThreadLocal<Integer> localCount = new ThreadLocal<Integer>() {
        @Override protected Integer initialValue() { return 0; }
    };

    public static void main(String[] args) throws Exception {
        Runnable task = () -> {
            sharedCount++;
            localCount.set(localCount.get() + 1);
            System.out.println(Thread.currentThread().getName() +
                " | volatile=" + sharedCount +
                " | threadLocal=" + localCount.get());
        };
        Thread t1 = new Thread(task, "T1");
        Thread t2 = new Thread(task, "T2");
        t1.start(); t2.start();
        t1.join(); t2.join();
    }
}
// Possible Output:
// T1 | volatile=1 | threadLocal=1
// T2 | volatile=2 | threadLocal=1  <- ThreadLocal is still 1, not 2!
```

#### When To Use Which?

**Use volatile when:**
```java
volatile boolean running = true; // Stop flag
volatile boolean ready = false; // Status flag
private static volatile Singleton instance; // Double-Checked Locking
```
Rule: **One writer, many readers, simple read/write. Never `volatile count++`.**

**Use ThreadLocal when:**
```java
ThreadLocal<SimpleDateFormat> sdf = ...;
ThreadLocal<User> currentUser = ...;
ThreadLocal<Integer> counter = ...;
```
Rule: **Data should NOT be shared. Each thread needs isolation.**

#### Biggest Mistakes

**Mistake 1 — Using volatile for counter:**
```java
volatile int count = 0;
count++; // WRONG! Read + Add + Write = 3 steps, race still happens
// Fix: Use AtomicInteger or synchronized
```

**Mistake 2 — Using ThreadLocal to share data:**
```java
ThreadLocal<String> data = new ThreadLocal<>();
// T1 sets "Hello", T2 does data.get() -> gets null, NOT "Hello"
// Fix: If you want to share, use volatile / ConcurrentHashMap
```

**One-line memory trick:**
> **Want all threads to SEE same value? -> `volatile`**
> **Want all threads to HAVE their own value? -> `ThreadLocal`**

---

### 1.2.3 `volatile` — What It Really Guarantees

`volatile` is older than Java 1.2: the keyword exists since 1.0. What changed is how seriously the memory model treated it. Before Java 5 (JSR-133), the JMM did not guarantee visibility and ordering well enough to make double-checked locking safe; Java 5 redefined `volatile` with happens-before semantics.

```java
class StopFlag {
    private volatile boolean running = true;

    void stop() {
        running = false; // visible to the worker immediately
    }

    void work() {
        while (running) {
            // do a chunk of work
        }
    }
}
```

Guarantees:
1. **Visibility** — a write is seen by the next read.
2. **Ordering** — no reordering across the volatile access (happens-before).
3. **Atomicity of a single read or write** — but **not** of `count++`.

Interview trap: `volatile` does not replace `synchronized` for invariants that span multiple fields. Use it for one independent flag or state field, not for compound updates.

#### Interview Q: What does volatile guarantee?
> Visibility, ordering (happens-before), and atomicity of the individual read/write. It does not give mutual exclusion and does not make compound actions atomic.

### 1.2.4 Synchronized Collections — and Their Traps

Before 1.2, `Vector` and `Hashtable` synchronized every method. The 1.2 Collections Framework replaced them with unsynchronized `ArrayList` and `HashMap`, plus optional wrappers:

```java
List<String> list = Collections.synchronizedList(new ArrayList<String>());
list.add("A");

// Iteration MUST hold the same lock
synchronized (list) {
    for (String s : list) {
        System.out.println(s);
    }
}
```

Traps:
- **Iteration** is not synchronized by the wrapper; without the `synchronized (list)` block the fail-fast iterator can throw `ConcurrentModificationException`.
- **Compound actions** are still races:

```java
if (!list.contains("A")) { // another thread can add here
    list.add("A");
}
```

- Every operation takes the whole collection lock, so throughput is poor under contention. Later releases added `ConcurrentHashMap` and `CopyOnWriteArrayList` for that reason.
- `Collections.synchronizedMap` is not the same as `ConcurrentHashMap`: reads still block each other.

---

## Java 1.3 (2000) — HotSpot, Timers, Shutdown Hooks

> Naming note: there was never an official release called "Java 3". JDK 1.2 was rebranded "Java 2", and the next releases were J2SE 1.3 and 1.4, then Java 5. "Java 3" in this timeline means J2SE 1.3.

### What
- **HotSpot becomes the default JVM** — adaptive optimization and faster synchronization/thread creation.
- **`java.util.Timer` and `TimerTask`** — a built-in scheduler that runs tasks on one background thread.
- **`Runtime.addShutdownHook(Thread)`** — run cleanup code when the JVM shuts down.
- Still no thread pool, no `Future`, and no `java.util.concurrent`.

### Why It Was Added
1. **Scheduling**: before 1.3, periodic work meant a hand-written `Thread` plus a `sleep` loop.
2. **Shutdown**: applications needed a portable way to flush buffers and close resources on Ctrl+C or `System.exit`.
3. **Performance**: HotSpot made uncontended `synchronized` cheap enough that thread-heavy code stopped paying a large penalty.

### Benefit
- `Timer` provided a standard scheduler.
- Shutdown hooks made graceful shutdown possible without a custom signal handler.
- HotSpot's adaptive compiler improved throughput for concurrent workloads.

### Downside / Limitation
- `Timer` uses **one** thread: one slow task delays every other task, and an uncaught exception kills the timer thread and cancels the whole schedule.
- `Timer` schedules can drift or bunch up after a stall.
- Shutdown hooks run concurrently, in unspecified order, and must be fast; they cannot assume other services are still alive.
- Server code still created one thread per request.

### Code — `Timer`

```java
import java.util.Timer;
import java.util.TimerTask;

Timer timer = new Timer(true); // true = daemon thread
timer.scheduleAtFixedRate(new TimerTask() {
    public void run() {
        System.out.println("tick " + System.currentTimeMillis());
    }
}, 0, 1000);
```

### Code — Shutdown Hook

```java
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    System.out.println("flushing buffers...");
}));
```

### Interview Q: Why is `java.util.Timer` considered legacy?
> It runs every task on a single thread, has no exception isolation, and has no real pool. One failing task cancels the schedule. Use `ScheduledExecutorService` (Java 5) instead.

---
