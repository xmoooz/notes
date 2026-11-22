---
title: "Java Thread & Concurrency Timeline: Java 18 to Java 20"
date: 2026-11-19T10:00:00Z
lastmod: 2026-11-19T10:00:00Z
draft: false
categories: ["java", "concurrency"]
tags: ["java", "concurrency", "threads", "virtual-threads", "structured-concurrency", "scoped-values", "java19", "java20"]
series: ["java concurrency timeline"]
author: "xmoooz"
description: "Java 18 to Java 20 concurrency, version by version: virtual threads preview, structured concurrency incubator/preview, scoped values incubator — what was added, what is deprecated, what to never use, and practical tips. Concurrency focus only."
url: 2026/11/19/java-concurrency-timeline-java18-to-java20
---

## 0. Global Vision — Java 18 to Java 20 in 2 Minutes

This file covers only Java 18 to Java 20. Nothing before, nothing after. Concurrency focus only.

The story inside this window is simple:

**Java 12-17 changed how you *write* concurrent code. Java 18-20 previews how you will *run* it: one cheap thread per task.**

| Version | What it added to the story (concurrency only) | What problem it was answering |
| :--- | :--- | :--- |
| **18 (2022)** | No new concurrency API | "No new concurrency feature — keep using 8-17 APIs." |
| **19 (2022)** | Virtual threads (preview) + Structured concurrency (incubator) | "What if a thread was cheap enough to create one per task?" |
| **20 (2023)** | Virtual threads (preview, round 2) + Structured concurrency (preview) + Scoped values (incubator) | "How do we pass data to a tree of tasks without ThreadLocal?" |

### The mental shift in 18-20

| In Java 8-17 | In Java 18-20 (preview) |
| :--- | :--- |
| Threads are expensive, so you pool them | Threads are cheap, so you create one per task |
| You reuse threads from a pool to save cost | You create a new virtual thread per task and throw it away |
| You pass data with `ThreadLocal` | You pass data with scoped values (incubator in 20) |
| You start subtasks and track them by hand | Structured concurrency starts subtasks in a scope and waits for all of them |

### The one sentence to remember for each version

- **Java 18:** No new concurrency API. The calm before the preview.
- **Java 19:** Virtual threads preview — cheap threads, one per task. Structured concurrency incubator — subtasks in a scope.
- **Java 20:** Virtual threads preview round 2, structured concurrency preview, scoped values incubator — the three pieces together for the first time.

### What is still relevant? What is dead?

**Still use every day (as final in Java 21+, see next file):** The *ideas* from 19-20 — virtual threads, structured concurrency, scoped values. The APIs in 19-20 themselves are preview / incubator, not production APIs.

**Use, but via modern wrapper:** Platform-thread pools for blocking work -> virtual threads (final in 21). `ThreadLocal` for request data in virtual threads -> scoped values (final in 25).

**Never use in new code:** Preview / incubator APIs in production without `--enable-preview`, `Thread.stop()` / `suspend()` / `resume()`, `ThreadGroup` as API, `Hashtable` / `Vector` / `Timer` for new code, `ThreadLocal` with virtual threads for large request data (see Section 3).

> Honest note: Java 18-20 is a preview era. Nothing in this file is final. The code in this file needs `--enable-preview` (19-20) and will change in Java 21+. Read this file to learn the shape, not to copy preview APIs into production.

---

## 1. Java 18 (2022) — No New Concurrency API

### What
No new `java.util.concurrent` API in Java 18. No new concurrency language feature in Java 18.

### Why
Java 18 is a short, non-LTS release. Its headline features (simple web server incubator, UTF-8 by default, code snippets) are *not* concurrency features. For concurrency, there is nothing new to learn in Java 18.

### Benefit
- Nothing to migrate. Java 17 concurrency code works unchanged.

### Downside
- Simple web server in Java 18 is incubator, not a concurrency API. Don't build production servers on it in Java 18.

### Status in Java 18
| API | Verdict |
| :--- | :--- |
| `CompletableFuture`, `Flow`, `ExecutorService`, `BlockingQueue`, concurrent collections, records, sealed (Java 17) | **STILL CORE. Keep using them.** |
| Any new concurrency API in Java 18 | **NONE. Do not invent one.** |
| Simple web server (incubator in 18) | **INCUBATOR ONLY. Do not use in production in Java 18.** |
| `Thread.stop()`, `suspend()`, `resume()` | **DEPRECATED / DO NOT USE.** |
| `ThreadGroup` as API, `Hashtable` / `Vector` / `Timer` for new code | **LEGACY. Do not use for new code.** |
| `SecurityManager` for new code | **DEPRECATED FOR REMOVAL (since Java 17). Do not build new code on it.** |

**Best Practice:**
- Don't treat Java 18 as a concurrency release. Keep the Java 17 patterns.
- Don't use incubator APIs in production in Java 18.

---

## 2. Java 19 (2022) — Virtual Threads Preview + Structured Concurrency Incubator

### What
- **Virtual threads (preview)** — `Thread.ofVirtual()`, `Thread.startVirtualThread()`, `Executors.newVirtualThreadPerTaskExecutor()`. Preview in Java 19, needs `--enable-preview`.
- **Structured concurrency (incubator)** — `StructuredTaskScope` (incubator API), `StructuredTaskScope.ShutdownOnFailure`, `ShutdownOnSuccess`. Incubator in Java 19, needs `--enable-preview` + incubator module.

### Why
Platform threads are expensive (OS thread, large stack, limited count). So you pool them, reuse them, and avoid blocking them. That forces async code, callbacks, and reactive chains for simple blocking work. Virtual threads are cheap (JVM-managed, small stack, millions possible). If a thread is cheap, you can write simple blocking code — one virtual thread per task — and let the JVM handle the scale. Structured concurrency fixes the other problem: when one task starts many subtasks, you need a scope that starts them, waits for all of them, and cancels the rest on failure. Without it, subtasks leak, errors get lost, and cancellation is hand-written.

### Benefit
- Simple blocking code scales: one virtual thread per task, no pool tuning, no async chains for simple cases.
- Millions of virtual threads possible (in practice, limited by memory / work, not by OS threads).
- Structured scope: subtasks are children of a scope, scope waits for all, failure cancels siblings.

### Downside
- Preview / incubator in Java 19. APIs can change. Needs `--enable-preview`.
- Virtual threads are not free: they still use memory, they still block on `synchronized` in Java 19 (pinning — see Best Practice), and they don't fix CPU-bound work (CPU is still the limit).
- `ThreadLocal` + virtual threads is a trap: millions of virtual threads each with a large `ThreadLocal` = memory blow-up.

### Status in Java 19
| API | Verdict |
| :--- | :--- |
| Virtual threads (preview in 19) | **PREVIEW ONLY. Learn it, do not build production code on it in Java 19.** |
| Structured concurrency (incubator in 19) | **INCUBATOR ONLY. Learn it, do not build production code on it in Java 19.** |
| `CompletableFuture`, `Flow`, `ExecutorService`, concurrent collections | **STILL CORE. Keep using them in production in Java 19.** |
| Platform-thread pools for production work in Java 19 | **STILL CORE in Java 19. Virtual threads are preview, not a replacement yet.** |
| Virtual threads in production on Java 19 without preview flag | **NEVER USE. They are preview in 19, final in 21.** |
| `ThreadLocal` for large request data with virtual threads | **NEVER USE as a pattern. Use scoped values (incubator in 20, final in 25).** |
| `synchronized` + blocking in virtual threads (pinning in 19) | **AVOID in virtual threads in 19. Prefer `ReentrantLock`.** |
| `Thread.stop()`, `suspend()`, `resume()` | **DEPRECATED / DO NOT USE.** |
| `ThreadGroup` as API, `Hashtable` / `Vector` / `Timer` for new code | **LEGACY. Do not use for new code.** |

---

### 2.1 Virtual Threads (Preview in 19) — One Cheap Thread Per Task

**What:** `Thread.ofVirtual()`, `Thread.startVirtualThread()`, `Executors.newVirtualThreadPerTaskExecutor()`.

**Why:** See Section 2 Why above — threads are cheap, so one per task.

**Real sample — preview code, needs `--enable-preview` in Java 19:**

```java
// Java 19 preview: --enable-preview
// Create one virtual thread per task, no pool to tune
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1000; i++) {
        int id = i;
        executor.submit(() -> {
            // Simple blocking code — this is the point of virtual threads
            System.out.println("Task " + id + " on " + Thread.currentThread());
            return id;
        });
    }
} // executor closes, waits for tasks
```

```java
// Java 19 preview: direct virtual thread
Thread vt = Thread.ofVirtual().name("my-virtual").start(() -> {
    System.out.println("Hello from virtual thread: " + Thread.currentThread());
});
vt.join();
System.out.println("Is virtual: " + vt.isVirtual());
```

**Best Practice (Java 19 preview):**
- Don't pool virtual threads. Pooling defeats the point — create one per task, let it die.
- Don't use `ThreadLocal` for large data with virtual threads. Millions of virtual threads + large `ThreadLocal` = memory problem.
- Avoid `synchronized` + blocking in virtual threads in Java 19 (pinning: virtual thread pins its carrier platform thread while in `synchronized`). Prefer `ReentrantLock` in virtual threads in this era.
- Don't use virtual threads for CPU-bound work to "make it faster." Virtual threads help blocking / I/O work, not CPU work. CPU work is still limited by cores.

### 2.2 Structured Concurrency (Incubator in 19) — Subtasks in a Scope

**What:** `StructuredTaskScope` (incubator), `fork()`, `join()`, `ShutdownOnFailure`, `ShutdownOnSuccess`.

**Why:** When one task starts many subtasks, you need: start all, wait for all, cancel the rest if one fails, propagate the first error. Hand-written code leaks subtasks and loses errors. Structured concurrency makes subtasks children of a scope — the scope enforces the structure.

**Real sample — incubator code, needs `--enable-preview` + incubator module in Java 19:**

```java
// Java 19 incubator: --enable-preview --add-modules jdk.incubator.concurrent
// Shape only — API changes in later versions
/*
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    StructuredTaskScope.Subtask<String> user = scope.fork(() -> loadUser());
    StructuredTaskScope.Subtask<String> orders = scope.fork(() -> loadOrders());
    scope.join();           // wait for all subtasks
    scope.throwIfFailed();  // propagate first failure, siblings cancelled
    System.out.println(user.get() + " / " + orders.get());
}
*/
```

**Best Practice:**
- Don't start subtasks and forget them. Always join the scope, always handle failure.
- Don't use structured concurrency to replace `CompletableFuture` for every async chain in Java 19 — it is incubator, and `CompletableFuture` is still the production API in 19.
- Don't ignore `throwIfFailed()` / failure handling. A scope that swallows errors hides failures.

---

## 3. Java 20 (2023) — Virtual Threads Preview Round 2 + Structured Concurrency Preview + Scoped Values Incubator

### What
- **Virtual threads (preview, round 2)** — same idea as Java 19, small API refinements. Still preview, needs `--enable-preview`.
- **Structured concurrency (preview)** — `StructuredTaskScope` moves from incubator to preview in Java 20. Still preview, needs `--enable-preview`.
- **Scoped values (incubator)** — `ScopedValue`, `ScopedValue.where(...).run(...)`, `ScopedValue.where(...).call(...)`. Incubator in Java 20, needs `--enable-preview` + incubator module.

### Why
Virtual threads need a way to pass data (request id, user, tenant) to a tree of subtasks without `ThreadLocal`. `ThreadLocal` is mutable, inherits by accident, and costs memory with millions of virtual threads. Scoped values are immutable, bound for a scope, and visible to all subtasks in that scope — including virtual threads and structured subtasks. When the scope ends, the binding ends. No leak, no mutation.

### Benefit
- Scoped values: immutable data for a scope, visible to subtasks, no `ThreadLocal` mutation.
- Virtual threads + structured concurrency + scoped values together: one thread per task, subtasks in a scope, data passed by scope — the full picture for the first time.
- Structured concurrency is now preview (not incubator) — closer to final, but still not final.

### Downside
- All three are still preview / incubator in Java 20. APIs can still change.
- Scoped values are incubator — the API in Java 20 is not the final API.
- `ThreadLocal` still exists and still works — migrating to scoped values in Java 20 is premature, because scoped values are incubator.

### Status in Java 20
| API | Verdict |
| :--- | :--- |
| Virtual threads (preview, round 2 in 20) | **PREVIEW ONLY. Learn it, do not build production code on it in Java 20.** |
| Structured concurrency (preview in 20) | **PREVIEW ONLY. Learn it, do not build production code on it in Java 20.** |
| Scoped values (incubator in 20) | **INCUBATOR ONLY. Learn it, do not build production code on it in Java 20.** |
| `CompletableFuture`, `Flow`, `ExecutorService`, concurrent collections | **STILL CORE. Keep using them in production in Java 20.** |
| Platform-thread pools for production work in Java 20 | **STILL CORE in Java 20. Virtual threads are preview, not a replacement yet.** |
| Preview / incubator APIs in production on Java 20 | **NEVER USE without `--enable-preview`. They are preview / incubator in 20, final in 21 / 25.** |
| `ThreadLocal` for large request data with virtual threads | **NEVER USE as a pattern. Scoped values are the direction, but incubator in 20.** |
| `synchronized` + blocking in virtual threads | **AVOID in virtual threads in 20. Prefer `ReentrantLock`.** |
| `Thread.stop()`, `suspend()`, `resume()` | **DEPRECATED / DO NOT USE.** |
| `ThreadGroup` as API, `Hashtable` / `Vector` / `Timer` for new code | **LEGACY. Do not use for new code.** |

---

### 3.1 Scoped Values (Incubator in 20) — Data for a Scope, Not a Thread

**What:** `ScopedValue`, `ScopedValue.where(value, binding).run(task)`, `.call(task)`.

**Why:** See Section 3 Why above — `ThreadLocal` is mutable and leaks, scoped values are immutable and scoped.

**Real sample — incubator code, needs `--enable-preview` + incubator module in Java 20:**

```java
// Java 20 incubator: --enable-preview --add-modules jdk.incubator.concurrent
// Shape only — API becomes preview in 21, final in 25
/*
final ScopedValue<String> REQUEST_ID = ScopedValue.newInstance();

void handle(String requestId) {
    // Bind for this scope only. Subtasks in this scope see the same value.
    ScopedValue.where(REQUEST_ID, requestId).run(() -> {
        System.out.println("Request: " + REQUEST_ID.get());
        // Subtasks started here also see REQUEST_ID
    });
    // After run() returns, binding is gone. No leak.
}
*/
```

**`ThreadLocal` vs Scoped Value:**

|  | `ThreadLocal` | Scoped Value |
| :--- | :--- | :--- |
| Mutability | Mutable (`set`) | Immutable (bound once for a scope) |
| Scope | Thread, until removed | Scope, until scope ends |
| Leak risk | Yes, if you forget `remove()` | No, binding ends with scope |
| With virtual threads | Costly, millions of threads + large locals | Designed for virtual threads |
| Subtasks | Not automatically visible | Visible to subtasks in scope |

**Best Practice:**
- Don't use `ThreadLocal` for request data in new virtual-thread code. Learn scoped values instead, but don't ship them in production in Java 20 — they are incubator.
- Don't mutate scoped values — they are immutable by design. If you need mutation, a scoped value is the wrong tool.
- Don't forget: scoped values are for *immutable, read-only* data (request id, user, tenant). Not for counters, not for mutable state.

---

## 4. Best Practices — The Cheat List (Only Java 18-20)

### Per-Version
- **Java 18:** No new concurrency API. Keep Java 17 patterns.
- **Java 19:** Virtual threads preview, structured concurrency incubator. Learn, don't ship in production in 19.
- **Java 20:** Virtual threads preview round 2, structured concurrency preview, scoped values incubator. Learn, don't ship in production in 20.

### Virtual Threads (Preview Era)
- One virtual thread per task. Don't pool virtual threads.
- Simple blocking code is the point. Don't write async chains for simple blocking work in virtual-thread code.
- Avoid `synchronized` + blocking in virtual threads in 19-20 (pinning). Prefer `ReentrantLock`.
- Virtual threads help I/O / blocking work, not CPU-bound work.

### Structured Concurrency (Preview Era)
- Subtasks in a scope. Scope joins all, cancels siblings on failure, propagates first error.
- Don't leak subtasks. Always join the scope.

### Scoped Values (Incubator Era)
- Immutable data for a scope, visible to subtasks. No `ThreadLocal` mutation, no leak.
- Only for immutable, read-only data. Not for counters or mutable state.

### Deprecated / Legacy — Never in New Java 18-20 Code
- Preview / incubator APIs in production without `--enable-preview`.
- `ThreadLocal` for large request data with virtual threads.
- `synchronized` + blocking in virtual threads (pinning in 19-20).
- Pooling virtual threads.
- `Thread.stop()` / `suspend()` / `resume()` — deprecated.
- `ThreadGroup` as API, `Hashtable` / `Vector` / `Timer` for new code — legacy.
- `SecurityManager` for new code — deprecated for removal.

---

## 5. Tips — 30-Second Answers (Only Java 18-20)

1.  **"What is a virtual thread?"**
    > A cheap, JVM-managed thread. Millions possible. One per task, no pool to tune. Simple blocking code scales.

2.  **"Virtual threads vs platform threads?"**
    > Platform threads are OS threads, expensive, pooled. Virtual threads are JVM threads, cheap, created per task, thrown away.

3.  **"Why not pool virtual threads?"**
    > Pooling defeats the point. Virtual threads are cheap to create. Pooling adds complexity for no gain.

4.  **"When do virtual threads *not* help?"**
    > CPU-bound work. CPU is still limited by cores. Virtual threads help blocking / I/O work, not CPU work.

5.  **"What is structured concurrency?"**
    > Subtasks are children of a scope. Scope starts them, waits for all, cancels siblings on failure, propagates first error. No leaked subtasks.

6.  **"What is a scoped value?"**
    > Immutable data bound for a scope, visible to subtasks in that scope. No mutation, no leak. The replacement for `ThreadLocal` for request data.

7.  **"Why not `ThreadLocal` with virtual threads?"**
    > `ThreadLocal` is mutable, leaks if you forget `remove()`, and costs memory with millions of virtual threads.

8.  **"Why `ReentrantLock` over `synchronized` in virtual threads in 19-20?"**
    > `synchronized` + blocking pins the virtual thread to its carrier platform thread in 19-20. `ReentrantLock` does not pin in the same way.

9.  **"What did Java 18 add for concurrency?"**
    > Nothing. No new concurrency API in Java 18.

10. **"Can I use virtual threads in production on Java 19-20?"**
    > No. They are preview in 19-20. Final in Java 21. Learn the shape in 19-20, ship on 21+.

---

## 6. Cheat Sheet — Java 18 to Java 20 Only

### Timeline

| Version | Year | Headline | Still Use? |
| :--- | :--- | :--- | :--- |
| **18** | 2022 | No new concurrency API | **No new API** |
| **19** | 2022 | Virtual threads (preview), Structured concurrency (incubator) | **Preview / incubator only in 19** |
| **20** | 2023 | Virtual threads (preview round 2), Structured concurrency (preview), Scoped values (incubator) | **Preview / incubator only in 20** |

### API Quick Reference

| API | What it does | Verdict |
| :--- | :--- | :--- |
| `Thread.ofVirtual()` / `startVirtualThread()` | Create virtual thread | Preview in 19-20, final in 21 |
| `Executors.newVirtualThreadPerTaskExecutor()` | One virtual thread per task | Preview in 19-20, final in 21 |
| `StructuredTaskScope` | Subtasks in a scope | Incubator in 19, preview in 20, final in 25 |
| `ScopedValue` | Immutable data for a scope | Incubator in 20, preview in 21-24, final in 25 |
| `CompletableFuture` / `Flow` / `ExecutorService` | Async core | Still core, unchanged |
| `Thread.stop/suspend/resume` | Force stop | **NEVER — deprecated** |
| `ThreadGroup` / `Hashtable` / `Vector` / `Timer` | Legacy APIs | **NEVER for new code** |
| Preview / incubator APIs in production | Preview | **NEVER in production in that version** |

### Decision Tree (Only Java 18-20)

```text
Which version are you on?
  ├─ 18? -> No new concurrency API, keep 17 patterns
  ├─ 19? -> Virtual threads preview, structured concurrency incubator, learn only
  └─ 20? -> Virtual threads preview, structured concurrency preview, scoped values incubator, learn only

Need to write production concurrent code in 18-20?
  └─ CompletableFuture / ExecutorService / Flow, same as 8-17. Preview APIs are learn-only.

Need to learn the future?
  ├─ Virtual threads -> one per task, no pooling, ReentrantLock over synchronized
  ├─ Structured concurrency -> subtasks in a scope, join all, handle failure
  └─ Scoped values -> immutable data for a scope, not ThreadLocal
```
