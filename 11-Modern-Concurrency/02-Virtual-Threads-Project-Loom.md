<!--
---
title: "Virtual Threads (Project Loom)"
---
-->

> **Learning Path Position**
>
> Phase 11: Modern Concurrency ➔ **Chunk 2: Virtual Threads (Project Loom)**

> **Prerequisites**
>
> *   Phase 1: Thread vs. Process (OS threads gurinchi teliyali)
> *   Phase 6: Executor Framework (Platform thread pools ela pani chestayo teliyali)

> **Coming After This**
>
> *   Phase 11, Chunk 3: Structured Concurrency
> *   Hands-On Mini-Project: Virtual Thread Web Server

---

### 🚀 1. What & Why: Virtual Threads?

Mawa, 지금까지 manam use chesina `Thread` anedi direct ga Operating System (OS) thread ki 1:1 mapping. Deenini ippudu manam **Platform Thread** antam. Prathi platform thread create cheyyadaniki chala time and memory (around 1-2 MB) padutundi. So, manam `Executors.newFixedThreadPool(200)` lanti pool create chesi, aa threads ni reuse chestu untam.

Kani, ee model lo oka pedda samasya undi. Web servers lanti applications lo, prathi incoming request ki oka thread ni assign chestam ("thread-per-request" model). Okavela aa thread database call or network call lanti I/O operation kosam wait chestunte? Aa OS thread antha sepu block aipotundi, em pani cheyyakunda memory ni waste chestu untundi. Manam aite 10,000 platform threads ni create cheyyalem, memory aipotundi!

Deeniki solution ee **Virtual Threads**, Project Loom lo part ga Java 19 lo preview chesi, Java 21 lo final chesaru.

**Virtual Threads ante enti?** Ive lightweight threads, JVM manage chestundi, OS ki teliyadu. Manam thousands, even millions of virtual threads ni create cheyyochu. Avi chala takkuva memory tesukuntayi.

**Deeni main goal enti?** Simple, blocking, sequential ga unna code ni, asynchronous code laaga high scalability tho run cheyyadam. Nuvvu `Thread.sleep(1000)` ani raasina, aa venakala unna OS thread block avvadu!

---

### analogy 2. Real-World Analogy: Restaurant Waiters 👨‍🍳 vs. Personal Assistants 🤖

*   **Platform Threads (Traditional Waiters):** Imagine oka high-end restaurant lo 10 mandi matrame chala experienced waiters unnaru. Prathi waiter oka table ki assign avutaru. Okavela aa table lo customers menu chustu unte (I/O wait), aa waiter em cheyyakunda, aalla daggare nilabadi untadu. Inko table ki help cheyyaledu. Ee 10 mandi waiters (threads) busy aipothe, restaurant (application) kottha customers ni handle cheyyaledu.

*   **Virtual Threads (Personal Assistants):** Ippudu inko restaurant imagine chesuko. Ikkada 10 mandi super-efficient chefs (Carrier Platform Threads) unnaru. Prathi customer ki, restaurant oka cheap, disposable robot assistant (`Virtual Thread`) istundi.
    *   Customer menu chustunnapudu, aa robot wait chestundi. Kani, aa venakala unna chef free ga unnadu, aayana inko robot ki unna order ni tayaru chestunnadu.
    *   Customer order ivvagane, aa robot order ni chef ki istundi.
    *   Ee model lo, manam thousands of customers ni okesari handle cheyyochu, endukante expensive chefs eppudu busy ga untaru, cheap robots matrame wait chestayi.

Virtual thread I/O kosam block ayinappudu, adi tana platform thread nunchi "unmount" aipotundi, aa platform thread inko virtual thread ni run chestundi.

---

### 🧠 3. How It Works: Mounting and Unmounting

Virtual threads anevi chala virtual threads ni konni platform threads mida multiplex chese magic. Aa venakala unna platform threads ni **Carrier Threads** antaru. Default ga, carrier threads pool anedi oka `ForkJoinPool`.

1.  Oka virtual thread run avvali anukunnapudu, adi oka carrier thread mida **mount** avutundi.
2.  Aa virtual thread code lo edaina blocking I/O operation (`InputStream.read()`, `Thread.sleep()`, etc.) vachinappudu, JVM adi chusi...
3.  ...aa virtual thread ni carrier thread nunchi **unmount** chesi, daani state ni heap lo save chestundi.
4.  Ippudu aa carrier thread free aipoindi! Adi ventane inko runnable virtual thread ni tesukuni run cheyyadam start chestundi.
5.  Eppaite first virtual thread I/O operation aipotundo, adi malli scheduler ki veltundi. Scheduler daanini oka free carrier thread mida malli **mount** chestundi.

Ee antha process chala fast ga, JVM level lo jarugutundi. OS ki ee unmounting/mounting gurinchi em teliyadu.

```mermaid
graph TD
    subgraph Carrier Platform Thread
        VT1("Virtual Thread 1 (Running)")
    end

    VT1 -- Blocks on I/O --> Unmounted

    subgraph Carrier Platform Thread
        VT2("Virtual Thread 2 (Running)")
    end

    subgraph Waiting for I/O (Heap)
        Unmounted("Virtual Thread 1 (Unmounted/Parked)")
    end

    Unmounted -- I/O Complete --> Rescheduled

    subgraph Carrier Platform Thread
        VT1_Resumed("Virtual Thread 1 (Running again)")
    end

    Rescheduled --> VT1_Resumed
```

---

### ✍️ 4. How to Use Virtual Threads

Virtual threads create cheyyadaniki kottha APIs vachayi.

**1. The `Thread.Builder` API:**
```java
// Direct ga start cheyyadaniki
Thread.ofVirtual().name("my-virtual-thread").start(() -> {
    System.out.println("Hello from Virtual Thread!");
});

// Unstarted thread create cheyyadaniki
Thread unstarted = Thread.ofVirtual().unstarted(myRunnable);
unstarted.start();
```

**2. The `Executors.newVirtualThreadPerTaskExecutor()` (Preferred Method):**
Idi oka kottha `ExecutorService` ni istundi. Ee executor prathi kottha task ki, oka kottha virtual thread ni create chestundi. Virtual threads chala cheap kabatti, vaatini pool cheyyalsina avasaram ledu. Ide recommended approach.

```java
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return "done";
        });
    }
} // try-with-resources valla executor automatic ga shutdown avutundi
```

---

### 💻 5. Code Example: Scalable Server Simulation

**Scenario:** 100,000 tasks ni submit cheddam, prathi okkati 1 second block avutundi. Mundu platform threads tho try cheddam, taruvatha virtual threads tho.

**👎 Failure Case (Platform Threads):**
Ee code run cheste, konni thousand threads create ayyaka, nee system `java.lang.OutOfMemoryError: unable to create native thread` ane exception tho crash avutundi.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.time.Duration;

public class PlatformThreadMemoryCrash {
    public static void main(String[] args) {
        // DON'T RUN THIS unless you want to crash your JVM!
        // try (ExecutorService executor = Executors.newFixedThreadPool(100_000)) {
        //     for (int i = 0; i < 100_000; i++) {
        //         executor.submit(() -> {
        //             Thread.sleep(Duration.ofSeconds(1));
        //             return i;
        //         });
        //     }
        // }
        System.out.println("Ee code run cheste, OutOfMemoryError confirm!");
    }
}
```

**✅ Success Case (Virtual Threads):**
Ee code chala easy ga, konni MBs of memory lo ne run avutundi.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.time.Duration;
import java.util.ArrayList;

public class VirtualThreadSuccess {
    public static void main(String[] args) {
        long startTime = System.currentTimeMillis();

        // Prathi task ki oka kottha virtual thread create chese executor
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {

            List<Future<Integer>> futures = new ArrayList<>();
            for (int i = 0; i < 100_000; i++) {
                final int taskNum = i;
                futures.add(executor.submit(() -> {
                    // Ee sleep call OS thread ni block cheyyadu
                    Thread.sleep(Duration.ofSeconds(1));
                    return taskNum;
                }));
            }
            // Optional: wait for one result (just to see)
            // System.out.println("Result of first task: " + futures.get(0).get());
        } // Executor automatic ga close avutundi

        long endTime = System.currentTimeMillis();
        System.out.println("100,000 tasks ni ~1 second lo purthi chesam!");
        System.out.println("Total time taken: " + (endTime - startTime) + " ms");
    }
}
```
Ee program almost 1 second lo ne aipotundi (anni tasks parallel ga sleep avutayi), and memory usage chala takkuva untundi.

---

### 🔗 6. Concept Connections

*   **Asynchronous Programming (`CompletableFuture`):** Virtual threads async programming ki alternative. `CompletableFuture` tho code chala complex ga, callbacks tho untundi. Virtual threads tho, manam simple, easy-to-read synchronous code rasi, asynchronous scalability పొందవచ్చు.
*   **Go (Goroutines), Erlang (Processes):** Ee languages lo lightweight concurrency chala kalam nunchi undi. Virtual threads anevi Java version of this powerful concept.

### 👎 7. Anti-Patterns & Common Mistakes

*   **Pinning with `synchronized`:** Okavela virtual thread `synchronized` block or method loki enter aite, adi tana carrier thread ki "pin" aipotundi. Ante, adi I/O lo block aite unmount avvaledu. Idi scalability ni debba teestundi. **Solution: `synchronized` badulu `java.util.concurrent.locks.ReentrantLock` vaadali.**
*   **Using for CPU-bound tasks:** Long-running CPU-intensive calculations (e.g., video encoding) kosam virtual threads vaadakudadu. Avi carrier thread ni eppatiki vadalavu, so work-stealing jaragadu. CPU-bound tasks kosam traditional platform thread pool ee better.
*   **Pooling Virtual Threads:** Virtual threads chala cheap. Vaatini pool cheyyalsina avasaram ledu. Prathi task ki oka kottha di create cheyyi. `newVirtualThreadPerTaskExecutor` ide chestundi.

### 🔑 8. Key Takeaways

1.  **Lightweight & Abundant:** Virtual threads anevi JVM-managed, lightweight threads. Manam millions create cheyyochu.
2.  **Best for I/O-bound tasks:** Blocking I/O (`sleep`, network calls, DB queries) unna tasks ki ive best.
3.  **Synchronous Code, Asynchronous Scale:** Manam simple, blocking code rasi, high scalability పొందవచ్చు.
4.  **Avoid `synchronized`:** `synchronized` pinning ki dari teestundi. Daani badulu `ReentrantLock` vaadali.
5.  **Don't Pool Them:** Virtual threads ni pool cheyyakudadu. Prathi task ki oka kottha di create cheyyi.

---

### ✅ Checkpoint

*   Platform thread ki and Virtual thread ki madhya unna main difference enti?
*   Oka virtual thread I/O kosam block ayinappudu, adi tana carrier thread nunchi "unmount" avvadam valla vache main benefit enti?
*   Virtual threads tho `synchronized` block vaadadam enduku oka bad idea? Daani badulu em vaadali?
*   CPU-bound task (e.g., oka pedda number ki prime factors kanukkodam) kosam nuvvu virtual thread ni enduku use cheyyakudadu?

---
