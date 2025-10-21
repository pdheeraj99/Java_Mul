---
# 🎯 11.2: Virtual Threads (Project Loom)

## 🗺️ Learning Path Position

**📍 You Are Here:** 11.2 - Virtual Threads

**✅ Prerequisites (Must Complete First):**
- [1.1: Core Concepts & Theory](../01-Foundation-Threading-Basics/01-Core-Concepts-and-Theory.md) - Platform (OS) threads ante ento clear idea undali.
- [6.1: Executor Framework](../06-Executor-Framework/01-Introduction-to-Executor-Framework.md) - `ExecutorService` gurinchi teliyali.

**🔜 Coming After This:**
- [11.3: Structured Concurrency](./03-Structured-Concurrency.md) - Virtual threads tho paatu vachina inko powerful new model gurinchi nerchukuntam.

**⏱️ Estimated Time:** 3.5 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: Threads are Heavy and Expensive! 🏋️‍♂️💸
Mawa, manam ippativaraku use chesina threads anni **Platform Threads**. Ante, prathi Java `Thread` ki, venakala oka Operating System (OS) thread map aypoyi untundi. Ee OS threads anevi chala heavy resources:
1.  **Limited Number:** Oka system lo manam kevalam konni thousands of platform threads matrame create cheyagalam. Ekkuva create cheste, `OutOfMemoryError` vastundi.
2.  **High Creation Cost:** Prathi thread create cheyadaniki konchem time and memory padutundi.
3.  **Inefficient for I/O:** Oka thread database query kosam wait chestunnapudu (`I/O-bound task`), aa OS thread antha block aypotundi. Adi CPU ni use cheyatledu, kaani resource matram hold chesi undi. Idi "thread-per-request" model lo unna web servers (like Tomcat) lo pedda bottleneck. Oka server kevalam konni thousands of concurrent requests matrame handle cheyagaladu.

### The Solution: Virtual Threads - Lightweight Threads Managed by the JVM 깃털 🧵
Java 19 (preview) and Java 21 (LTS) lo vachina **Virtual Threads** (Project Loom) ee problem ni solve chestayi.
- **Core Idea:** Virtual threads anevi OS threads kadu. Avi JVM manage chese lightweight Java objects. Manam **millions** of virtual threads ni create cheyochu.
- **The Magic:** Oka virtual thread blocking I/O operation deggara wait chestunnapudu, JVM aa virtual thread ni daani carrier (platform thread) nunchi **unmount** chesi, aa platform thread meeda inko virtual thread ni run chestundi. Platform thread eppudu idle ga undadu! I/O complete ayyaka, JVM malli aa original virtual thread ni edoka available platform thread meeda **mount** chestundi.
- **Result:** Throughput anedi dramatically perugutundi, especially I/O-bound applications lo. Manam మళ్ళీ simple "thread-per-request" style code rayochu, without worrying about thread limits.

### Real-World Analogy: Super-Efficient Call Center 📞
- **Platform Threads:** Call center lo 10 mandi agents unnaru. Prathi agent (`platform thread`) oka customer call (`task`) matrame handle cheyagaladu. Agent customer information kosam system lo wait chestunnapudu (`I/O block`), aa agent em pani cheyakunda kurchuntadu.
- **Virtual Threads:** Call center lo 10 mandi "super agents" (`platform threads`) unnaru. Prathi super agent deggara 1000 customer files (`virtual threads`) unnayi.
  - Agent-1 customer A tho matladutunnadu. Customer A information kosam wait cheyalsi vachindi.
  - Ventane, Agent-1 customer A file ni pakkana petti (`unmount`), customer B file teskuni (`mount`) matladadam start chestadu.
  - Ee process antha chala fast ga jarugutundi. Ee 10 mandi super agents, లక్షల calls ni handle cheyagalరు, endukante వాళ్ళు eppudu waiting lo time waste cheyaru.

---

## 📚 Detailed Explanation: Platform vs. Virtual Threads

| Feature | Platform Thread | Virtual Thread |
|---|---|---|
| **Analogy**| Dedicated Truck 🚚 | Mailman with many letters 💌 |
| **Managed By**| Operating System (OS) | Java Virtual Machine (JVM) |
| **Weight**| Heavyweight (wraps an OS thread) | Lightweight (just a Java object) |
| **Number**| Limited (thousands) | Practically unlimited (millions) |
| **Creation Cost**| High | Very Low |
| **Best For**| CPU-bound tasks | I/O-bound tasks |
| **Key Behavior**| Blocks the OS thread on I/O | Unmounts from the OS thread on I/O |

---

## 💻 Code Examples

### Example 1: The Old Way - Crashing with Platform Threads

```java
// File: com/example/threads/PlatformThreadLimit.java
package com.example.threads;

public class PlatformThreadLimit {
    public static void main(String[] args) {
        // Try to create 20,000 platform threads
        // 💣 This will likely throw an OutOfMemoryError!
        for (int i = 0; i < 20_000; i++) {
            final int taskNum = i;
            new Thread(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {}
            }).start();
            System.out.println("Started platform thread #" + taskNum);
        }
    }
}
```

### Example 2: The New Way - Millions of Virtual Threads!
```java
// File: com/example/threads/VirtualThreadDemo.java
package com.example.threads;

public class VirtualThreadDemo {
    public static void main(String[] args) throws InterruptedException {
        // Create 1 million virtual threads
        // ✅ This runs perfectly fine!
        for (int i = 0; i < 1_000_000; i++) {
            final int taskNum = i;
            // Use the Thread.ofVirtual() factory
            Thread.ofVirtual().start(() -> {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {}
                if (taskNum % 100_000 == 0) {
                    System.out.println("Virtual thread #" + taskNum + " finished.");
                }
            });
        }

        // Wait for a bit to see some output
        Thread.sleep(2000);
        System.out.println("Main thread finished.");
    }
}
```

### Example 3: The Recommended Way - `newVirtualThreadPerTaskExecutor()`
Direct ga `Thread.ofVirtual()` vadadam kanna, `ExecutorService` vadadam better practice. Java 21 lo, deeni kosam kotha factory method vachindi.
```java
// File: com/example/threads/VirtualThreadExecutor.java
package com.example.threads;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class VirtualThreadExecutor {
    public static void main(String[] args) {
        // ✅ Create an executor that starts a new virtual thread for each task
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 100_000; i++) {
                final int taskNum = i;
                executor.submit(() -> {
                    System.out.println("Executing task " + taskNum + " on thread: " + Thread.currentThread());
                    try { TimeUnit.SECONDS.sleep(1); } catch (Exception e) {}
                });
            }
        } // try-with-resources automatically calls shutdown() and awaitTermination()
    }
}
```
**Output Analysis:** `Thread.currentThread()` output chuste, adi `VirtualThread[#ID]/runnable@ForkJoinPool-1-worker-1` laaga kanipistundi. Ante, virtual thread anedi `ForkJoinPool` (carrier pool) lo unna oka worker (platform) thread meeda run avtundi ani ardham.

---

## ⚠️ Caveats: `synchronized` and "Pinning"
- **Pinning:** Oka virtual thread `synchronized` block/method loki enter ayinappudu, adi daani carrier (platform) thread ki "pinned" aypotundi.
- **Problem:** Ee `synchronized` block loపల, virtual thread blocking I/O operation cheste, adi unmount avvadu. Aa platform thread antha block aypotundi. Idi virtual threads yokka main advantage ni debba testundi.
- **Solution:** `synchronized` badulu, `java.util.concurrent.locks.ReentrantLock` vadali. `ReentrantLock` anedi pinning cause cheyadu.

---

## ✅ Checkpoint: Did You Master This?
- [ ] Platform thread ki Virtual thread ki main difference enti?
- [ ] Oka virtual thread I/O deggara block aite, venakala em jarugutundi?
- [ ] Millions of virtual threads create cheyadam possible ye na? Yela?
- [ ] `synchronized` block ni virtual thread lo vadithe, em problem vache chance undi? Daani solution enti?

**✅ Ready?** → [Next: Structured Concurrency](./03-Structured-Concurrency.md)
