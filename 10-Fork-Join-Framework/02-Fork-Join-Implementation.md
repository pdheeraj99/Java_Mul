---
# 🎯 10.2: Fork/Join Implementation in Practice

## 🗺️ Learning Path Position

**📍 You Are Here:** 10.2 - Fork/Join Implementation

**✅ Prerequisites (Must Complete First):**
- [10.1: Fork/Join Basics](./01-Fork-Join-Basics.md) - `RecursiveTask` and `RecursiveAction` ante ento, and the "divide and conquer" concept teliyali.

**🔜 Coming After This:**
- [10.3: Parallel Streams and Performance](./03-Parallel-Streams-and-Performance.md) - Fork/Join framework ni Java 8 Streams tho ela vadalo chustam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 🤔 What & Why

### The Problem: Theory is Easy, But How Do I Actually Write the Code?
Mawa, manam `compute()` method lo "break the problem" and "combine the results" ani cheppadam easy, kaani daanini code lo ela rayali? `fork()`, `join()`, `invoke()`, `invokeAll()` - ee methods madhyalo teda enti? Ekkada edi vadali? Threshold ni ela decide cheyali? Ee implementation details chala important.

### The Solution: A Practical Guide to `compute()`

A well-structured `compute()` method is the heart of any Fork/Join task. Let's break down the key implementation choices.

#### 1. Choosing a Good Threshold
- **Threshold ante enti?** Adi mana base case. Problem size anedi ee threshold kanna takkuva unte, manam daanini inka divide cheyam, direct ga solve chestam.
- **How to choose?** Idi oka magic number kadu. Idi nee problem type and hardware meeda depend avtundi.
  - **Too large:** Parallelism anedi takkuva untundi. Nee cores antha waste avtayi.
  - **Too small:** Sub-tasks create cheyadaniki and manage cheyadaniki overhead ekkuva aypotundi.
  - **Good starting point:** Problem ni 100 to 10,000 operations unna chunks ga divide cheyadam try chey. Performance testing chesi, nee specific use case ki fine-tune cheyali.

#### 2. `fork()` vs. `invoke()` vs. `invokeAll()`
- **`fork()`:** Asynchronous. Task ni `ForkJoinPool` ki submit chestundi, kaani daani completion kosam wait cheyadu. Ventane return aypotundi.
- **`join()`:** `fork()` chesina task yokka result kosam wait chestundi (blocking call).
- **`invoke()`:** Synchronous. Task ni pool lo execute chesi, adi complete ayyevaraku wait chesi, result ni return chestundi. Basically, `fork()` followed by `join()`.
- **`invokeAll(task1, task2, ...)`:** Multiple tasks ni submit chesi, anni complete ayyevaraku wait chestundi.

**Best Practice Pattern:** Oka task ni rendu sub-tasks ga divide chesinappudu, performance kosam ee pattern vadali:
```java
// DON'T DO THIS (creates too many threads)
left.fork();
right.fork();
result = left.join() + right.join();

// DO THIS (re-uses the current thread for one sub-task)
right.fork(); // Run the right half in a new worker thread
leftResult = left.compute(); // Run the left half in the CURRENT thread
rightResult = right.join(); // Wait for the forked task
result = leftResult + rightResult;
```
Ee second approach lo, `left.compute()` call cheyadam valla, current worker thread anedi idle ga undakunda, ade pani chestundi. Deenivalla unnecessary thread creation and context switching taggutundi.

---

## 💻 Code Example: `RecursiveTask` for Summing an Array

**Scenario:** Manam oka pedda `long` array yokka sum ni parallel ga calculate cheddam. Ee sari manaki result (`long` sum) kavali, so `RecursiveTask<Long>` vadali.

```java
// File: com/example/implementation/ParallelSumTask.java
package com.example.implementation;

import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ParallelSumTask extends RecursiveTask<Long> {

    private static final int THRESHOLD = 10_000;
    private final long[] numbers;
    private final int start;
    private final int end;

    public ParallelSumTask(long[] numbers) {
        this(numbers, 0, numbers.length);
    }

    private ParallelSumTask(long[] numbers, int start, int end) {
        this.numbers = numbers;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            // Base case: Calculate sum sequentially
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += numbers[i];
            }
            return sum;
        } else {
            // Recursive step: Fork the task
            int mid = start + length / 2;
            ParallelSumTask leftTask = new ParallelSumTask(numbers, start, mid);
            ParallelSumTask rightTask = new ParallelSumTask(numbers, mid, end);

            // ✅ Best Practice: Fork one, compute the other
            rightTask.fork(); // Asynchronously execute the right half
            long leftSum = leftTask.compute(); // Synchronously execute the left half in the current thread
            long rightSum = rightTask.join(); // Wait for the right half to complete

            // Join the results
            return leftSum + rightSum;
        }
    }

    public static void main(String[] args) {
        long[] data = new long[10_000_000];
        for (int i = 0; i < data.length; i++) {
            data[i] = i + 1;
        }

        // --- Sequential Calculation (for comparison) ---
        long startTime = System.currentTimeMillis();
        long sequentialSum = 0;
        for (long n : data) sequentialSum += n;
        long endTime = System.currentTimeMillis();
        System.out.println("Sequential Sum: " + sequentialSum + " | Time: " + (endTime - startTime) + " ms");

        // --- Parallel Calculation ---
        ForkJoinPool pool = ForkJoinPool.commonPool();
        ParallelSumTask task = new ParallelSumTask(data);

        startTime = System.currentTimeMillis();
        long parallelSum = pool.invoke(task);
        endTime = System.currentTimeMillis();
        System.out.println("Parallel Sum:   " + parallelSum + " | Time: " + (endTime - startTime) + " ms");
    }
}
```
**Expected Output (on a multi-core machine):**
```
Sequential Sum: 50000005000000 | Time: 25 ms
Parallel Sum:   50000005000000 | Time: 8 ms
```
Multi-core machine meeda, parallel version anedi sequential version kanna significantly fast ga undali.

---

## ✅ Checkpoint: Did You Master This?
- [ ] `fork()` chesina tarvata `join()` call cheyakapothe em avtundi?
- [ ] `left.fork(); right.fork();` enduku vadakudadu? Best practice enti?
- [ ] `invokeAll(left, right)` anedi synchronous or asynchronous?
- [ ] Nee `RecursiveTask` eppudu `compute()` ni direct ga call cheyali, and eppudu `fork()` cheyali?

**✅ Ready?** → [Next: Parallel Streams and Performance](./03-Parallel-Streams-and-Performance.md)
