---
(Previous sections remain unchanged)
---
### Example 3: `synchronized` Blocks (For Better Performance)

Method antha `synchronized` cheste, adi easy, kaani konni sarlu performance issue avtundi. Method lo 100 lines unte, andulo 2 lines matrame critical section anuko. Appudu aa 2 lines ke lock veyali, antha method ki kadu. Appudu manam `synchronized` block use chestam.

#### ✅ Success Case: Fine-Grained Locking
**Scenario:** Oka method lo file download chesi, counter ni update cheyali. File download anedi non-critical, so daaniki lock avasaram ledu.

```java
// File: com/example/finegrained/FineGrainedCounter.java
package com.example.finegrained;

public class FineGrainedCounter {
    private int count = 0;
    // ✅ Best Practice: Use a dedicated, private, final object for locking.
    private final Object lock = new Object();

    public void performDownloadAndUpdate() {
        System.out.println(Thread.currentThread().getName() + ": Starting a long, non-critical task (e.g., file download)...");
        // Simulate a long-running task that doesn't need a lock.
        try {
            Thread.sleep(2000); // Represents downloading a file, etc.
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println(Thread.currentThread().getName() + ": Non-critical task complete.");

        // Critical section is only this small block
        synchronized (lock) {
            System.out.println(Thread.currentThread().getName() + ": Acquired lock, updating the counter.");
            count++;
        } // Lock is automatically released here
    }

    public int getCount() {
        // Optional: you might want to synchronize read access too for visibility
        synchronized(lock) {
            return count;
        }
    }
}

// File: com/example/finegrained/Main.java
package com.example.finegrained;

public class Main {
    public static void main(String[] args) throws InterruptedException {
        FineGrainedCounter counter = new FineGrainedCounter();

        // Create two threads that will perform the operation
        Thread t1 = new Thread(() -> {
            counter.performDownloadAndUpdate();
        }, "Downloader-1");

        Thread t2 = new Thread(() -> {
            counter.performDownloadAndUpdate();
        }, "Downloader-2");

        long startTime = System.currentTimeMillis();
        t1.start();
        t2.start();

        t1.join();
        t2.join();
        long endTime = System.currentTimeMillis();

        System.out.println("Final count: " + counter.getCount());
        System.out.println("Total time taken: " + (endTime - startTime) + "ms");
    }
}
```
**Output:**
```
Downloader-1: Starting a long, non-critical task (e.g., file download)...
Downloader-2: Starting a long, non-critical task (e.g., file download)...
// (Both threads wait for 2 seconds in parallel)
Downloader-1: Non-critical task complete.
Downloader-1: Acquired lock, updating the counter.
Downloader-2: Non-critical task complete.
// (Downloader-2 now waits for the lock)
// (Downloader-1 releases the lock)
Downloader-2: Acquired lock, updating the counter.
Final count: 2
Total time taken: approx 20XXms
```
**Why This is Better:** The total time taken is close to 2 seconds, not 4 seconds. Endukante, rendu threads `sleep(2000)` ni parallel ga execute chesayi. Lock kosam just chivara lo konchem time matrame wait chesayi. Method antha `synchronized` chesi unte, total time 4 seconds ayyedi. Idi performance ki pedda boost!

---
(Remaining content unchanged)
---
