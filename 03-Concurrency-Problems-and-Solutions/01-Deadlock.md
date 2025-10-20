---
# 🎯 3.1: Deadlock

## 🗺️ Learning Path Position

**📍 You Are Here:** 3.1 - Deadlock

**✅ Prerequisites (Must Complete First):**
- [2.2: The `synchronized` Keyword](../02-Synchronization-Thread-Safety/02-Synchronized-Keyword.md) - `synchronized` and intrinsic locks meeda solid understanding undali, endukante ade deadlock ki karanam avtundi.

**🔜 Coming After This:**
- [3.2: Livelock and Starvation](./02-Livelock-and-Starvation.md) - Deadlock ki cousins lanti Livelock and Starvation gurinchi nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 🤔 What & Why

### The Problem: The Eternal Wait 🥶
Deadlock ante, rendu (or ekkuva) threads okari kosam okaru forever wait chestu undipovadam. Thread 1, Thread 2 release cheyalsina lock kosam wait chestundi. Ade time lo, Thread 2, Thread 1 release cheyalsina lock kosam wait chestundi. Iddaru eppatiki munduku vellaru. App antha hang aipotundi.

### Real-World Analogy: Traffic Gridlock 🚦
Oka four-way traffic signal anuko.
- Car A (Thread A) wants to go straight, needs intersection part 2 (Resource 2). It currently holds part 1 (Resource 1).
- Car B (Thread B) wants to go straight, needs intersection part 1 (Resource 1). It currently holds part 2 (Resource 2).
Car A, Car B kosam wait chestundi. Car B, Car A kosam wait chestundi. Traffic jam! Evaru munduku vellaru. Idhe Deadlock.

---

## 📚 The Four Conditions for Deadlock

Deadlock ravali ante, ee 4 conditions **okesari** satisfy avvali. Okati miss ayina, deadlock radu.
1.  **Mutual Exclusion (ఒకరికే సొంతం):** At least oka resource non-sharable ga undali. Ante, trial room laaga, okate thread at a time use cheyagalali. (`synchronized` idi guarantee chestundi).
2.  **Hold and Wait (ఒకటి పట్టుకుని, ఇంకో దాని కోసం వెయిటింగ్):** Oka thread already oka resource ni hold chesi, inko resource kosam wait chestu undali.
3.  **No Preemption (లాక్కోవడం కుదరదు):** System, oka thread nunchi resource ni balavanthanga teeskoledu. Thread ade voluntarily release cheyali.
4.  **Circular Wait (చక్రీయ నిరీక్షణ):** Oka circular chain of threads form avvali, where each thread is waiting for a resource held by the next thread in the chain. (e.g., T1 waits for T2, T2 waits for T1).

---

## 💻 Code Example: A Guaranteed Deadlock

**Scenario:** Rendu bank accounts madhyalo money transfer cheyali. Account A nunchi B ki, and Account B nunchi A ki, okesari. Transfer chese mundu, manam rendu accounts ni lock cheyali, so vere transaction disturb cheyakudadu.

### ❌ The Failure Case: Deadlock in Action!
```java
// File: com/example/deadlock/DeadlockExample.java
package com.example.deadlock;

public class DeadlockExample {

    // Two shared resources (locks)
    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        // Thread 1: Tries to lock A then B
        Thread thread1 = new Thread(() -> {
            synchronized (lockA) {
                System.out.println("Thread 1: Acquired lock A");
                try {
                    // Introduce a delay to increase the chance of deadlock
                    Thread.sleep(100);
                } catch (InterruptedException e) {}

                System.out.println("Thread 1: Trying to acquire lock B...");
                synchronized (lockB) {
                    System.out.println("Thread 1: Acquired lock B"); // This line is never reached
                }
            }
        }, "Thread-1");

        // Thread 2: Tries to lock B then A
        Thread thread2 = new Thread(() -> {
            synchronized (lockB) {
                System.out.println("Thread 2: Acquired lock B");
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {}

                System.out.println("Thread 2: Trying to acquire lock A...");
                synchronized (lockA) {
                    System.out.println("Thread 2: Acquired lock A"); // This line is never reached
                }
            }
        }, "Thread-2");

        thread1.start();
        thread2.start();
    }
}
```
**Result:**
The program will run and then hang forever. You'll have to manually stop it.
```
Thread 1: Acquired lock A
Thread 2: Acquired lock B
Thread 1: Trying to acquire lock B...
Thread 2: Trying to acquire lock A...
// <-- Application freezes here -->
```
**Why This Fails (Circular Wait):**
1.  Thread 1 acquires `lockA`.
2.  Thread 2 acquires `lockB`.
3.  Thread 1 tries to acquire `lockB`, but it's held by Thread 2. So, Thread 1 waits.
4.  Thread 2 tries to acquire `lockA`, but it's held by Thread 1. So, Thread 2 waits.
5.  Thread 1 is waiting for Thread 2, and Thread 2 is waiting for Thread 1. **Circular wait** condition satisfied. Deadlock!

### 🐛 How to Detect Deadlock in a Real App?
Production lo app hang aythe, manam **Thread Dump** teskuntam.
1.  Get the Process ID (PID) of your Java app: `jps -l`
2.  Take a thread dump: `jstack <PID>`
Ee dump lo, JVM clear ga cheptundi "Found 1 deadlock" ani, and a a threads a a locks kosam wait chestunnayo chupistundi. Chala powerful debugging tool!

### ✅ The Solution: Consistent Lock Ordering
Deadlock ni prevent cheyadaniki a best and most common way aంటంటే, **Circular Wait** condition ni break cheyadam. Ante, andaru threads locks ni okate fixed order lo teskovali.

```java
// File: com/example/deadlock_fix/NoDeadlockExample.java
package com.example.deadlock_fix;

public class NoDeadlockExample {

    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        // Thread 1: Locks A then B
        Thread thread1 = new Thread(() -> {
            // ✅ Rule: Always lock A before B
            synchronized (lockA) {
                System.out.println("Thread 1: Acquired lock A");
                try { Thread.sleep(100); } catch (InterruptedException e) {}

                System.out.println("Thread 1: Trying to acquire lock B...");
                synchronized (lockB) {
                    System.out.println("Thread 1: Acquired lock B successfully");
                }
            }
        }, "Thread-1");

        // Thread 2: Also locks A then B
        Thread thread2 = new Thread(() -> {
            // ✅ Rule: Always lock A before B
            synchronized (lockA) {
                System.out.println("Thread 2: Acquired lock A");
                try { Thread.sleep(100); } catch (InterruptedException e) {}

                System.out.println("Thread 2: Trying to acquire lock B...");
                synchronized (lockB) {
                    System.out.println("Thread 2: Acquired lock B successfully");
                }
            }
        }, "Thread-2");

        thread1.start();
        thread2.start();
    }
}
```
**Result:**
The program now runs to completion without freezing.
```
Thread 1: Acquired lock A
Thread 1: Trying to acquire lock B...
Thread 1: Acquired lock B successfully
// (After Thread 1 finishes, Thread 2 starts)
Thread 2: Acquired lock A
Thread 2: Trying to acquire lock B...
Thread 2: Acquired lock B successfully
```
**Why This Works:**
Thread 2 `lockA` kosam try chesinappudu, adi already Thread 1 hold chesindi. So, Thread 2 wait chestundi. Kaani, ikkada Thread 1 `lockB` kosam wait cheyatledu, endukante daanini evaru hold cheyaledu. So, Thread 1 `lockB` teskuni, pani complete chesi, rendu locks release chestundi. Appudu Thread 2 munduku veltundi. Circular wait anedi impossible ikkada.

---

## 🌟 Best Practices to Avoid Deadlocks

1.  **Lock Ordering:** Ee method a best. Project antha, multiple locks teskovalsi vaste, eppudu okate order follow avvali.
2.  **Avoid Nested Locks:** Asal nested synchronized blocks (oka lock teskuni, daani lopala inko lock teskovadam) avoid cheyadaniki try chey.
3.  **Use Lock Timeouts:** `synchronized` keyword tho timeout possible kadu. Kaani, manam future lo nerchukune `java.util.concurrent.locks.Lock` interface (`lock.tryLock(timeout, unit)`) use cheste, oka lock konni seconds try chesi, dorakkapothe give up cheyochu. Idi "Hold and Wait" condition ni break chestundi.

---

## ✅ Checkpoint: Did You Master This?

- [ ] Deadlock ravali ante 4 conditions enti?
- [ ] "Circular Wait" ante enti? Manam code lo a a break chesam?
- [ ] Oka running application lo deadlock vachindi ani a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a anukuntam? (Hint: `jstack`).
- Deadlock ni prevent cheyadaniki most common technique aంటi?

**✅ Ready?** → [Next: Livelock and Starvation](./02-Livelock-and-Starvation.md)

**😕 Need Review?** → Paiki scroll chesi a 4 conditions malli chudu.
