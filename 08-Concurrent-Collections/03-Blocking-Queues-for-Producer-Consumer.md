---
# 🎯 8.3: Blocking Queues for Producer-Consumer

## 🗺️ Learning Path Position

**📍 You Are Here:** 8.3 - Blocking Queues

**✅ Prerequisites (Must Complete First):**
- [2.4: Thread Communication (wait, notify)](../02-Synchronization-Thread-Safety/04-Thread-Communication-wait-notify.md) - `wait()` and `notify()` tho producer-consumer implement cheyadam lo unna complexity teliste, `BlockingQueue` entha easy no ardham avtundi.

**🔜 Coming After This:**
- [8.4: Advanced Queues and Deques](./04-Advanced-Queues-and-Deques.md) - `DelayQueue`, `PriorityBlockingQueue` lanti special purpose queues gurinchi nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: Coordinating Hand-offs Between Threads is Hard! 🤝
Mawa, imagine oka thread (`Producer`) data ni generate chestundi (e.g., tasks, log messages), and inko thread (`Consumer`) aa data ni process chestundi. Ee iddaru threads madhyalo coordination chala kastam:
- **What if the producer is too fast?** Consumer inka old data ni process chestu unte, producer kotha data generate chesi ekkada pedtadu? Buffer antha nindipotundi (`OutOfMemoryError`).
- **What if the consumer is too fast?** Buffer lo em data lekapothe, consumer em cheyali? Continuously check chestu CPU waste cheyala (`busy-waiting`)?
Ee coordination ni maname `wait()`, `notify()`, and `synchronized` tho handle cheyali ante, chala complex and error-prone code rayali.

### The Solution: `BlockingQueue` - The Thread-Safe Assembly Line  conveyor belt 🏭
`BlockingQueue<E>` anedi ee problem ni beautifully solve chestundi. Idi oka thread-safe queue with a special power:
- **If a producer tries to `put()` an element into a full queue, it BLOCKS** (waits) until a consumer takes an element out.
- **If a consumer tries to `take()` an element from an empty queue, it BLOCKS** (waits) until a producer puts an element in.

Ee blocking behavior antha `BlockingQueue` implementation lone handle aypotundi. Manam `wait()` or `notify()` lanti low-level primitives use cheyalsina avasaram ledu.

### Real-World Analogy: Biryani Restaurant Kitchen 👨‍🍳
- **Chefs (Producers):** Continuously biryani prepare chestu untaru.
- **Serving Counter (BlockingQueue):** Prepare chesina biryanis ni ee counter meeda pedtaru. Ee counter meeda maximum 5 handis matrame pettagalaru (bounded queue).
- **Waiters (Consumers):** Counter nunchi biryani handis teskuni, customers ki serve chestaru.
- **Coordination:**
  - Counter antha full (`queue is full`) ga unte, chefs kotha biryani prepare chesina, akkade wait chestaru (`put()` blocks).
  - Counter meeda em biryani lekapothe (`queue is empty`), waiters orders teskovadaniki akkade wait chestaru (`take()` blocks).

---

## 📚 Detailed Explanation

`BlockingQueue` anedi oka interface. Daani main implementations:
- **`ArrayBlockingQueue<E>`:**
  - Bounded (fixed size). Size ni constructor lo cheppali.
  - Internally array use chestundi.
  - Good for predictable, controlled memory usage.
- **`LinkedBlockingQueue<E>`:**
  - Optionally bounded. Constructor lo size ivvakapothe, `Integer.MAX_VALUE` teskuntundi (effectively unbounded).
  - **WARNING:** Unbounded `LinkedBlockingQueue` use cheste, producer fast ga unte `OutOfMemoryError` vache chance undi. Production lo eppudu bounded vadadam better.

### Key Methods: Blocking vs. Non-Blocking

| Operation | Blocking Method | Non-Blocking Method |
|---|---|---|
| **Add Element** | `put(e)` - Waits if queue is full. | `offer(e)` - Returns `false` if queue is full. |
| **Remove Element**| `take()` - Waits if queue is empty. | `poll()` - Returns `null` if queue is empty. |

---

## 💻 Code Example: The Producer-Consumer Pattern

**Scenario:** Manam oka log processing system build cheddam. Multiple `Producers` log messages generate chesi, oka central `BlockingQueue` lo pedtayi. Multiple `Consumers` aa queue nunchi messages teskuni, process chestayi.

```java
// File: com/example/producerconsumer/ProducerConsumerDemo.java
package com.example.producerconsumer;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

// The shared resource
class MessageQueue {
    // A bounded queue with a capacity of 10
    private final BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

    public void produce(String message) throws InterruptedException {
        System.out.println("  -> Producing: " + message);
        queue.put(message); // Blocks if queue is full
    }

    public String consume() throws InterruptedException {
        String message = queue.take(); // Blocks if queue is empty
        System.out.println("      <- Consuming: " + message);
        return message;
    }
}

// The Producer task
class Producer implements Runnable {
    private final MessageQueue messageQueue;
    public Producer(MessageQueue queue) { this.messageQueue = queue; }

    @Override
    public void run() {
        try {
            for (int i = 0; i < 20; i++) {
                messageQueue.produce("Message-" + i);
                Thread.sleep(100); // Simulate time taken to generate a message
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

// The Consumer task
class Consumer implements Runnable {
    private final MessageQueue messageQueue;
    public Consumer(MessageQueue queue) { this.messageQueue = queue; }

    @Override
    public void run() {
        try {
            while (true) { // Consumers run indefinitely
                messageQueue.consume();
                Thread.sleep(500); // Simulate time taken to process a message
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}

// Main class to run the demo
public class ProducerConsumerDemo {
    public static void main(String[] args) {
        MessageQueue queue = new MessageQueue();
        ExecutorService executor = Executors.newFixedThreadPool(5);

        // Start 2 Producers and 3 Consumers
        executor.execute(new Producer(queue));
        executor.execute(new Producer(queue));
        executor.execute(new Consumer(queue));
        executor.execute(new Consumer(queue));
        executor.execute(new Consumer(queue));

        // In a real app, you'd have a graceful shutdown mechanism.
        // For this demo, we let it run for a while.
        // try { Thread.sleep(10000); } catch (Exception e) {}
        // executor.shutdownNow();
    }
}
```

---

## ✅ Checkpoint: Did You Master This?
- [ ] Producer fast ga unte, and Consumer slow ga unte, `BlockingQueue` ee problem ni ela solve chestundi?
- [ ] `put()` ki `offer()` ki teda enti?
- [ ] `take()` ki `poll()` ki teda enti?
- [ ] Unbounded `LinkedBlockingQueue` use cheyadam enduku dangerous?

**✅ Ready?** → [Next: Advanced Queues and Deques](./04-Advanced-Queues-and-Deques.md)
