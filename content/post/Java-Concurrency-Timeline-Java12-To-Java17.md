---
title: "Java Thread & Concurrency Timeline: Java 12 to Java 17"
date: 2026-11-15T18:00:00Z
lastmod: 2026-11-19T10:00:00Z
draft: false
categories: ["java", "concurrency"]
tags: ["java", "concurrency", "threads", "records", "sealed", "pattern-matching", "java17"]
series: ["java concurrency timeline"]
author: "xmoooz"
description: "Java 12 to Java 17 concurrency focus: records for immutable messages, sealed classes for exhaustive task handling, pattern matching and switch expressions for dispatch, and what did not change in java.util.concurrent — concurrency focus only."
url: 2026/11/15/java-concurrency-timeline-java12-to-java17
---

## 0. Global Vision — Java 12 to Java 17 in 2 Minutes

This file covers only Java 12 to Java 17. Nothing before, nothing after. Concurrency focus only — no general language tour.

The story inside this window is different from Java 5 or Java 8:

**Java 12-17 adds almost no new `java.util.concurrent` API. It changes the *shape* of concurrent code: how you represent messages between threads, and how you handle every possible result of an async task.**

| Problem in Java 8-11 | What Java 12-17 gives you |
| :--- | :--- |
| Messages between threads are mutable classes — easy to mutate after sending | **Records** — immutable, compact messages by default |
| Task results are `Object` / `String` codes — easy to forget a case | **Sealed classes + pattern matching** — compiler checks you handled every result type |
| Dispatch on task result is nested `if` / `instanceof` + cast | **`switch` expressions + pattern matching for `instanceof`** — exhaustive, no cast, expression form |
| `CompletableFuture` chains are hard to read when branching on result type | **Sealed result types + `switch`** — one place that handles every outcome |

### The mental shift in 12-17

| In Java 8-11 | In Java 12-17 |
| :--- | :--- |
| You pass mutable objects between threads and hope nobody mutates them | You pass records — immutable by construction |
| You handle async results with `if (x instanceof Y)` + cast | You handle them with pattern matching — no cast, compiler-checked |
| You hope you covered every task outcome | Sealed hierarchy + `switch` makes the compiler tell you when you missed one |
| You write boilerplate for message classes (`equals`, `hashCode`, `toString`) | Record generates it, correctly, for immutable data |

### The one sentence to remember for each part

- **Records:** Immutable messages between threads, with no boilerplate.
- **Sealed classes:** A closed set of result types. The compiler knows all of them.
- **Pattern matching + `switch` expressions:** Dispatch on result type, exhaustively, without casts.
- **What did *not* change:** `CompletableFuture`, `ExecutorService`, `Flow`, `ConcurrentHashMap` are still the concurrency core. Java 12-17 does not replace them — it makes the code around them safer.

### What to take away from this era

- The biggest concurrency win in 12-17 is not a new pool or lock. It is making illegal states unrepresentable: immutable messages + closed result types.
- If a message can be mutated after sending, it *will* be mutated. Records remove that class of bug.
- If a `switch` can miss a case, it *will* miss one in production. Sealed + `switch` removes that class of bug.

> Honest note: Java 12-17 is a language era, not a concurrency-API era. If you are looking for a new executor, lock, or queue, you won't find one here. The value is in how you write the code that *uses* those APIs.

---

## 1. Records — Immutable Messages Between Threads

### What
`record` (Java 16, final): compact class declaration for immutable data. `record OrderCreated(String orderId, int amount) {}`.

### Why
Concurrent code passes messages between threads: tasks, events, results, commands. If those messages are mutable, any thread can mutate them after sending — race, stale read, or a message that changes mid-processing. Before records, writing a correct immutable message class meant boilerplate: `final` fields, constructor, getters, `equals`, `hashCode`, `toString`, no setters. Most code skipped it and used mutable classes.

### Benefit
- Immutable by construction: fields are `private final`, no setters.
- No boilerplate: `equals`, `hashCode`, `toString` generated correctly.
- Compact: one line declares the message shape.
- Safe to share: an immutable record can be passed between threads without locking.

### Downside
- Records are final and shallowly immutable. If a record holds a mutable list / map, that list can still be mutated. Immutability is only as deep as the components.
- Records are not for entities with identity / mutable lifecycle (e.g. a bank account). They are for messages, events, results, snapshots.
- No default constructor, no setters — if you need mutation, a record is the wrong tool.

### Status in Java 12-17
| API | Verdict |
| :--- | :--- |
| `record` for messages, events, results, snapshots, task inputs | **CORE. This is the Java 16+ way to write concurrent messages.** |
| Mutable class for a message passed between threads | **DO NOT USE as a pattern.** Use a record. |
| Record holding a mutable collection, exposed directly | **DO NOT USE.** Copy / wrap it, or use an immutable collection. |

### Real Sample Code — Record as a Message

```java
import java.util.concurrent.*;

// Message between threads: immutable, no boilerplate
record OrderCreated(String orderId, int amount) {}
record OrderFailed(String orderId, String reason) {}

public class RecordMessageDemo {
    public static void main(String[] args) throws Exception {
        BlockingQueue<OrderCreated> queue = new LinkedBlockingQueue<>();
        queue.put(new OrderCreated("o-1", 100));

        // Consumer: message cannot be mutated by producer after sending
        OrderCreated msg = queue.take();
        System.out.println(msg.orderId() + " / " + msg.amount());
        // msg.amount() = 200; // DOES NOT COMPILE — no setter, field is final
    }
}
```

### Real Sample Code — Record + CompletableFuture Result

```java
import java.util.concurrent.*;

record UserLoaded(String userId, String name) {}
record UserMissing(String userId) {}

public class RecordFutureDemo {
    public static void main(String[] args) {
        CompletableFuture<Object> f = CompletableFuture.supplyAsync(() -> {
            // In real code this would be a sealed result type (Section 2)
            // Here: record makes the success value immutable and clear
            return (Object) new UserLoaded("u-1", "Ada");
        });
        System.out.println(f.join());
    }
}
```

**Best Practice:**
- Use records for everything that crosses a thread boundary: task input, event, result, snapshot, command.
- If a record holds a collection, copy it in the compact constructor or use an immutable copy. Don't expose a mutable list from a record.
- Don't use records for mutable entities. Use a regular class with explicit locking for those.
- Don't put blocking / I/O logic in a record. Records are data, not workers.

```java
// GOOD: defensive copy in compact constructor
record OrderBatch(String batchId, List<String> orderIds) {
    OrderBatch {
        orderIds = List.copyOf(orderIds); // immutable copy, caller cannot mutate after sending
    }
}
```

---

## 2. Sealed Classes — A Closed Set of Task Results

### What
`sealed` classes / interfaces + `permits` (Java 17, final): a type that can only be implemented by a fixed, known set of subtypes.

### Why
Async code has a fixed set of outcomes: success, not-found, timeout, validation error. Before sealed classes, you represented that as `Object`, `String` codes, or an open hierarchy anyone could extend. The compiler could not tell you when you forgot an outcome. With sealed classes, the compiler knows *all* subtypes — and a `switch` over them can be checked for exhaustiveness.

### Benefit
- Closed hierarchy: no surprise subtype from another module.
- Exhaustive `switch`: compiler errors if you miss a case (with pattern matching, Section 3).
- Clear contract: "a task result is *exactly* one of these."

### Downside
- Sealed is restrictive by design. If you need an open hierarchy (plugins, user extensions), sealed is the wrong tool.
- Sealed + `switch` exhaustiveness needs pattern matching (Java 17). In earlier versions in this range, the check is weaker.

### Status in Java 12-17
| API | Verdict |
| :--- | :--- |
| `sealed` for task results, events, commands with a fixed set of outcomes | **CORE in Java 17.** |
| Open hierarchy / `String` codes for task results | **DO NOT USE as a pattern.** Use sealed types. |

### Real Sample Code — Sealed Result Type

```java
// A task result is exactly one of these three. Nothing else can implement it.
sealed interface TaskResult permits TaskResult.Ok, TaskResult.NotFound, TaskResult.Failed {
    record Ok(String value) implements TaskResult {}
    record NotFound(String id) implements TaskResult {}
    record Failed(String id, String reason) implements TaskResult {}
}

public class SealedDemo {
    static TaskResult load(String id) {
        if (id.equals("1")) return new TaskResult.Ok("Ada");
        if (id.equals("2")) return new TaskResult.NotFound(id);
        return new TaskResult.Failed(id, "db error");
    }
    public static void main(String[] args) {
        System.out.println(load("1"));
        System.out.println(load("2"));
    }
}
```

**Best Practice:**
- Use sealed types for results, events, and commands that have a fixed set of cases.
- Keep sealed hierarchies small. If you have 20 subtypes, the hierarchy is too broad — split it.
- Put sealed result types in the same package / module as the code that produces them, so the closed set is visible.

---

## 3. Pattern Matching + Switch Expressions — Exhaustive Dispatch

### What
Pattern matching for `instanceof` (Java 16, final): `if (x instanceof TaskResult.Ok ok)`. `switch` expressions (Java 14, final): `switch` as an expression that returns a value, with `->` arms. Pattern matching for `switch` is preview in Java 17, not final — see note below.

### Why
Dispatching on async result type used to be nested `if` + `instanceof` + cast. Easy to forget a case, easy to cast wrong. Pattern matching removes the cast. `switch` expressions make dispatch a single expression that returns a value.

> Note on scope: Pattern matching *for `switch`* (e.g. `case Ok(String v)`) is preview in Java 17, not final. This file uses only what is final in 12-17: pattern matching for `instanceof`, and `switch` expressions over sealed types with explicit handling. It does not rely on preview `switch` patterns.

### Benefit
- No cast: `instanceof` binds the variable for you.
- `switch` expression returns a value — no fall-through, no forgotten `break`.
- With sealed types, the compiler can check you handled every case (when using `switch` over the sealed type).

### Downside
- `switch` expressions are still easy to misuse if you use them for side effects instead of returning a value.
- Pattern matching for `switch` being preview in 17 means code that uses it needs `--enable-preview` in 17 — don't rely on it in this era for production code in this file's scope.

### Status in Java 12-17
| API | Verdict |
| :--- | :--- |
| Pattern matching for `instanceof` (Java 16) | **CORE. Use it instead of `instanceof` + cast.** |
| `switch` expressions (Java 14) | **CORE. Use them for dispatch on sealed results.** |
| Pattern matching for `switch` (preview in 17) | **DO NOT USE as a required pattern in this file.** Preview only in 17. |

### Real Sample Code — Pattern Matching for `instanceof`

```java
// Before: instanceof + cast
// if (result instanceof TaskResult.Ok) {
//     TaskResult.Ok ok = (TaskResult.Ok) result;
//     ...
// }

// Java 16+: instanceof binds the variable, no cast
static String describe(TaskResult result) {
    if (result instanceof TaskResult.Ok ok) {
        return "ok: " + ok.value();
    }
    if (result instanceof TaskResult.NotFound nf) {
        return "not found: " + nf.id();
    }
    if (result instanceof TaskResult.Failed f) {
        return "failed: " + f.id() + " / " + f.reason();
    }
    throw new IllegalStateException("unknown: " + result);
}
```

### Real Sample Code — Switch Expression Over Sealed Type

```java
static String describeWithSwitch(TaskResult result) {
    return switch (result) {
        case TaskResult.Ok ok -> "ok: " + ok.value();
        case TaskResult.NotFound nf -> "not found: " + nf.id();
        case TaskResult.Failed f -> "failed: " + f.id() + " / " + f.reason();
        // No default needed: sealed type is closed, compiler knows all cases
        // (in Java 17, exhaustiveness for sealed types in switch is supported
        //  for the forms shown here; do not rely on preview switch patterns)
    };
}
```

### Real Sample Code — Records + Sealed + CompletableFuture Together

```java
import java.util.concurrent.*;

sealed interface LoadResult permits LoadResult.Ok, LoadResult.Missing {
    record Ok(String name) implements LoadResult {}
    record Missing(String id) implements LoadResult {}
}

public class CombinedDemo {
    static CompletableFuture<LoadResult> loadAsync(String id) {
        return CompletableFuture.supplyAsync(() ->
            id.equals("1") ? (LoadResult) new LoadResult.Ok("Ada") : new LoadResult.Missing(id));
    }
    public static void main(String[] args) {
        LoadResult result = loadAsync("1").join();
        // Dispatch: exhaustive, no cast, no String codes
        String text = switch (result) {
            case LoadResult.Ok ok -> "found: " + ok.name();
            case LoadResult.Missing m -> "missing: " + m.id();
        };
        System.out.println(text);
    }
}
```

**Best Practice:**
- Use `switch` expressions for dispatch on sealed results, not nested `if` / `else`.
- Return a value from `switch`, don't use it only for side effects.
- Don't use preview `switch` pattern matching in 17 production code in this file's scope. Use `instanceof` patterns + `switch` over sealed types, which are final.

---

## 4. What Did Not Change — And What to Keep Using

### What Did Not Change in 12-17
- `CompletableFuture`, `ExecutorService`, `Flow`, `ConcurrentHashMap`, `BlockingQueue`, `StampedLock`, `LongAdder` — all still the concurrency core. No replacement in 12-17.
- No new executor, lock, or queue in this range that replaces Java 5-8 APIs for general concurrent code.

### What This Means in Practice
- Don't rewrite working `CompletableFuture` chains to use records / sealed types everywhere. Use records / sealed where they remove a real bug class (mutable messages, missed cases). Keep the async core as-is.
- Don't use records / sealed as a concurrency *mechanism*. They are a *shape* for data. The mechanism is still `CompletableFuture` / `ExecutorService` / `Flow`.

### Status in Java 12-17
| Topic | Verdict |
| :--- | :--- |
| `CompletableFuture` / `ExecutorService` / `Flow` / concurrent collections | **STILL CORE. Keep using them.** |
| Records / sealed for messages / results | **CORE in 16-17. Use them for new concurrent messages.** |
| Rewriting all concurrent code to records / sealed | **DO NOT USE as a goal.** Use them where they fix a bug class. |

### Real Sample Code — Old Message vs New Message (Same Mechanism)

```java
// Before (mutable message — race if producer mutates after sending):
// class Msg { public String id; public int amount; }

// Java 16+ (immutable message — safe to share):
record Msg(String id, int amount) {}

// Mechanism is unchanged:
BlockingQueue<Msg> queue = new LinkedBlockingQueue<>();
queue.put(new Msg("m-1", 10));
Msg taken = queue.take();
```

---

## 5. Best Practices — The Cheat List (Only Java 12-17)

### Messages & Results
- Messages between threads: records, immutable, defensive copy for collections.
- Results with fixed outcomes: sealed types, not `String` codes / `Object`.
- Dispatch: `switch` expression + pattern matching for `instanceof`, no cast, no nested `if`.

### Concurrency Core (Unchanged)
- Keep using `CompletableFuture`, `ExecutorService`, `Flow`, concurrent collections. They are still the core.
- Don't block the common pool. Don't mutate shared state in parallel streams. Same rules as Java 8-11 still apply.

### Never in New Java 12-17 Code
- Mutable message classes passed between threads — use records.
- `String` / `int` codes for task outcomes — use sealed types.
- `instanceof` + manual cast — use pattern matching for `instanceof`.
- `switch` statements with fall-through for dispatch — use `switch` expressions.
- Preview `switch` pattern matching as a required pattern in Java 17 — it is preview in this era.
- Rewriting all concurrent code to records / sealed — use them where they fix a bug class.

---

## 6. Tips — 30-Second Answers (Only Java 12-17)

1.  **"Why records for concurrent messages?"**
    > Records are immutable by construction. A message that cannot be mutated after sending cannot be raced on.

2.  **"What if a record holds a mutable list?"**
    > Then it is only shallowly immutable. Copy the list in the compact constructor with `List.copyOf()`.

3.  **"Why sealed classes for task results?"**
    > A sealed type is a closed set. The compiler knows all outcomes, so a `switch` can be checked for exhaustiveness. No surprise subtype.

4.  **"Why pattern matching for `instanceof`?"**
    > It binds the variable and removes the cast. Less code, fewer cast bugs.

5.  **"Why `switch` expressions over `switch` statements?"**
    > `switch` expression returns a value, has no fall-through, and — with sealed types — can be checked for exhaustiveness.

6.  **"What concurrency API did Java 12-17 add?"**
    > Almost none for general concurrent code. The win is language shape: records + sealed + pattern matching make the code around `CompletableFuture` / `ExecutorService` safer.

7.  **"When not to use a record?"**
    > When the data needs mutation, identity, or a lifecycle. Records are for messages, events, results, snapshots — not for mutable entities.

8.  **"When not to use sealed?"**
    > When the hierarchy needs to be open (plugins, user extensions). Sealed is restrictive by design.

9.  **"Is pattern matching for `switch` final in Java 17?"**
    > No. It is preview in Java 17. This file uses only what is final in 12-17: pattern matching for `instanceof` + `switch` expressions.

10. **"Should I rewrite all concurrent code to records / sealed?"**
    > No. Use them for new messages / results where they remove a bug class. Keep the async core (`CompletableFuture`, `ExecutorService`, `Flow`) as-is.

---

## 7. Cheat Sheet — Java 12 to Java 17 Only

### Timeline

| Part | Headline | Verdict in This Era |
| :--- | :--- | :--- |
| **Records (16)** | Immutable messages | **Core for messages** |
| **Sealed classes (17)** | Closed result types | **Core for fixed outcomes** |
| **Pattern matching for `instanceof` (16)** | No cast | **Core** |
| **`switch` expressions (14)** | Exhaustive dispatch | **Core** |
| **Concurrency core (5-11)** | `CompletableFuture`, `ExecutorService`, `Flow`, collections | **Still core, unchanged** |

### API Quick Reference

| API | What it does | Verdict |
| :--- | :--- | :--- |
| `record` | Immutable data, no boilerplate | Core for messages / results |
| `sealed` + `permits` | Closed hierarchy | Core for fixed outcomes |
| `instanceof` pattern | Bind + no cast | Core |
| `switch` expression | Dispatch, returns value | Core |
| `CompletableFuture` / `ExecutorService` / `Flow` | Async core | Still core, unchanged |

### Decision Tree (Only Java 12-17)

```text
Need to pass data between threads?
  └─ Record, defensive copy for collections

Need to represent task outcome?
  ├─ Fixed set of outcomes? -> Sealed type + records for each case
  └─ Open / extensible? -> Regular interface, not sealed

Need to dispatch on result type?
  ├─ Pattern matching for instanceof, no cast
  └─ switch expression, return a value, no fall-through

Need async work / pooling / streaming?
  └─ Same as Java 8-11: CompletableFuture / ExecutorService / Flow
```
