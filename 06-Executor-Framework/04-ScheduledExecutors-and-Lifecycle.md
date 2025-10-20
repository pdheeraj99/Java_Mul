---
# 🎯 6.4 & 6.5: ScheduledExecutors & Lifecycle

## 🗺️ Learning Path Position

**📍 You Are Here:** 6.4 & 6.5 - ScheduledExecutors & Lifecycle

**✅ Prerequisites (Must Complete First):**
- [6.3: ThreadPoolExecutor Deep Dive](./03-ThreadPoolExecutor-Deep-Dive.md) - Custom thread pools ela create cheyalo teliyali.

**🔜 Coming After This:**
- [7.1: Callable and Future](../07-Callable-Future-CompletableFuture/01-Callable-and-Future.md) - Executor framework tho tasks nunchi results ni return cheyadam nerchukuntam.
- [Phase End] - We're wrapping up the Executor Framework! 🎉

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: Running Tasks "Later" or "Repeatedly" ⏰
Mawa, ఇప్పటివరకు మనం tasks ని submit cheyagane, అవి immediately execute avvadaniki ready aypoyayi. Kaani real world lo, manaki ilanti requirements untayi:
- "Ee task ni correct ga 10 seconds tarvata run chey." (e.g., OTP timeout message)
- "Prathi 5 nimishalaki, mana database health check chey." (e.g., monitoring)
- "Prathi roju midnight 12 ki, daily reports generate chey." (e.g., cron job)
`ThreadPoolExecutor` tho idi direct ga cheyalem.

Inko problem: manam `executor.shutdown()` gurinchi matladam, kaani daani venaka unna details ento inka chudaledu. Shutdown process ni ela control cheyali? Running tasks ni madhyalo aapali ante ela?

### The Solution: `ScheduledExecutorService` & Graceful Shutdown
1.  **`ScheduledExecutorService`:** Idi `ExecutorService` ni extend chese oka special interface. Idi tasks ni oka specific delay tarvata or periodic ga run cheyadaniki manaki methods istundi.
2.  **Lifecycle Methods (`shutdown`, `shutdownNow`):** `ExecutorService` manaki shutdown process meeda fine-grained control istundi. Manam decide cheyochu - new tasks aapi, running tasks ni complete avvanivvala (`shutdown`) or anni threads ni immediately interrupt cheyala (`shutdownNow`).

### Real-World Analogy: Your Smartphone Alarms 📱
- **`schedule()`:** Nuvvu "Remind me in 15 minutes to check the oven" ani alarm pettinattu. Okasari mogutundi, aypoindi.
- **`scheduleAtFixedRate()`:** Nuvvu prathi roju morning 7 AM ki alarm pettinattu. Correct ga 7 ki mogutundi, nuvvu lechina lekapoyina. Okaవేళ nuvvu snooze chesi late ga lechina, next alarm malli correct ga 7 AM ke mogutundi. It's based on a **fixed clock time**.
- **`scheduleWithFixedDelay()`:** Nuvvu "Remind me every 1 hour *after I finish my work*" ani alarm pettinattu. Nuvvu work cheyadaniki 30 mins teskunte, alarm 1 hour 30 mins tarvata mogutundi. Work cheyadaniki 2 hours padithe, alarm 3 hours tarvata mogutundi. It's based on the **completion time of the previous task**.

---

## 📚 Detailed Explanation

### 1. `ScheduledExecutorService` Methods

| Method | Description | Use Case |
|---|---|---|
| `schedule(Runnable/Callable, long delay, TimeUnit unit)` | Task ni oka specific `delay` tarvata **okasari** matrame execute chestundi. | OTP validity check, one-time notifications. |
| `scheduleAtFixedRate(Runnable, long initialDelay, long period, TimeUnit unit)`| Task ni `initialDelay` tarvata start chesi, tarvata prathi `period` ki execute chestundi. **Start time** of tasks is fixed. | Prathi minute ki stock price update cheyadam. Health checks. |
| `scheduleWithFixedDelay(Runnable, long initialDelay, long delay, TimeUnit unit)`| Task ni `initialDelay` tarvata start chesi, oka execution **complete ayina tarvata** `delay` aagi, next execution start chestundi. | Previous task output meeda depend ayye background jobs. |

#### 🔥 `scheduleAtFixedRate` vs `scheduleWithFixedDelay` - The CRITICAL Difference

Idi chala common interview question, mawa.
- **`scheduleAtFixedRate`:** Execution start time is constant. Task run avvadaniki `period` kanna ekkuva time padithe, next task immediately start avtundi (or even run concurrently if the pool has multiple threads). Task executions can bunch up.
- **`scheduleWithFixedDelay`:** Guarantees a fixed gap (`delay`) between the end of one task and the start of the next. The execution frequency will be lower if the task duration is long.

### 2. Executor Lifecycle Management

| Method | Description | Behavior |
|---|---|---|
| `shutdown()` | Initiates a **graceful shutdown**. | New tasks accept cheyadu. Already running tasks continue chestayi. Queued tasks ni kuda execute chestundi. |
| `shutdownNow()` | Initiates a **forceful shutdown**. | New tasks accept cheyadu. Running threads ni `Thread.interrupt()` tho aapadaniki try chestundi. Queue lo unna tasks ni execute cheyadu, vaatini return chestundi. |
| `awaitTermination(long timeout, TimeUnit unit)`| Current thread ni block chesi, shutdown complete ayyevaraku (or timeout) wait chestundi.| `shutdown()` call chesina tarvata idi call cheyali to ensure all tasks are finished before moving on. |

---

## 💻 Code Examples

### Example 1: `schedule()` & `scheduleAtFixedRate()`
```java
// File: com/example/scheduling/SchedulingTasks.java
package com.example.scheduling;

import java.util.concurrent.*;

public class SchedulingTasks {
    public static void main(String[] args) {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        System.out.println("Current time: " + System.currentTimeMillis());

        // Task 1: Run once after 3 seconds
        scheduler.schedule(() -> {
            System.out.println("One-time task executed at: " + System.currentTimeMillis());
        }, 3, TimeUnit.SECONDS);

        // Task 2: Run every 2 seconds after an initial 5-second delay
        Runnable periodicTask = () -> {
            System.out.println("Periodic task (Fixed Rate) running at: " + System.currentTimeMillis());
        };
        scheduler.scheduleAtFixedRate(periodicTask, 5, 2, TimeUnit.SECONDS);

        // Let the scheduler run for 15 seconds then shut it down
        try {
            Thread.sleep(15000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("Shutting down scheduler.");
        scheduler.shutdown();
    }
}
```

### Example 2: `FixedRate` vs `FixedDelay` - The Showdown 🔥

**Scenario:** Manaki oka task undi, adi run avvadaniki 3 seconds padutundi. Manam daanini prathi 2 seconds ki run cheyalani try cheddam.

#### ❌ Common Confusion with `scheduleAtFixedRate`
```java
// File: com/example/ratevsdelay/RateVsDelay.java
package com.example.ratevsdelay;

import java.util.concurrent.*;

public class RateVsDelay {
    public static void main(String[] args) throws InterruptedException {
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

        System.out.println("Starting scheduleAtFixedRate test at: " + System.currentTimeMillis());
        Runnable longTask = () -> {
            try {
                System.out.println("--> Task Start: " + System.currentTimeMillis());
                Thread.sleep(3000); // 🕒 Task takes 3 seconds
                System.out.println("<-- Task End:   " + System.currentTimeMillis());
            } catch (InterruptedException e) { e.printStackTrace(); }
        };

        // Schedule to run every 2 seconds. But the task itself takes 3 seconds!
        scheduler.scheduleAtFixedRate(longTask, 0, 2, TimeUnit.SECONDS);

        Thread.sleep(10000);
        scheduler.shutdownNow();

        System.out.println("\n-----------------------------------\n");

        scheduler = Executors.newScheduledThreadPool(1);
        System.out.println("Starting scheduleWithFixedDelay test at: " + System.currentTimeMillis());
        // Schedule with a 2-second delay between completions
        scheduler.scheduleWithFixedDelay(longTask, 0, 2, TimeUnit.SECONDS);

        Thread.sleep(15000);
        scheduler.shutdownNow();
    }
}
```
**Output:**
```
Starting scheduleAtFixedRate test at: 1668853200000
--> Task Start: 1668853200000
<-- Task End:   1668853203000
--> Task Start: 1668853203000  // 🚨 No delay! Starts immediately after previous ends.
<-- Task End:   1668853206000
--> Task Start: 1668853206000  // 🚨 No delay again!
<-- Task End:   1668853209000

-----------------------------------

Starting scheduleWithFixedDelay test at: 1668853210000
--> Task Start: 1668853210000
<-- Task End:   1668853213000
--> Task Start: 1668853215000  // ✅ Guaranteed 2-second delay after end.
<-- Task End:   1668853218000
--> Task Start: 1668853220000  // ✅ Guaranteed 2-second delay again.
<-- Task End:   1668853223000
```
**Why This Happens:** `scheduleAtFixedRate` tries to maintain the schedule (e.g., at 0s, 2s, 4s, 6s...). Kaani 0s lo start ayina task 3s varaku run avvadam valla, 2s schedule miss aipoindi. So, adi next task ni immediately start chesestundi to catch up. `scheduleWithFixedDelay` alanti tension em teskodu, "Okati aypoyaka, 2 seconds aagi, next start chey" ani simple ga untundi.

### Example 3: `shutdown()` vs `shutdownNow()`

```java
// File: com/example/shutdown/ShutdownDemo.java
package com.example.shutdown;

import java.util.List;
import java.util.concurrent.*;

public class ShutdownDemo {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = Executors.newFixedThreadPool(1);

        // Submit a task that can be interrupted
        executor.submit(() -> {
            try {
                System.out.println("Task started, will run for 10 seconds...");
                Thread.sleep(10000);
                System.out.println("Task finished normally."); // Ee line eppudu print avvadu
            } catch (InterruptedException e) {
                System.out.println("Task was interrupted!");
            }
        });

        // Submit more tasks that will wait in the queue
        executor.submit(() -> System.out.println("Queued Task 1"));
        executor.submit(() -> System.out.println("Queued Task 2"));

        Thread.sleep(1000); // Running task konchem sepu run avvani

        // --- Now, shutdown forcefully ---
        System.out.println("Calling shutdownNow()...");
        List<Runnable> unexecutedTasks = executor.shutdownNow();

        System.out.println("Executor is shutting down.");
        System.out.println(unexecutedTasks.size() + " tasks were in the queue.");

        if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
            System.err.println("Executor did not terminate in time.");
        }
        System.out.println("Executor terminated.");
    }
}
```
**Output:**
```
Task started, will run for 10 seconds...
Calling shutdownNow()...
Executor is shutting down.
Task was interrupted!
2 tasks were in the queue.
Executor terminated.
```
**Why This Works:** `shutdownNow()` first running thread ki `interrupt()` signal pampindi, daanivalla `InterruptedException` vachi task aagipoindi. Tarvata, queue lo unna 2 tasks ni process cheyakunda return chesesindi.

---
## 🐛 Debugging Guide

### Common Issue: Silent Task Failure in Schedulers
**Symptom:** Nuvvu oka periodic task schedule chesav. Adi konni sarlu run ayyi, tarvata aagipoindi. Error logs lo em kanipinchadu.
**Meaning:** Idi `ScheduledExecutorService` yokka most dangerous behavior. Task lo edaina unhandled exception (`RuntimeException` or `Error`) vasthe, scheduler aa task yokka future executions ni silently cancel chestundi. No warning, no logging.
**Solution:**
**ALWAYS** wrap your `Runnable`'s logic in a `try-catch (Throwable)` block.
```java
// ❌ DANGEROUS CODE
Runnable badTask = () -> {
    System.out.println("Running...");
    if (Math.random() > 0.5) {
        throw new RuntimeException("Something went wrong!");
    }
};

// ✅ SAFE CODE
Runnable goodTask = () -> {
    try {
        System.out.println("Running safely...");
        if (Math.random() > 0.5) {
            throw new RuntimeException("Something went wrong!");
        }
    } catch (Throwable t) {
        // Log the error properly with a logging framework
        System.err.println("ERROR in scheduled task: " + t.getMessage());
    }
};

// Use goodTask in your scheduler
// scheduler.scheduleAtFixedRate(goodTask, 1, 1, TimeUnit.SECONDS);
```

---

## 🧪 Testing Examples

### Unit Test: Testing Graceful Shutdown Logic
**Scenario:** Manam oka `ServiceManager` class ni test cheddam. Adi `shutdown()` call chesinappudu, running tasks ni complete avvanichi, queued tasks ni execute cheyకుండా untundo ledo verify cheddam.
```java
// File: com/example/testing/ServiceManager.java
package com.example.testing;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.TimeUnit;

public class ServiceManager {
    private final ExecutorService executor;

    public ServiceManager(ExecutorService executor) {
        this.executor = executor;
    }

    public void gracefullyShutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }
    }
}

// File: com/example/testing/ServiceManagerTest.java
package com.example.testing;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class ServiceManagerTest {
    @Test
    void testGracefulShutdownCompletesAllTasks() throws InterruptedException {
        // Arrange
        ExecutorService executor = Executors.newSingleThreadExecutor();
        ServiceManager manager = new ServiceManager(executor);
        AtomicInteger counter = new AtomicInteger(0);

        // Submit a task that runs for 1 second
        executor.submit(() -> {
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
            counter.incrementAndGet();
        });

        // Submit another task that will stay in the queue
        executor.submit(counter::incrementAndGet);

        Thread.sleep(100); // Let the first task start

        // Act
        manager.gracefullyShutdown(); // This should wait for BOTH tasks to complete.

        // Assert
        assertTrue(executor.isTerminated());
        assertEquals(2, counter.get(), "shutdown() should wait for running AND queued tasks to complete.");
    }
}
```

---

## ❌ Anti-Patterns

### Anti-Pattern: Using `Thread.sleep()` instead of a Scheduler
**What's Wrong:** Oka task ni periodic ga run cheyadaniki, `while(true)` loop lo `Thread.sleep()` pettadam.
```java
// ❌ BAD CODE
new Thread(() -> {
    while (true) {
        try {
            System.out.println("Doing work...");
            Thread.sleep(5000); // Manually sleeping
        } catch (InterruptedException e) {
            // ...
        }
    }
}).start();
```
**Why Bad:**
1.  **Inaccurate:** `sleep` time guarantee kadu. OS antha kante ekkuva time kuda teskovachu.
2.  **No Central Management:** Ee thread gurinchi `ExecutorService` ki teliyadu. Shutdown cheyali ante, daaniki separate logic rayali.
3.  **Error Handling is a Pain:** Exception vasthe, ee loop break avtundi and task silent ga aagipotundi.
**✅ Correct Way:** Use `ScheduledExecutorService`. It's designed for this exact purpose.

---

## ✨ Key Takeaways (Top 5)

1.  **Schedulers are for "When", not just "What" 🗓️:** Regular executors tasks ni *em* cheyalo cheptayi. Schedulers *eppudu* cheyalo kuda cheptayi.
2.  **Rate vs Delay is a Critical Choice ⚖️:** `scheduleAtFixedRate` anedi clock-time ki sync avvadaniki try chestundi. `scheduleWithFixedDelay` anedi tasks madhyalo gap ivvadaniki guarantee istundi. Nee requirement batti choose chesko.
3.  **Exceptions are Silent Killers 🤫:** Scheduled tasks lo unhandled exceptions future executions ni aapistayi. Always use `try-catch(Throwable)`.
4.  **Shutdown is a Two-Step Process 🚶‍♂️🚶‍♂️:** First, `shutdown()` tho graceful ga try chey. `awaitTermination` tho wait chey. Work avvakapothe, appudu `shutdownNow()` tho force chey.
5.  **`shutdownNow()` is not a Guarantee:** Idi `Thread.interrupt()` ni matrame call chestundi. Nee task `InterruptedException` ni handle cheyakapothe, adi aagadu.

---

## 💪 Practice Exercises

### Exercise 1 (Medium): ⭐⭐
**Problem:** Oka `ScheduledExecutorService` use chesi, prathi 3 seconds ki current timestamp ni print chey. Kaani, task lo random ga (`Math.random() > 0.7`) oka `RuntimeException` throw chey. Proper exception handling (`try-catch`) lekunda run chesi em avtundo chudu, tarvata `try-catch` petti fix chey.

### Exercise 2 (Hard): ⭐⭐⭐
**Problem:** `shutdown()` vs `shutdownNow()` behavior ni demonstrate chese oka program rayi.
1.  Oka `FixedThreadPool` create chey.
2.  Oka long-running, interruptible task (e.g., loop with `Thread.sleep()`) submit chey.
3.  Konni tasks ni queue lo pettadaniki submit chey.
4.  First, `shutdown()` use chesi chudu - running task complete avtundi, queued tasks kuda execute avtayi.
5.  Tarvata, adhe setup ni `shutdownNow()` tho run chey - running task interrupt avvali, queued tasks return avvali.

---

## 📝 Quick Reference Card

| Method | Use For | Key Behavior |
|---|---|---|
| `schedule()` | One-time delayed task | Runs once after a delay. |
| `scheduleAtFixedRate()`| Fixed-frequency tasks | Tries to maintain start time. Executions can bunch up. |
| `scheduleWithFixedDelay()`| Tasks with a gap | Guarantees a delay between task *end* and next *start*. |
| `shutdown()` | Graceful shutdown | Stops accepting new tasks, waits for existing to finish. |
| `shutdownNow()` | Forceful shutdown | Interrupts running tasks, discards queued tasks. |
| `awaitTermination()`| Blocking wait for shutdown| Use after `shutdown()` to wait for completion. |

---

## ✅ Checkpoint: Did You Master This?
- [ ] `scheduleAtFixedRate(task, 0, 5, SECONDS)` use cheste, and task run avvadaniki 6 seconds padithe, em avtundi?
- [ ] `shutdown()` ki `shutdownNow()` ki unna main difference enti? `shutdownNow()` running task ni ela stop chestundi?
- [ ] oka periodic task lo `NullPointerException` vasthe em avtundi?
- [ ] `awaitTermination()` enduku use chestam?

**✅ Ready?** → [Next: Callable and Future](../07-Callable-Future-CompletableFuture/01-Callable-and-Future.md) - Let's move to the next phase!

**😕 Need Review?** → Paiki scroll chesi, Smartphone Alarms analogy and the Rate vs Delay code example ni malli chudu.
