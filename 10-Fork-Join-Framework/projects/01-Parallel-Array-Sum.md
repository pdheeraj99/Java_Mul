# 🏗️ Mini Project: Parallel Array Sum

## 🤔 Problem Statement
Mawa, manam ee project lo, Fork/Join framework yokka classic textbook example ni build cheddam: oka pedda array of numbers yokka sum ni parallel ga calculate cheyadam. Idi "divide and conquer" strategy ni perfectly demonstrate chestundi.

**Requirements:**
1.  Oka `RecursiveTask<Long>` ni implement cheyali.
2.  Ee task anedi array ni recursively chinna chinna sub-arrays ga divide cheyali, oka `THRESHOLD` reach ayyevaraku.
3.  Base case lo (array size <= THRESHOLD), sum anedi sequentially calculate avvali.
4.  Recursive step lo, task anedi `fork()` and `join()` use chesi, sub-tasks ni parallel ga run chesi, vaati results ni combine cheyali.
5.  Final ga, manam sequential sum tho and parallel sum tho performance ni compare cheddam.

**This Project Uses Concepts From:**
- ✅ [10.1: Fork/Join Basics](../01-Fork-Join-Basics.md)
- ✅ [10.2: Fork/Join Implementation](../02-Fork-Join-Implementation.md)

---

## 🏗️ Architecture
Manam `ParallelSumTask` ane `RecursiveTask` subclass ni create chestam. Main thread anedi oka `ForkJoinPool` ni create chesi, ee `ParallelSumTask` yokka main instance ni submit chestundi. `compute()` method lo unna logic antha array ni recursively split chestundi. Prathi split lo, oka sub-task ni asychronously `fork()` chesi, inko sub-task ni current thread lone `compute()` chestam (performance optimization). Final ga, `join()` tho results anni combine chestam.

```mermaid
graph TD
    A[Main Task (0..10M)] --> B{Size > THRESHOLD?};
    B -- Yes --> C[Split into (0..5M) and (5M..10M)];

    subgraph ForkJoinPool
        C --> D[Task(0..5M)];
        C --> E[Task(5M..10M)];

        D --> F{Size > THRESHOLD?};
        E --> G{Size > THRESHOLD?};

        F -- Yes --> H[...continues splitting...];
        G -- Yes --> I[...continues splitting...];

        F -- No --> J[Compute Sequentially];
        G -- No --> K[Compute Sequentially];
    end

    J -- join() --> L[Combine Results];
    K -- join() --> L;
    H -- join() --> L;
    I -- join() --> L;

    L --> M[Final Sum];

```

---

## 💻 Complete Code

#### File 1: `ParallelSumTask.java`
```java
// File: src/com/example/sum/ParallelSumTask.java
package com.example.sum;

import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;
import java.util.stream.LongStream;

public class ParallelSumTask extends RecursiveTask<Long> {

    // A good threshold is crucial. Too small = too much overhead. Too large = not enough parallelism.
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
        // Base case: If the task is small enough, compute it sequentially.
        if (length <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += numbers[i];
            }
            return sum;
        } else {
            // Recursive step: The task is large, split it into two sub-tasks.
            int mid = start + length / 2;
            ParallelSumTask leftTask = new ParallelSumTask(numbers, start, mid);
            ParallelSumTask rightTask = new ParallelSumTask(numbers, mid, end);

            // ✅ Best Practice: Fork one task to run in parallel, and compute the other
            //    in the current thread. This is more efficient than forking both.
            rightTask.fork(); // Asynchronously execute the right half.

            long leftSum = leftTask.compute(); // Synchronously execute the left half.

            long rightSum = rightTask.join(); // Wait for the right half to complete.

            // Join (combine) the results.
            return leftSum + rightSum;
        }
    }

    public static void main(String[] args) {
        // Create a large array
        long[] data = LongStream.rangeClosed(1, 20_000_000).toArray();

        // --- 1. Sequential Calculation (for comparison) ---
        long startTime = System.currentTimeMillis();
        long sequentialSum = 0;
        for (long n : data) {
            sequentialSum += n;
        }
        long endTime = System.currentTimeMillis();
        System.out.println("Sequential Sum: " + sequentialSum);
        System.out.println("Time taken (Sequential): " + (endTime - startTime) + " ms");

        System.out.println("\n-----------------------------------\n");

        // --- 2. Parallel Calculation ---
        // Use the common ForkJoinPool (introduced in Java 8)
        ForkJoinPool commonPool = ForkJoinPool.commonPool();
        System.out.println("Parallelism level of common pool: " + commonPool.getParallelism());

        ParallelSumTask task = new ParallelSumTask(data);

        startTime = System.currentTimeMillis();
        // The invoke method submits the task and waits for its result
        long parallelSum = commonPool.invoke(task);
        endTime = System.currentTimeMillis();

        System.out.println("Parallel Sum:   " + parallelSum);
        System.out.println("Time taken (Parallel):   " + (endTime - startTime) + " ms");
    }
}
```

---

## 🚀 How to Run & Test

### Step 1: Compile the Code
```bash
# Assuming you are in the project root
javac -d out src/com/example/sum/*.java
```

### Step 2: Run the Project
```bash
java -cp out com.example.sum.ParallelSumTask
```

### Expected Output & Analysis
(Your times will vary based on your CPU cores)
```
Sequential Sum: 200000010000000
Time taken (Sequential): 45 ms

-----------------------------------

Parallelism level of common pool: 8
Parallel Sum:   200000010000000
Time taken (Parallel):   15 ms
```
**Analysis:**
- **Correctness:** Rendu sums okate. Mana parallel logic correct.
- **Performance:** Multi-core machine meeda, parallel version anedi sequential version kanna 2x, 3x, or even more faster ga untundi. Idi Fork/Join framework yokka power. CPU cores anni 100% busy ga pani chesayi.
- **Threshold Tuning:** `THRESHOLD` value ni marchi chudu (e.g., 100 or 1,000,000). Performance ela change avtundo observe chey. Prathi problem ki oka optimal threshold untundi.

Ee project tho, nuvvu a complete, from-scratch Fork/Join task ni ela build cheyalo, and daani performance benefits ni chusav. Great work, mawa!
