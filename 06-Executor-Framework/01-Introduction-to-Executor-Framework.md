---
# 🎯 6.1: Introduction to the Executor Framework

## 🗺️ Learning Path Position

**📍 You Are Here:** 6.1 - Introduction to the Executor Framework

**✅ Prerequisites (Must Complete First):**
- [1.2: Creating & Managing Threads](../01-Foundation-Threading-Basics/02-Creating-and-Managing-Threads.md) - Manual ga threads ela create cheyalo gurtundali.
- [2.2: The `synchronized` Keyword](../02-Synchronization-Thread-Safety/02-Synchronized-Keyword.md) - Basic synchronization concepts gurtu techuko.

**🔜 Coming After This:**
- [6.2: Understanding Thread Pools](./02-Understanding-Thread-Pools.md) - `newFixedThreadPool`, `newCachedThreadPool` lanti different pool types gurinchi deep dive cheddam.
- [6.3: ThreadPoolExecutor Deep Dive](./03-ThreadPoolExecutor-Deep-Dive.md) - Thread pools ni fine-tune cheyadaniki unna core components ni explore cheddam.

**⏱️ Estimated Time:** 2 hours
**📊 Difficulty Level:** 🟢 Beginner

---

## 🤔 What & Why (Asal Enduku Ee Topic?)

### The Problem: Manual Thread Management is a Nightmare! 😫
Mawa, manam ippativaraku `new Thread()` use chesi threads create chesam. Chinna applications ki idi okay, kaani pedda, production-level systems lo idi chala pedda headache:

1.  **Resource Waste 💸:** Prathi chinna task ki oka kotha thread create chesi, pani aypogane destroy cheyadam chala expensive. Thread creation anedi OS level lo konchem heavy operation.
2.  **No Control 🕹️:** Enni threads run avtunnayo mana control lo undadu. Okavela 1000 tasks vachi, 1000 threads create aite, system resources (CPU, memory) anni aypotayi, and application crash avtundi (`OutOfMemoryError`).
3.  **Complex Logic 🤯:** Thread lifecycle ni manage cheyadam, tasks ni queue cheyadam, results ni collect cheyadam, shutdown logic rayadam... antha mana meede untundi. Chala boilerplate code rayali.

### The Solution: The Executor Framework - A "Manager" for Your Threads 👨‍💼
Java 5 lo `java.util.concurrent` package tho paatu `Executor Framework` introduce chesaru. Idi thread management ni mana nunchi teskuni, ade handle chestundi.

**Core Idea:** Task submission ni task execution nunchi separate cheyadam.
- **Mana Pani (Developer):** Tasks (`Runnable` or `Callable`) ni create chesi, Executor ki ivvadam.
- **Executor Pani:** Ee tasks ni execute cheyadaniki threads ni ela manage cheyalo ade chuskuntundi (thread creation, reuse, scheduling).

### Real-World Analogy: Restaurant Kitchen 🍽️
Imagine a popular restaurant.
- **Manual Threading (Bad Approach):** Prathi order (task) ki, restaurant owner oka kotha chef (`Thread`) ni hire cheskuni, pani aypogane pampistadu. Idi chala inefficient and expensive!
- **Executor Framework (Good Approach):** Restaurant lo already కొంతమంది chefs (`Thread Pool`) untaru. Orders (tasks) రాగానే, manager (`ExecutorService`) aa orders ni chefs ki assign chestadu. Chef khali lekapothe order queue lo wait chestundi. Chef pani aypogane, next order teskuntadu. Kotha chefs ni hire cheyadam, fire cheyadam undadu. Existing chefs ne efficiently reuse cheskuntaru.

---

## 📚 Detailed Explanation

### The Core Components

Executor Framework lo 3 main building blocks unnayi:

1.  **`Executor` (Interface):**
    - Idi framework ki foundation. Deentlo okate okka method untundi: `void execute(Runnable command)`.
    - Deeni main purpose: "Naku oka `Runnable` task ivvu, nenu daanini execute chesta". Ekkada, eppudu, ela execute cheyalo cheppadu. Just oka contract matrame.

2.  **`ExecutorService` (Interface):**
    - Idi `Executor` interface ni extend chestundi. Ante `execute()` method deentlo kuda untundi.
    - Kaani idi inka chala powerful features istundi:
      - **Task Submission with Results:** `Runnable` matrame kakunda, result ni return chese `Callable` tasks ni kuda submit cheyochu (`submit()` method). Result anedi oka `Future` object lo vastundi. ( దీని గురించి మనం నెక్స్ట్ phase లో నేర్చుకుంటాం!)
      - **Lifecycle Management:** Executor service ni graceful ga shutdown cheyadaniki methods (`shutdown()`, `shutdownNow()`). Idi chala important!
      - **Batch Submission:** Multiple tasks ni okesari submit cheyochu (`invokeAll()`, `invokeAny()`).

3.  **`Executors` (Utility Class):**
    - Idi oka factory class anuko. Manam direct ga `ExecutorService` implementations ni create cheyakunda, ee class lo unna static factory methods use chesi common configurations unna thread pools ni create cheskovachu.
    - Examples: `Executors.newSingleThreadExecutor()`, `Executors.newFixedThreadPool(10)`.

#### 🧠 Mental Model Diagram: Components

```mermaid
graph TD
    subgraph You (Developer)
        Task1[Runnable Task]
        Task2[Callable Task]
    end

    subgraph Executor Framework
        ExecutorService[ExecutorService]
        subgraph Thread Pool
            Thread1[Thread 1]
            Thread2[Thread 2]
        end
    end

    Task1 -->|submit()| ExecutorService
    Task2 -->|submit()| ExecutorService
    ExecutorService -->|assigns| Thread1
    ExecutorService -->|assigns| Thread2

    style You fill:#90EE90
    style ExecutorService fill:#FFD700
```

---

## 💻 Code Examples

### Example 1: The Old Way vs. The New Way ✨

#### ❌ Failure Case: Manual Thread Creation Hell
**Scenario:** Manam 5 different tasks ni execute cheyali. Pratidi oka separate thread lo.
```java
// File: com/example/manual/OldWay.java
package com.example.manual;

public class OldWay {
    public static void main(String[] args) {
        System.out.println("Main thread started: " + Thread.currentThread().getName());

        // ❌ PRATHI TASK KI OKA KOTHA THREAD!
        for (int i = 1; i <= 5; i++) {
            final int taskId = i;
            Thread taskThread = new Thread(() -> {
                System.out.println("Executing task " + taskId + " in thread: " + Thread.currentThread().getName());
                try {
                    Thread.sleep(1000); // Simulating work
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
            taskThread.start(); // Thread ni start cheyadam marchipothe, task run avvadu!
        }

        System.out.println("Main thread finished: " + Thread.currentThread().getName());
    }
}
```
**Output (Example):**
```
Main thread started: main
Main thread finished: main
Executing task 1 in thread: Thread-0
Executing task 2 in thread: Thread-1
Executing task 3 in thread: Thread-2
Executing task 4 in thread: Thread-3
Executing task 5 in thread: Thread-4
```
**Why This is Bad:**
1.  **Boilerplate Code:** Task logic kanna thread creation logic ekkuva undi.
2.  **Resource Heavy:** 5 tasks kosam 5 separate threads create chesam. 1000 tasks unte? App antha crash avtundi.
3.  **No Graceful Shutdown:** Main thread aypoyina, background threads inka run avtune unnayi. Vatini manage cheyadaniki maname extra logic rayali.

---

#### ✅ Success Case: Using `newSingleThreadExecutor`
**Scenario:** Same 5 tasks, kaani ee sari Executor Framework tho.
```java
// File: com/example/executor/NewWay.java
package com.example.executor;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class NewWay {
    public static void main(String[] args) {
        System.out.println("Main thread started: " + Thread.currentThread().getName());

        // 1. ✅ Create an ExecutorService. Idi okate thread ni manage chestundi.
        ExecutorService executor = Executors.newSingleThreadExecutor();

        for (int i = 1; i <= 5; i++) {
            final int taskId = i;
            // 2. ✅ Task ni create chey, thread ni kadu!
            Runnable task = () -> {
                System.out.println("Executing task " + taskId + " in thread: " + Thread.currentThread().getName());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            };
            // 3. ✅ Task ni submit chey, Executor chuskuntundi.
            executor.execute(task);
        }

        // 4. 🛑 IMPORTANT! Executor ni shutdown cheyali. Leda app exit avvadu.
        executor.shutdown(); // New tasks accept cheyadu, but existing tasks complete avtayi.

        try {
            // Optional: Main thread ni shutdown complete ayyevaraku wait cheyinchu
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow(); // Wait time aypoyina tasks complete kakapothe, force shutdown.
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }

        System.out.println("Main thread finished: " + Thread.currentThread().getName());
    }
}
```
**Output:**
```
Main thread started: main
Main thread finished: main
Executing task 1 in thread: pool-1-thread-1
Executing task 2 in thread: pool-1-thread-1
Executing task 3 in thread: pool-1-thread-1
Executing task 4 in thread: pool-1-thread-1
Executing task 5 in thread: pool-1-thread-1
```
**Why This is Awesome:**
1.  **Clean Code:** Manam just tasks meeda focus chesam. Thread management antha framework chuskundi.
2.  **Resource Efficient:** Anni 5 tasks ni okate thread (`pool-1-thread-1`) sequentially execute chesindi. Thread ni reuse cheskundi.
3.  **Lifecycle Management:** `shutdown()` and `awaitTermination()` tho graceful shutdown chala easy aypoindi.

#### 🔗 Concept Connections:
- **💭 Uses from previous:** Manam `Runnable` interface gurinchi [Chapter 1.2](../01-Foundation-Threading-Basics/02-Creating-and-Managing-Threads.md) lo nerchukunnam. Ade ikkada task ga use chestunnam.
- **📌 Will be used in:** Ee `ExecutorService` anedi foundation. Manam next `Callable`, `Future` and `CompletableFuture` (Phase 7) lo deeni meede build chestam.

---

## 🐛 Debugging Guide

### Error 1: Application Doesn't Exit (Hangs)
**Symptom:** `main` thread aypoyina, nee program exit avvakunda hang avtundi.
**Meaning:** Idi ante bro, nuvvu `executor.shutdown()` call cheyadam marchipoyav. `ExecutorService` lo unna threads non-daemon threads, so avi running lo unte JVM exit avvadu.
**Solution:**
1.  Nee code lo `ExecutorService` create chesina prathi chota, `finally` block petti andulo `shutdown()` call chey.
2.  Try-with-resources (Java 19+) use cheste inka better, but `ExecutorService` anedi `AutoCloseable` kadu by default, so custom implementation kavali. `finally` is the standard way.

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
try {
    // ... use executor ...
} finally {
    executor.shutdown(); // ✅ Always do this!
}
```

### Error 2: `RejectedExecutionException`
**Symptom:** Executor ki task submit chestunte ee exception vastundi.
**Meaning:** Nuvvu `shutdown()` call chesina tarvata, inko kotha task ni submit cheyadaniki try chestunnav. Once shutdown, an executor won't accept new tasks.
**Solution:**
1.  Nee application logic ni check chesko. Shutdown process start ayyaka kotha tasks enduku generate avtunnayo chudu.
2.  `executor.isShutdown()` or `executor.isTerminated()` lanti methods tho executor state ni check cheskuni, appude tasks submit chey.

---

## 🧪 Testing Examples

Executor framework tho pani chese code ni test cheyadam konchem tricky, endukante adi asynchronous. Manam `CountDownLatch` or `awaitTermination` lanti tools use cheyali.

### Unit Test Example 1: Verifying Task Execution
**Scenario:** Manam oka `TaskSubmitter` class ni test cheddam. Adi task ni executor ki correctly submit chestunda leda ani.
```java
// File: com/example/testing/TaskSubmitter.java
package com.example.testing;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.atomic.AtomicBoolean;

public class TaskSubmitter {
    private final ExecutorService executor;

    public TaskSubmitter(ExecutorService executor) {
        this.executor = executor;
    }

    public void submitTask(AtomicBoolean flag) {
        executor.execute(() -> {
            // Simulate some work
            System.out.println("Task is running...");
            flag.set(true); // Set the flag to indicate execution
        });
    }
}

// File: com/example/testing/TaskSubmitterTest.java
package com.example.testing;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

public class TaskSubmitterTest {
    @Test
    public void testTaskIsExecuted() throws InterruptedException {
        // Arrange
        ExecutorService executor = Executors.newSingleThreadExecutor();
        TaskSubmitter submitter = new TaskSubmitter(executor);
        AtomicBoolean hasRun = new AtomicBoolean(false);

        // Act
        submitter.submitTask(hasRun);

        // Assert
        // We need to wait for the task to complete
        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS); // Wait for the task to run

        assertTrue(hasRun.get(), "The task should have run and set the flag to true.");
    }
}
```
**Why This Works:** Test method lo task submit chesina tarvata, manam `shutdown()` and `awaitTermination()` use chesi, asynchronous task complete ayyevaraku wait chestunnam. Appudu manam result (`hasRun` flag) ni safely check cheyochu.

---

## ❌ Anti-Patterns

### Anti-Pattern 1: Creating Executors in a Loop or Method
**What's Wrong:** Prathi request ki or prathi method call lo `Executors.newFixedThreadPool(10)` lantiవి create cheyadam.
```java
// ❌ BAD CODE
public void handleRequest() {
    ExecutorService executor = Executors.newFixedThreadPool(10); // Don't do this!
    executor.execute(() -> { /* ... */ });
    // ... you'll forget to shut it down, and you create a new pool every time!
}
```
**Why Bad:** Idi manual thread creation (`new Thread()`) kanna worst. Prathi sari kotha thread pool create chestunnav, daaniki memory overhead untundi, and vatini shutdown cheyadam marchipothe resource leak avtundi.
**✅ Correct Way:** ExecutorService ni oka singleton laaga or application scope lo okate instance create chesi, daanini reuse cheyali. Spring lo bean ga define cheyadam is a good practice.

---

## ⚡ Performance & Optimization

**Memory Usage:**
- Prathi thread ki oka separate stack memory (usually 1MB per thread) allocate avtundi. So, 1000 threads unte, 1GB of stack memory use avtundi. Unnecessary ga ekkuva threads create cheyakudadu.
- `ExecutorService` object itself konchem memory teskuntundi (for queue, stats, etc.).

**Optimization Tips:**
- **Reuse Threads:** Executor framework main advantage ide. Thread reuse cheyadam valla, thread creation/destruction overhead taggutundi.
- **Don't use it for trivial tasks:** Oka chinna calculation (`a+b`) ki task create chesi executor ki submit chesthe, aa task chese pani kanna submission overhead ekkuva avtundi. Alantivi direct ga cheyadam better.

---

## ✨ Key Takeaways (Top 5)

1.  **Decoupling is King 👑:** Executor Framework's main power is separating task submission from execution. Idi code ni clean and manageable ga chestundi.
2.  **Threads are Expensive 💸:** `new Thread()` ni vadakam maneyali. Thread pools use chesi resources save cheyali.
3.  **Always, Always Shutdown! 🛑:** `executor.shutdown()` call cheyadam marchipothe, nee app hang avtundi. Use a `finally` block.
4.  **Start Simple:** `Executors` factory methods (like `newSingleThreadExecutor`) are great for learning, but be aware of their limitations (like unbounded queues).
5.  **Task, Not a Thread:** From now on, think about creating `Runnable` tasks, not `Thread` objects. Let the framework handle the threads.

---

## 💪 Practice Exercises

### Exercise 1 (Easy): ⭐
**Problem:** Oka `ExecutorService` (`newSingleThreadExecutor`) create chesi, 1 nunchi 10 varaku numbers ni print cheyadaniki 10 separate tasks submit chey. Final ga, executor ni graceful ga shutdown chey.

**Hints:** 💡
- Loop lo tasks create chey.
- `shutdown()` and `awaitTermination()` use cheyadam marchipoku.

**Solution:**
<details>
<summary>Click to see solution</summary>

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class PrintNumbers {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        try {
            for (int i = 1; i <= 10; i++) {
                final int num = i;
                executor.execute(() -> System.out.println("Printing " + num + " on thread " + Thread.currentThread().getName()));
            }
        } finally {
            executor.shutdown();
        }
        System.out.println("All tasks submitted.");
    }
}
```
</details>

### Exercise 2 (Medium): ⭐⭐
**Problem:** Manual thread creation example (`OldWay.java`) ni `Executors.newFixedThreadPool(3)` use chesi rewrite chey. 5 tasks ni submit chey, and observe how threads are reused.

**Hints:** 💡
- Pool size 3 ante, at a time maximum 3 tasks run avtayi.
- Thread names ni observe chey.

**Solution:**
<details>
<summary>Click to see solution</summary>

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ReworkOldWay {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(3);
        try {
            for (int i = 1; i <= 5; i++) {
                final int taskId = i;
                executor.execute(() -> {
                    System.out.println("Executing task " + taskId + " on thread " + Thread.currentThread().getName());
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });
            }
        } finally {
            executor.shutdown();
        }
        executor.awaitTermination(1, TimeUnit.MINUTES);
        System.out.println("All tasks finished.");
    }
}
// Output will show that the 5 tasks are executed by the 3 threads in the pool (e.g., pool-1-thread-1, 2, 3).
```
</details>

---

## 📝 Quick Reference Card

| Component | Purpose | Key Method(s) |
|---|---|---|
| `Executor` | Basic interface for executing `Runnable`s. | `execute(Runnable)` |
| `ExecutorService`| Extends `Executor`, adds lifecycle management. | `submit()`, `shutdown()`, `shutdownNow()`, `awaitTermination()`|
| `Executors` | Utility class to create pre-configured pools. | `newSingleThreadExecutor()`, `newFixedThreadPool()` |
| **Shutdown Rule**| Always call `shutdown()` in a `finally` block.| `try { ... } finally { executor.shutdown(); }`|

---

## ✅ Checkpoint: Did You Master This?

Munduku velle mundu, ee questions ki confident ga answer cheyagalava leda chusko:

- [ ] Manual thread management lo unna 3 main problems enti?
- [ ] Executor Framework ee problems ni ela solve chestundi?
- [ ] `Executor` ki `ExecutorService` ki teda enti?
- `executor.shutdown()` call cheyakapothe em avtundi?

**✅ Ready?** → [Next: Understanding Thread Pools](./02-Understanding-Thread-Pools.md)

**😕 Need Review?** → Paiki scroll chesi, Restaurant analogy malli chudu.

**💬 Questions?** → Note chesko, manam next concepts tho link cheddam!
