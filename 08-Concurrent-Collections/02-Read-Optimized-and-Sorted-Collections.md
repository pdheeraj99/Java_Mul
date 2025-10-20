---
# 🎯 8.2: Read-Optimized & Sorted Collections

## 🗺️ Learning Path Position

**📍 You Are Here:** 8.2 - Read-Optimized & Sorted Collections

**✅ Prerequisites (Must Complete First):**
- [8.1: Intro to Concurrent Collections](./01-Intro-and-ConcurrentHashMap.md) - `ConcurrentHashMap` gurinchi teliste, ee collections yokka trade-offs better ga ardham avtayi.

**🔜 Coming After This:**
- [8.3: Blocking Queues for Producer-Consumer](./03-Blocking-Queues-for-Producer-Consumer.md) - Producer-Consumer pattern ki backbone ayina `BlockingQueue`s gurinchi nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: One Size Still Doesn't Fit All!
`ConcurrentHashMap` anedi general-purpose, high-performance map. Kaani, manaki `List` or `SortedMap` thread-safe versions kavalante?
- **Read-Heavy `ArrayList`:** Manaki oka list undi, daanini 99% of the time threads read matrame chestayi, eppudo okasari modify chestayi (e.g., a list of event listeners). `Collections.synchronizedList` use cheste, prathi read operation ki kuda lock avasaram. Idi performance ni debba testundi.
- **Sorted `TreeMap`:** Manaki sorted order lo keys maintain chese map kavali (e.g., a leaderboard with scores). `TreeMap` thread-safe kadu. `Collections.synchronizedSortedMap` use cheste, malli adhe single-lock performance problem.

### The Solution: Specialized Concurrent Collections

#### 1. `CopyOnWriteArrayList<E>`
- **Core Idea:** "Copy-On-Write". Reads are cheap, writes are expensive.
- **How it Works:**
  - **Read Operations (`get`, `iterator`):** Chala fast, no locks needed. Readers anevi immutable snapshot (a copy) of the array meeda pani chestayi.
  - **Write Operations (`add`, `remove`):** Chala expensive. Prathi write operation ki, internal array antha **copy** aypoyi, aa kotha copy meeda modification jarigi, tarvata reference anedi aa kotha copy ki point cheyabadutundi.
- **When to Use:** Read operations >> Write operations. (e.g., Listeners, configurations that rarely change).
- **Analogy:** Oka team antha oka shared document ni chustunnaru (reads). Evaraina edit cheyali anukunte, వాళ్ళు aa document ni copy cheskuni (`Copy`), valla changes chesi (`on-Write`), tarvata "idi final version" ani team ki share chestaru. Aa process lo, migatha andaru old version ni chustune untaru without any interruption.

#### 2. `ConcurrentSkipListMap<K, V>`
- **Core Idea:** A thread-safe `SortedMap`.
- **How it Works:** Idi `TreeMap` (Red-Black Tree) laaga balanced binary tree use cheyadu. Daani badulu, **Skip List** ane oka probabilistic data structure ni use chestundi. Skip list anedi multiple layers of linked lists laaga untundi, idi search, insert, and delete operations ni average `O(log n)` time lo cheyadaniki allow chestundi, lock-free algorithms use chesi.
- **When to Use:** Nuvvu `TreeMap` thread-safe version kavali anukunnapudu. Keys anevi sorted order lo undali.
- **Analogy:** Idi oka city map laantidi. Kindest level lo anni streets (elements) untayi. Paina level lo main roads matrame untayi. Inka paina level lo highways matrame untayi. Nuvvu oka place nunchi inko place ki vellali ante, first highway teskuni fast ga daggariki velli, tarvata main roads, tarvata small streets loki enter avtav. Idi normal linked list (anni streets chusukuntu velladam) kanna chala fast.

---

## 💻 Code Examples

### Example 1: `CopyOnWriteArrayList` for Event Listeners
**Scenario:** Oka application undi, adi events ni fire chestundi. Multiple listener threads aa events ni listen chestunnayi. New listeners rarely add avtaru or remove avtaru.

#### ❌ Failure Case: `ArrayList` with `ConcurrentModificationException`
```java
// File: com/example/cow/FailFastIterator.java
package com.example.cow;

import java.util.ArrayList;
import java.util.List;

public class FailFastIterator {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        list.add("one"); list.add("two");

        // Thread 1 starts iterating
        new Thread(() -> {
            for (String s : list) {
                System.out.println("Iterating: " + s);
                try { Thread.sleep(100); } catch (Exception e) {}
            }
        }).start();

        // Thread 2 modifies the list while Thread 1 is iterating
        new Thread(() -> {
            try { Thread.sleep(50); } catch (Exception e) {}
            System.out.println("Trying to add 'three'...");
            list.add("three"); // 💣 This will cause ConcurrentModificationException
        }).start();
    }
}
```

#### ✅ Success Case: `CopyOnWriteArrayList` with Fail-Safe Iterator
```java
// File: com/example/cow/CopyOnWriteDemo.java
package com.example.cow;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

public class CopyOnWriteDemo {
    public static void main(String[] args) {
        List<String> list = new CopyOnWriteArrayList<>();
        list.add("one"); list.add("two");

        new Thread(() -> {
            // Iterator is created on a snapshot of ["one", "two"]
            for (String s : list) {
                System.out.println("Iterating: " + s);
                try { Thread.sleep(100); } catch (Exception e) {}
            }
            System.out.println("Iteration finished.");
        }).start();

        new Thread(() -> {
            try { Thread.sleep(50); } catch (Exception e) {}
            System.out.println("Trying to add 'three'...");
            list.add("three"); // ✅ This is safe. It creates a new underlying array.
            System.out.println("List is now: " + list);
        }).start();
    }
}
```
**Output:**
```
Trying to add 'three'...
Iterating: one
List is now: [one, two, three]
Iterating: two
Iteration finished.
```
**Why This Works:** First thread yokka iterator create ayinappudu, list lo `["one", "two"]` matrame unnayi. Second thread `add("three")` call chesinappudu, `CopyOnWriteArrayList` oka kotha array `["one", "two", "three"]` ni create chesindi. Kaani, first thread iterator inka aa old snapshot (`["one", "two"]`) ne chustundi. So, `ConcurrentModificationException` radu.

---

### Example 2: `ConcurrentSkipListMap` for a Live Leaderboard
**Scenario:** Oka gaming leaderboard undi. Multiple players తమ scores ni concurrently update chestunnaru. Manam eppudu leaderboard ni highest score to lowest score order lo display cheyali.
```java
// File: com/example/sorted/Leaderboard.java
package com.example.sorted;

import java.util.Comparator;
import java.util.concurrent.ConcurrentNavigableMap;
import java.util.concurrent.ConcurrentSkipListMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Leaderboard {
    public static void main(String[] args) {
        // We want a sorted map, from highest score to lowest
        ConcurrentNavigableMap<Integer, String> leaderboard = new ConcurrentSkipListMap<>(Comparator.reverseOrder());

        ExecutorService executor = Executors.newFixedThreadPool(5);

        // Simulate 5 players updating their scores concurrently
        executor.execute(() -> leaderboard.put(95, "Player1"));
        executor.execute(() -> leaderboard.put(85, "Player3"));
        executor.execute(() -> leaderboard.put(100, "Player5"));
        executor.execute(() -> leaderboard.put(95, "Player2")); // Same score as Player1
        executor.execute(() -> leaderboard.put(98, "Player4"));

        try { Thread.sleep(100); } catch (Exception e) {}

        System.out.println("--- Live Leaderboard ---");
        leaderboard.forEach((score, player) -> System.out.println(player + ": " + score));

        executor.shutdown();
    }
}
```
**Output:**
```
--- Live Leaderboard ---
Player5: 100
Player4: 98
Player2: 95
Player3: 85
```
**Why This Works:** Multiple threads `put` operations chesina kuda, `ConcurrentSkipListMap` antha internally handle cheskuni, map ni eppudu sorted state lo ne maintain chestundi. `forEach` kuda fail-safe.

---

## ✅ Checkpoint: Did You Master This?
- [ ] `CopyOnWriteArrayList` ye scenario lo vadali? Yenduku adi write operations ki slow ga untundi?
- [ ] `CopyOnWriteArrayList` iterator `ConcurrentModificationException` enduku throw cheyadu?
- `ConcurrentSkipListMap` anedi `TreeMap` kanna better a? Thread-safe `TreeMap` kavali anukunte edi vadali?
- [ ] Oka `ConcurrentSkipListMap` meeda iterate avutunnapudu, inko thread oka element ni remove cheste em avtundi?

**✅ Ready?** → [Next: Blocking Queues for Producer-Consumer](./03-Blocking-Queues-for-Producer-Consumer.md)
