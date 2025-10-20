# 🏗️ Mini Project: Log Processing Pipeline

## 🤔 Problem Statement
Mawa, manam ee project lo, oka multi-stage log processing pipeline ni build cheddam. Idi chala real-world scenario, data engineering and backend systems lo common ga kanipistundi.
- **Stage 1 (Producers):** Multiple "source" threads high volume lo log messages ni generate chestayi.
- **Stage 2 (Queue):** Ee messages anni oka central, thread-safe `BlockingQueue` lo ki veltayi.
- **Stage 3 (Consumers):** Multiple "processor" threads aa queue nunchi messages ni teskuni, "process" (e.g., parse, enrich) chestayi.

**This Project Uses Concepts From:**
- ✅ [8.3: Blocking Queues for Producer-Consumer](../03-Blocking-Queues-for-Producer-Consumer.md) (`ArrayBlockingQueue`)
- ✅ [6.3: Custom ThreadPoolExecutor](../../06-Executor-Framework/03-ThreadPoolExecutor-Deep-Dive.md) (to manage our producer and consumer threads)
- ✅ [5.1: Atomic Variables](../../05-Atomic-Operations/01-Atomic-Variables.md) (`AtomicInteger` for metrics)

---

## 🏗️ Architecture
Manam rendu `ExecutorService`s create chestam: okati producers kosam, inkoti consumers kosam. Renditi madhyalo, manam oka `ArrayBlockingQueue` ni shared buffer laaga pedtam. Producers tasks generate chesi queue lo `put` chestayi. Consumers tasks queue nunchi `take` chesi process chestayi. `BlockingQueue` anedi automatically producers and consumers ni coordinate chestundi.

```mermaid
graph TD
    subgraph Producer Pool (2 Threads)
        P1[Producer 1] --> Q;
        P2[Producer 2] --> Q;
    end

    subgraph Shared Buffer
        Q[ArrayBlockingQueue (Capacity: 100)];
    end

    subgraph Consumer Pool (4 Threads)
        Q --> C1[Consumer 1];
        Q --> C2[Consumer 2];
        Q --> C3[Consumer 3];
        Q --> C4[Consumer 4];
    end

    style Shared Buffer fill:#FFD700
```

---

## 💻 Complete Code

#### File 1: `LogProcessingPipeline.java` (Main Class)
```java
// File: src/com/example/pipeline/LogProcessingPipeline.java
package com.example.pipeline;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class LogProcessingPipeline {

    private static final int QUEUE_CAPACITY = 100;
    private static final int NUM_PRODUCERS = 2;
    private static final int NUM_CONSUMERS = 4;
    private static final int MESSAGES_PER_PRODUCER = 500;
    // Special message to signal the end of production
    private static final String POISON_PILL = "END_OF_MESSAGES";

    public static void main(String[] args) throws InterruptedException {
        // ✅ The shared, bounded queue
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(QUEUE_CAPACITY);

        // ✅ Custom executors for producers and consumers
        ExecutorService producerExecutor = Executors.newFixedThreadPool(NUM_PRODUCERS);
        ExecutorService consumerExecutor = Executors.newFixedThreadPool(NUM_CONSUMERS);

        // Metrics
        AtomicInteger producedCount = new AtomicInteger(0);
        AtomicInteger consumedCount = new AtomicInteger(0);

        // 🚀 Start Producers
        for (int i = 0; i < NUM_PRODUCERS; i++) {
            producerExecutor.execute(new LogProducer(queue, MESSAGES_PER_PRODUCER, producedCount));
        }

        // 🚀 Start Consumers
        for (int i = 0; i < NUM_CONSUMERS; i++) {
            consumerExecutor.execute(new LogConsumer(queue, consumedCount));
        }

        // --- Graceful Shutdown Logic ---
        // 1. Stop accepting new tasks in the producer pool
        producerExecutor.shutdown();
        // 2. Wait for all producers to finish generating messages
        producerExecutor.awaitTermination(5, TimeUnit.MINUTES);

        // 3. Now that all messages are produced, signal consumers to stop
        //    by sending one "poison pill" for each consumer.
        System.out.println("All producers finished. Sending poison pills to consumers...");
        for (int i = 0; i < NUM_CONSUMERS; i++) {
            queue.put(POISON_PILL);
        }

        // 4. Stop accepting new tasks in the consumer pool
        consumerExecutor.shutdown();
        // 5. Wait for all consumers to finish processing the remaining messages
        consumerExecutor.awaitTermination(5, TimeUnit.MINUTES);

        System.out.println("\n🎉 Pipeline finished.");
        System.out.println("Total messages produced: " + producedCount.get());
        System.out.println("Total messages consumed: " + consumedCount.get());
    }

    // --- Producer Task ---
    static class LogProducer implements Runnable {
        private final BlockingQueue<String> queue;
        private final int messageCount;
        private final AtomicInteger producedCount;

        LogProducer(BlockingQueue<String> queue, int messageCount, AtomicInteger producedCount) {
            this.queue = queue;
            this.messageCount = messageCount;
            this.producedCount = producedCount;
        }

        @Override
        public void run() {
            try {
                for (int i = 0; i < messageCount; i++) {
                    String log = "Log-" + Thread.currentThread().getName() + "-" + i;
                    queue.put(log);
                    producedCount.incrementAndGet();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    // --- Consumer Task ---
    static class LogConsumer implements Runnable {
        private final BlockingQueue<String> queue;
        private final AtomicInteger consumedCount;

        LogConsumer(BlockingQueue<String> queue, AtomicInteger consumedCount) {
            this.queue = queue;
            this.consumedCount = consumedCount;
        }

        @Override
        public void run() {
            try {
                while (true) {
                    String log = queue.take();
                    // Check for the poison pill to stop
                    if (POISON_PILL.equals(log)) {
                        System.out.println("Consumer " + Thread.currentThread().getName() + " received poison pill. Exiting.");
                        break; // Exit the loop
                    }

                    // Simulate processing
                    Thread.sleep(10);
                    consumedCount.incrementAndGet();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

---

## 🚀 How to Run & Test

### Step 1: Compile the Code
```bash
# Assuming you are in the project root
javac -d out src/com/example/pipeline/*.java
```

### Step 2: Run the Pipeline
```bash
java -cp out com.example.pipeline.LogProcessingPipeline
```

### Expected Output & Analysis
You'll see a lot of interleaved output from producers and consumers. The important part is the end:
```
...
All producers finished. Sending poison pills to consumers...
Consumer pool-2-thread-1 received poison pill. Exiting.
Consumer pool-2-thread-2 received poison pill. Exiting.
Consumer pool-2-thread-3 received poison pill. Exiting.
Consumer pool-2-thread-4 received poison pill. Exiting.

🎉 Pipeline finished.
Total messages produced: 1000
Total messages consumed: 1000
```
**Analysis:**
- **Decoupling:** Producers ki consumers gurinchi teliyadu, consumers ki producers gurinchi teliyadu. Vaatiki kevalam `BlockingQueue` gurinchi matrame telusu. This is great for modularity.
- **Back-Pressure:** Okaవేళ consumers slow ga unte (e.g., `Thread.sleep(100)` pedithe), queue antha nindipotundi. Appudu producers `queue.put(log)` deggara automatically block avtayi. System antha crash avvadu. Ide **back-pressure**.
- **Graceful Shutdown:** "Poison Pill" anedi oka common pattern to signal consumers that there is no more data to process. Simple ga `shutdownNow()` call cheste, queue lo unna messages anni lost avtayi. Ee approach tho, manam anni messages process ayyayi ani guarantee cheyochu.

Ee project tho, nuvvu a complete, robust, and production-style Producer-Consumer system ni `BlockingQueue` tho ela build cheyalo nerchukunnav. Excellent work, mawa!
