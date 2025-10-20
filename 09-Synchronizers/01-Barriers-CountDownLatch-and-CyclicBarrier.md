---
# 🎯 9.1 & 9.2: Barriers - CountDownLatch & CyclicBarrier

## 🗺️ Learning Path Position

**📍 You Are Here:** 9.1 & 9.2 - Barriers

**✅ Prerequisites (Must Complete First):**
- [1.2: Creating & Managing Threads](../01-Foundation-Threading-Basics/02-Creating-and-Managing-Threads.md) - `Thread.join()` gurinchi teliste, `CountDownLatch` ni appreciate cheskovadam easy.
- [6.1: Executor Framework](../06-Executor-Framework/01-Introduction-to-Executor-Framework.md) - Multiple threads ni manage cheyadaniki manam `ExecutorService` vadatam.

**🔜 Coming After This:**
- [9.3: Permit-Based Access - Semaphore](./02-Permit-Based-Access-Semaphore.md) - Limited resources ki access ni ela control cheyalo nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: How Do Threads Wait for Each Other? 🚦
Mawa, manaki chala sarlu ilanti situations vastayi:
- "Main thread anedi, 3 worker threads valla pani complete chesevaraku wait cheyali." (e.g., Application start ayye mundu, anni required services load avvali).
- "Multiple threads anni oka point deggara reach ayyaka, andaru kalisi okate sari munduku vellali." (e.g., Parallel computation lo, oka phase complete ayyaka, anni threads kalisi next phase start cheyali).
`Thread.join()` lanti tools unna, avi chala basic. Manaki inka flexible and powerful constructs kavali.

### The Solution: `CountDownLatch` and `CyclicBarrier`

#### 1. `CountDownLatch`
- **Core Idea:** A one-time use "gate". Oka counter (latch) zero ayyevaraku, oka thread or multiple threads wait chestu untayi.
- **How it Works:**
  - Nuvvu oka count tho initialize chestav (e.g., `new CountDownLatch(3)`).
  - Worker threads valla pani aypogane, `latch.countDown()` ani call chestayi. Ee call counter ni okati taggistundi.
  - Main thread `latch.await()` deggara block avtundi.
  - Counter `0` avvagane, `await()` nunchi main thread release avtundi and munduku veltundi.
- **Key Property:** **One-time use.** Latch count zero ayyaka, daanini reset cheyalem.
- **Analogy:** Rocket Launch 🚀. Launch control (`main` thread) anedi `latch.await()` deggara wait chestundi. 3 different teams (System Check, Fuel Check, Weather Check) untayi. Prathi team valla pani aypogane, `latch.countDown()` call chestaru. Andaru "GO" cheppagane (count = 0), rocket launch avtundi (`await()` releases).

#### 2. `CyclicBarrier`
- **Core Idea:** A reusable "gate" where a fixed number of threads must wait for each other.
- **How it Works:**
  - Nuvvu oka number of parties (threads) tho initialize chestav (e.g., `new CyclicBarrier(4)`).
  - Prathi thread oka point ki reach avvagane, `barrier.await()` call chestundi. Appudu aa thread block avtundi.
  - Eppudaithe 4 threads `await()` call chesayo, barrier "breaks", and anni threads okate sari release avtayi.
  - Tarvata, aa barrier automatically reset avtundi. Malli vere 4 threads wait cheyochu.
- **Key Property:** **Reusable (Cyclic).**
- **Analogy:** 4 friends hiking 🧗 plan chesaru. Andaru first meeting point (e.g., base camp) deggara kalisi, tarvata next point ki start avvali. Evaru mundu vachina, migatha vallu vachevarku `barrier.await()` deggara wait chestaru. Andaru (4) vachaka, andaru kalisi next leg of the journey start chestaru.

### `CountDownLatch` vs. `CyclicBarrier`

| Feature | CountDownLatch | CyclicBarrier |
|---|---|---|
| **Analogy** | Rocket Launch (One-way) | Hiking Meetup Point (Reusable) |
| **Main Use**| One thread waits for N others. | N threads wait for each other. |
| **Reusability**| No, one-time use. | Yes, resets automatically. |
| **Action** | `countDown()` and `await()` | Just `await()` |

---

## 💻 Code Examples

### Example 1: `CountDownLatch` for Service Initialization
**Scenario:** Application start ayye mundu, 3 services (`DatabaseService`, `CacheService`, `LoggingService`) initialize avvali. Main thread antha varku wait cheyali.
```java
// File: com/example/barriers/LatchDemo.java
package com/example/barriers;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

class ServiceInitializer implements Runnable {
    private final String serviceName;
    private final CountDownLatch latch;

    ServiceInitializer(String name, CountDownLatch latch) { this.serviceName = name; this.latch = latch; }

    @Override
    public void run() {
        System.out.println("Initializing " + serviceName + "...");
        try { Thread.sleep((long) (Math.random() * 2000)); } catch (Exception e) {}
        System.out.println(serviceName + " initialized.");
        latch.countDown(); // ✅ Signal that this service is done
    }
}

public class LatchDemo {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(3); // Wait for 3 services
        ExecutorService executor = Executors.newFixedThreadPool(3);

        executor.execute(new ServiceInitializer("Database", latch));
        executor.execute(new ServiceInitializer("Cache", latch));
        executor.execute(new ServiceInitializer("Logging", latch));

        System.out.println("Main thread is waiting for services to start...");
        latch.await(); // 🛑 Blocks until the count reaches zero

        System.out.println("🎉 All services are up. Application is starting!");
        executor.shutdown();
    }
}
```

### Example 2: `CyclicBarrier` for a Multi-Player Game
**Scenario:** Oka 4-player game lo, andaru players ready ayyaka game start avvali. Prathi round aypoyaka, andaru next round ki ready ayyevaraku wait cheyali.
```java
// File: com/example/barriers/BarrierDemo.java
package com/example/barriers;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

class Player implements Runnable {
    private final String name;
    private final CyclicBarrier barrier;

    Player(String name, CyclicBarrier barrier) { this.name = name; this.barrier = barrier; }

    @Override
    public void run() {
        try {
            System.out.println(name + " has joined the game.");
            barrier.await(); // Wait for all players to join

            // Round 1
            System.out.println(name + " is playing Round 1...");
            Thread.sleep((long) (Math.random() * 2000));
            System.out.println(name + " finished Round 1.");
            barrier.await(); // Wait for all players to finish Round 1

            // Round 2
            System.out.println(name + " is playing Round 2...");

        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    }
}

public class BarrierDemo {
    public static void main(String[] args) {
        // A barrier for 4 players, and a "barrier action" that runs when barrier is broken
        CyclicBarrier barrier = new CyclicBarrier(4, () -> {
            System.out.println("\n--- All players are ready. Starting next round! ---\n");
        });

        ExecutorService executor = Executors.newFixedThreadPool(4);
        executor.execute(new Player("Player 1", barrier));
        executor.execute(new Player("Player 2", barrier));
        executor.execute(new Player("Player 3", barrier));
        executor.execute(new Player("Player 4", barrier));

        executor.shutdown();
    }
}
```

---

## ✅ Checkpoint: Did You Master This?
- [ ] Main thread 5 worker threads kosam wait cheyali. `CountDownLatch` use chestava or `CyclicBarrier`? Why?
- [ ] `CountDownLatch` ni reset cheyochha?
- [ ] `CyclicBarrier` lo oka thread `await()` deggara wait chestunnapudu interrupt aite, em avtundi? (Hint: `BrokenBarrierException`)
- [ ] `CyclicBarrier` constructor lo `Runnable` enduku pass chestam?

**✅ Ready?** → [Next: Permit-Based Access - Semaphore](./02-Permit-Based-Access-Semaphore.md)
