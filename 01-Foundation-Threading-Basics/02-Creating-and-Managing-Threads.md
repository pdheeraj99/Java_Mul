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
            System.out.println("Thread Name: " + Thread.currentThread().getName() + " - Number: " + i);
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
        printerThread.setName("NumberPrinter");
        printerThread.start();
        System.out.println("Main thread finished!");
    }
}
```
**Output (Example):**
```
Main thread started!
Main thread finished!
Thread Name: NumberPrinter - Number: 1
Thread Name: NumberPrinter - Number: 2
...
```

#### ❌ Failure Case #1: Calling `run()` instead of `start()`
**What Developers Do Wrong:** `start()` badulu `run()` method ni direct ga call chestaru.

```java
// Inside Main.java's main method
// ...
System.out.println("Calling run() directly...");
printerThread.run(); // ❌ Mistake
// ...
```
**Output:**
```
Main thread started!
Calling run() directly...
Thread Name: main - Number: 1
Thread Name: main - Number: 2
...
Main thread finished!
```
**Why This Fails:** `run()` ni direct ga call cheste, adi normal method laage **main thread** lo ne execute avtundi. Kotha thread create avvaledu!

#### ❌ Failure Case #2: Starting the same Thread twice
**What Developers Do Wrong:** Oka `Thread` object ni okasari `start()` chesina tarvata, malli `start()` cheyadaniki try chestaru.
```java
// Inside Main.java
public static void main(String[] args) throws InterruptedException {
    NumberPrinterThread printerThread = new NumberPrinterThread();
    printerThread.start(); // First call is OK
    printerThread.join(); // Wait for it to finish
    System.out.println("Trying to start the same thread again...");
    printerThread.start(); // ❌ Second call will fail
}
```
**Error Message:** `java.lang.IllegalThreadStateException`
**Why This Fails:** Oka thread `TERMINATED` state ki vellina tarvata, daanini malli start cheyalemu.

#### 🔗 Concept Connections:
- **💭 Uses from previous:** [Thread Lifecycle](./01-Core-Concepts-and-Theory.md#4-thread-lifecycle--states-🚦)
- **📌 Will be used in:** [Executor Framework](../06-Executor-Framework/01-Executor-Basics.md)

---

### Example 2: Creating a Thread by Implementing `Runnable` (❤️ The Better Way)

#### ✅ Success Case: File Downloader Task
```java
// File: com/example/runnable/FileDownloaderTask.java
package com.example.runnable;
public class FileDownloaderTask implements Runnable {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + ": Starting file download...");
        try { Thread.sleep(3000); } catch (InterruptedException e) {}
        System.out.println(Thread.currentThread().getName() + ": File download complete!");
    }
}
// File: com/example/runnable/Main.java
package com.example.runnable;
public class Main {
    public static void main(String[] args) {
        FileDownloaderTask task = new FileDownloaderTask();
        Thread workerThread = new Thread(task, "Downloader");
        workerThread.start();
        System.out.println("Main thread: Continuing with other work...");
    }
}
```
**Why This Works:** Manam task (`Runnable`) ni worker (`Thread`) nunchi separate chesam. Idi clean design.

---

### Example 3: `join()` - Waiting for a Thread to Finish

#### ✅ Success Case: Using `join()`
```java
// File: com/example/join/CalculationTask.java
package com.example.join;
public class CalculationTask implements Runnable {
    private long result;
    @Override
    public void run() {
        long sum = 0;
        for (int i = 0; i < 1000000; i++) { sum += i; }
        result = sum;
    }
    public long getResult() { return result; }
}

// File: com/example/join/Main.java
package com.example.join;
public class Main {
    public static void main(String[] args) throws InterruptedException {
        CalculationTask task = new CalculationTask();
        Thread calcThread = new Thread(task, "Calculator");
        calcThread.start();
        calcThread.join(); // ✅ Main thread waits here
        System.out.println("Result is " + task.getResult());
    }
}
```
**Output:** `Result is 499999500000`

#### ❌ Failure Case #1: Not using `join()`
```java
// Inside Main.java
public static void main(String[] args) {
    CalculationTask task = new CalculationTask();
    Thread calcThread = new Thread(task, "Calculator");
    calcThread.start();
    // ❌ We don't wait!
    System.out.println("Result is " + task.getResult());
}
```
**Output:** `Result is 0`
**Why This Fails:** Main thread calculation aypoyaka mundare `getResult()` call chesindi.

---

### Example 4: `interrupt()` - Stopping a Thread Gracefully

#### ✅ Success Case: Stoppable Worker
```java
// File: com/example/interrupt/StoppableWorker.java
package com.example.interrupt;
public class StoppableWorker implements Runnable {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            System.out.println("Worker: I am working...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                System.out.println("Worker: Interrupted! Cleaning up.");
                Thread.currentThread().interrupt(); // Restore interrupt status
                break;
            }
        }
        System.out.println("Worker: Stopped.");
    }
}

// File: com/example/interrupt/Main.java
package com.example.interrupt;
public class Main {
    public static void main(String[] args) throws InterruptedException {
        StoppableWorker workerTask = new StoppableWorker();
        Thread workerThread = new Thread(workerTask, "StoppableWorker");
        workerThread.start();
        Thread.sleep(3000);
        System.out.println("Main: Requesting worker to stop.");
        workerThread.interrupt();
    }
}
```
---

### Example 5: Daemon Threads - The Background Helpers

#### ✅ Correct Version: Daemon Thread
```java
// File: com/example/daemon/Main.java
package com.example.daemon;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        Runnable healthChecker = () -> {
            while (true) {
                System.out.println("Health Checker: Checking system health...");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        };

        Thread healthThread = new Thread(healthChecker, "Health-Checker");
        healthThread.setDaemon(true); // ✅ Set as daemon BEFORE starting
        healthThread.start();

        System.out.println("Main thread is working for 3 seconds...");
        Thread.sleep(3000);
        System.out.println("Main thread is finishing. JVM will now exit.");
    }
}
```
**Result:** Main thread finish avvagane, program exit aipotundi.

---
## 🌟 Best Practices
1.  **Prefer `Runnable` over `Thread` extension.**
2.  **NEVER call `run()` directly.**
3.  **Use `join()` for coordination.**
4.  **Handle `InterruptedException` properly.**
5.  **Name your threads.**
6.  **Don't depend on Thread Priorities.**

---

## ✅ Checkpoint: Did You Master This?
- [ ] `Thread` vs `Runnable` - Edi better and enduku?
- [ ] `start()` ki `run()` ki teda enti?
- [ ] `join()` lekunda code raste em problem vastundi?
- [ ] Oka running thread ni gracefully ga stop cheyadam ela?
- [ ] Daemon thread ante enti?

**✅ Ready?** → [Next: Java Memory Model](./03-Java-Memory-Model.md)
**😕 Need Review?** → Scroll back and review the examples.
---
