---
# 🎯 10.1: Fork/Join Basics - Divide and Conquer

## 🗺️ Learning Path Position

**📍 You Are Here:** 10.1 - Fork/Join Basics

**✅ Prerequisites (Must Complete First):**
- [6.2: Understanding Thread Pools](../06-Executor-Framework/02-Understanding-Thread-Pools.md) - `Executors.newWorkStealingPool()` gurinchi brief ga chusam, ikkada daani venaka unna `ForkJoinPool` ni deep dive chestam.

**🔜 Coming After This:**
- [10.2: Fork/Join Implementation](./02-Fork-Join-Implementation.md) - Theory ni practice lo petti, `compute()` method ni ela implement cheyalo nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: How to Maximize CPU Power for Big Tasks? 💻
Mawa, imagine neeku oka pedda task undi: oka 10 million elements unna array lo prathi number ni square cheyali. Ee pani antha oka single thread meeda cheste, nee multi-core processor lo unna migatha cores anni idle ga untayi. Chala waste of power!

Ee task ni manam chala chinna chinna sub-tasks ga divide cheyochu (e.g., first 1 million elements ni square chey, next 1 million, and so on). Ee sub-tasks anni parallel ga veru veru CPU cores meeda run avvali. Kaani, ee sub-tasks ni create cheyadam, manage cheyadam, and results ni combine cheyadam ela? Regular `ExecutorService` tho idi konchem complex.

### The Solution: The Fork/Join Framework 🍴
Java 7 lo introduce chesina Fork/Join framework, ee "divide and conquer" algorithm ni implement cheyadaniki design chesaru.
- **Fork:** Oka pedda task ni chinna chinna sub-tasks ga break cheyadam. Ee process recursively jarugutundi, oka "threshold" (minimum size) reach ayyevaraku.
- **Join:** Chinna sub-tasks anni complete ayyaka, vaati results ni kalipi, original task yokka final result ni produce cheyadam.

### The Secret Sauce: Work-Stealing Algorithm 훔치는
Fork/Join framework performance ki main reason ee **work-stealing** algorithm.
- **Traditional Thread Pool:** Anni threads oka central `BlockingQueue` nunchi tasks teskuntayi. High contention unte, ee single queue anedi bottleneck avtundi.
- **Fork/Join Pool:** Prathi worker thread ki oka **local deque** (double-ended queue) untundi.
  - Oka thread tana local deque lo front nunchi tasks teskuni pani chestundi.
  - Oka thread pani aypoyi, daani deque khali aite, adi idle ga undadu. Adi vere worker thread yokka deque lo **last nunchi** (`tail`) oka task ni "dongalistundi" (`steals`).
- **Why this is brilliant:** Deenivalla lock contention chala taggutundi, and CPU cores anni eppudu busy ga untayi, maximizing throughput.

### Real-World Analogy: Multiple Chefs Preparing a Feast 👨‍🍳
Imagine a head chef (`ForkJoinPool`) ki oka pedda pani vachindi: 1000 vegetables cut cheyali.
- **Fork:** Head chef aa pani ni 10 junior chefs (`worker threads`) ki divide chestadu, prathi okariki 100 vegetables isthadu. Prathi junior chef, daanini inka chinna batches (10 vegetables okasari) ga divide cheskuntadu.
- **Work-Stealing:** Chef-A tana 100 vegetables fast ga cut chesesadu. Chef-B inka slow ga chestunnadu. Chef-A idle ga undakunda, Chef-B table deggariki velli, "Nuvvu inka chala cheyali kada, konni naku ivvu" ani, Chef-B yokka uncut vegetables nunchi konni teskuni cut cheyadam start chestadu.
- **Join:** Andaru chefs valla pani aypoyaka, cut chesina vegetables anni kalipi, head chef ki istaru.

---

## 📚 Detailed Explanation: The Core Components

### 1. `ForkJoinPool`
- Idi `ExecutorService` yokka special implementation. Idi work-stealing algorithm ni implement chestundi.
- Generally, manam daani constructor ni direct ga call cheyam. `ForkJoinPool.commonPool()` (Java 8+) ane static method tho JVM-wide common pool ni vadukuntam, or `new ForkJoinPool(parallelism)` tho custom pool create cheskuntam.

### 2. `RecursiveAction`
- **Purpose:** Result return cheyani task kosam. `Runnable` laantidi.
- **Key Method:** `protected void compute()`. Ee method lo manam mana logic rastam.

### 3. `RecursiveTask<V>`
- **Purpose:** Result return chese task kosam. `Callable` laantidi.
- **Key Method:** `protected V compute()`.

### How the `compute()` method works:
```java
if (problem is small enough) {
  // Solve the problem directly (base case)
} else {
  // 1. Fork: Break the problem into two or more sub-tasks
  subtask1 = new MyTask(...);
  subtask2 = new MyTask(...);

  // 2. Invoke the sub-tasks in parallel
  invokeAll(subtask1, subtask2); // or subtask1.fork(); subtask2.compute();

  // 3. Join: Combine the results
  result = subtask1.join() + subtask2.join();
  return result;
}
```

---

## 💻 Code Example: `RecursiveAction`

**Scenario:** Oka `int` array lo unna prathi element ni `10` tho increment cheddam. Result em return avasaram ledu, array ne modify cheddam.

```java
// File: com/example/basics/IncrementAction.java
package com.example.basics;

import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;
import java.util.Arrays;

public class IncrementAction extends RecursiveAction {

    private static final int THRESHOLD = 10; // Base case: process 10 elements at a time
    private final int[] array;
    private final int start;
    private final int end;

    public IncrementAction(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected void compute() {
        if (end - start <= THRESHOLD) {
            // Base case: Do the work directly
            for (int i = start; i < end; i++) {
                array[i] += 10;
            }
        } else {
            // Recursive step: Fork the task
            int mid = start + (end - start) / 2;
            IncrementAction leftTask = new IncrementAction(array, start, mid);
            IncrementAction rightTask = new IncrementAction(array, mid, end);

            // Run sub-tasks in parallel
            invokeAll(leftTask, rightTask);
        }
    }

    public static void main(String[] args) {
        int[] data = new int[100];
        Arrays.fill(data, 1);

        ForkJoinPool pool = new ForkJoinPool();
        IncrementAction task = new IncrementAction(data, 0, data.length);

        // Submit the main task to the pool
        pool.invoke(task);

        System.out.println("First 20 elements after increment: "
            + Arrays.toString(Arrays.copyOfRange(data, 0, 20)));
    }
}
```

---

## ✅ Checkpoint: Did You Master This?

- [ ] "Divide and Conquer" ante nee own words lo explain cheyagalava?
- [ ] Work-stealing algorithm anedi normal thread pool kanna enduku efficient?
- [ ] `RecursiveAction` ki `RecursiveTask` ki teda enti?
- [ ] `compute()` method lo "threshold" or "base case" enduku important?

**✅ Ready?** → [Next: Fork/Join Implementation](./02-Fork-Join-Implementation.md)
