---
# 🎯 4.2: ReadWriteLock

## 🗺️ Learning Path Position

**📍 You Are Here:** 4.2 - `ReadWriteLock`

**✅ Prerequisites (Must Complete First):**
- [4.1: The `Lock` Interface and `ReentrantLock`](./01-ReentrantLock.md) - `Lock` interface, `lock()` and `unlock()` usage gurinchi clear ga teliyali.

**🔜 Coming After This:**
- [4.3: StampedLock](./03-StampedLock.md) - Inko advanced locking mechanism with "optimistic reads" gurinchi nerchukuntam.

**⏱️ Estimated Time:** 2.5 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 🤔 What & Why

### The Problem: Readers Have to Wait Unnecessarily
Imagine oka shared configuration object undi. Hundreds of threads ee configuration ni continuously read chestunnayi. Appatlo okasari, oka admin thread ee configuration ni update chestundi (write).
- `ReentrantLock` or `synchronized` use cheste emavtundi? At a time, **okate thread** matrame configuration ni read cheyagaladu. Migatha andaru readers wait cheyalsi vastundi, even though they are not changing anything! Idi performance ki pedda debba.

### The Solution: `ReadWriteLock`
`ReadWriteLock` ane interface ee problem ni solve chestundi. Deeniki rendu separate locks untayi:
1.  **Read Lock (`readLock()`):** Idi shared lock. Multiple threads ee lock ni okesari acquire cheyochu.
2.  **Write Lock (`writeLock()`):** Idi exclusive lock. `synchronized` laaga, at a time okate thread ee lock ni acquire cheyagaladu.

**Rules:**
- Multiple threads can hold the read lock simultaneously.
- Only one thread can hold the write lock.
- Write lock hold chesi unte, ye thread kuda read lock teskoledu.
- At least oka read lock hold chesi unte, ye thread kuda write lock teskoledu.

### Real-World Analogy: Wikipedia Article 📖
- **Article:** The shared resource.
- **Readers (Read Threads):** Entha mandaina okesari article ni chadavochu. No problem.
- **Editor (Write Thread):** Oka editor article ni edit chestunnapudu, adi lock aipotundi. Aa time lo vere vallu chadavaleru (latest version kosam wait chestaru), and inkokaru edit cheyaleru.

---

## 💻 Code Example: A Thread-Safe Cache

**Scenario:** Manam oka simple in-memory cache implement cheddam. Ee cache lo `get()` operations (reads) chala ekkuva, `put()` operations (writes) chala takkuva.

### ❌ The Failure Case: Using a Simple `ReentrantLock`
```java
public class SlowCache {
    private final Map<String, String> map = new HashMap<>();
    private final Lock lock = new ReentrantLock();

    public String get(String key) {
        lock.lock();
        try {
            // Even for a simple read, all other readers must wait.
            return map.get(key);
        } finally {
            lock.unlock();
        }
    }

    public void put(String key, String value) {
        lock.lock();
        try {
            map.put(key, value);
        } finally {
            lock.unlock();
        }
    }
}
```
**Why this is Slow:** 100 threads `get()` ni call cheste, avi anni okati tarvata okati serial ga execute avtayi. Pedda performance bottleneck.

### ✅ The Success Case: Using `ReentrantReadWriteLock`
```java
// File: com/example/readwrite/FastCache.java
package com.example.readwrite;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class FastCache {
    private final Map<String, String> map = new HashMap<>();
    // 1. Create a ReadWriteLock instance
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    // 2. Get the specific read and write locks from it
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // READ operation
    public String get(String key) {
        // Acquire the read lock (shared)
        readLock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " is reading.");
            // Multiple threads can be inside this block at the same time!
            Thread.sleep(100); // Simulate some read work
            return map.get(key);
        } catch(InterruptedException e) {
            return null;
        } finally {
            System.out.println(Thread.currentThread().getName() + " finished reading.");
            readLock.unlock();
        }
    }

    // WRITE operation
    public void put(String key, String value) {
        // Acquire the write lock (exclusive)
        writeLock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " is WRITING.");
            // Only one thread can be inside this block.
            Thread.sleep(1000); // Simulate some write work
            map.put(key, value);
        } catch(InterruptedException e) {
            // handle
        } finally {
            System.out.println(Thread.currentThread().getName() + " finished WRITING.");
            writeLock.unlock();
        }
    }

    public static void main(String[] args) {
        FastCache cache = new FastCache();

        // One writer thread
        Thread writer = new Thread(() -> {
            cache.put("key1", "value1");
        }, "Writer");

        // Multiple reader threads
        Runnable readerTask = () -> {
            String value = cache.get("key1");
            System.out.println(Thread.currentThread().getName() + " got value: " + value);
        };

        Thread reader1 = new Thread(readerTask, "Reader-1");
        Thread reader2 = new Thread(readerTask, "Reader-2");
        Thread reader3 = new Thread(readerTask, "Reader-3");

        writer.start(); // Start writer first
        try{ Thread.sleep(200); } catch(InterruptedException e){}
        reader1.start();
        reader2.start();
        reader3.start();
    }
}
```
**Output:**
```
Writer is WRITING.
// (Readers start but wait because the write lock is held)
Writer finished WRITING.
// (Now all readers can acquire the read lock and proceed in parallel)
Reader-1 is reading.
Reader-2 is reading.
Reader-3 is reading.
// (All finish reading at approx the same time)
Reader-1 finished reading.
Reader-1 got value: value1
Reader-2 finished reading.
Reader-2 got value: value1
Reader-3 finished reading.
Reader-3 got value: value1
```
**Why This is Faster:** Reader threads anni okesari parallel ga run avtayi. `get()` operation time anedi number of readers meeda depend avvadu. Performance chala better ga untundi.

---

### 🔥 Lock Downgrading and Upgrading

- **Lock Downgrading (Possible):** Oka thread **write lock** hold chesi unte, adi **read lock** ni kuda acquire cheyochu. Endukante, write lock holder ki anni permissions untayi. Write lock hold chesi, read lock acquire chesi, tarvata original write lock ni release cheste, daanini **lock downgrading** antaru.
- **Lock Upgrading (NOT Possible):** Oka thread **read lock** hold chesi unte, adi **write lock** ni acquire cheyaldu. Endukante, multiple readers undochu. Oka reader write lock ki upgrade avadaniki try cheste, deadlock vastundi. Read lock ni release chesi, tarvata write lock teskovali.

---
## ✅ Checkpoint: Did You Master This?

- [ ] `ReentrantLock` ki `ReadWriteLock` ki main difference enti?
- [ ] Ye scenario lo `ReadWriteLock` use cheste performance baga improve avtundi?
- [ ] Write lock hold chesi unnapudu, vere threads read lock teskovacha?
- [ ] Lock "upgrading" (read to write) enduku possible kadu?

**✅ Ready?** → [Next: StampedLock](./03-StampedLock.md)

**😕 Need Review?** → Paiki scroll chesi Wikipedia analogy malli chudu.
