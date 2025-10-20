---
# 🎯 4.1: The `Lock` Interface and `ReentrantLock`

## 🗺️ Learning Path Position

**📍 You Are Here:** 4.1 - `ReentrantLock`

**✅ Prerequisites (Must Complete First):**
- [2.2: The `synchronized` Keyword](../02-Synchronization-Thread-Safety/02-Synchronized-Keyword.md) - `synchronized` ela pani chestundo teliste, deeni advantages ardham avtayi.
- [3.1: Deadlock](../03-Concurrency-Problems-and-Solutions/01-Deadlock.md) - Deadlock ante ento gurtundali, endukante `ReentrantLock` daanini prevent cheyadaniki help chestundi.

**🔜 Coming After This:**
- [4.2: ReadWriteLock](./02-ReadWriteLock.md) - Read-heavy scenarios lo performance ni inka improve chese inko type of lock gurinchi nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 💭 Quick Recap

💭 **Gurtunda ra?** `synchronized` anedi oka **implicit lock**. Ante, manam `synchronized` keyword pedatam, JVM αυτόματικά lock/unlock chuskuntundi. Chala convenient, kaani limited features. `ReentrantLock` anedi oka **explicit lock**. Maname manually `lock()` and `unlock()` cheyali. Idi manaki chala extra power istundi.

---

## 🤔 What & Why

### The Problem: `synchronized` is Not Enough
`synchronized` chala situations lo perfect. Kaani, ee kindi scenarios lo adi help cheyaledu:
- Oka lock kosam nenu just 1 second wait cheyali, dorakkapothe vere pani cheskovali anukunte? (`tryLock` with timeout).
- Rendu locks teskovali, okati doriki, rendodi dorakkapothe, modati daanini vadilesi malli try cheyali anukunte? (Deadlock prevention).
- Lock kosam wait chestunna queue lo, first vachina thread ke first chance ivvali ani guarantee cheyalante? (Fairness).

### The Solution: `ReentrantLock`
`java.util.concurrent.locks.Lock` interface ee problems ki solution istundi. Daani most popular implementation `ReentrantLock`. "Reentrant" ante, `synchronized` laage, oka thread already hold chestunna lock ni malli acquire cheyochu ani.

### Real-World Analogy: House Key 🔑 vs. Hotel Keycard 🏨
- **`synchronized` (House Key):** Simple, oka lock, oka key. Easy to use.
- **`ReentrantLock` (Hotel Keycard):** Chala flexible.
  - **`lock()`:** Keycard use chesi door open chey.
  - **`unlock()`:** Door close chey.
  - **`tryLock()`:** Card swipe chesi chudu, door open avtunda leda ani ventane telusko.
  - **`tryLock(time)`:** Card ni 5 seconds door daggara petti chudu, open avvakapothe alarm moginchu (vere pani chey).
  - **Fairness:** Reception daggara queue form chesi, first vachina vallake keycard ivvadam.

---

## 💻 Code Examples

### Example 1: Basic Usage - The `try-finally` Block is Your Best Friend!

#### ✅ Success Case: `SafeCounter` with `ReentrantLock`
```java
// File: com/example/reentrant/SafeCounter.java
package com.example.reentrant;

import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class SafeCounter {
    private int count = 0;
    // 1. Create a Lock instance
    private final Lock lock = new ReentrantLock();

    public void increment() {
        // 2. Acquire the lock
        lock.lock();
        try {
            // 3. This is our critical section
            count++;
        } finally {
            // 4. 🔥 CRITICAL: Release the lock in a finally block.
            // Exception vachina kuda, lock anedi guarantee ga release avtundi.
            lock.unlock();
        }
    }

    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
// Main class is the same as the one for synchronized SafeCounter
```
**Why This Works:** Idi `synchronized` method laage pani chestundi. `lock.lock()` call chesinappudu thread lock teskuntundi, vere threads wait chestayi. `unlock()` chesinappudu vere thread ki chance vastundi.

#### ❌ Failure Case #1: Forgetting to `unlock()`
```java
public void badIncrement() {
    lock.lock();
    // What if an exception happens here? The unlock() is never called!
    count++;
    // Or what if we just forget to call unlock()?
    // lock.unlock(); // <-- We forgot this line
}
```
**Result:** Oka thread `badIncrement()` call chesi, lock teskuni, release cheyadam marchipothe, inka ye thread kuda ee method ni access cheyaledu. App antha akkade aagipotundi. **Always use `try-finally`!**

---

### Example 2: `tryLock()` - A Powerful Way to Avoid Deadlocks

#### ✅ Success Case: Transferring money without Deadlock
Manam Phase 3 lo chusina bank account transfer deadlock ni `tryLock` tho solve cheddam.
```java
// File: com/example/deadlock_prevention/Account.java
package com.example.deadlock_prevention;

import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class Account {
    private int balance = 10000;
    private final Lock lock = new ReentrantLock();

    public void deposit(int amount) { balance += amount; }
    public void withdraw(int amount) { balance -= amount; }
    public int getBalance() { return balance; }
    public Lock getLock() { return lock; }

    public static void transfer(Account from, Account to, int amount) {
        while (true) {
            // 1. Try to acquire locks
            if (from.getLock().tryLock()) {
                System.out.println(Thread.currentThread().getName() + ": Acquired lock on account " + from);
                try {
                    if (to.getLock().tryLock()) {
                        System.out.println(Thread.currentThread().getName() + ": Acquired lock on account " + to);
                        try {
                            // 2. If both locks are acquired, perform the transfer
                            if (from.getBalance() < amount) {
                                throw new IllegalStateException("Insufficient funds");
                            }
                            from.withdraw(amount);
                            to.deposit(amount);
                            System.out.println(Thread.currentThread().getName() + ": Transfer successful.");
                            break; // Exit the loop
                        } finally {
                            // 3. Always release the second lock
                            to.getLock().unlock();
                        }
                    }
                } finally {
                    // 4. Always release the first lock
                    from.getLock().unlock();
                }
            }
            // 5. If we couldn't get both locks, wait a bit and retry
            System.out.println(Thread.currentThread().getName() + ": Could not acquire all locks, backing off...");
            try { Thread.sleep(100); } catch (InterruptedException e) {}
        }
    }
}
```
**Why This Works:** Ee code lo thread eppudu rendu locks ni hold chesi wait cheyadu. Okati doriki, rendodi dorakkapothe, modati daanini kuda release chesi, malli try chestundi. Idi Deadlock yokka **"Hold and Wait"** condition ni break chestundi.

---

### Example 3: Fairness Policy

#### ✅ Success Case: Fair Lock Demo
```java
// File: com/example/fairness/FairLockDemo.java
package com.example.fairness;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class FairLockDemo {
    // Try changing this to 'false' to see the difference
    private static final Lock lock = new ReentrantLock(true); // Fair lock

    public static void main(String[] args) {
        Runnable task = () -> {
            for (int i = 0; i < 2; i++) {
                lock.lock();
                try {
                    System.out.println(Thread.currentThread().getName() + " acquired the lock.");
                    Thread.sleep(500); // Hold the lock for a while
                } catch (InterruptedException e) {
                } finally {
                    System.out.println(Thread.currentThread().getName() + " is releasing the lock.");
                    lock.unlock();
                }
            }
        };

        Thread t1 = new Thread(task, "Thread-1");
        Thread t2 = new Thread(task, "Thread-2");
        Thread t3 = new Thread(task, "Thread-3");

        t1.start();
        t2.start();
        t3.start();
    }
}
```
**Output (with `true` - Fair Lock):** Threads will likely take turns in a predictable order, like 1, 2, 3, 1, 2, 3.
**Output (with `false` - Unfair Lock):** Oka thread (e.g., Thread-3) malli malli lock teskovachu, vere threads wait chestu undanga. Output might be 1, 3, 2, 3, 1, 2.

**Why This Matters:** Starvation ni prevent cheyadaniki fairness chala important. Kaani, fairness ki performance cost untundi, endukante JVM oka queue ni maintain cheyali. By default, `ReentrantLock` anedi **unfair**.

---
## 📊 `synchronized` vs `ReentrantLock`

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| **Type** | Implicit Lock | Explicit Lock |
| **Usage** | Keyword (easier) | Class (more verbose) |
| **`unlock()`** | Automatic | Manual (must use `try-finally`) |
| **Flexibility**| Low | High |
| **`tryLock()`** | No | Yes |
| **Interruptible Lock** | No | Yes (`lockInterruptibly()`) |
| **Fairness** | Not guaranteed | Yes (configurable) |
| **Conditions** | `wait/notify` | `Condition` objects (more powerful) |

---
## ✅ Checkpoint: Did You Master This?

- [ ] `ReentrantLock` use chesinappudu, `unlock()` ni `finally` block lo enduku pettali?
- [ ] `tryLock()` anedi deadlock ni a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a-condition ni break chesi, a a a prevent chestundi?
- "Fair" lock ki "Unfair" lock ki a a a main a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a-difference aంటi?
- Simple case lo, `synchronized` a, `ReentrantLock` a, aది prefer chestaru? Enduku?

**✅ Ready?** → [Next: ReadWriteLock](./02-ReadWriteLock.md)

**😕 Need Review?** → Paiki scroll chesi `try-finally` block importance malli chudu.
