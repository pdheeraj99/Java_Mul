---
# 🎯 5.1: Atomic Variables

## 🗺️ Learning Path Position

**📍 You Are Here:** 5.1 - Atomic Variables

**✅ Prerequisites (Must Complete First):**
- [2.1: Race Conditions & Critical Sections](../02-Synchronization-Thread-Safety/01-Race-Conditions-and-Critical-Sections.md) - `i++` enduku atomic kado clear ga gurtundali.
- [2.2: The `synchronized` Keyword](../02-Synchronization-Thread-Safety/02-Synchronized-Keyword.md) - Locking mechanism tho ee lock-free approach ni compare cheyadaniki.

**🔜 Coming After This:**
- [5.2: Adders and Accumulators](./02-Adders-and-Accumulators.md) - High-contention scenarios lo `AtomicLong` kanna better performance iche `LongAdder` gurinchi nerchukuntam.

**⏱️ Estimated Time:** 2.5 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: Locking is (Sometimes) Slow
`synchronized` and `ReentrantLock` anevi chala powerful, kaani vati valla konni problems unnayi:
- **Overhead:** Lock teskovadam, release cheyadam anedi konchem expensive operation.
- **Blocking:** Oka thread lock hold chestunte, vere threads block avtayi. Context switching jarugutundi.
- **Complexity:** Deadlocks, Livelocks lanti problems vache chance undi.

Simple operations ki (oka counter ni increment cheyadaniki), intha overhead avasarama?

### The Solution: Atomic Variables (Lock-Free Thread Safety)
`java.util.concurrent.atomic` package lo unna classes (like `AtomicInteger`) ee problem ni solve chestayi. Ivvi locks use cheyavu. Vaati badulu, **CAS (Compare-And-Swap)** ane oka special, low-level CPU instruction ni use cheskuntayi.

**CAS ante enti?**
Idi oka atomic instruction. Simple ga, idi 3 parameters teskuntundi:
1.  **Memory location (V):** Manam update cheyalsina variable.
2.  **Expected old value (A):** Manam anukuntunna current value.
3.  **New value (B):** Manam set cheyalsina kotha value.

CPU ee operation ni **atomically** chestundi: "Memory location V lo unna value inka A ga ne unte, daanini B ga marchu. Lekapothe, em cheyaku."

`incrementAndGet()` lanti methods ee CAS ni loop lo use cheskuntayi:
1.  Current value ni read chey (`100`).
2.  New value ni calculate chey (`101`).
3.  CAS try chey: `compareAndSet(100, 101)`.
    - **Success aite:** Super, operation complete.
    - **Fail aite:** Ante, madhyalo inko thread value ni `100` nunchi `105` ki marchesindi. Parledu, malli step 1 nunchi try chey (current value `105` teskuni).

Ee process antha chala fast ga jarugutundi and threads block avvavu.

---

## 💻 Code Examples

### Example 1: `AtomicInteger` - The Thread-Safe Counter without Locks

#### ✅ Success Case: Solving the `UnsafeCounter` problem
```java
// File: com/example/atomic/AtomicCounter.java
package com.example.atomic;

import java.util.concurrent.atomic.AtomicInteger;

public class AtomicCounter {
    // 1. Use AtomicInteger instead of int
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        // 2. This is a thread-safe, atomic operation!
        count.incrementAndGet();
    }

    public int getCount() {
        return count.get();
    }
}

// File: com/example/atomic/Main.java
package com.example.atomic;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        AtomicCounter counter = new AtomicCounter();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) counter.increment();
        });
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) counter.increment();
        });

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        // Output is ALWAYS 20000
        System.out.println("Final count: " + counter.getCount());
    }
}
```
**Output (Every time):** `Final count: 20000`
**Why This Works:** `incrementAndGet()` anedi atomic. Rendu threads okesari call chesina, CAS mechanism valla "lost update" problem radu. And most importantly, manam `synchronized` keyword or `Lock` object వాడలేదు!

---

### Example 2: `compareAndSet()` - The Building Block of Atomics

#### ✅ Success Case: Ensuring a one-time initialization
**Scenario:** Oka resource ni okate thread initialize cheyali.
```java
// File: com/example/cas/ResourceInitializer.java
package com.example.cas;

import java.util.concurrent.atomic.AtomicBoolean;

public class ResourceInitializer {
    private final AtomicBoolean initialized = new AtomicBoolean(false);

    public void initialize() {
        // Use CAS to ensure this block runs only once
        if (initialized.compareAndSet(false, true)) {
            System.out.println(Thread.currentThread().getName() + ": Initializing the resource...");
            // ... perform one-time setup ...
        } else {
            System.out.println(Thread.currentThread().getName() + ": Resource is already initialized by another thread.");
        }
    }

    public static void main(String[] args) {
        ResourceInitializer initializer = new ResourceInitializer();

        // Create multiple threads that try to initialize
        Thread t1 = new Thread(() -> initializer.initialize(), "Thread-1");
        Thread t2 = new Thread(() -> initializer.initialize(), "Thread-2");

        t1.start();
        t2.start();
    }
}
```
**Output (Example):**
```
Thread-1: Initializing the resource...
Thread-2: Resource is already initialized by another thread.
```
**Why This Works:** First thread `compareAndSet(false, true)` call chesinappudu, adi success avtundi. Rendo thread vachesariki, `initialized` value `true` ga untundi, so `compareAndSet(false, true)` fail avtundi. Idi lock lekunda "check-then-act" operation ni thread-safe ga cheyadaniki perfect example.

---

### Example 3: `AtomicReference` - Making Object References Atomic

#### ✅ Success Case: A Thread-Safe Singleton without `synchronized`
```java
// File: com/example/atomicref/Singleton.java
package com.example.atomicref;

import java.util.concurrent.atomic.AtomicReference;

class MySingleton {
    private MySingleton() {} // private constructor
    private static final AtomicReference<MySingleton> INSTANCE = new AtomicReference<>();

    public static MySingleton getInstance() {
        while (true) {
            MySingleton existingInstance = INSTANCE.get();
            if (existingInstance != null) {
                return existingInstance;
            }

            MySingleton newInstance = new MySingleton();
            // Try to set the new instance ONLY if the current one is null
            if (INSTANCE.compareAndSet(null, newInstance)) {
                return newInstance;
            }
        }
    }
}
```
**Why This Works:** Idi "double-checked locking" pattern ki lock-free alternative. `compareAndSet` valla, multiple threads `getInstance` ni call chesina, constructor okate sari call avtundi ani guarantee.

---

## 🌟 Best Practices
1.  **Prefer Atomics for Simple Counters/Flags:** Simple state updates (counters, flags, on/off switches) ki `synchronized` or `ReentrantLock` kanna Atomics better performance istayi.
2.  **Use `getAndUpdate` or `updateAndGet` for Complex Logic:** `i -> i * 2` lanti lambda functions use chesi complex atomic updates cheyochu.
3.  **Remember: Atomics Don't Compose:** Rendu separate atomic operations (e.g., `atomicInt1.increment()` and `atomicInt2.increment()`) ni kalipi oka single atomic operation ga cheyalemu. Appudu manaki `synchronized` or `Lock` avasaram.

---
## ✅ Checkpoint: Did You Master This?
- [ ] Lock-based synchronization ki and Atomic variables ki main difference enti?
- [ ] CAS (Compare-And-Swap) ante nee own words lo explain cheyagalava?
- [ ] `AtomicInteger` anedi `count++` race condition ni a a a solve chestundi?
- [ ] Rendu `AtomicInteger` variables ni update cheyali. Ee operation atomic ga undalante, em cheyali? (Hint: Can atomics do it?)

**✅ Ready?** → [Next: Adders and Accumulators](./02-Adders-and-Accumulators.md)
**😕 Need Review?** → Paiki scroll chesi CAS explanation malli chudu.
