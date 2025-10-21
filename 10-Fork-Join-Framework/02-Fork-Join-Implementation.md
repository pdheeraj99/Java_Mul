<!--
---
title: "Implementing a Fork/Join Task"
---
-->

> **Learning Path Position**
>
> Phase 10: Fork/Join Framework ➔ **Chunk 2: Fork/Join Implementation**

> **Prerequisites**
>
> *   Phase 10, Chunk 1: Fork/Join Basics (especially `RecursiveTask` and `compute()`).
> *   Solid understanding of recursion.

> **Coming After This**
>
> *   Phase 10, Chunk 3: Parallel Streams Internals
> *   Hands-On Mini-Project: Parallel File System Counter

---

### 🚀 1. What & Why: Practical Implementation

Mawa, poyina chunk lo manam theory chusam. Ippudu, manam hands-on cheddam. Oka `ForkJoinTask` ni nijanga ela rayali? `compute()` method lo logic ela pettali? Results ni ela combine cheyyali? Ee chunk lo, manam ee practical aspects mida focus cheddam.

Ee implementation pattern chala common ga untundi and chala "divide and conquer" problems ki apply avutundi. Okasari ee pattern neeku ardham aite, nuvvu chala problems ni Fork/Join framework use chesi solve cheyyochu.

**Ee chunk lo em nerchukuntam? 🤔**

*   `compute()` method ni implement cheyyadaniki oka standard template.
*   `THRESHOLD` (threshold) ni ela choose chesukovali and daani importance enti.
*   `fork()`, `compute()`, and `join()` methods ni correct order lo ela use cheyyali.
*   Sub-tasks nunchi vachina results ni ela kalapali.

---

### 🧠 2. The `compute()` Method: Standard Template

Prathi `ForkJoinTask` ki `compute()` method anedi daani brain laantidi. Daani logic almost eppudu ide pattern follow avutundi:

```java
if (समस्या छोटी है) { // Check if the problem is small enough
    सीधे समाधान करें; // Solve it directly (sequentially)
} else {
    समस्या को दो या अधिक भागों में तोड़ें; // Break the problem into two or more parts

    सभी उप-कार्यों को फोर्क करें; // Fork all sub-tasks to run them in parallel

    सभी उप-कार्यों के परिणामों को जॉइन करें; // Join the results of all sub-tasks

    परिणामों को मिलाएं; // Combine the results
}
```

Ee template ni manam oka `RecursiveTask` use chesi, oka pedda array sum calculate cheyyadaniki implement cheddam.

---

### ✍️ 3. Step-by-Step Implementation Guide

Manam oka `SummingTask` ane `RecursiveTask` ni create cheddam.

#### Step 1: Define the Task and its State

Mana task ki em kavali? Sum cheyyalsina array, and aa array lo ഏ ഭാഗം (start and end index) sum cheyyalo anedi. So, ivi mana class variables avutayi.

```java
class SummingTask extends RecursiveTask<Long> {
    private final int[] data;
    private final int start;
    private final int end;
    // ...
}
```
Result type `Long` endukante, sum peddaga avvochu.

#### Step 2: Choose a `THRESHOLD`

Idi chala mukhyamaina decision. Ee `THRESHOLD` cheptundi, "Task entha chinna aite, daanni inka split cheyyakunda direct ga solve cheyyali" ani.
*   **Too small:** Chala chinna threshold (e.g., 1 or 2) pedithe, manam chala ekkuva tasks create chestam. Task creation and management overhead perigipotundi.
*   **Too large:** Chala pedda threshold pedithe, manam takkuva tasks create chestam. Parallelism taggipotundi, and work-stealing antha effective ga undadu.

Correct value ni kanukkodaniki benchmarking cheyyali, kani oka reasonable starting point entante, pani sequential ga cheyyadaniki chala takkuva time pattali (e.g., < 1 millisecond). Manam oka `static final int` variable create cheddam.

```java
private static final int THRESHOLD = 1000; // 1000 elements unte direct ga sum cheyyi
```

#### Step 3: Implement the `compute()` Method

Ippudu manam paina cheppina template ni follow avudam.

```java
@Override
protected Long compute() {
    int length = end - start;
    if (length <= THRESHOLD) {
        // 1. Pani chinnadi aite, direct ga solve cheyyi
        return computeSequentially();
    } else {
        // 2. Pani ni rendu bhagaluga break cheyyi
        int mid = start + length / 2;
        SummingTask leftTask = new SummingTask(data, start, mid);
        SummingTask rightTask = new SummingTask(data, mid, end);

        // 3. Sub-tasks ni fork cheyyi
        leftTask.fork(); // Left sub-task ni asynchronus ga run cheyyi

        // 4. Right sub-task ni current thread lo ne compute cheyyi
        long rightResult = rightTask.compute();

        // 5. Left sub-task result kosam join (wait) cheyyi
        long leftResult = leftTask.join();

        // 6. Results ni combine cheyyi
        return leftResult + rightResult;
    }
}

private long computeSequentially() {
    long sum = 0;
    for (int i = start; i < end; i++) {
        sum += data[i];
    }
    return sum;
}
```

**Important Optimization Note:** Manam `leftTask.fork()` and `rightTask.fork()` chesi, taruvatha `rightTask.join()` and `leftTask.join()` cheyyochu. Kani paina chesina pattern (`fork`, `compute`, `join`) inkonchem better. Endukante, adi current thread ni oka sub-task (`rightTask`) cheyyadaniki reuse chestundi, deeni valla kottha task create chese overhead taggutundi.

---

### 💻 4. Complete Code Example: Parallel Summation

Ippudu anni kalipi, oka complete, runnable program chuddam.

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;
import java.util.stream.IntStream;

public class ForkJoinSummingTask extends RecursiveTask<Long> {

    private final int[] data;
    private final int start;
    private final int end;
    private static final int THRESHOLD = 10_000; // Threshold peddaga pettam

    public ForkJoinSummingTask(int[] data, int start, int end) {
        this.data = data;
        this.start = start;
        this.end = end;
    }

    // Main compute logic
    @Override
    protected Long compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            // Base case: pani chinnadi aite, direct ga chesey
            return computeSequentially();
        } else {
            // Recursive case: pani ni break cheyyi
            int mid = start + length / 2;
            ForkJoinSummingTask leftTask = new ForkJoinSummingTask(data, start, mid);
            ForkJoinSummingTask rightTask = new ForkJoinSummingTask(data, mid, end);

            // Left task ni asynchronus ga start cheyyi
            leftTask.fork();

            // Right task ni ee thread lo ne cheyyi
            long rightResult = rightTask.compute();

            // Left task result kosam wait cheyyi
            long leftResult = leftTask.join();

            // Rendu results ni kalupu
            return leftResult + rightResult;
        }
    }

    // Sequential computation
    private long computeSequentially() {
        long sum = 0;
        for (int i = start; i < end; i++) {
            sum += data[i];
        }
        return sum;
    }

    public static void main(String[] args) {
        int[] data = IntStream.rangeClosed(1, 20_000_000).toArray();

        // Common ForkJoinPool ni use cheddam
        ForkJoinPool pool = ForkJoinPool.commonPool();
        System.out.println("Pool parallelism: " + pool.getParallelism());

        ForkJoinSummingTask mainTask = new ForkJoinSummingTask(data, 0, data.length);

        System.out.println("Parallel computation start chestunnam...");
        long startTime = System.currentTimeMillis();
        long result = pool.invoke(mainTask);
        long endTime = System.currentTimeMillis();

        System.out.println("Parallel Sum: " + result);
        System.out.println("Time taken (parallel): " + (endTime - startTime) + " ms");

        // Sequential ga cheste entha time padutundo chuddam
        System.out.println("\nSequential computation start chestunnam...");
        startTime = System.currentTimeMillis();
        long seqResult = 0;
        for (int val : data) {
            seqResult += val;
        }
        endTime = System.currentTimeMillis();

        System.out.println("Sequential Sum: " + seqResult);
        System.out.println("Time taken (sequential): " + (endTime - startTime) + " ms");
    }
}
```
**Output Analysis:** Ee program ni run cheste, nuvvu parallel version sequential version kanna chala fast ga undadam chustav (nee system lo multiple cores unte).

---

### 👎 5. Common Mistakes

*   **`join()` ni `fork()` chesina task mida call cheyyadam marchipovadam:** Okavela nuvvu `leftTask.fork()` chesi, taruvatha `leftTask.join()` call cheyyadam marchipothe, nee final result thappu vastundi, endukante nuvvu aa sub-task result ni consider cheyyatledu.
*   **Rendu tasks ni `compute()` cheyyadam:** `leftTask.compute()` and `rightTask.compute()` ani sequential ga call cheste, adi normal recursion laage pani chestundi, parallelism undadu.
*   **Wrong `join()` order:** `leftTask.join()` ni `rightTask.compute()` kanna mundu call cheste, current thread block aipotundi, and `rightTask` ni cheyyadaniki available undadu. Idi performance ni debba teestundi.

### 🔑 6. Key Takeaways

1.  **Standard Template:** `compute()` method ki oka standard template undi: `if (small) { solve } else { split, fork, compute, join, combine }`.
2.  **Threshold is Key:** Correct `THRESHOLD` ni choose chesukovadam performance ki chala mukhyam.
3.  **`fork()`-`compute()`-`join()` Pattern:** Oka sub-task ni `fork` chesi, inko sub-task ni current thread lo `compute` chesi, taruvatha first task ni `join` cheyyadam anedi oka common and efficient pattern.
4.  **Combine Correctly:** Sub-task results ni correct ga combine cheyyadam anedi final result correct ga ravalaniki chala mukhyam.

---

### ✅ Checkpoint

*   `ForkJoinTask` lo, `THRESHOLD` chala peddaga pedithe emi avutundi? Chala chinnaga pedithe?
*   Nuvvu `task1.fork(); task2.fork(); task1.join(); task2.join();` pattern enduku prefer cheyyakudadu? (Hint: current thread ni use chesukovadam gurinchi alochinchu).
*   `RecursiveTask` lo, oka thread `join()` call chesinappudu emi avutundi?
*   `compute()` method lo, base case (sequential computation) enduku avasaram?

---
