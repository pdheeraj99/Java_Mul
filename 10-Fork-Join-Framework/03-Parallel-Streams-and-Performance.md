---
# 🎯 10.3: Parallel Streams & Performance

## 🗺️ Learning Path Position

**📍 You Are Here:** 10.3 - Parallel Streams & Performance

**✅ Prerequisites (Must Complete First):**
- [10.2: Fork/Join Implementation](./02-Fork-Join-Implementation.md) - Fork/Join Pool ela pani chestundo clear ga teliyali.
- Basic knowledge of Java 8 Streams API (`stream()`, `map()`, `reduce()`).

**🔜 Coming After This:**
- [Project: Parallel Array Sum](./projects/01-Parallel-Array-Sum.md) - Manam nerchukunna concepts tho a hands-on project build cheddam.

**⏱️ Estimated Time:** 2.5 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: Fork/Join is Powerful, but Verbose  verbose 📝
Mawa, manam `ParallelSumTask` example lo chusam, Fork/Join framework chala powerful, kaani daaniki manam chala boilerplate code rayali: `RecursiveTask` ni extend cheyali, `compute` method ni override cheyali, `fork`, `join`... Anni common operations (like sum, max, filter) ki, ee code antha malli malli rayadam kastam.

### The Solution: Parallel Streams - Fork/Join Made Easy! ✨
Java 8 lo vachina Streams API lo, ee problem ki oka beautiful solution icharu: **`.parallelStream()`**.
Oka collection meeda `.parallelStream()` call cheste, Java antha venakala ee Fork/Join logic ni automatically handle chestundi.
- **How it Works:** Stream source (like a `List`) anedi chinna chinna chunks ga divide avtundi (`spliterator`). Ee chunks anni `ForkJoinPool.commonPool()` ki tasks laaga submit avtayi. Final ga, results anni combine avtayi.
- **Example:** Manam `ParallelSumTask` lo rasina 50 lines of code, parallel stream tho okate line lo rayochu:
  `long sum = data.stream().parallel().mapToLong(i -> i).sum();`

### So, When Should I Use Raw Fork/Join vs. Parallel Streams?
- **Use Parallel Streams (95% of the time):** Data processing operations (filter, map, reduce, sum) kosam eppudu parallel streams ye vadu. Idi clean, less error-prone, and expressive.
- **Use Raw Fork/Join (5% of the time):** Nuvvu non-standard data structures (like a custom tree) meeda pani chestunnappudu, or task logic anedi simple data processing kanna complex ga unnappudu, appudu matrame raw `RecursiveTask`/`RecursiveAction` rayadam sense avtundi.

---

## 📚 Detailed Explanation: Performance Considerations

"Parallel is always faster" anedi oka pedda myth, mawa. Konni sarlu, parallel stream anedi sequential stream kanna **slower** ga untundi. Endukante, parallelism ki kuda konni costs untayi:
- Task splitting overhead.
- Thread management overhead.
- Result combining overhead.

### When Parallel Streams HURT Performance 🤕

1.  **Small Datasets:** Oka 100 elements unna list ni parallel ga process cheyadaniki try cheste, aa data ni process cheyadaniki kante, daanini sub-tasks ga divide chesi, threads ki ivvadanike ekkuva time padutundi.
2.  **Wrong Data Structure:** `ArrayList` anedi easy ga and efficiently split avtundi. Kaani `LinkedList` anedi kadu. Oka `LinkedList` lo middle element kanukovalante, manam start nunchi traverse cheyali. So, daanini split cheyadam chala expensive. `LinkedList` meeda parallel stream vadithe performance chala darunanga untundi.
3.  **CPU-Bound Tasks in a Web Server:** Parallel streams anevi by default `ForkJoinPool.commonPool()` ni vadatayi. Idi JVM-wide shared pool. Nuvvu oka web server lo, prathi request ki parallel stream vadithe, andaru (`.parallel()`) aa common pool kosame compete avtaru. Deenivalla thread starvation vachi, application antha slow aypotundi.
4.  **Blocking I/O Operations:** Stream pipeline madhyalo oka blocking I/O call (e.g., database query, network call) undanuko. Parallel stream lo unna threads anni ee I/O call deggara block aypotayi. `commonPool` lo unna threads anni block aite, JVM lo unna vere parallel streams kuda affect avtayi.

**Rule of Thumb:** Parallel streams anevi **CPU-intensive operations** on **large, splittable data sources** (like `ArrayList`) ki matrame best. **Never use them for I/O-bound work.**

---

## 💻 Code Example: The "Cost of Parallelism"

**Scenario:** Oka `ArrayList` vs `LinkedList` meeda simple operation chesi, sequential vs parallel performance chuddam.
```java
// File: com/example/performance/StreamPerformance.java
package com.example.performance;

import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.LongStream;

public class StreamPerformance {
    public static void main(String[] args) {
        final int SIZE = 20_000_000;

        // --- ArrayList ---
        List<Long> arrayList = LongStream.rangeClosed(1, SIZE).boxed().collect(Collectors.toCollection(ArrayList::new));

        long startTime = System.currentTimeMillis();
        long sumSequential = arrayList.stream().reduce(0L, Long::sum);
        long endTime = System.currentTimeMillis();
        System.out.println("ArrayList Sequential Sum: " + (endTime - startTime) + " ms");

        startTime = System.currentTimeMillis();
        long sumParallel = arrayList.parallelStream().reduce(0L, Long::sum);
        endTime = System.currentTimeMillis();
        System.out.println("ArrayList Parallel   Sum: " + (endTime - startTime) + " ms");

        System.out.println("\n-----------------------------------\n");

        // --- LinkedList ---
        List<Long> linkedList = LongStream.rangeClosed(1, SIZE).boxed().collect(Collectors.toCollection(LinkedList::new));

        startTime = System.currentTimeMillis();
        sumSequential = linkedList.stream().reduce(0L, Long::sum);
        endTime = System.currentTimeMillis();
        System.out.println("LinkedList Sequential Sum: " + (endTime - startTime) + " ms");

        startTime = System.currentTimeMillis();
        sumParallel = linkedList.parallelStream().reduce(0L, Long::sum);
        endTime = System.currentTimeMillis();
        System.out.println("LinkedList Parallel   Sum: " + (endTime - startTime) + " ms");
    }
}
```
**Expected Output:**
```
ArrayList Sequential Sum: 150 ms
ArrayList Parallel   Sum: 40 ms   // ✅ Much faster!

-----------------------------------

LinkedList Sequential Sum: 160 ms
LinkedList Parallel   Sum: 1200 ms  // 😱 Much SLOWER!
```

---

## ✅ Checkpoint: Did You Master This?
- [ ] Parallel streams venakala ye thread pool untundi?
- [ ] `ArrayList` meeda parallel stream fast ga, `LinkedList` meeda enduku slow ga untundi?
- [ ] Oka web request handle chese code lo, database nunchi data fetch cheyadaniki `.parallelStream()` vadoccha? Why or why not?
- [ ] Nuvvu `.parallelStream()` use chesinappudu, performance improve avvatledu. Yemayi undochu? (List 3 possible reasons).

**✅ Ready?** → [Project: Parallel Array Sum](./projects/01-Parallel-Array-Sum.md)
