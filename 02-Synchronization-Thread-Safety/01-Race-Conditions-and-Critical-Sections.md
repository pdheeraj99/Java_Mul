---
# 🎯 2.1: Race Conditions & Critical Sections

## 🗺️ Learning Path Position

**📍 You Are Here:** 2.1 - Race Conditions & Critical Sections

**✅ Prerequisites (Must Complete First):**
- [1.2: Creating & Managing Threads](../01-Foundation-Threading-Basics/02-Creating-and-Managing-Threads.md) - Multiple threads ni create cheyadam and `join()` use cheyadam teliyali.
- [1.3: Java Memory Model](../01-Foundation-Threading-Basics/03-Java-Memory-Model.md) - Threads memory ni ela share cheskuntayo (Heap vs Stack) clear idea undali.

**🔜 Coming After This:**
- [2.2: The `synchronized` Keyword](./02-Synchronized-Keyword.md) - Ee race conditions ni solve cheyadaniki powerful weapon `synchronized` gurinchi nerchukuntam.

**⏱️ Estimated Time:** 2 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 💭 Quick Recap

💭 **Gurtunda ra?** Manam Java Memory Model lo "Visibility Problem" gurinchi matladukunnam. Ante, oka thread chesina change inko thread ki kanipinchakapovachu ani. Ippudu manam inko pedda problem chustam: **Atomicity Problem**. Rendu threads okate data ni *at the same time* modify cheste emavtundo chustam.

---

## 🤔 What & Why

### The Problem: Two Hands in One Jar 🍯
Imagine, bank account lo 10,000 rupees unnai. Nuvvu ATM nunchi 5,000 withdraw chesav. Almost అదే time lo, nee partner kuda online lo 5,000 withdraw chesindi. Rendu transactions okesari start ayyayi.

1.  **Your ATM:** Balance chusindi -> 10,000.
2.  **Partner's App:** Balance chusindi -> 10,000.
3.  **Your ATM:** 10,000 - 5,000 = 5,000. New balance ni update chesindi.
4.  **Partner's App:** 10,000 - 5,000 = 5,000. New balance ni update chesindi.

Final balance enta undali? **Zero**. Kaani enta undi? **5,000**. Bank ki 5,000 loss! Endukante, iddari operations interleaving (kalisiపోయి) ayyayi. Ee disaster ne **Race Condition** antaru.

### The Solution: The "Critical Section"
Ee problem unna code area ni (e.g., account balance update chese logic), manam **Critical Section** antam. Solution entante, ee critical section ni okate thread at a time execute chesela cheyadam. Daaniki manam "locks" use chestam. Idi bathroom door lock lantidi. Oka person use chestunnapudu, vere vallu bayata wait cheyali.

---

## 💻 Code Example: The Unsafe Counter

Ee example lo, manam oka shared counter ni 2 threads tho 10,000 sarlu increment cheddam. Final value enta ravali? `20,000` ravali. Chuddam, vastundo ledo.

### ❌ The Failure Case: Race Condition in Action!
**Scenario:** Oka website lo, entha mandi users visit chesaro count chese counter undi. Multiple users okesari visit cheste, ee counter sarigga pani cheyademo chuddam.

```java
// File: com/example/race/UnsafeCounter.java
package com.example.race;

public class UnsafeCounter {
    private int count = 0;

    // Ee method thread-safe KADU!
    public void increment() {
        // Step 1: Read the current value of count
        // Step 2: Increment the value by 1
        // Step 3: Write the new value back to count
        // Ee 3 steps madhyalo inkoka thread vachi disturb cheyochu!
        count++;
    }

    public int getCount() {
        return count;
    }
}

// File: com/example/race/Main.java
package com.example.race;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        UnsafeCounter counter = new UnsafeCounter();

        // Rendu threads create cheddam
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                counter.increment();
            }
        }, "Thread-1");

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                counter.increment();
            }
        }, "Thread-2");

        // Start both threads
        t1.start();
        t2.start();

        // Wait for both to finish
        t1.join();
        t2.join();

        // Expected: 20000. But what do we get?
        System.out.println("Final count: " + counter.getCount());
    }
}
```
**Output (First Run):**
`Final count: 12457`

**Output (Second Run):**
`Final count: 18912`

**Output (Third Run):**
`Final count: 15321`

**Why This Fails (The Deep Dive):**
Chusava mawa? Prathi sari oka kotha result vastundi, and eppudu `20,000` ravatledu. Endukante `count++` anedi **atomic operation kadu**. Adi background lo 3 steps ga jarugutundi:
1.  **Read:** Get current value of `count`.
2.  **Modify:** Add 1 to it.
3.  **Write:** Save the new value back.

Ee 3 steps madhyalo em jarugutundo chudu:
1.  `count` is `100`.
2.  **Thread 1** reads `count` (value `100`).
3.  **Context Switch!** CPU Thread 1 ni aapi, Thread 2 ki chance istundi.
4.  **Thread 2** reads `count` (value is still `100`).
5.  **Thread 2** increments its local value to `101`.
6.  **Thread 2** writes `101` back to `count`. So `count` is now `101`.
7.  **Context Switch!** CPU Thread 2 ni aapi, Thread 1 ki chance istundi.
8.  **Thread 1** ki `count` value `100` ani gurtu undi. Adi daaniki 1 add chesi `101` chestundi.
9.  **Thread 1** writes `101` back to `count`.

Rendu threads increment chesayi, kani final result `102` బదులు `101` vachindi. Oka increment miss aypoindi! Ee "lost update" problem valla manaki final count thappuga vastundi.

### What is a Critical Section Here?
Ee example lo, `increment()` method antha **Critical Section**. Ante, aa code block ni okate thread at a time access cheyali.

### What is Thread Safety?
Oka class or method multiple threads tho call chesinappudu kuda correct ga behave cheste, daanini **thread-safe** antaru. Mana `UnsafeCounter` class ippudu **not thread-safe**.

### 💡 So, How Do We Fix This? (A Teaser)
Ee `increment()` method ni okate thread at a time execute cheyali. Daaniki manam `synchronized` ane keyword use chestam. Adi ela anedi next session lo chustam.

---
## 🔥 Key Takeaways

1.  **Race Condition:** Multiple threads shared data ni access chesi, andulo minimum okati write chesinappudu, final result timing meeda depend aythe, adi race condition.
2.  **Atomicity:** Oka operation anedi madhyalo aagakunda, poorthiga okesari jarigithe adi atomic. `count++` lanti operations atomic kaavu.
3.  **Critical Section:** Code lo a bhagam aite race condition ki potential undo, daanini critical section antaru.
4.  **Thread-Unsafe:** Mana `UnsafeCounter` lanti classes correct output ivvavu when used by multiple threads.

---

## ✅ Checkpoint: Did You Master This?

- [ ] Race Condition ante enti? Oka real-world (non-technical) example cheppagalava?
- [ ] `i++` anedi enduku atomic kadu? Background lo em jarugutundi?
- [ ] "Critical Section" ante nee own words lo explain cheyagalava?
- [ ] Oka code thread-safe na kada ani ela cheppagalam?

**✅ Ready?** → [Next: The `synchronized` Keyword](./02-Synchronized-Keyword.md) (Finally, the solution!)

**😕 Need Review?** → Paiki scroll chesi `count++` explanation malli chudu.
