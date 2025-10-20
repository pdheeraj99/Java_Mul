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

## 💭 Quick Recap

💭 **Gurtunda ra?** Manam last session lo Thread ante ento nerukunnam - oka process lo unde chinna execution path ani. Ippudu aa threads ni manam ela create cheyali, valla lifecycle ni ela control cheyalo nerukuntam.

---

## 🤔 What & Why

### The Problem: How to do multiple tasks?
Theoretically anni threads, multitasking anukunnam. But practically, aa threads ni create cheyadam ela? Vati pani aypoyaka ela manage cheyali? Oka thread start avvaka mundu inkoti aagali ante ela? Ee questions ki answers ee session lo dorukutayi.

### The Solution: The `Thread` Class and `Runnable` Interface
Java manaki two primary ways ichindi threads create cheyadaniki:
1.  **`Thread` class ni extend cheyadam:** Inheritance use cheskuni.
2.  **`Runnable` interface ni implement cheyadam:** Composition use cheskuni. (🔥 **Idi Best Practice!**)

### Real-World Analogy: Restaurant Manager 👨‍💼
Imagine nuvvu oka restaurant manager vi.
- **`Runnable`:** Idi oka recipe book (`Task`). Andulo "How to make Biryani" ane steps untayi. Recipe book নিজে Biryani cheyaledu.
- **`Thread`:** Idi mana chef (`Worker`). Nuvvu chef ki recipe book isthe, chef aa pani chestadu.
- **`start()`:** "Chef, ee recipe tesko, pani start chey" ani cheppadam. Appudu chef velli separate ga pani start chestadu.
- **`run()`:** Nuvve manager vi ayyundi, chef pakkana nunchuni, "Step 1 chey, Step 2 chey" ani cheptunnav. Ikkada nuvve pani chestunnav, chef kadu!

---

## 💻 Code Examples

### Example 1: Creating a Thread by Extending `Thread` Class

#### ✅ Success Case: Simple Number Printer
**Scenario:** Manam 1 nunchi 5 varaku numbers print chese oka separate thread create cheddam.

```java
// File: com/example/threads/NumberPrinterThread.java
package com.example.threads;

public class NumberPrinterThread extends Thread {
    @Override
    public void run() {
        for (int i = 1; i <= 5; i++) {
            System.out.println("Thread ID: " + Thread.currentThread().getId() + " - Number: " + i);
            try {
                Thread.sleep(500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

// File: com/example/threads/Main.java
package com.example.threads;

public class Main {
    public static void main(String[] args) {
        System.out.println("Main thread started!");
        NumberPrinterThread printerThread = new NumberPrinterThread();
        printerThread.start();
        System.out.println("Main thread finished!");
    }
}
```
**Output (Example):**
```
Main thread started!
Main thread finished!
Thread ID: 17 - Number: 1
Thread ID: 17 - Number: 2
...
```
**Why This Works:** `start()` call cheyagane, a new thread is created and it moves to the `RUNNABLE` state. The main thread continues its own work.

#### ❌ Failure Case #1: Calling `run()` instead of `start()` (BIGGEST MISTAKE!)
**What Developers Do Wrong:** `start()` badulu `run()` method ni direct ga call chestaru.

```java
// Inside Main.java
// ...
// ❌ Common Mistake: Direct ga run() call chestunnaru.
printerThread.run();
// ...
```
**Output:**
```
Main thread started!
Thread ID: 1 - Number: 1
Thread ID: 1 - Number: 2
...
Main thread finished!
```
**Why This Fails:** `run()` ni direct ga call cheste, adi normal method laage **main thread** lo ne execute avtundi. Kotha thread create avvaledu! 🔥 **`start()` matrame kotha thread ni create chestundi.**

#### ❌ Failure Case #2: Starting the same Thread twice
**What Developers Do Wrong:** Oka `Thread` object ni okasari `start()` chesina tarvata, malli `start()` cheyadaniki try chestaru.

```java
// Inside Main.java
public static void main(String[] args) {
    NumberPrinterThread printerThread = new NumberPrinterThread();
    printerThread.start(); // First call is OK

    try {
        // Let the thread finish its work before we try to start it again
        printerThread.join();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    System.out.println("Trying to start the same thread again...");
    // ❌ Second call will fail
    printerThread.start();
}
```
**Error Message:**
```
Exception in thread "main" java.lang.IllegalThreadStateException
    at java.base/java.lang.Thread.start(Thread.java:793)
    ...
```
**Why This Fails:** Oka thread `TERMINATED` state ki vellina tarvata, daanini malli `RUNNABLE` state ki teeskuni ralevu. Thread anedi one-time use object anuko. Malli start cheyalante, kotha object create cheyalsinde.

#### 🔗 Concept Connections:
- **💭 Uses from previous:** [Thread Lifecycle](./01-Core-Concepts-and-Theory.md#4-thread-lifecycle--states-🚦) - `start()` method call cheste, thread `NEW` nunchi `RUNNABLE` state ki move avtundi.
- **📌 Will be used in:** [Executor Framework](../06-Executor-Framework/01-Executor-Basics.md) - Manam manually `new Thread().start()` cheyakunda, thread pools ee pani ni manage chestayi.

---

### Example 2: Creating a Thread by Implementing `Runnable` (❤️ The Better Way)
**Why `Runnable`?** Java lo multiple inheritance ledu. `Runnable` anedi interface, so daanini implement cheskuni, vere class ni extend cheskovachu. Chala flexible!

#### ✅ Success Case: File Downloader Task
**Scenario:** Oka file download chese task anukundam.
```java
// File: com/example/runnable/FileDownloaderTask.java
package com.example.runnable;

public class FileDownloaderTask implements Runnable {
    @Override
    public void run() {
        System.out.println("Downloader Thread: Starting file download...");
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Downloader Thread: File download complete!");
    }
}
// File: com/example/runnable/Main.java
package com.example.runnable;
public class Main {
    public static void main(String[] args) {
        FileDownloaderTask task = new FileDownloaderTask();
        Thread workerThread = new Thread(task);
        workerThread.start();
        System.out.println("Main thread: Continuing with other work...");
    }
}
```
**Output:**
```
Main thread: Starting application.
Main thread: Continuing with other work...
Downloader Thread: Starting file download...
Downloader Thread: File download complete!
```
**Why This Works:** Manam task (`Runnable`) ni worker (`Thread`) nunchi separate chesam. Idi chala clean design.

#### ❌ Failure Case #1: Passing a `null` Runnable
**What Developers Do Wrong:** Accidental ga `Thread` constructor ki `null` pass cheyadam.
```java
// Inside Main.java
public static void main(String[] args) {
    FileDownloaderTask task = null;
    // ❌ Passing null to the Thread constructor
    Thread workerThread = new Thread(task);
    workerThread.start();
}
```
**Error Message:**
```
Exception in thread "main" java.lang.NullPointerException
	at java.base/java.lang.Thread.<init>(Thread.java:423)
	...
```
**Why This Fails:** `Thread` constructor `null` `Runnable` ni accept cheyadu. It needs a valid task to execute.

#### 🔗 Concept Connections:
- **💭 Uses from previous:** [Separation of Concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) - Task logic (`Runnable`) ni execution mechanism (`Thread`) nunchi separate chestunnam.
- **📌 Will be used in:** [Callable Interface](../07-Callable-Future/01-Callable-Interface.md) - `Runnable` laage `Callable` kuda oka task ni represent chestundi, kaani adi oka value ni return cheyagaladu.

---

### Example 3: `join()` - Waiting for a Thread to Finish

#### ❌ Failure Case #1: Not using `join()` (Race Condition)
**Scenario:** Oka background thread lo calculation jarugutundi. Main thread aa calculation result kosam wait cheyakapothe emavtundi?
```java
// Code for CalculationTask (same as before)
// ...
public class Main {
    public static void main(String[] args) {
        CalculationTask task = new CalculationTask();
        Thread calcThread = new Thread(task);
        calcThread.start();
        // ❌ Mistake: Wait cheyakunda result adigestunnam.
        System.out.println("Main thread: The result is " + task.getResult());
    }
}
```
**Output:**
`Main thread: The result is 0`
**Why This Fails:** Main thread `start()` call chesi ventane `getResult()` adigesindi. Appatiki calculation thread inka pani complete cheyaledu. So, default value `0` vachindi.

#### ❌ Failure Case #2: `join()` on a thread that hasn't started
**What Developers Do Wrong:** `start()` call cheyakunda `join()` call cheyadam.
```java
// Inside Main.java
public static void main(String[] args) throws InterruptedException {
    CalculationTask task = new CalculationTask();
    Thread calcThread = new Thread(task);
    // ❌ Mistake: start() was never called!
    calcThread.join(); // What happens here?
    System.out.println("Main thread finished waiting.");
}
```
**Output:**
`Main thread finished waiting.`
**Why This is a Bug:** No exception vachindi, kaani program behave chesina vidhanam wrong. `join()` anedi thread `isAlive()` check chestundi. Thread inka `NEW` state lo undi kabatti, `isAlive()` returns `false`, and `join()` immediately returns. Manam anukuntam thread pani aypoindi ani, kaani asalu pani start ee avvaledu!

#### ✅ Correct Version: Using `join()`
```java
// Inside Main.java's main method
public static void main(String[] args) throws InterruptedException {
    // ... (task and thread creation)
    calcThread.start();
    // ✅ Fix: calcThread pani aypoye varaku wait cheyi.
    calcThread.join();
    System.out.println("Main thread: The result is " + task.getResult());
}
```
**Output:**
`Main thread: The result is 100`
**Why This Works:** `join()` call cheste, main thread `WAITING` state loki veltundi. `calcThread` `TERMINATED` ayyaka, main thread resume avtundi.

#### 🔗 Concept Connections:
- **💭 Uses from previous:** [Thread Lifecycle](./01-Core-Concepts-and-Theory.md#4-thread-lifecycle--states-🚦) - `join()` call cheste current thread `WAITING` state loki veltundi.
- **📌 Will be used in:** [Java Memory Model](./03-Java-Memory-Model.md#3-happens-before-relationship--guarantee) - `join()` creates a happens-before relationship. Ante, child thread lo jarigina changes anni `join()` return ayyaka main thread ki visible ga untayi.

---

### Example 4: `interrupt()` - Stopping a Thread Gracefully

#### ❌ Failure Case #1: Swallowing `InterruptedException`
**What Developers Do Wrong:** `InterruptedException` ni catch chesi, em cheyakunda ignore chestaru.
```java
public class BadWorker implements Runnable {
    @Override
    public void run() {
        while (true) { // Loop that should stop
            System.out.println("Worker: I am working...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                // ❌ Mistake: Exception ni ignore chesaru!
                // The thread's interrupted status is cleared here, but we don't act on it.
            }
        }
    }
}
```
**Result:** Ee thread ni `interrupt()` chesina, adi eppatiki aagadu! Endukante `InterruptedException` catch chesinappudu loop break cheyatledu.

#### ✅ Correct Version: Stoppable Worker
```java
// File: com/example/interrupt/StoppableWorker.java
public class StoppableWorker implements Runnable {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            System.out.println("Worker: I am working...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                System.out.println("Worker: I was interrupted! Cleaning up and stopping.");
                // ✅ Fix: Restore the interrupted status and break
                Thread.currentThread().interrupt(); // Good practice!
                break;
            }
        }
        System.out.println("Worker: Stopped.");
    }
}
```

#### 🔗 Concept Connections:
- **📌 Will be used in:** [ExecutorService Shutdown](../06-Executor-Framework/06-Executor-Lifecycle.md) - `executor.shutdownNow()` threads ni interrupt cheyadaniki try chestundi. Manam tasks ni ilaga correct ga handle cheste, shutdown anedi graceful ga untundi.

---

### Example 5: Daemon Threads - The Background Helpers

#### ❌ Failure Case #1: Forgetting to set Daemon
**Result:** Main application exit ayina, `healthChecker` thread run avutune untundi, so JVM shutdown avvadu.

#### ❌ Failure Case #2: Setting Daemon status *after* starting
**What Developers Do Wrong:** Thread `start()` chesina tarvata `setDaemon(true)` call chestaru.
```java
// Inside a Main class
public static void main(String[] args) {
    Thread healthThread = new Thread(() -> { /* ... */ });
    healthThread.start();
    // ❌ Too late!
    healthThread.setDaemon(true);
}
```
**Error Message:**
```
Exception in thread "main" java.lang.IllegalThreadStateException
    ...
```
**Why This Fails:** Thread ni start chesina tarvata daani "daemon" status ni change cheyalemu. Adi `start()` ki mundare cheyali.

#### ✅ Correct Version: Daemon Thread
```java
// Inside a Main class
public static void main(String[] args) {
    Thread healthThread = new Thread(() -> { /* ... */ });
    // ✅ Fix: Ee thread ni daemon ga set chey. start() ki mundu cheyali!
    healthThread.setDaemon(true);
    healthThread.start();
    System.out.println("Main thread is finishing.");
}
```
**Result:** Main thread aypoyagane JVM exits.

#### 🔗 Concept Connections:
- **💭 Uses from previous:** JVM architecture. JVM exits when only daemon threads are left.
- **📌 Will be used in:** Library/Framework code. For example, garbage collector anedi oka high-priority daemon thread.

---
## 🌟 Best Practices
(Same as before)
---
## ✅ Checkpoint: Did You Master This?
(Same as before)
---
