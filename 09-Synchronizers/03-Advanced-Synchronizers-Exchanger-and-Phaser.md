---
# 🎯 9.4 & 9.5: Advanced Synchronizers - Exchanger & Phaser

## 🗺️ Learning Path Position

**📍 You Are Here:** 9.4 & 9.5 - Advanced Synchronizers

**✅ Prerequisites (Must Complete First):**
- [9.1 & 9.2: Barriers](./01-Barriers-CountDownLatch-and-CyclicBarrier.md) - `CyclicBarrier` gurinchi teliste, `Phaser` yokka flexibility better ga ardham avtundi.

**🔜 Coming After This:**
- [Project: Concurrent Service Initializer](./projects/01-Concurrent-Service-Initializer.md) - Manam nerchukunna `CountDownLatch` and `Semaphore` ni oka project lo vadudam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 🤔 What & Why

### The Problem: Even `CyclicBarrier` Isn't Flexible Enough!
Mawa, manam `CyclicBarrier` chusam, adi fixed number of parties kosam pani chestundi. Kaani, real-world lo manaki inka dynamic situations untayi:
- "Oka group of threads pani start chesayi, kaani madhyalo inkonni kotha threads join avvali." `CyclicBarrier` lo idi possible kadu.
- "Rendu threads madhyalo data ni swap cheskovali. Veedi data vaadiki, vaadi data veediki ivvali, at the same time." `BlockingQueue` lanti tools tho idi cheyochu, kaani adi konchem overkill. Manaki direct handoff mechanism kavali.

### The Solution: `Exchanger` and `Phaser`

#### 1. `Exchanger<V>`
- **Core Idea:** Correct ga rendu threads madhyalo data ni swap (exchange) cheyadaniki oka synchronization point.
- **How it Works:**
  - Rendu threads `exchanger.exchange(data)` ani call chestayi.
  - First thread `exchange()` call chesinappudu, adi block avtundi and second thread kosam wait chestundi.
  - Second thread `exchange()` call cheyagane, handoff jarugutundi: first thread ki second thread data vastundi, second thread ki first thread data vastundi.
- **When to Use:** Genetic algorithms, pipeline designs lo stages madhyalo data pass cheyadaniki.
- **Analogy:** Relay Race 🏃. Rendu runners unnaru. Runner A oka baton (`data`) tho run chestu vachi, oka specific point deggara Runner B kosam wait chestadu. Runner B inko baton tho vachi, aa point deggara Runner A ni kalustadu. Okate sari, A tana baton ni B ki, B tana baton ni A ki istaru.

#### 2. `Phaser`
- **Core Idea:** `CyclicBarrier` ki inka flexible, dynamic version anuko.
- **Key Features over `CyclicBarrier`:**
  - **Dynamic Parties:** Number of parties (threads) anedi fixed kadu. Running lo unnappudu kotha parties ni register cheskovachu (`register`), or unregister cheskovachu (`arriveAndDeregister`).
  - **Phases:** Barriers laage, `Phaser` kuda phases lo pani chestundi. Prathi phase number untundi (`getPhase()`).
  - **Flexible await:** `awaitAdvance(phase)` method use chesi, oka specific phase complete ayyevaraku wait cheyochu. `arriveAndAwaitAdvance()` anedi `CyclicBarrier.await()` laantidi.
- **When to Use:** Highly dynamic parallel computations, where the number of threads can change over time.
- **Analogy:** Tour group 🚌. Initially 10 mandi start avtaru (`Phaser phaser = new Phaser(10)`). Andaru first monument deggara kalustaru (`arriveAndAwaitAdvance`). Akkada inko 2 friends join avtaru (`register`). Ippudu total 12 mandi. Andaru next monument ki veltaru. Akkada oka person intiki vellipothadu (`arriveAndDeregister`). Ippudu total 11 mandi. Chala flexible.

---

## 💻 Code Examples

### Example 1: `Exchanger` for a Producer-Consumer Swap
**Scenario:** Producer oka buffer ni data tho fill chesi, Consumer ki istadu. At the same time, consumer tana empty buffer ni producer ki istadu. Ee swap valla, memory allocation taggutundi and performance perugutundi.
```java
// File: com/example/advanced/ExchangerDemo.java
package com/example/advanced;

import java.util.concurrent.Exchanger;

public class ExchangerDemo {
    public static void main(String[] args) {
        Exchanger<String> exchanger = new Exchanger<>();

        // Producer fills a buffer and exchanges it for an empty one
        new Thread(() -> {
            String buffer = "Initial Data";
            try {
                System.out.println("Producer has: " + buffer);
                // Exchange the full buffer for an empty one from the consumer
                buffer = exchanger.exchange(buffer);
                System.out.println("Producer now has: " + buffer);
            } catch (InterruptedException e) {}
        }).start();

        // Consumer processes a buffer and exchanges it for a full one
        new Thread(() -> {
            String buffer = "Empty Buffer";
            try {
                System.out.println("Consumer has: " + buffer);
                Thread.sleep(2000); // Consumer is slow
                // Exchange the empty buffer for a full one from the producer
                buffer = exchanger.exchange(buffer);
                System.out.println("Consumer now has: " + buffer);
            } catch (InterruptedException e) {}
        }).start();
    }
}
```

### Example 2: `Phaser` for a Dynamic Fork-Join Style Task
**Scenario:** Manam 3 worker threads tho oka task start cheddam. First phase lo, prathi worker oka pani chesi, inko 2 sub-workers ni create chestadu. Andaru workers (original + new) second phase complete cheyali.
```java
// File: com/example/advanced/PhaserDemo.java
package com/example/advanced;

import java.util.concurrent.Phaser;

public class PhaserDemo {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(1); // 1 registered party (the main thread)

        // Start 3 initial worker threads
        for (int i = 0; i < 3; i++) {
            phaser.register(); // Register a new party
            new Thread(new Worker(phaser, "Worker-" + i)).start();
        }

        System.out.println("Main thread is waiting for Phase 0 to complete...");
        phaser.arriveAndAwaitAdvance(); // Wait for the 3 workers + main thread

        System.out.println("Phase 0 complete. All workers and sub-workers are now running.");

        // Final wait for all work to finish
        phaser.arriveAndDeregister();
        System.out.println("All phases complete. Main thread is done.");
    }

    static class Worker implements Runnable {
        private final Phaser phaser;
        private final String name;
        Worker(Phaser phaser, String name) { this.phaser = phaser; this.name = name; }

        @Override
        public void run() {
            System.out.println(name + " starting Phase 0.");
            // Do Phase 0 work
            phaser.arriveAndAwaitAdvance(); // Arrive at the barrier

            // In Phase 1, create sub-workers (only for initial workers)
            if (name.startsWith("Worker")) {
                phaser.register(); // Register a sub-worker
                new Thread(new SubWorker(phaser, name + "-Sub")).start();
            }

            System.out.println(name + " starting Phase 1.");
            // Do Phase 1 work
            phaser.arriveAndDeregister(); // Arrive and deregister
        }
    }

    static class SubWorker implements Runnable {
        // ... similar logic, but doesn't create more workers
        private final Phaser phaser;
        private final String name;
        SubWorker(Phaser phaser, String name) { this.phaser = phaser; this.name = name; }

        @Override
        public void run() {
            System.out.println(name + " starting Phase 1.");
            phaser.arriveAndDeregister();
        }
    }
}
```

---

## ✅ Checkpoint: Did You Master This?
- [ ] `Exchanger` ni 3 threads tho use cheyochha? Why or why not?
- [ ] `Phaser` ki `CyclicBarrier` ki unna main advantage enti?
- [ ] `phaser.register()` and `phaser.arriveAndDeregister()` eppudu use chestam?
- [ ] `Phaser` lo "phase" number ela teluskuntam?

**✅ Ready?** → [Project: Concurrent Service Initializer](./projects/01-Concurrent-Service-Initializer.md)
