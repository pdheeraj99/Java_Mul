---
# 🎯 1.2: Creating & Managing Threads

## 🗺️ Learning Path Position

**📍 You Are Here:** 1.2 - Creating & Managing Threads

**✅ Prerequisites (Must Complete First):**
- [1.1: Core Concepts & Theory](./01-Core-Concepts-and-Theory.md) - Process, Thread, Concurrency ante ento teliyali.

**🔜 Coming After This:**
- [1.3: Java Memory Model](./03-Java-Memory-Model.md) - Threads memory ni ela share cheskuntayo inka deep ga chustam.

**⏱️ Estimated Time:** 2.5 hours
**📊 Difficulty Level:** 🟢 Beginner

---
(Sections 1 & 2 are unchanged)
---
### Example 3: `join()` - Waiting for a Thread to Finish

**Scenario:** Oka background thread lo calculation jarugutundi. Main thread aa calculation result kosam wait cheyali.

#### ✅ Success Case: Using `join()`
```java
// File: com/example/join/CalculationTask.java
package com.example.join;

public class CalculationTask implements Runnable {
    private long result;

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + ": Performing a long calculation...");
        long sum = 0;
        for (int i = 0; i < 1000000; i++) {
            sum += i; // Simulate some work
        }
        result = sum;
        System.out.println(Thread.currentThread().getName() + ": Calculation complete.");
    }

    public long getResult() {
        return result;
    }
}

// File: com/example/join/Main.java
package com.example.join;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        CalculationTask task = new CalculationTask();
        Thread calcThread = new Thread(task, "Calculator-Thread");

        System.out.println("Main thread: Starting the calculation thread.");
        calcThread.start();

        System.out.println("Main thread: Waiting for the calculation to finish...");
        // ✅ Magic line: Main thread BLOCKS here until calcThread is TERMINATED
        calcThread.join();

        System.out.println("Main thread: Calculation finished. The result is " + task.getResult());
    }
}
```
**Output:**
```
Main thread: Starting the calculation thread.
Main thread: Waiting for the calculation to finish...
Calculator-Thread: Performing a long calculation...
Calculator-Thread: Calculation complete.
Main thread: Calculation finished. The result is 499999500000
```
**Why This Works:** `calcThread.join()` call cheyagane, main thread `WAITING` state loki veltundi. Eppudaithe `calcThread` `TERMINATED` avtundo, appudu main thread resume ayyi correct result ni print chestundi.

#### ❌ Failure Case #1: Not using `join()` (Race Condition)
```java
// Inside Main.java's main method
public static void main(String[] args) {
    CalculationTask task = new CalculationTask();
    Thread calcThread = new Thread(task, "Calculator-Thread");
    calcThread.start();

    // ❌ Mistake: Wait cheyakunda result adigestunnam.
    System.out.println("Main thread: The result is " + task.getResult());
}
```
**Output:**
`Main thread: The result is 0`
**Why This Fails:** Main thread `start()` call chesi ventane `getResult()` adigesindi. Appatiki calculation thread inka pani complete cheyaledu. So, default value `0` vachindi.

---
(Unchanged sections)
---

### Example 5: Daemon Threads - The Background Helpers

**Scenario:** Main application run avutunnapudu background lo health checks chese thread. Main app aagipothe, idi kuda aagipovali.

#### ✅ Correct Version: Daemon Thread
```java
// File: com/example/daemon/Main.java
package com.example.daemon;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        Runnable healthChecker = () -> {
            // ✅ This is a complete, runnable lambda
            while (true) {
                System.out.println("Health Checker: Checking system health...");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    // This allows the thread to be interrupted and exit gracefully
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        };

        Thread healthThread = new Thread(healthChecker, "Health-Checker-Thread");

        // ✅ Fix: Ee thread ni daemon ga set chey. start() ki mundu cheyali!
        healthThread.setDaemon(true);
        healthThread.start();

        System.out.println("Main thread is doing some work for 3 seconds...");
        Thread.sleep(3000);

        System.out.println("Main thread is finishing. JVM will now exit.");
    }
}
```
**Result:** "Main thread is finishing..." print avvagane, program exit aypotundi. Endukante, main thread aypoyindi, inka migilindi daemon thread okkate. So, JVM says "I can exit now".

#### ❌ Failure Case #1: Forgetting to set Daemon (User Thread)
```java
// Inside Main.java
public static void main(String[] args) throws InterruptedException {
    Runnable healthChecker = () -> { /* ... same as above ... */ };
    Thread healthThread = new Thread(healthChecker, "Health-Checker-Thread");

    // ❌ Mistake: We forgot to set it as a daemon. It's a user thread by default.
    healthThread.start();

    System.out.println("Main thread is finishing.");
}
```
**Result:** Ee program eppatiki aagadu! "Main thread is finishing." print ayina tarvata kuda, health checker print chestune untundi, endukante JVM user threads anni complete ayyevaraku wait chestundi.

---
(Remaining content unchanged)
---
