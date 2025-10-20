---
# 🎯 8.4: Advanced Queues and Deques

## 🗺️ Learning Path Position

**📍 You Are Here:** 8.4 - Advanced Queues and Deques

**✅ Prerequisites (Must Complete First):**
- [8.3: Blocking Queues for Producer-Consumer](./03-Blocking-Queues-for-Producer-Consumer.md) - `ArrayBlockingQueue` and `LinkedBlockingQueue` gurinchi clear ga teliyali.

**🔜 Coming After This:**
- [Project: Log Processing Pipeline](./projects/01-Log-Processing-Pipeline.md) - `BlockingQueue` concept ni use chesi, oka hands-on project build cheddam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 🤔 What & Why

### The Problem: Simple FIFO Queues Aren't Always Enough
Mawa, `ArrayBlockingQueue` and `LinkedBlockingQueue` anevi FIFO (First-In, First-Out) order lo pani chestayi. Vachina message ni vachinattu process chestayi. Kaani, manaki ilanti special requirements unte:
- "High-priority alerts ni normal log messages kanna mundu process cheyali."
- "Oka task ni correct ga 5 minutes tarvata run cheyali, అంత ముందు kadu."
- "Producer and Consumer madhyalo zero buffer undali. Producer data ichinappude consumer teskovali, leda producer wait cheyali."
Ee scenarios ki regular queues saripovu.

### The Solution: Specialized Queues for Special Jobs ✨

#### 1. `PriorityBlockingQueue<E>`
- **Core Idea:** An unbounded blocking queue that uses ordering based on priority.
- **How it Works:** Elements anevi `Comparable` interface ni implement cheyali, or manam `Comparator` ni constructor lo pass cheyali. `take()` method eppudu **highest priority** unna element ni return chestundi.
- **When to Use:** Task processing lo priority aadharanga scheduling cheyali anukunnappudu. (e.g., handling emergency alerts over regular notifications).

#### 2. `DelayQueue<E extends Delayed>`
- **Core Idea:** An unbounded blocking queue where elements can only be taken when their delay has expired.
- **How it Works:** Queue lo pette prathi element `Delayed` interface ni implement cheyali. Ee interface lo `getDelay(TimeUnit)` ane method untundi. Oka element yokka delay zero or less aite tappa, `take()` method daanini return cheyadu.
- **When to Use:** Caching systems (e.g., expire items after 30 minutes), scheduling temporary resource cleanup.

#### 3. `SynchronousQueue<E>`
- **Core Idea:** A queue with **zero capacity**. It's a handoff mechanism.
- **How it Works:** `put()` call anedi, inko thread `take()` call chesevaraku block avtundi. Vice-versa, `take()` call anedi, inko thread `put()` call chesevaraku block avtundi.
- **When to Use:** `Executors.newCachedThreadPool()` deenine internally use chestundi. Producer and Consumer threads ni tightly couple cheyali anukunnapudu. Chala high-performance, low-latency scenarios lo use avtundi.

#### 4. `ConcurrentLinkedDeque<E>`
- **Core Idea:** A thread-safe, unbounded **Deque** (Double-Ended Queue).
- **How it Works:** Idi `BlockingQueue` kadu! Queue empty ga unte `poll()` `null` istundi, kaani `take()` laaga block avvadu. Elements ni rendu ends (first and last) nunchi add and remove cheyochu.
- **When to Use:** Work-stealing algorithms lo. Oka thread (thief) inko thread (victim) yokka queue nunchi "steal" cheskovadaniki. Thief anedi victim yokka task list lo last nunchi (tail) teskuntundi, victim anedi front nunchi (head) teskuntundi. Deenivalla contention taggutundi.

---

## 💻 Code Examples

### Example 1: `PriorityBlockingQueue` for Prioritized Tasks
```java
// File: com/example/advanced/PriorityQueueDemo.java
package com.example.advanced;

import java.util.concurrent.PriorityBlockingQueue;

// Task class that implements Comparable for priority
class PrioritizedTask implements Comparable<PrioritizedTask> {
    private final int priority;
    private final String name;
    public PrioritizedTask(String name, int priority) { this.name = name; this.priority = priority; }
    public String getName() { return name; }

    @Override
    public int compareTo(PrioritizedTask other) {
        // Higher number = higher priority
        return Integer.compare(other.priority, this.priority);
    }
}

public class PriorityQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        PriorityBlockingQueue<PrioritizedTask> queue = new PriorityBlockingQueue<>();

        // Add tasks with different priorities
        queue.put(new PrioritizedTask("Regular Task", 1));
        queue.put(new PrioritizedTask("Critical Alert", 10));
        queue.put(new PrioritizedTask("Important Task", 5));

        // Elements are taken out based on priority, not insertion order
        System.out.println(queue.take().getName()); // Critical Alert
        System.out.println(queue.take().getName()); // Important Task
        System.out.println(queue.take().getName()); // Regular Task
    }
}
```

### Example 2: `DelayQueue` for a Simple Cache
```java
// File: com/example/advanced/DelayQueueDemo.java
package com.example.advanced;

import java.util.concurrent.DelayQueue;
import java.util.concurrent.Delayed;
import java.util.concurrent.TimeUnit;

class DelayedCacheItem implements Delayed {
    private final String data;
    private final long expiryTime;

    public DelayedCacheItem(String data, long delayMs) {
        this.data = data;
        this.expiryTime = System.currentTimeMillis() + delayMs;
    }
    public String getData() { return data; }

    @Override
    public long getDelay(TimeUnit unit) {
        long diff = expiryTime - System.currentTimeMillis();
        return unit.convert(diff, TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        return Long.compare(this.expiryTime, ((DelayedCacheItem) o).expiryTime);
    }
}

public class DelayQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        DelayQueue<DelayedCacheItem> cache = new DelayQueue<>();

        cache.put(new DelayedCacheItem("Item-1", 3000)); // Expires in 3s
        cache.put(new DelayedCacheItem("Item-2", 1000)); // Expires in 1s

        System.out.println("Taking from cache at " + System.currentTimeMillis());
        // take() will block until the item with the nearest expiry time is ready
        System.out.println("Got: " + cache.take().getData() + " at " + System.currentTimeMillis()); // Item-2
        System.out.println("Got: " + cache.take().getData() + " at " + System.currentTimeMillis()); // Item-1
    }
}
```

### Example 3: `SynchronousQueue` for a Direct Handoff
```java
// File: com/example/advanced/SynchronousQueueDemo.java
package com.example.advanced;

import java.util.concurrent.SynchronousQueue;

public class SynchronousQueueDemo {
    public static void main(String[] args) {
        SynchronousQueue<String> handoffQueue = new SynchronousQueue<>();

        // Consumer thread - tries to take, but blocks
        new Thread(() -> {
            try {
                System.out.println("Consumer waiting to take...");
                String item = handoffQueue.take();
                System.out.println("Consumer took: " + item);
            } catch (InterruptedException e) {}
        }).start();

        // Producer thread - puts an item, which unblocks the consumer
        new Thread(() -> {
            try {
                Thread.sleep(2000);
                System.out.println("Producer putting item...");
                handoffQueue.put("Handoff Item");
                System.out.println("Producer put successful.");
            } catch (InterruptedException e) {}
        }).start();
    }
}
```

---

## ✅ Checkpoint: Did You Master This?
- [ ] `PriorityBlockingQueue` lo elements ni ela order chestam?
- [ ] `DelayQueue` lo element `getDelay` method negative value return cheste em avtundi?
- [ ] `SynchronousQueue` capacity enta? Ye scenario lo idi useful?
- [ ] `ConcurrentLinkedDeque` anedi `BlockingQueue` na? Kaada? Why?

**✅ Ready?** → [Project: Log Processing Pipeline](./projects/01-Log-Processing-Pipeline.md)
