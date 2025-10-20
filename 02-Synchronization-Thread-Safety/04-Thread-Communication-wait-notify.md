---
# 🎯 2.4: Thread Communication (`wait`, `notify`, `notifyAll`)

## 🗺️ Learning Path Position

**📍 You Are Here:** 2.4 - Thread Communication

**✅ Prerequisites (Must Complete First):**
- [2.2: The `synchronized` Keyword](./02-Synchronized-Keyword.md) - `synchronized` block/method and intrinsic locks meeda solid understanding undali.

**🔜 Coming After This:**
- **Phase 3:** [Concurrency Problems & Solutions](../03-Concurrency-Problems/01-Deadlock.md) - Deadlock lanti complex problems gurinchi nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 🤔 What & Why

### The Problem: Inefficient Waiting
Imagine, oka thread (Producer) data ni prepare chesi oka list lo pedutundi. Inko thread (Consumer) aa list nunchi data teskuni process chestundi.
- **Consumer Problem:** List empty ga unte, Consumer em cheyali? Continuously list empty na kada ani check chestu undali (`while (list.isEmpty()) {}`). Idi **busy-waiting** antaru. Idi CPU ni anavasaranga 100% use cheskuntundi. Power waste, performance waste.
- **Producer Problem:** List ki oka max size unte, adi full ayinappudu Producer em cheyali? Adi kuda busy-waiting cheyalsi vastundi.

Manaki oka better mechanism kavali. Thread ki pani leనప్పుడు, adi nidrapovali (`WAITING` state), and eppudైతే pani vastundo, appudu levali.

### The Solution: `wait()`, `notify()`, `notifyAll()`
Ee methods `java.lang.Object` class lo untayi, ante prathi Java object ki ee methods untayi.
- **`wait()`:** Oka thread ee method ni call cheste, adi hold chesina lock ni **release** chesi, `WAITING` state loki velli nidrapotundi.
- **`notify()`:** Ade object meeda wait chestunna threads lo *oka* thread ni leputundi (`RUNNABLE` state ki testundi). Evariని leputundo guarantee ledu.
- **`notifyAll()`:** Ade object meeda wait chestunna **anni** threads ni leputundi.

🔥 **Golden Rule:** Ee 3 methods ni **`synchronized` block or method lopala matrame** call cheyali. Leda `IllegalMonitorStateException` vastundi.

### Real-World Analogy: Coffee Shop ☕
- **Shared Resource:** Order Queue on the counter.
- **Producer (Cashier):** Takes orders and adds to the queue.
- **Consumer (Barista):** Takes orders from the queue and makes coffee.
- **`synchronized` block:** Only one person can touch the queue at a time.
- **Barista sees queue is empty:** Barista says "Nenu wait chestunna" (`queue.wait()`) and goes to rest, leaving the counter free.
- **Cashier adds an order:** Cashier adds an order and shouts "Order vachindi!" (`queue.notify()`) to wake up the barista.
- **Barista wakes up,** takes the order, and starts working.

---

## 💻 Code Example: The Producer-Consumer Problem

**Scenario:** Manam oka fixed-size buffer (queue) teskuni, producer numbers ni produce chesi andulo pedtadu, consumer daanini teskuni print chestadu.

### ❌ Failure Case #1: Busy Waiting (No `wait`/`notify`)
```java
// This is VERY BAD for CPU. Don't do this in production.
public void consume() {
    while (true) {
        synchronized (queue) {
            if (!queue.isEmpty()) {
                int value = queue.remove();
                // process value
                break; // for demonstration
            }
        }
        // If queue was empty, the loop continues, wasting CPU cycles.
    }
}
```
**Why this is Bad:** Thread sleep loki velladu, `RUNNABLE` state lo ne untu continuously loop tirugutundi, CPU resources ni bokka pedutundi.

### ❌ Failure Case #2: `IllegalMonitorStateException`
```java
public void consume() throws InterruptedException {
    if (queue.isEmpty()) {
        // ❌ Mistake: Calling wait() outside a synchronized block.
        queue.wait();
    }
}
```
**Error:** `java.lang.IllegalMonitorStateException: current thread is not owner`. `wait` call chese mundu, nuvvu aa object (`queue`) meeda lock hold cheyali.

### ✅ The Correct Implementation
```java
// File: com/example/prodcons/ProducerConsumer.java
package com.example.prodcons;

import java.util.LinkedList;
import java.util.Queue;

public class ProducerConsumer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int CAPACITY = 5;

    // PRODUCER LOGIC
    public void produce() throws InterruptedException {
        int value = 0;
        while (true) {
            synchronized (this) { // Lock on the shared resource monitor
                // 1. Check if the queue is full. Use 'while' to handle spurious wakeups.
                while (queue.size() == CAPACITY) {
                    System.out.println("Queue is full, producer is waiting...");
                    wait(); // Releases the lock and waits
                }

                // 2. Add the value to the queue
                System.out.println("Produced: " + value);
                queue.add(value++);

                // 3. Notify the consumer that an item is available
                notifyAll(); // Wakes up all waiting threads (producers and consumers)

                // Simulate some time
                Thread.sleep(1000);
            }
        }
    }

    // CONSUMER LOGIC
    public void consume() throws InterruptedException {
        while (true) {
            synchronized (this) { // Lock on the same monitor
                // 1. Check if the queue is empty.
                while (queue.isEmpty()) {
                    System.out.println("Queue is empty, consumer is waiting...");
                    wait(); // Releases the lock and waits
                }

                // 2. Remove the value from the queue
                int value = queue.poll();
                System.out.println("Consumed: " + value);

                // 3. Notify the producer that space is available
                notifyAll();

                // Simulate some time
                Thread.sleep(1000);
            }
        }
    }
}

// File: com/example/prodcons/Main.java
package com.example.prodcons;

public class Main {
    public static void main(String[] args) {
        ProducerConsumer pc = new ProducerConsumer();

        // Producer Thread
        Thread producerThread = new Thread(() -> {
            try {
                pc.produce();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        // Consumer Thread
        Thread consumerThread = new Thread(() -> {
            try {
                pc.consume();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        producerThread.start();
        consumerThread.start();
    }
}
```
**Output:**
```
Produced: 0
Consumed: 0
Produced: 1
Consumed: 1
...
Produced: 5
Queue is full, producer is waiting...
Consumed: 5
Produced: 6
...
```
**Why This Works:**
- Producer and Consumer rendu `synchronized(this)` use chesi **okate lock** meeda pani chestunnayi.
- Queue full ayinappudu, producer `wait()` call chesi lock release chestadu, so consumer run avagaladu.
- Queue empty ayinappudu, consumer `wait()` call chesi lock release chestadu, so producer run avagaladu.
- `notifyAll()` ensures that waiting threads are woken up to re-check the condition.

---

### 🔥 `notify()` vs `notifyAll()` - The Danger of `notify()`

- **`notify()`:** Wakes up **one** randomly chosen thread waiting on this object's monitor.
- **`notifyAll()`:** Wakes up **all** threads waiting on this object's monitor.

**Problem with `notify()`:**
Manam multiple producers and multiple consumers unnarani anukundam. Queue full ga undi, so andaru producers wait chestunnaru. Oka consumer item teskuni `notify()` call chestadu. System a a `notify()` ni inko producer ki pampisthe emavtundi? Aa producer lechi chustadu, queue inka full gane undi, so malli wait chestadu. Kaani asalu pani cheyalsina consumer matram nidrapotune untadu. Idi **deadlock** ki dari teeyochu.

**Rule of Thumb:** 🔥 **Always prefer `notifyAll()` over `notify()`.** Idi safer and less error-prone. Performance impact chala takkuva untundi.

---

## ✅ Checkpoint: Did You Master This?

- [ ] `wait()` call chesinappudu, thread hold chesina lock emavtundi?
- [ ] `wait()`/`notify()` ni `synchronized` block bayata enduku call cheyakudadu?
- [ ] `wait()` ni eppudu `while` loop lo enduku pettali (`if` lo enduku pettakudadu)? (Hint: Spurious Wakeups)
- [ ] `notify()` ki `notifyAll()` ki teda enti? Edi safer and enduku?

**✅ Ready?** → [Next: Phase 3 - Concurrency Problems (Deadlock)](../03-Concurrency-Problems/01-Deadlock.md)

**😕 Need Review?** → Paiki scroll chesi producer-consumer code flow malli chudu.
