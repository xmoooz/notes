---
title: "Java Thread & Concurrency Timeline: Java 9 to Java 11"
date: 2026-11-05T23:30:00Z
lastmod: 2026-11-05T23:30:00Z
draft: false
categories: ["java", "concurrency", "timeline"]
tags: ["java", "concurrency", "timeline", "threads", "flow", "httpclient", "completablefuture", "java9", "java11"]
series: ["java concurrency timeline"]
author: "xmoooz"
description: "Java 9 to Java 11 concurrency, version by version: Flow API, CompletableFuture timeouts, HttpClient async, and modules — what was added, what is deprecated, what to never use, and practical tips. Concurrency focus only."
url: 2026/11/05/java-concurrency-timeline-java9-to-java11
---

## 0. Global Vision — Java 9 to Java 11 in 2 Minutes

This file covers only Java 9 to Java 11. Nothing before, nothing after. Concurrency focus only.

The story inside this window is simple:

**Java 8 gave you async chains. Java 9-11 gave you control over the flow of async data, and a standard async HTTP client to use it with.**

| Version | What it added to the story | What problem it was answering |
| :--- | :--- | :--- |
| **9 (2017)** | `Flow` API + `CompletableFuture` timeouts + Modules | "How do we stream many results without flooding the consumer? How do we time out async work? How do we close JDK internals?" |
| **10 (2018)** | No new concurrency API | "No new concurrency feature — this is a short, non-LTS release. Keep using Java 8-9 APIs." |
| **11 (2018, LTS)** | `HttpClient` final + `CompletableFuture` improvements usable in LTS | "How do we do async HTTP with the standard library, on an LTS?" |

### The mental shift in 9-11

| In Java 8 | In Java 9-11 |
| :--- | :--- |
| You chain one result to the next | You also control the *rate* of many results |
| You push data as fast as you can | Consumer pulls data by signalling demand |
| You write your own HTTP async wrapper | Standard `HttpClient` returns `CompletableFuture` |
| You use `CompletableFuture` and hope it finishes | You can time out, complete on timeout, or delay it |

### The one sentence to remember for each version

- **Java 9:** Async streams with backpressure (`Flow`), timeouts for `CompletableFuture`, and modules closing JDK internals.
- **Java 10:** No new concurrency API. The version to skip for concurrency — keep using 8-9 APIs.
- **Java 11:** Standard async HTTP (`HttpClient`) on an LTS. `sendAsync` returns `CompletableFuture`.

### What is still relevant? What is dead?

**Still use every day:** `Flow` contract, `HttpClient.sendAsync`, `CompletableFuture.orTimeout` / `completeOnTimeout`, `delayedExecutor`.

**Use, but via modern wrapper:** Hand-rolled timeout with scheduler + `complete()` -> `orTimeout`. Custom HTTP async wrapper -> `HttpClient`.

**Never use in new code:** `Flow` with unbounded demand, blocking `send()` loop for many HTTP calls, hand-rolled timeouts, reaching into JDK internals for locks / atomics, `--add-opens` as a permanent fix.

---

## 1. Java 9 (2017) — Flow + Timeouts + Modules

### What
- **`java.util.concurrent.Flow`** — `Publisher`, `Subscriber`, `Subscription`, `Processor`, plus `SubmissionPublisher`.
- **`CompletableFuture` improvements** — `orTimeout`, `completeOnTimeout`, `newIncompleteFuture`, `delayedExecutor`, `copy`, `completedStage`, `failedStage`, `completedFuture`.
- **Modules (JPMS)** — `module-info.java`, `exports`, `requires`, strong encapsulation. `java.util.concurrent` lives in `java.base`.

### Why
`CompletableFuture` handles one result. Parallel streams handle a fixed collection. Neither handles a *stream* of results where the producer is faster than the consumer. Without backpressure, a fast producer floods a slow consumer — unbounded queue, OOM. `Flow` makes demand explicit: consumer requests N, producer sends at most N. Java 8 `CompletableFuture` also has no timeout — a hanging task hangs `join()` forever. Java 9 adds timeouts. Modules close JDK internals that concurrent code used to reach into for hand-rolled locks / atomics.

### Benefit
- Backpressure is part of the contract, not a hand-rolled queue bound.
- Timeout / fallback for async work, no manual scheduler needed.
- Clear module boundary: public API is stable, internal API is not.

### Downside
- `Flow` alone is verbose and low-level. Easy to get wrong: forgetting `request(n)` means nothing arrives. Requesting `Long.MAX_VALUE` means no backpressure.
- `orTimeout` does not cancel the underlying work by itself. Task keeps running unless it checks interruption / cancellation.
- Old code that used `Unsafe` / internal APIs breaks or warns on modules. Migration is real work.

### Status in Java 9
| API | Verdict |
| :--- | :--- |
| `Flow.Publisher` / `Subscriber` / `Subscription` / `Processor` | **STILL CORE as the contract.** Know it, even if you use a library on top. |
| `SubmissionPublisher` | **STILL CORE for simple cases.** Built-in publisher, handles demand. |
| `CompletableFuture.orTimeout`, `completeOnTimeout`, `delayedExecutor`, `copy`, `newIncompleteFuture` | **STILL CORE. Use instead of hand-rolled timeouts.** |
| Modules, `module-info.java`, public `java.util.concurrent` API | **STILL CORE.** |
| Raw `Flow` for everyday async work | **USE WITH CARE.** Verbose. Prefer `CompletableFuture` / queue unless you need streaming + backpressure. |
| `Flow` with `request(Long.MAX_VALUE)` | **NEVER USE.** That is no backpressure. |
| Hand-rolled timeout with scheduler + `complete()` | **NEVER USE for new code. Use `orTimeout`.** |
| Reaching into JDK internals (`sun.misc.Unsafe`, internal packages) for locks / atomics / queues | **DEPRECATED / DO NOT USE.** Modules encapsulate them. Use public `java.util.concurrent` API. |
| `--add-opens` / `--add-exports` as a permanent fix | **NEVER USE as a permanent fix.** Fix the code to use public API. |
| `Thread.stop()`, `suspend()`, `resume()` | **DEPRECATED / DO NOT USE.** Still deprecated in Java 9. |
| `ThreadGroup` as API, `Hashtable` / `Vector` / `Timer` for new code | **LEGACY. Do not use for new code.** |

---

### 1.1 Flow API — Async Streams With Backpressure

**What:** `Publisher`, `Subscriber`, `Subscription`, `Processor`, `SubmissionPublisher`.

**Why:** See Section 1 Why above — fast producer, slow consumer, no flooding.

**Real sample — `SubmissionPublisher` + subscriber with demand:**

```java
import java.util.concurrent.*;
import java.util.concurrent.Flow.*;

public class FlowDemo {
    public static void main(String[] args) throws Exception {
        SubmissionPublisher<String> publisher = new SubmissionPublisher<>();

        publisher.subscribe(new Subscriber<>() {
            Subscription sub;
            public void onSubscribe(Subscription s) {
                sub = s;
                s.request(1); // ask for 1 first — backpressure starts here
            }
            public void onNext(String item) {
                System.out.println("Got: " + item);
                sub.request(1); // ask for next 1 only after processing this one
            }
            public void onError(Throwable t) { t.printStackTrace(); }
            public void onComplete() { System.out.println("Done"); }
        });

        publisher.submit("a");
        publisher.submit("b");
        publisher.submit("c");
        publisher.close();
        Thread.sleep(500);
    }
}
```

**`Flow` contract — the rules that matter:**

1.  Producer must not send more than the consumer requested. `request(n)` is a hard limit.
2.  Consumer must call `request(n)` to get anything. No request = no items.
3.  `onNext` must not be called concurrently for the same subscriber (in a correct implementation).
4.  `onError` / `onComplete` are terminal — nothing after them.
5.  `cancel()` stops delivery. Always cancel when you no longer need the stream.

**`Flow` vs `CompletableFuture` vs `BlockingQueue`:**

|  | `CompletableFuture` | `BlockingQueue` | `Flow` |
| :--- | :--- | :--- | :--- |
| Results | One | Many, via queue | Many, via stream |
| Backpressure | No | Yes, via bounded queue + blocking | Yes, via demand (`request(n)`) |
| Blocking? | No (chain) | Yes (`put` / `take` block) | No (async) |
| Use when | One async result, chaining | Passing work between threads | Streaming many results, fast producer, slow consumer |

**Best Practice:**
- Always `request(n)` in small batches. Never `Long.MAX_VALUE` unless you truly have no backpressure need.
- Always handle `onError` and `cancel()` on shutdown. A subscriber that never cancels keeps the publisher alive.
- Don't do slow work inside `onNext` without requesting less. `onNext` is the consumer's bottleneck.
- Don't use raw `Flow` to replace a simple `CompletableFuture` chain. Use `Flow` only when you have a *stream* of items and a *rate* problem.

### 1.2 CompletableFuture Improvements — Timeout, Delay, Copy

**What:** `orTimeout`, `completeOnTimeout`, `delayedExecutor`, `newIncompleteFuture`, `copy`.

**Why:** Java 8 `CompletableFuture` has no timeout. A hanging task hangs `join()` forever.

> Note on scope: `exceptionallyCompose` is *not* in Java 9. It comes in Java 12. This file does not use it here.

**Real sample:**

```java
import java.util.concurrent.*;

public class TimeoutDemo {
    public static void main(String[] args) {
        // orTimeout: fail after 2 seconds
        CompletableFuture<String> f1 = CompletableFuture
            .supplyAsync(() -> slowWork())
            .orTimeout(2, TimeUnit.SECONDS);
        try {
            System.out.println(f1.join());
        } catch (CompletionException e) {
            System.out.println("Timed out: " + e.getCause()); // TimeoutException
        }

        // completeOnTimeout: fallback value instead of failing
        CompletableFuture<String> f2 = CompletableFuture
            .supplyAsync(() -> slowWork())
            .completeOnTimeout("fallback", 2, TimeUnit.SECONDS);
        System.out.println(f2.join()); // "fallback" if slowWork takes too long

        // delayedExecutor: run after a delay, no scheduler needed
        CompletableFuture<String> f3 = CompletableFuture
            .supplyAsync(() -> "delayed", CompletableFuture.delayedExecutor(1, TimeUnit.SECONDS));
        System.out.println(f3.join());
    }
    static String slowWork() {
        try { Thread.sleep(5000); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        return "slow";
    }
}
```

**Best Practice:**
- Use `orTimeout` for any async work that must not hang forever — HTTP, DB, external calls.
- Use `completeOnTimeout` only when a fallback value is genuinely safe. Log when the timeout fires.
- Remember: timeout completes the *future*, not the *task*. If the task does I/O, it may keep running.

### 1.3 Modules (JPMS) — What Changes for Concurrent Code

**What:** `module-info.java`, `exports`, `requires`, strong encapsulation. `java.util.concurrent` in `java.base`.

**Why:** Before modules, concurrent code could reach into JDK internals to hand-roll locks, atomics, queues. Modules close that door by default.

**Real sample:**

```java
// module-info.java — minimal, concurrency code needs nothing special
module com.example.app {
    requires java.base; // java.util.concurrent, java.util.concurrent.Flow live here
    requires java.net.http; // HttpClient, only if you use it (Java 11)
}
```

**Best Practice:**
- Don't fight encapsulation with `--add-opens` / `--add-exports` as a permanent fix. Fix the code to use public API.
- If a library needs reflection on your concurrent classes, open only that package, only to that module.
- Keep concurrent code in `java.base` types — they are always available, no extra `requires` needed beyond `java.base`.

---

## 2. Java 10 (2018) — No New Concurrency API

### What
No new `java.util.concurrent` API in Java 10.

### Why
Java 10 is a short, non-LTS release (6-month cadence). Its headline is local-variable type inference (`var`), which is *not* a concurrency feature. For concurrency, there is nothing new to learn in Java 10.

### Benefit
- Nothing to migrate. Java 8-9 concurrency code works unchanged.

### Downside
- `var` does not change the thread model. `var future = CompletableFuture.supplyAsync(...)` is the same future, just less typing. It does not make async code safer or faster by itself.

### Status in Java 10
| API | Verdict |
| :--- | :--- |
| `CompletableFuture`, `Flow`, `ExecutorService`, concurrent collections (Java 8-9) | **STILL CORE. Keep using them.** |
| `var` for task / future variables | **OPTIONAL. Style only, not concurrency.** |
| Any new concurrency API in Java 10 | **NONE. Do not invent one.** |
| `Thread.stop()`, `suspend()`, `resume()` | **DEPRECATED / DO NOT USE.** |
| `ThreadGroup` as API, `Hashtable` / `Vector` / `Timer` for new code | **LEGACY. Do not use for new code.** |

**Real sample — what Java 10 actually changes for concurrent code (only style):**

```java
import java.util.concurrent.*;

// Java 10: var is style only, same CompletableFuture as Java 8-9
var future = CompletableFuture.supplyAsync(() -> "hello").thenApply(String::toUpperCase);
System.out.println(future.join()); // HELLO
```

**Best Practice:**
- Don't treat Java 10 as a concurrency release. Keep the Java 8-9 patterns.
- Don't use `var` to hide the type of a future / executor when the type matters for reading. `CompletableFuture<String> f = ...` is clearer than `var f = ...` in async chains.

---

## 3. Java 11 (2018, LTS) — Standard Async HTTP

### What
- **`java.net.http.HttpClient`** — final in Java 11 (incubator in Java 9-10). `HttpClient`, `HttpRequest`, `HttpResponse`, `sendAsync` returning `CompletableFuture`.
- **`CompletableFuture` improvements from Java 9, now on an LTS** — `orTimeout`, `completeOnTimeout`, `delayedExecutor` are now usable on a supported LTS.
- **No new `java.util.concurrent` API in Java 11 itself** — the concurrency story in Java 11 is `HttpClient` + Java 9 timeouts, on LTS.

### Why
Before Java 11, there was no standard async HTTP client. Every project picked a different library, with a different async model. Java 11 gives you one client that returns `CompletableFuture`, so it fits the Java 8 chaining model directly, on an LTS you can actually stay on.

### Benefit
- Async by default with `sendAsync`. No thread blocked waiting for HTTP.
- Returns `CompletableFuture` — chain, combine, timeout, handle errors the Java 8 way.
- HTTP/2 support, same client for sync and async.

### Downside
- `sendAsync` still needs an executor if you want control over the pool. Default uses the common pool.
- Response body handling is explicit — you choose `BodyHandlers.ofString()`, `ofByteArray()`, `discarding()`, and so on.

### Status in Java 11
| API | Verdict |
| :--- | :--- |
| `HttpClient`, `sendAsync`, `HttpRequest`, `HttpResponse` | **STILL CORE for async HTTP in Java 11.** |
| `CompletableFuture.orTimeout` / `completeOnTimeout` on LTS | **STILL CORE.** |
| Blocking `send()` for many parallel HTTP calls | **NEVER USE as a pattern. Use `sendAsync` + `allOf`.** |
| Custom HTTP async wrapper library, when standard `HttpClient` fits | **LEGACY for new code. Use standard `HttpClient`.** |
| `Thread.stop()`, `suspend()`, `resume()` | **DEPRECATED / DO NOT USE.** |
| `ThreadGroup` as API, `Hashtable` / `Vector` / `Timer` for new code | **LEGACY. Do not use for new code.** |
| `java.util.concurrent` APIs removed in Java 11 | **NONE. No concurrency API was removed in Java 11.** |

---

### 3.1 HttpClient — Standard Async HTTP

**Real sample — async HTTP + chaining:**

```java
import java.net.*;
import java.net.http.*;
import java.util.concurrent.*;

public class HttpClientDemo {
    public static void main(String[] args) {
        HttpClient client = HttpClient.newBuilder()
            .executor(Executors.newFixedThreadPool(4)) // control the pool, don't rely on default
            .build();

        HttpRequest req = HttpRequest.newBuilder()
            .uri(URI.create("https://example.com"))
            .GET()
            .build();

        CompletableFuture<String> body = client
            .sendAsync(req, HttpResponse.BodyHandlers.ofString())
            .thenApply(HttpResponse::body)
            .exceptionally(ex -> "fallback");

        System.out.println(body.join());
    }
}
```

**Real sample — many requests in parallel:**

```java
import java.net.*;
import java.net.http.*;
import java.util.*;
import java.util.concurrent.*;

public class ParallelHttpDemo {
    public static void main(String[] args) {
        HttpClient client = HttpClient.newHttpClient();
        List<String> urls = List.of("https://example.com", "https://example.org");

        List<CompletableFuture<String>> futures = urls.stream()
            .map(url -> client.sendAsync(
                HttpRequest.newBuilder().uri(URI.create(url)).build(),
                HttpResponse.BodyHandlers.ofString())
                .thenApply(HttpResponse::body))
            .toList();

        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        List<String> bodies = futures.stream().map(CompletableFuture::join).toList();
        System.out.println(bodies);
    }
}
```

**Best Practice:**
- Use `sendAsync` for parallel HTTP. Never loop `send()` for many URLs — that serializes them.
- Give `HttpClient` an explicit executor in production. Don't rely on the default pool for I/O.
- Chain with `thenApply` / `exceptionally`, same as any `CompletableFuture`. Don't call `join()` immediately after `sendAsync()`.
- Use `orTimeout` for HTTP calls that must not hang forever.

---

## 4. Best Practices — The Cheat List (Only Java 9-11)

### Flow / Backpressure
- Consumer requests in small batches. Never `Long.MAX_VALUE` unless you truly need no backpressure.
- Handle `onError`, `cancel()` on shutdown.
- Don't do slow work in `onNext` without reducing demand.

### HttpClient / CompletableFuture
- `sendAsync` for parallel HTTP, `allOf` to wait for many.
- Explicit executor for `HttpClient` in production.
- `orTimeout` for any async work that must not hang. `completeOnTimeout` only with a safe fallback + logging.
- Chain, don't block immediately. Handle errors at the end.

### Modules
- Use public `java.util.concurrent` API. Don't reach into JDK internals.
- Open only the packages that need reflection, only to the modules that need it.

### Deprecated / Legacy — Never in New Java 9-11 Code
- Hand-rolled timeout with a scheduler + `complete()` — use `orTimeout`.
- Blocking `send()` loop for many HTTP calls — use `sendAsync`.
- `Flow` with unbounded demand — that is no backpressure.
- Reaching into JDK internals for locks / atomics — deprecated / encapsulated.
- `--add-opens` as a permanent fix.
- `Thread.stop()` / `suspend()` / `resume()` — deprecated.
- `ThreadGroup` as API, `Hashtable` / `Vector` / `Timer` for new code — legacy.

---

## 5. Tips — 30-Second Answers (Only Java 9-11)

1.  **"What is backpressure?"**
    > Consumer controls the rate. Producer sends at most what consumer requested. Without it, a fast producer floods a slow consumer.

2.  **"`Flow` vs `CompletableFuture`?"**
    > `CompletableFuture` = one result. `Flow` = many results as a stream, with demand. Use `Flow` only for streaming + rate problems.

3.  **"Why must a subscriber call `request(n)`?"**
    > No request = no items. `Flow` sends nothing until the consumer asks. Forgetting `request()` is the most common `Flow` bug.

4.  **"Why not `request(Long.MAX_VALUE)`?"**
    > That is unbounded demand — no backpressure. The producer can flood you again.

5.  **"Why `HttpClient.sendAsync` over `send()`?"**
    > `send()` blocks one thread per request. `sendAsync` returns `CompletableFuture`, so many requests run in parallel and you chain results.

6.  **"What does `orTimeout` cancel?"**
    > It completes the future with a timeout, but does not cancel the underlying task by itself. Design tasks to be interruptible if that matters.

7.  **"`completeOnTimeout` vs `orTimeout`?"**
    > `completeOnTimeout` gives a fallback value and the chain continues as success. `orTimeout` fails the future with `TimeoutException`.

8.  **"When to use `Flow` vs `BlockingQueue`?"**
    > `BlockingQueue` blocks threads to apply backpressure. `Flow` applies backpressure without blocking, via demand. Use `Flow` for async streaming, queue for passing work between threads.

9.  **"What did modules change for concurrency?"**
    > JDK internals are encapsulated. Use public `java.util.concurrent` API instead of reaching into internal classes.

10. **"What did Java 10 add for concurrency?"**
    > Nothing. No new concurrency API in Java 10. Keep using Java 8-9 APIs.

---

## 6. Cheat Sheet — Java 9 to Java 11 Only

### Timeline

| Version | Year | Headline | Still Use? |
| :--- | :--- | :--- | :--- |
| **9** | 2017 | `Flow`, `CompletableFuture` timeouts, Modules | **Yes — Flow contract + timeouts** |
| **10** | 2018 | No new concurrency API | **No new API — keep 8-9 APIs** |
| **11** | 2018, LTS | `HttpClient` final | **Yes — async HTTP** |

### API Quick Reference

| API | What it does | Verdict |
| :--- | :--- | :--- |
| `Flow.Publisher` / `Subscriber` / `Subscription` | Async stream contract | Core as contract |
| `SubmissionPublisher` | Built-in publisher | Core for simple cases |
| `HttpClient.sendAsync` | Async HTTP, returns `CompletableFuture` | Core (Java 11) |
| `CompletableFuture.orTimeout` | Fail after timeout | Core |
| `CompletableFuture.completeOnTimeout` | Fallback after timeout | Core, log when it fires |
| `CompletableFuture.delayedExecutor` | Run after delay | Core |
| `CompletableFuture.copy` / `newIncompleteFuture` | Copy / create incomplete future | Core |
| `module-info.java` | Module boundary | Know it, don't fight it |
| `Thread.stop/suspend/resume` | Force stop | **NEVER — deprecated** |
| JDK internals (`Unsafe`) for locks / atomics | Hand-rolled concurrency | **NEVER — encapsulated** |
| `ThreadGroup` / `Hashtable` / `Vector` / `Timer` | Legacy APIs | **NEVER for new code** |

### Decision Tree (Only Java 9-11)

```text
Need async work?
  ├─ One result? -> CompletableFuture, orTimeout for safety
  ├─ Many HTTP calls? -> HttpClient.sendAsync + allOf
  └─ Streaming many results, fast producer? -> Flow, request in small batches

Need backpressure?
  ├─ Blocking threads ok? -> BlockingQueue (bounded)
  └─ Async, no blocking? -> Flow, demand via request(n)

Need timeout?
  └─ orTimeout / completeOnTimeout, not hand-rolled scheduler

Need to reach JDK internals for concurrency?
  └─ Don't. Use public java.util.concurrent API.
```
