---
# 🎯 5.2: Adders and Accumulators

## 🗺️ Learning Path Position

**📍 You Are Here:** 5.2 - Adders & Accumulators

**✅ Prerequisites (Must Complete First):**
- [5.1: Atomic Variables](./01-Atomic-Variables.md) - `AtomicLong` and CAS gurinchi clear ga teliyali.

**🔜 Coming After This:**
- [5.3: CAS and the ABA Problem](./03-CAS-and-ABA-Problem.md) - Atomics venaka unna CAS algorithm ni inka deep ga chustam.

**⏱️ Estimated Time:** 2 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 🤔 What & Why

### The Problem: `AtomicLong` under High Contention
`AtomicLong` use chesinappudu, anni threads okate central `value` variable ni update cheyadaniki try chestayi.
- 100 threads okesari `incrementAndGet()` call cheste, okate thread CAS lo success avtundi.
- Migatha 99 threads fail ayi, malli retry cheyali.
- Ee continuous retries valla, performance drop avtundi. Ee problem ni **high contention** antaru.

### The Solution: `LongAdder` (Divide and Conquer)
`LongAdder` (Java 8 lo introduce chesaru) ee problem ni chala cleverly solve chestundi. Okate central value badulu, adi multiple variables (a "striped" array of counters) ni maintain chestundi.
- Thread 1 vachi `add(1)` call cheste, `LongAdder` daaniki array lo unna counter 1 istundi.
- Thread 2 vachi `add(1)` call cheste, daaniki counter 2 istundi.
- Threads veru veru counters ni update chestayi కాబట్టి, contention undadu. CAS failures chala takkuva.
- `sum()` aney method call chesinappudu, `LongAdder` ee anni counters yokka values ni add chesi final result istundi.

### Real-World Analogy: Supermarket Checkout Counters 🛒
- **`AtomicLong`:** Oka pedda supermarket lo okate okka checkout counter. Andaru customers (threads) ade line lo wait cheyali. Chala contention.
- **`LongAdder`:** Ade supermarket lo 10 checkout counters. Customers veru veru counters ki veltaru. Contention chala takkuva, throughput (performance) chala ekkuva. `sum()` ante, anni counters lo total collection entha ani calculate cheyadam.

---

## 💻 Code Example: `AtomicLong` vs `LongAdder` Performance

**Scenario:** Manam chala threads (e.g., 10 threads) create chesi, prathi thread oka counter ni 1 million sarlu increment cheyali. `AtomicLong` entha time teskuntundo, `LongAdder` entha time teskuntundo compare cheddam.

```java
// File: com/example/adders/AdderPerformanceDemo.java
package com.example.adders;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;
import java.util.concurrent.atomic.LongAdder;

public class AdderPerformanceDemo {
    private static final int NUM_THREADS = 10;
    private static final int INCREMENTS_PER_THREAD = 1_000_000;

    public static void testAtomicLong() throws InterruptedException {
        AtomicLong atomicCounter = new AtomicLong(0);
        ExecutorService executor = Executors.newFixedThreadPool(NUM_THREADS);

        long startTime = System.currentTimeMillis();

        for (int i = 0; i < NUM_THREADS; i++) {
            executor.submit(() -> {
                for (int j = 0; j < INCREMENTS_PER_THREAD; j++) {
                    atomicCounter.incrementAndGet();
                }
            });
        }

        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.MINUTES);

        long endTime = System.currentTimeMillis();
        System.out.println("--- AtomicLong ---");
        System.out.println("Final Count: " + atomicCounter.get());
        System.out.println("Time taken: " + (endTime - startTime) + " ms");
    }

    public static void testLongAdder() throws InterruptedException {
        LongAdder longAdder = new LongAdder();
        ExecutorService executor = Executors.newFixedThreadPool(NUM_THREADS);

        long startTime = System.currentTimeMillis();

        for (int i = 0; i < NUM_THREADS; i++) {
            executor.submit(() -> {
                for (int j = 0; j < INCREMENTS_PER_THREAD; j++) {
                    longAdder.increment(); // or longAdder.add(1)
                }
            });
        }

        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.MINUTES);

        long endTime = System.currentTimeMillis();
        System.out.println("--- LongAdder ---");
        System.out.println("Final Count: " + longAdder.sum());
        System.out.println("Time taken: " + (endTime - startTime) + " ms");
    }

    public static void main(String[] args) throws InterruptedException {
        testAtomicLong();
        System.out.println();
        testLongAdder();
    }
}
```
**Output (Example - your mileage may vary):**
```
--- AtomicLong ---
Final Count: 10000000
Time taken: 350 ms

--- LongAdder ---
Final Count: 10000000
Time taken: 50 ms
```
**Why `LongAdder` is Faster:** Chusava mawa? `LongAdder` is almost 7 times faster in this high-contention scenario! Endukante, prathi thread veru veru internal counters ni update chesindi, CAS retries chala takkuva.

---

## 🤔 When to use `AtomicLong` vs `LongAdder`?

| Scenario | `AtomicLong` | `LongAdder` |
|---|---|---|
| **Low to Moderate Contention** | Good. Faster than `LongAdder` because `get()` is cheap. | Okay, but slightly slower `sum()`. |
| **High Contention** | Performance degrades due to CAS retries. | **Excellent**. Designed for this. |
| **Need the current value frequently** | Good. `get()` is a single memory read. | Not ideal. `sum()` is more expensive as it adds up all internal counters. |

**Rule of Thumb:** 🔥
- Nuvvu frequent ga counter ni update chesi, chivarilo okate sari final `sum()` teskunte (e.g., statistics gathering), **`LongAdder` is the winner**.
- Nuvvu counter value ni frequent ga read cheyali, and aa value meeda decisions teskovali anukunte (e.g., `if (counter.get() > 100)`), **`AtomicLong` is better**.

---

## 🚀 `LongAccumulator`: The Customizable `LongAdder`

`LongAccumulator` anedi `LongAdder` ki generalization. Idi oka initial value and oka `LongBinaryOperator` (lambda function) teskuntundi.
- `LongAdder` anedi `add()` kosam matrame.
- `LongAccumulator` tho manam `max`, `min`, or any custom operation cheyochu.

### ✅ Success Case: Finding the Maximum Value in Parallel
```java
// File: com/example/accumulator/MaxFinder.java
package com.example.accumulator;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.atomic.LongAccumulator;

public class MaxFinder {
    public static void main(String[] args) throws InterruptedException {
        // 1. Function: (currentMax, newValue) -> if (newValue > currentMax) return newValue else return currentMax
        // 2. Initial value: Long.MIN_VALUE
        LongAccumulator maxAccumulator = new LongAccumulator(Math::max, Long.MIN_VALUE);
        ExecutorService executor = Executors.newFixedThreadPool(10);

        for (int i = 0; i < 10; i++) {
            executor.submit(() -> {
                long randomValue = ThreadLocalRandom.current().nextLong(10000);
                System.out.println(Thread.currentThread().getName() + " found value: " + randomValue);
                maxAccumulator.accumulate(randomValue); // Apply the function
            });
        }

        // ... shutdown executor ...
        System.out.println("Final Max Value: " + maxAccumulator.get());
    }
}
```
---

## ✅ Checkpoint: Did You Master This?
- [ ] High contention lo `LongAdder` enduku `AtomicLong` kanna faster?
- [ ] `LongAdder` yokka `sum()` method enduku `AtomicLong` yokka `get()` kanna slower?
- [ ] Oka web server lo request count ni track cheyali. `AtomicLong` a, `LongAdder` a, edi better? Enduku?
- [ ] `LongAdder` ki `LongAccumulator` ki teda enti?

**✅ Ready?** → [Next: CAS and the ABA Problem](./03-CAS-and-ABA-Problem.md)
**😕 Need Review?** → Paiki scroll chesi supermarket analogy malli chudu.
