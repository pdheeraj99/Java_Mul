---
# 🎯 2.2: The `synchronized` Keyword

## 🗺️ Learning Path Position

**📍 You Are Here:** 2.2 - The `synchronized` Keyword

**✅ Prerequisites (Must Complete First):**
- [2.1: Race Conditions & Critical Sections](./01-Race-Conditions-and-Critical-Sections.md) - Race condition and critical section gurinchi clear ga teliyali.

**🔜 Coming After This:**
- [2.3: The `volatile` Keyword](./03-Volatile-Keyword.md) - Inko synchronization approach `volatile` gurinchi nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🟡 Intermediate
---

## 💭 Quick Recap
💭 **Gurtunda ra?** Manam `UnsafeCounter` example lo `count++` operation valla final result ela corrupt ayyindo chusam. `increment()` method anedi critical section ani identify chesam. Ippudu daanini protect cheddam.
---

## 🤔 What & Why
### The Problem: How to Protect the Critical Section?
Critical section ni at a time okate thread execute cheyali. Deeniki manaki oka "lock" kavali.
### The Solution: `synchronized` Keyword
Java ee locking mechanism ni `synchronized` keyword tho provide chestundi. Prathi Java object ki oka **intrinsic lock** (or **monitor lock**) untundi, `synchronized` daanini use cheskuntundi.
### Real-World Analogy: Trial Room Lock 🚪
- **Trial Room:** The Critical Section.
- **Lock on the door:** The `synchronized` keyword.
- **Person (Thread):** Tries to enter. If locked, waits in a queue.
---

## 💻 Code Examples
### Example 1: `synchronized` Methods (The Easiest Way)
#### ✅ Success Case: The `SafeCounter`
**Scenario:** `UnsafeCounter` ni thread-safe ga marchadam.
```java
// File: com/example/safe/SafeCounter.java
package com.example.safe;
public class SafeCounter {
    private int count = 0;
    // ✅ Just add the 'synchronized' keyword!
    public synchronized void increment() {
        count++;
    }
    public int getCount() {
        return count;
    }
}

// File: com/example/safe/Main.java
package com.example.safe;
public class Main {
    public static void main(String[] args) throws InterruptedException {
        SafeCounter counter = new SafeCounter();
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) counter.increment();
        });
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) counter.increment();
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
        System.out.println("Final count: " + counter.getCount());
    }
}
```
**Output (Every Single Run):** `Final count: 20000`
**Why This Works:** Oka thread `increment()` ni execute chestunnapudu, adi `counter` object yokka lock ni hold chestundi. Inko thread `BLOCKED` state lo wait chestundi. Ee valla `count++` operation atomic ga jarugutundi.

#### 🔗 Concept Connections:
- **💭 Uses from previous:** [Race Conditions](./01-Race-Conditions-and-Critical-Sections.md)
- **📌 Will be used in:** [ReentrantLock](../04-Advanced-Locking/01-ReentrantLock.md)
---

### Example 2: `synchronized` Static vs. Instance Methods

- **Instance Method:** `this` object meeda lock chestundi.
- **Static Method:** `ClassName.class` object meeda lock chestundi.

#### ❌ Failure Case: Confusing Instance and Static Locks
```java
// File: com/example/mixed/MixedCounter.java
package com.example.mixed;
public class MixedCounter {
    private static int staticCount = 0;
    private int instanceCount = 0;

    // Locks on the instance ('this')
    public synchronized void incrementInstance() { instanceCount++; }
    // Locks on the 'MixedCounter.class' object
    public static synchronized void incrementStatic() { staticCount++; }

    public static void main(String[] args) throws InterruptedException {
        MixedCounter c1 = new MixedCounter();
        MixedCounter c2 = new MixedCounter();

        Thread t1 = new Thread(() -> c1.incrementInstance()); // Locks on c1
        Thread t2 = new Thread(() -> c2.incrementInstance()); // Locks on c2
        Thread t3 = new Thread(() -> MixedCounter.incrementStatic()); // Locks on MixedCounter.class
        Thread t4 = new Thread(() -> MixedCounter.incrementStatic()); // Locks on MixedCounter.class

        // t1 and t2 can run in parallel (different locks).
        // t3 and t4 are synchronized with each other, but not with t1 or t2.
        t1.start(); t2.start(); t3.start(); t4.start();
        t1.join(); t2.join(); t3.join(); t4.join();

        System.out.println("c1.instanceCount = " + c1.instanceCount); // 1
        System.out.println("c2.instanceCount = " + c2.instanceCount); // 1
        System.out.println("staticCount = " + staticCount); // 2
    }
}
```
**Why This is a Common Pitfall:** Developers anukuntaru `synchronized` ante anni methods lock aypotayi ani, kaani **ye object meeda lock** chestunnam anedi chala important.
---

### Example 3: `synchronized` Blocks (For Better Performance)
Method antha kakunda, kevalam critical section varake lock cheyadaniki `synchronized` block use chestaru.

#### ✅ Success Case: Fine-Grained Locking
**Scenario:** File download chesi, counter ni update cheyali. Download non-critical, so daaniki lock avasaram ledu.
```java
// File: com/example/finegrained/FineGrainedCounter.java
package com.example.finegrained;
public class FineGrainedCounter {
    private int count = 0;
    private final Object lock = new Object(); // ✅ Best Practice: Private final lock object

    public void performDownloadAndUpdate() {
        System.out.println(Thread.currentThread().getName() + ": Starting long task...");
        try { Thread.sleep(2000); } catch (InterruptedException e) {}
        System.out.println(Thread.currentThread().getName() + ": Task complete.");

        synchronized (lock) {
            System.out.println(Thread.currentThread().getName() + ": Acquired lock, updating counter.");
            count++;
        }
    }
    public int getCount() {
        synchronized(lock) { return count; }
    }
}

// File: com/example/finegrained/Main.java
package com.example.finegrained;
public class Main {
    public static void main(String[] args) throws InterruptedException {
        FineGrainedCounter counter = new FineGrainedCounter();
        Thread t1 = new Thread(() -> counter.performDownloadAndUpdate(), "Downloader-1");
        Thread t2 = new Thread(() -> counter.performDownloadAndUpdate(), "Downloader-2");

        long startTime = System.currentTimeMillis();
        t1.start(); t2.start();
        t1.join(); t2.join();
        long endTime = System.currentTimeMillis();
        System.out.println("Final count: " + counter.getCount());
        System.out.println("Total time: " + (endTime - startTime) + "ms");
    }
}
```
**Output:** Total time taken approx `20XXms`, not `40XXms`. Rendu threads download ni parallel ga chesayi.
---

### 🔥 What is Reentrancy?
`synchronized` anedi **reentrant**. Ante, oka thread already oka lock hold chestunte, adi ade lock require chese inko `synchronized` method ni call cheyochu.
```java
// File: com/example/reentrant/ReentrantExample.java
package com.example.reentrant;
public class ReentrantExample {
    public synchronized void methodA() {
        System.out.println("In method A, calling method B");
        methodB();
    }
    public synchronized void methodB() {
        System.out.println("In method B");
    }
    public static void main(String[] args) {
        new ReentrantExample().methodA();
    }
}
```
**Output:** `In method A...` `In method B`. Deadlock radu.
---

## 🌟 Best Practices
1.  **Minimize Lock Scope:** Performance kosam `synchronized` blocks ni prefer chey.
2.  **Use a Private Final Lock Object:** `synchronized(this)` badulu `private final Object lock = new Object();` use chey.
3.  **Don't Lock on Publicly Accessible Objects.**
4.  **Document your Locking Strategy.**
---

## ✅ Checkpoint: Did You Master This?
- [ ] `synchronized` method vs block: performance lo edi better?
- [ ] Instance vs Static `synchronized`: lock ye object meeda untundi?
- [ ] "Reentrancy" ante enti?
- [ ] `private final Object lock` enduku better?

**✅ Ready?** → [Next: The `volatile` Keyword](./03-Volatile-Keyword.md)
**😕 Need Review?** → Paiki scroll chesi `synchronized` block example malli chudu.
---
