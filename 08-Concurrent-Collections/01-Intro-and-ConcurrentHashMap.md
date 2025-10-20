---
# 🎯 8.1 & 8.2: Intro to Concurrent Collections & ConcurrentHashMap

## 🗺️ Learning Path Position

**📍 You Are Here:** 8.1 & 8.2 - Intro & ConcurrentHashMap

**✅ Prerequisites (Must Complete First):**
- [2.2: The `synchronized` Keyword](../02-Synchronization-Thread-Safety/02-Synchronized-Keyword.md) - `synchronized` blocks and methods gurinchi clear idea undali.
- [4.1: `java.util.concurrent.locks`](../04-Advanced-Locking-Mechanisms/01-ReentrantLock.md) - `ReentrantLock` gurinchi teliste, `ConcurrentHashMap` internal working ardham cheskovadam easy.

**🔜 Coming After This:**
- [8.2: Read-Optimized and Sorted Collections](./02-Read-Optimized-and-Sorted-Collections.md) - `CopyOnWriteArrayList` lanti special-purpose collections gurinchi nerchukuntam.

**⏱️ Estimated Time:** 3.5 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: Simple Collections Aren't Thread-Safe! 💣
Mawa, manam regular ga use chese `ArrayList`, `HashMap` lanti collections **thread-safe kavu**. Ante, multiple threads okate `HashMap` ni modify cheste (e.g., okati `put` chestu, inkoti `remove` chestu), adi corrupt aypotundi. Data loss avvachu, or worse, program infinite loop lo ki vellipovachu.

### The "Old" Solution: `Collections.synchronizedMap()`
Ee problem ni solve cheyadaniki, Java developers `Collections.synchronizedMap(new HashMap<>())` lanti wrapper methods use chesevaru. Ee wrappers `HashMap` yokka prathi method ni (`get`, `put`, `remove`) `synchronized` block lo wrap chestayi.

**But this has a HUGE performance problem:**
- **Single Lock for the Whole Map:** Ee wrapper antha map ki okate lock (`intrinsic lock`) use chestundi. Ante, Thread-A `get()` call chestunte, Thread-B `put()` cheyalekunda block avtundi. Read operations kuda write operations ni block chestayi. Idi high-contention scenarios lo chala slow (poor scalability).
- **`ConcurrentModificationException`:** Nuvvu `synchronizedMap` meeda iterate avutunnapudu, inko thread aa map ni modify cheste, `ConcurrentModificationException` vastundi.

### The "Modern" Solution: `ConcurrentHashMap` (and other concurrent collections) 🚀
Java 5 lo `java.util.concurrent` package tho paatu, `ConcurrentHashMap` lanti highly optimized, thread-safe collections vachayi. Ivvi `synchronized` wrappers kanna chala better performance istayi.

**`ConcurrentHashMap`'s Magic: Lock Striping (and more)**
`ConcurrentHashMap` antha map ki okate lock use cheyadu. Adi map ni chala chinna chinna segments/partitions ga divide chestundi. Prathi segment ki oka separate lock untundi.
- Thread-A segment-1 lo write chestunte, Thread-B segment-10 lo write cheyochu. Rendu parallel ga jarugutayi!
- Read operations (`get`) anevi chala varaku lock-free ga untayi. `volatile` reads use cheskuntayi.
- Deenivalla, multiple threads map ni concurrently access chesina, contention chala takkuva untundi and performance chala ekkuva untundi.

### Real-World Analogy: Supermarket Billing Counters 🛒
- **`synchronizedMap`:** Oka pedda supermarket lo okate okka billing counter undi. Andaru customers (threads) adhe line lo wait cheyali. Chala slow.
- **`ConcurrentHashMap`:** Adhe supermarket lo 16 billing counters (segments) unnayi. Customers different counters ki velli, parallel ga bill cheyinchukovachu. Oka counter deggara rush unna, vere counters free ga untayi. Chala efficient.

---

## 📚 Detailed Explanation: `ConcurrentHashMap`

`ConcurrentHashMap` anedi `HashMap` ki thread-safe replacement. Idi `null` keys or values ni allow cheyadu.

### Key Features:
- **High Concurrency:** Optimized for high number of reads and concurrent writes.
- **Fail-Safe Iterators:** Iterator create chesina tarvata, vere threads map ni modify chesina, `ConcurrentModificationException` రాదు. Iterator anedi aa time lo unna snapshot meeda pani chestundi.
- **Atomic Operations:** `putIfAbsent()`, `compute()`, `merge()` lanti powerful atomic operations ni support chestundi.

### Important Methods:

| Method | Description | Atomicity |
|---|---|---|
| `put(K key, V value)` | Standard `put` operation. | Atomic |
| `get(Object key)` | Standard `get`. Usually lock-free. | Atomic |
| `putIfAbsent(K key, V value)` | Key lekapothe matrame, value ni put chestundi. | Atomic |
| `compute(K key, BiFunction fn)` | Key ki unna current value tho oka computation chesi, daanni update chestundi. | Atomic |
| `merge(K key, V value, BiFunction fn)`| Key lekapothe, new value ni put chestundi. Unte, old and new values tho function run chesi result ni update chestundi.| Atomic |

---

## 💻 Code Examples

### Example 1: `synchronizedMap` vs. `ConcurrentHashMap` Performance

#### ❌ The Slow `synchronizedMap`
```java
// File: com/example/maps/SyncMapDemo.java
// Note: This is a conceptual test. Real microbenchmarking is more complex.
package com.example.maps;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class SyncMapDemo {
    public static void main(String[] args) throws InterruptedException {
        Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
        ExecutorService executor = Executors.newFixedThreadPool(10);

        long startTime = System.nanoTime();
        for (int i = 0; i < 100000; i++) {
            final int key = i;
            executor.execute(() -> syncMap.put("key" + key, key));
        }

        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.MINUTES);
        long endTime = System.nanoTime();

        System.out.println("Time taken for synchronizedMap: " + (endTime - startTime) / 1_000_000 + " ms");
    }
}
```

#### ✅ The Fast `ConcurrentHashMap`
```java
// File: com/example/maps/ConcurrentMapDemo.java
package com.example.maps;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ConcurrentMapDemo {
    public static void main(String[] args) throws InterruptedException {
        Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();
        ExecutorService executor = Executors.newFixedThreadPool(10);

        long startTime = System.nanoTime();
        for (int i = 0; i < 100000; i++) {
            final int key = i;
            executor.execute(() -> concurrentMap.put("key" + key, key));
        }

        executor.shutdown();
        executor.awaitTermination(1, TimeUnit.MINUTES);
        long endTime = System.nanoTime();

        System.out.println("Time taken for ConcurrentHashMap: " + (endTime - startTime) / 1_000_000 + " ms");
    }
}
```
**Expected Output:** `ConcurrentHashMap` time anedi `synchronizedMap` kanna chala takkuva untundi, especially on multi-core machines.

### Example 2: Atomic "Check-then-Act" with `putIfAbsent`
**Scenario:** Oka resource ni okasari matrame initialize cheyali.
#### ❌ The Old Way with Race Condition
```java
if (map.get("resource") == null) {
    map.put("resource", "Initialized!"); // RACE CONDITION HERE!
}
```
#### ✅ The `ConcurrentHashMap` Way
```java
// File: com/example/atomic/AtomicOperations.java
package com.example.atomic;

import java.util.concurrent.ConcurrentHashMap;

public class AtomicOperations {
    public static void main(String[] args) {
        ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();

        // Multiple threads might call this simultaneously
        // but only one will succeed in putting the value.
        map.putIfAbsent("resource", "Initialized by Thread-1");
        map.putIfAbsent("resource", "Initialized by Thread-2"); // This will fail silently

        System.out.println(map.get("resource")); // Always prints "Initialized by Thread-1"
    }
}
```

### Example 3: `compute` to Implement a Concurrent Counter
**Scenario:** Prathi word occurrence ni count cheyali.
```java
// File: com/example/atomic/WordCounter.java
package com.example.atomic;

import java.util.concurrent.ConcurrentHashMap;

public class WordCounter {
    public static void main(String[] args) {
        ConcurrentHashMap<String, Integer> wordCounts = new ConcurrentHashMap<>();
        String text = "hello world hello mawa";

        for (String word : text.split(" ")) {
            // ✅ Atomically update the count
            wordCounts.compute(word, (key, value) -> (value == null) ? 1 : value + 1);
        }

        System.out.println(wordCounts); // {world=1, mawa=1, hello=2}
    }
}
```

---

## ✅ Checkpoint: Did You Master This?

- [ ] `Collections.synchronizedMap` ki `ConcurrentHashMap` ki main performance difference enti?
- [ ] `ConcurrentHashMap` lo `get` operation enduku fast ga untundi?
- [ ] `putIfAbsent` anedi race condition ni ela prevent chestundi?
- [ ] `ConcurrentHashMap` lo `null` keys enduku allow cheyaru? (Hint: `get(key)` returns `null` for two reasons...)

**✅ Ready?** → [Next: Read-Optimized and Sorted Collections](./02-Read-Optimized-and-Sorted-Collections.md)
