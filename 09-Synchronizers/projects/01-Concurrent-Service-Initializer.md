# 🏗️ Mini Project: Concurrent Service Initializer

## 🤔 Problem Statement
Mawa, manam oka application startup sequence ni simulate cheddam. Modern applications lo, main application start ayye mundu, chala independent services (like Database Connection Pool, Cache, Messaging Client) ready ga undali. Ee services anni parallel ga start avvali to reduce startup time. Main thread anedi, ee critical services anni start ayyevaraku wait cheyali.

Inka, manam oka `SharedResource` (e.g., a special analytics client) ni create cheddam, daanini at a time limited number of threads matrame access cheyagalali.

**Requirements:**
1.  Application startup lo 3 core services (`Database`, `Cache`, `Messaging`) ni parallel ga initialize chey.
2.  Main application thread anedi ee 3 services antha `UP` ayyevaraku block avvali. Idi `CountDownLatch` tho cheddam.
3.  Manam oka `LimitedResource` ni kuda simulate cheddam, daaniki kevalam 2 "permits" matrame untayi.
4.  Multiple threads ee `LimitedResource` ni access cheyadaniki try chestaru, kaani `Semaphore` use chesi, access ni limit cheddam.

**This Project Uses Concepts From:**
- ✅ [9.1: CountDownLatch](../01-Barriers-CountDownLatch-and-CyclicBarrier.md)
- ✅ [9.3: Semaphore](../02-Permit-Based-Access-Semaphore.md)
- ✅ [6.1: Executor Framework](../../06-Executor-Framework/01-Introduction-to-Executor-Framework.md)

---

## 🏗️ Architecture
Main thread anedi oka `CountDownLatch(3)` ni, oka `Semaphore(2)` ni, and oka `ExecutorService` ni create chestundi. Tarvata, adi 3 `ServiceInitializer` tasks ni (`Database`, `Cache`, `Messaging`) submit chestundi, prathi task ki latch ni pass chestu. Main thread `latch.await()` deggara wait chestundi. Prathi service start ayyaka, adi `latch.countDown()` call chestundi.

Anni services start ayyaka, main thread unblock avtundi. Tarvata, manam inkonni tasks ni submit chesi `LimitedResource` ni access cheyadaniki try cheddam. `Semaphore` anedi access ni restrict chestundi.

```mermaid
graph TD
    subgraph Startup Phase
        A[Main Thread starts] --> B(Create CountDownLatch(3));
        B --> C(Create ExecutorService);
        C --> D[Submit 3 ServiceInitializers];
        D --> E[main.await()];
    end

    subgraph Service Initialization (Parallel)
        S1[DB Service] -- countDown() --> Latch;
        S2[Cache Service] -- countDown() --> Latch;
        S3[Messaging Service] -- countDown() --> Latch;
        Latch -- count == 0 --> E;
    end

    subgraph Post-Startup Phase
        E -- unblocks --> F[App is Running!];
        F --> G[Submit 5 ResourceUsers];
    end

    subgraph Resource Access (Throttled)
      G --> R[LimitedResource];
      R -- uses --> Sem(Semaphore(2));
    end

    style Startup Phase fill:#87CEEB
    style Post-Startup Phase fill:#90EE90
```

---

## 💻 Complete Code

#### File 1: `ConcurrentServiceInitializer.java` (Main Class)
```java
// File: src/com/example/startup/ConcurrentServiceInitializer.java
package com.example.startup;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class ConcurrentServiceInitializer {

    public static void main(String[] args) throws InterruptedException {
        // --- Part 1: Service Initialization with CountDownLatch ---
        final CountDownLatch latch = new CountDownLatch(3);
        ExecutorService startupExecutor = Executors.newFixedThreadPool(3);

        System.out.println("🚀 Starting core services...");
        startupExecutor.execute(new Service("Database", 2000, latch));
        startupExecutor.execute(new Service("Cache", 1000, latch));
        startupExecutor.execute(new Service("Messaging", 2500, latch));

        System.out.println("... Main thread waiting for services to be operational.");
        // Main thread blocks here until the latch count is zero
        latch.await();
        System.out.println("🎉 All core services are UP! Main application can now start.\n");
        startupExecutor.shutdown();

        // --- Part 2: Throttling access with Semaphore ---
        final Semaphore resourceLimiter = new Semaphore(2, true); // 2 permits, fair
        LimitedResource resource = new LimitedResource(resourceLimiter);
        ExecutorService workerExecutor = Executors.newFixedThreadPool(5);

        System.out.println("🔥 5 worker threads will now try to access the limited resource (2 permits)...");
        for (int i = 0; i < 5; i++) {
            workerExecutor.execute(() -> {
                try {
                    resource.use();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            });
        }

        workerExecutor.shutdown();
    }
}

// Represents a service that needs to be started
class Service implements Runnable {
    private final String name;
    private final int startupTime;
    private final CountDownLatch latch;

    Service(String name, int time, CountDownLatch latch) {
        this.name = name;
        this.startupTime = time;
        this.latch = latch;
    }

    @Override
    public void run() {
        try {
            System.out.println("  -> Initializing " + name + "...");
            Thread.sleep(startupTime);
            System.out.println("  <- " + name + " is UP.");
            latch.countDown(); // Signal that this service is ready
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

// Represents a resource with limited concurrent access
class LimitedResource {
    private final Semaphore semaphore;

    LimitedResource(Semaphore semaphore) {
        this.semaphore = semaphore;
    }

    public void use() throws InterruptedException {
        System.out.println("   " + Thread.currentThread().getName() + " is trying to access the resource...");
        semaphore.acquire();
        try {
            System.out.println("✅ " + Thread.currentThread().getName() + " has acquired the resource.");
            Thread.sleep(1500); // Simulate using the resource
        } finally {
            System.out.println("❌ " + Thread.currentThread().getName() + " is releasing the resource.");
            semaphore.release();
        }
    }
}
```

---

## 🚀 How to Run & Test

### Step 1: Compile the Code
```bash
# Assuming you are in the project root
javac -d out src/com/example/startup/*.java
```

### Step 2: Run the Initializer
```bash
java -cp out com.example.startup.ConcurrentServiceInitializer
```

### Expected Output & Analysis
```
🚀 Starting core services...
... Main thread waiting for services to be operational.
  -> Initializing Database...
  -> Initializing Cache...
  -> Initializing Messaging...
  <- Cache is UP.
  <- Database is UP.
  <- Messaging is UP.
🎉 All core services are UP! Main application can now start.

🔥 5 worker threads will now try to access the limited resource (2 permits)...
   pool-2-thread-1 is trying to access the resource...
✅ pool-2-thread-1 has acquired the resource.
   pool-2-thread-2 is trying to access the resource...
✅ pool-2-thread-2 has acquired the resource.
   pool-2-thread-3 is trying to access the resource...
   pool-2-thread-4 is trying to access the resource...
   pool-2-thread-5 is trying to access the resource...
❌ pool-2-thread-1 is releasing the resource.
   pool-2-thread-3 has acquired the resource.
...
```
**Analysis:**
- **`CountDownLatch` in Action:** "All core services are UP!" ane message anedi, 3 services `countDown()` call chesevaraku print avvaledu. Main thread antha sepu `await()` deggara block aindi.
- **`Semaphore` in Action:** At any given time, "✅ ... has acquired the resource" ane message maximum rendu threads matrame print chesayi. Migatha threads anni `semaphore.acquire()` deggara wait chesayi, oka permit release ayyevaraku.

Ee project tho, nuvvu `CountDownLatch` and `Semaphore` lanti powerful synchronizers ni real-world startup and resource-throttling scenarios lo ela use cheyalo chusav. Chala manchi practice idi. Great job, mawa!
