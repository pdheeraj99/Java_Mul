---
# 🎯 4.3: StampedLock (Java 8+)

## 🗺️ Learning Path Position

**📍 You Are Here:** 4.3 - `StampedLock`

**✅ Prerequisites (Must Complete First):**
- [4.2: ReadWriteLock](./02-ReadWriteLock.md) - `ReadWriteLock` gurinchi teliste, `StampedLock` deenikanna a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a-better o ardham avtundi.

**🔜 Coming After This:**
- **Phase 5:** [Atomic Operations](../05-Atomic-Operations/01-Atomic-Variables.md) - Lock-free programming ki foundation ayina Atomics gurinchi nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🔴🔴 SUPER Advanced

---

## 🤔 What & Why

### The Problem: Even Read Locks Can Be Slow
`ReadWriteLock` manchide, kaani chala mandi readers unte, adi kuda slow avvochu. Prathi reader lock teskovali, unlock cheyali. Ee coordination ki kuda kontha overhead untundi. Inka, readers unnapudu writer starve avvochu.

Manaki inka fast ga, asalu lock teskokunda read chese option unte bagundu kada?

### The Solution: `StampedLock` and Optimistic Reading
`StampedLock` anedi `ReadWriteLock` ki alternative. Deenilo 3 modes untayi:
1.  **Writing (`writeLock()`):** `ReadWriteLock` lo laage, exclusive lock.
2.  **Pessimistic Reading (`readLock()`):** `ReadWriteLock` lo laage, shared lock.
3.  **Optimistic Reading (`tryOptimisticRead()`):** 🔥 **Idi kotha feature!** Ee mode lo, manam asalu lock cheyamu. Oka "stamp" (version number lanti `long` value) teskuntam. Data read chesi, a a a tarvata, a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a-stamp inka valid o ledo check chestam.
    - **Valid aite:** Super! Manam lock lekunda data read chesesam. Chala fast.
    - **Invalid aite:** Ante, manam data read chese time lo, vere thread write chesindi. Parledu, ippudu manam normal `readLock()` teskuni, data malli read cheddam.

### Real-World Analogy: Checking Train Timetable 🚆
- **Pessimistic Read:** Nuvvu station ki velli, information counter daggara lock (queue) lo nunchuni, "Train time cheppu" ani adugutunnav.
- **Optimistic Read:** Nuvvu display board chusi time note cheskuntav. Tarvata, "Announcement emaina vachinda?" ani chustav.
  - **No announcement:** Nee time correct.
  - **Announcement vachindi ("Train delayed"):** Ayyayyo, nee time wrong. Ippudu nuvvu pessimistic ga queue lo nunchuni correct time adugutav.

---
## 💻 Code Example: Point on a 2D Plane
**Scenario:** Oka point (`x`, `y`) coordinates ni store cheyali. Multiple threads a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a-distance from origin calculate chestayi (reads). Appudappudu, oka thread a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a a-point ni move chestundi (write).

```java
// File: com/example/stamped/Point.java
package com.example.stamped;

import java.util.concurrent.locks.StampedLock;

public class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();

    // WRITE method
    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock(); // Exclusive lock
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }

    // OPTIMISTIC READ method
    public double distanceFromOrigin() {
        // 1. Get an optimistic "stamp" without blocking
        long stamp = lock.tryOptimisticRead();

        // 2. Read the values into local variables
        double currentX = x;
        double currentY = y;

        // 3. Check if the stamp is still valid.
        // If another thread got a write lock, this will fail.
        if (!lock.validate(stamp)) {
            System.out.println(Thread.currentThread().getName() + ": Optimistic read failed! Falling back to pessimistic read lock.");

            // 4. Fallback: Get a real read lock and read again
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        } else {
            System.out.println(Thread.currentThread().getName() + ": Optimistic read successful!");
        }

        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```
**Why This is Potentially Faster:**
- 99% of the time, `tryOptimisticRead()` and `validate()` will succeed, because writes are rare.
- Ee successful cases lo, manam asalu lock teskovatledu, so no blocking, no contention. Performance super fast.
- Only when a write happens during the read, manam normal `readLock()` ki fallback avtam.

---

## ⚠️ `StampedLock` - Important Warnings

1.  **It is NOT Reentrant:** `synchronized` or `ReentrantLock` laaga, oka thread already hold chestunna lock ni malli acquire cheyaledu. Idi deadlock ki karanam avvochu if you are not careful.
2.  **Stamps are Important:** `unlockWrite(stamp)`, `unlockRead(stamp)` lo correct stamp pass cheyali. `try-finally` block lo `lock()` teskuni, `unlock()` chese pattern work avvadu, endukante stamp anedi `try` block bayata undali.
3.  **It is more complex:** Ee code chala careful ga rayali. `tryOptimisticRead` logic correct ga handle cheyakapothe, inconsistent data read chese risk undi.

## 📊 `ReadWriteLock` vs `StampedLock`

| Feature | `ReadWriteLock` | `StampedLock` |
|---|---|---|
| **Primary Feature** | Shared Read Lock, Exclusive Write Lock | Optimistic Reads, Pessimistic Reads, Exclusive Writes |
| **Performance** | Good for read-heavy scenarios | Potentially better if reads are very frequent and writes are very rare |
| **Reentrant?** | Yes | **No** |
| **Complexity** | Moderate | High |
| **Starvation** | Writer starvation is possible | Provides `asReadLock()` etc. but generally less prone to starvation |

**Rule of Thumb:** 🔥 Start with `ReentrantReadWriteLock`. Performance inka saripokapothe, and you have a very high read-to-write ratio, then *carefully* consider `StampedLock`.

---
## ✅ Checkpoint: Did You Master This?

- [ ] `StampedLock` yokka "optimistic read" ante enti? Adi a a a performance ni improve chestundi?
- [ ] Optimistic read fail aythe, em cheyali?
- [ ] `StampedLock` ni `ReentrantLock` laaga reentrant anukuni vadite emavtundi?
- [ ] Ye scenario lo `StampedLock` better `ReadWriteLock` kanna?

**✅ Ready?** → [Next: Phase 5 - Atomic Operations](../05-Atomic-Operations/01-Atomic-Variables.md)

**😕 Need Review?** → Paiki scroll chesi optimistic read flow malli chudu.
