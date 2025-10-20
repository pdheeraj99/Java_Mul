---
# 🎯 9.3: Permit-Based Access - Semaphore

## 🗺️ Learning Path Position

**📍 You Are Here:** 9.3 - Semaphore

**✅ Prerequisites (Must Complete First):**
- [9.1 & 9.2: Barriers](./01-Barriers-CountDownLatch-and-CyclicBarrier.md) - Barriers gurinchi teliste, Semaphore yokka different purpose clear ga ardham avtundi.

**🔜 Coming After This:**
- [9.4 & 9.5: Advanced Synchronizers](./03-Advanced-Synchronizers-Exchanger-and-Phaser.md) - `Exchanger` and `Phaser` lanti inka specialized tools gurinchi nerchukuntam.

**⏱️ Estimated Time:** 2.5 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: How to Limit Access to a Pool of Resources?
Mawa, manaki konni sarlu ilanti situation vastundi: manadeggara konni limited resources unnayi (e.g., 5 database connections, 3 file handles), kaani 50 threads aa resources kosam try chestunnayi. Andari threads ni okate sari allow cheste, system crash avtundi. Manam at any given time, kevalam 5 threads ni matrame aa database connections ni use cheyadaniki allow cheyali.

Ee access ni `synchronized` block tho control cheyalem, endukante manaki okati kanna ekkuva (but still limited) threads ni allow cheyali.

### The Solution: `Semaphore` - The Bouncer of Concurrency 🕴️
`Semaphore` anedi ee problem ni solve chestundi. Idi oka counter (`permits`) ni maintain chestundi.
- **How it Works:**
  - Nuvvu oka number of permits tho `Semaphore` ni initialize chestav (e.g., `new Semaphore(5)`).
  - Resource access cheyali anukunna prathi thread, `semaphore.acquire()` ani call cheyali.
    - Permit available unte (`counter > 0`), semaphore counter ni okati tagginchi, thread ni allow chestundi.
    - Permits em lekapothe (`counter == 0`), `acquire()` method block ayyi, thread wait chestundi.
  - Thread tana pani complete chesaka, **it must call `semaphore.release()`**. Ee call permit ni tirigi istundi (counter ni increment chestundi). Appudu waiting lo unna vere thread aa permit ni teskuni munduku veltundi.
- **Key Property:** It controls access to a pool of resources, not just a single critical section (like a lock).

### Real-World Analogy: Library with Limited Study Rooms 📚
- **Study Rooms (The Resource):** Library lo kevalam 3 study rooms unnayi.
- **Semaphore:** Library entrance deggara `new Semaphore(3)` ane oka board undi, 3 keys tho.
- **Students (Threads):** Chala mandi students study rooms kosam vastaru.
- **`acquire()`:** Oka student vachi, oka key (`permit`) teskuni, room loki veltadu. Board meeda 2 keys migilayi.
- **Blocking:** 4th student vachinappudu, board meeda em keys levu. So, aa student akkade wait cheyali (`acquire()` blocks).
- **`release()`:** Lopalunna student tana pani aypogane, vachi key ni board ki tirigi pedtadu. Appudu waiting lo unna 4th student aa key teskuni lopaliki veltadu.

---

## 💻 Code Examples

### Example 1: Limiting Access to a Database Connection Pool
**Scenario:** Manaki oka connection pool undi, daantlo 3 connections matrame unnayi. Multiple threads aa connections ni use cheskuni queries run cheyali. `Semaphore` use chesi, at a time 3 threads kanna ekkuva connections use cheyakunda restrict cheddam.

```java
// File: com/example/semaphore/ConnectionPoolDemo.java
package com/example/semaphore;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

// A dummy connection pool resource
class ConnectionPool {
    private final Semaphore semaphore;

    ConnectionPool(int maxConnections) {
        // Fair semaphore: waiting threads get permits in FIFO order
        this.semaphore = new Semaphore(maxConnections, true);
    }

    public void useConnection() throws InterruptedException {
        // 1. Acquire a permit
        semaphore.acquire();

        try {
            // 2. Use the resource
            System.out.println(Thread.currentThread().getName() + " got a connection. "
                + (semaphore.availablePermits()) + " permits left.");
            Thread.sleep(1000); // Simulate doing work with the connection
        } finally {
            // 3. MUST release the permit in a finally block
            System.out.println(Thread.currentThread().getName() + " releasing connection.");
            semaphore.release();
        }
    }
}

public class ConnectionPoolDemo {
    public static void main(String[] args) {
        ConnectionPool pool = new ConnectionPool(3);
        ExecutorService executor = Executors.newFixedThreadPool(10);

        // 10 threads will try to use the pool of 3 connections
        for (int i = 0; i < 10; i++) {
            executor.execute(() -> {
                try {
                    pool.useConnection();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        executor.shutdown();
    }
}
```
**Output (Example):**
```
pool-1-thread-1 got a connection. 2 permits left.
pool-1-thread-2 got a connection. 1 permits left.
pool-1-thread-3 got a connection. 0 permits left.
... (Other threads will wait here)
pool-1-thread-1 releasing connection.
pool-1-thread-4 got a connection. 0 permits left.
...
```

### Binary Semaphore (Mutex)
Semaphore ni `1` permit tho create cheste, adi normal lock (`Mutex` - MUTual EXclusion) laaga pani chestundi. At a time okate thread ni allow chestundi.
`Semaphore binarySemaphore = new Semaphore(1);`
Kaani, deeniki `synchronized` or `ReentrantLock` ki unna ownership concept undadu. Ye thread `acquire` chesindo, ade `release` cheyalane rule ledu. Vere thread kuda `release` cheyochu. Idi advanced scenarios lo use avtundi (signaling).

---

## ✅ Checkpoint: Did You Master This?
- [ ] `Semaphore` ki `ReentrantLock` ki main difference enti?
- [ ] `acquire()` call chesina tarvata, `release()` ni `finally` block lo enduku pettali? Pettakapothe em avtundi?
- [ ] Oka `Semaphore` ni `0` permits tho initialize cheste em avtundi?
- [ ] "Fair" semaphore ante enti (`new Semaphore(5, true)`)?

**✅ Ready?** → [Next: Advanced Synchronizers - Exchanger & Phaser](./03-Advanced-Synchronizers-Exchanger-and-Phaser.md)
