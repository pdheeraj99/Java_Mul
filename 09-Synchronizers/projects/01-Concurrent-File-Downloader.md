<!--
---
title: "Mini-Project: Concurrent File Downloader"
---
-->

> **Learning Path Position**
>
> Phase 9: Synchronizers ➔ **Hands-On Mini-Project: Concurrent File Downloader**

> **Project Goal**
>
> Manam oka concurrent file downloader ni build cheddam, adi oka list of files ni parallel ga download cheyyagaladu. Ee project `Semaphore` ni use chesukuni simultaneous downloads ni limit chestundi (network mida ekkuva load padakunda) and `CountDownLatch` ni use chesukuni main thread anni downloads aipoye varaku wait chestundi. Idi oka perfect real-world example, complex concurrent task ni orchestrate cheyyadaniki multiple synchronizers ni ela kalapalo chupistundi.

---

### 🚀 1. Project Overview

Mawa, imagine chesuko nuvvu 10 pedda files ni internet nunchi download cheyyali. Okati taruvatha okati cheste chala slow avutundi. Anni 10 okesari cheste, nee internet connection saturate aipovachu and inka slow avvochu, leda server ninnu block cheyyochu.

Optimal solution entante, konni files ni parallel ga download cheyyadam— উদাহরনস্বরূপ, okesari 3. Mana application ade chestundi. Adi download tasks queue ni manage chestundi, kani `Semaphore` use chesi, specified number kanna ekkuva downloads okesari active ga lekunda chusukuntundi. Main application thread anni download tasks ni fire chesi, taruvatha `CountDownLatch` use chesi last download aipoyevaraku efficiently wait chesi, taruvatha shutdown avutundi.

###🎯 2. Learning Objectives

*   Oka practical problem solve cheyyadaniki multiple synchronizers (`Semaphore` and `CountDownLatch`) ni kalapadam.
*   `Semaphore` use chesi oka throttling/rate-limiting mechanism implement cheyyadam.
*   "Master" thread ni multiple "worker" threads tho coordinate cheyyadaniki `CountDownLatch` use cheyyadam.
*   Permits (`Semaphore`) release cheyyadaniki and latches (`CountDownLatch`) count down cheyyadaniki `try...finally` blocks entha mukhyamo reinforce cheyyadam.

### 🏛️ 3. Project Structure

Manam structure ni simple ga unchi, rendu classes matrame create cheddam:

1.  `DownloaderTask.java`: Oka `Runnable`, adi oka single file download ni simulate chestundi. Idi semaphore and latch rendu tho interact avutundi.
2.  `Main.java`: Main application class, idi synchronizers ni setup chestundi, thread pool create chestundi, anni tasks ni submit chestundi, and avi aipoye varaku wait chestundi.

### 🛠️ 4. Step-by-Step Implementation

Let's build it.

#### Step 1: Create the `DownloaderTask`

Ee class core worker. Deeni `run` method oka single download lifecycle ni define chestundi.

1.  Mundu `Semaphore` nunchi oka permit acquire cheyyali. Download capacity max aipothe idi block avvochu.
2.  Permit dorikaka, adi download process ni simulate chestundi.
3.  Critically, adi `try...finally` block ni use chestundi. `release()` and `countDown()` calls **must** be in the `finally` block. Deeni valla download "fail" ayina (exception vachina), permit return avutundi and latch count down avutundi.

```java
// DownloaderTask.java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class DownloaderTask implements Runnable {

    private final String fileUrl;
    private final Semaphore semaphore;
    private final CountDownLatch latch;

    public DownloaderTask(String fileUrl, Semaphore semaphore, CountDownLatch latch) {
        this.fileUrl = fileUrl;
        this.semaphore = semaphore;
        this.latch = latch;
    }

    @Override
    public void run() {
        try {
            // 1. Download start cheyyadaniki oka permit acquire cheyyi. Pool full aite idi block avutundi.
            semaphore.acquire();

            // --- CRITICAL SECTION: DOWNLOAD LOGIC ---
            System.out.println("⬇️ " + fileUrl + " download start avutondi " + Thread.currentThread().getName() + " dwara");
            // Download time ni simulate cheddam
            TimeUnit.SECONDS.sleep(2 + (long)(Math.random() * 3));
            System.out.println("✅ " + fileUrl + " download aipoindi");
            // --- END CRITICAL SECTION ---

        } catch (InterruptedException e) {
            System.err.println("❌ " + fileUrl + " download fail aindi");
            Thread.currentThread().interrupt();
        } finally {
            // 2. Permit ni release cheyyi, so inko thread start avvochu.
            semaphore.release();
            // 3. Ee task aipoindi ani signal ivvadaniki latch ni count down cheyyi.
            latch.countDown();
        }
    }
}
```

#### Step 2: Create the `Main` Class

Ee class orchestrator.

1.  Download cheyyalsina files list ni define chestundi.
2.  Kavalsina concurrency limit tho `Semaphore` create chestundi (e.g., 3).
3.  Total number of files tho `CountDownLatch` create chestundi.
4.  Oka `ExecutorService` (thread pool) create chestundi.
5.  Files loop chesi, prathi file ki oka `DownloaderTask` create chesi, pool ki submit chestundi.
6.  Taruvatha `latch.await()` call chestundi, adi `main` thread ni latch count zero ayye varaku block chestundi.
7.  Final ga, oka completion message print chesi, executor ni shut down chestundi.

```java
// Main.java
import java.util.List;
import java.util.ArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class Main {
    // Maximum number of concurrent downloads
    private static final int MAX_CONCURRENT_DOWNLOADS = 3;

    public static void main(String[] args) {
        List<String> filesToDownload = List.of(
            "file-1.zip", "file-2.zip", "file-3.zip", "file-4.zip", "file-5.zip",
            "file-6.zip", "file-7.zip", "file-8.zip", "file-9.zip", "file-10.zip"
        );

        int totalFiles = filesToDownload.size();

        // Concurrent downloads ni limit cheyyadaniki Semaphore
        Semaphore semaphore = new Semaphore(MAX_CONCURRENT_DOWNLOADS);

        // Anni downloads aipoye varaku wait cheyyadaniki Latch
        CountDownLatch completionLatch = new CountDownLatch(totalFiles);

        ExecutorService executor = Executors.newFixedThreadPool(10);

        System.out.println("Anni " + totalFiles + " download tasks submit chestunnam. Max concurrent: " + MAX_CONCURRENT_DOWNLOADS);

        for (String fileUrl : filesToDownload) {
            executor.submit(new DownloaderTask(fileUrl, semaphore, completionLatch));
        }

        try {
            System.out.println("Main thread ippudu anni downloads aipovadaniki wait chestondi...");
            // Latch count zero ayye varaku block cheyyi
            completionLatch.await();
            System.out.println("\n🎉🎉🎉 Anni files successfully download aipoyayi! 🎉🎉🎉");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.err.println("Main thread interrupt aindi.");
        } finally {
            // Thread pool ni cleanly shut down cheyyi
            executor.shutdown();
            try {
                if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
                    executor.shutdownNow();
                }
            } catch (InterruptedException e) {
                executor.shutdownNow();
            }
        }
    }
}
```

### 🏃 5. Running the Simulation

`Main` class ni run chesinappudu, neeku ilanti output kanipistundi:

```
Submitting all 10 download tasks. Max concurrent: 3
Main thread ippudu anni downloads aipovadaniki wait chestondi...
⬇️ file-1.zip download start avutondi pool-1-thread-1 dwara
⬇️ file-2.zip download start avutondi pool-1-thread-2 dwara
⬇️ file-3.zip download start avutondi pool-1-thread-3 dwara
// <-- Chudu, mundu 3 downloads matrame start ayyayi. Migatha tasks semaphore permit kosam wait chestunnayi. -->
✅ file-1.zip download aipoindi
⬇️ file-4.zip download start avutondi pool-1-thread-4 dwara
// <-- file-1 aipogane, ventane kottha download (file-4) start aindi. -->
✅ file-3.zip download aipoindi
⬇️ file-5.zip download start avutondi pool-1-thread-5 dwara
... (anni files download ayyevaraku ide continue avutundi) ...

🎉🎉🎉 Anni files successfully download aipoyayi! 🎉🎉🎉
```

**Output Analysis:**

*   **Throttling:** Okate sari 3 kanna ekkuva "Starting download" messages ravu. `Semaphore` pani ni correct ga throttle chestondi.
*   **Coordination:** "Main thread is now waiting..." message taruvatha, application aagipotundi. 10th and final file aipoyake "All files have been downloaded" ane message vastundi. `CountDownLatch` main thread ni correct ga wait cheyistondi.

### 🔑 6. Key Takeaways from the Project

1.  **Synergy of Synchronizers:** Ee project synchronizers anevi separate tools kadu ani chupistundi. Avi building blocks, manam vaatini kalipi sophisticated and robust concurrency control logic create cheyyochu.
2.  **Separation of Concerns:**
    *   `Semaphore` **rate limiting** mida concentrate chesindi (ippudu enni threads run avvochu).
    *   `CountDownLatch` **completion** mida concentrate chesindi (anni tasks aipoyaya leda).
3.  **Robust Worker Design:** Worker `Runnable` lo `try...finally` block use chesi cleanup (`release` and `countDown`) guarantee cheyyadam anede ikkada most important lesson. Idi system ni errors nunchi resilient ga chestundi.
