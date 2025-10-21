<!--
---
title: "Mini-Project: Parallel File System Counter"
---
-->

> **Learning Path Position**
>
> Phase 10: Fork/Join Framework ➔ **Hands-On Mini-Project: Parallel File System Counter**

> **Project Goal**
>
> Manam oka application build cheddam, adi oka pedda directory structure lo unna total number of files ni parallel ga count chestundi. Idi Fork/Join framework ki oka classic, real-world use case. Oka directory ni scan cheyyadam anedi naturally recursive pani (prathi sub-directory ni malli scan cheyyali). Ee recursive nature ni manam Fork/Join tho parallel ga chesi, single-threaded approach tho compare cheste entha performance gain vastundo chuddam.

---

### 🚀 1. Project Overview

Mawa, nee computer lo `C:\Program Files` or `/usr/lib` lanti pedda folder lo enni files unnayo count cheyyali anuko. Okate thread tho cheste, adi prathi sub-folder loki velli, malli daani loni sub-folders loki velli... ilaga chala time padutundi.

Kani ee pani ni manam chala easy ga parallel cheyyochu. Main thread "Program Files" ni chusi, daanilo unna prathi sub-directory ki ("Google", "Microsoft", etc.) oka separate worker thread ki pani assign cheyyochu. Aa worker threads malli vaati kinda unna sub-directories ki inko workers ki pani ivvochu. Ide "divide and conquer". Ee project lo, manam ide logic ni `RecursiveTask` use chesi implement cheddam.

###🎯 2. Learning Objectives

*   `RecursiveTask` ni oka real-world, recursive problem kosam implement cheyyadam.
*   Directory traversal lanti recursive logic ni `compute()` method lo ela rayalo nerchukovadam.
*   Sub-tasks ni create chesi `fork()` cheyyadam and vaati results ni `join()` chesi combine cheyyadam practice cheyyadam.
*   Fork/Join framework valla vache performance improvement ni single-threaded version tho compare chesi chudatam.

### 🏛️ 3. Project Structure

Manam rendu classes create cheddam:

1.  `FileCounterTask.java`: Idi mana `RecursiveTask` implementation. Oka directory ni input ga tesukuni, daanilo unna files count ni return chestundi. Sub-directories unte, vaati kosam kottha sub-tasks ni create chestundi.
2.  `Main.java`: Idi mana driver class. Fork/Join pool ni set up chestundi, main task ni start chestundi, result ni print chestundi, and performance ni sequential version tho compare chestundi.

### 🛠️ 4. Step-by-Step Implementation

Let's start building.

#### Step 1: Create the `FileCounterTask`

Idi mana project lo main logic unna class.

*   Idi `RecursiveTask<Integer>` ni extend chestundi, endukante manam file count (oka integer) ni return chestunnam.
*   Deeni state lo `File` object (directory) untundi.
*   `compute()` method lo, manam directory lo unna items ni loop chestam.
    *   Item `File` aite, counter penchutam.
    *   Item `Directory` aite, daani kosam oka kottha `FileCounterTask` create chesi, daanni `fork()` cheddam.
*   Anni forked sub-tasks ni `join()` chesi, vaati results ni mana local counter ki add chesi, final ga return cheddam.

```java
// FileCounterTask.java
import java.io.File;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.RecursiveTask;

public class FileCounterTask extends RecursiveTask<Integer> {

    private final File directory;

    public FileCounterTask(File directory) {
        this.directory = directory;
    }

    @Override
    protected Integer compute() {
        int fileCount = 0;
        List<FileCounterTask> subTasks = new ArrayList<>();

        // Ee directory lo unna files and sub-directories ni get chesuko
        File[] filesAndDirs = directory.listFiles();
        if (filesAndDirs == null) {
            return 0; // Access lekapothe or empty aite
        }

        for (File item : filesAndDirs) {
            if (item.isFile()) {
                // Idi file aite, counter penchu
                fileCount++;
            } else if (item.isDirectory()) {
                // Idi directory aite, deeni kosam oka kottha sub-task create cheyyi
                FileCounterTask subTask = new FileCounterTask(item);
                // Daanni asynchronus ga run cheyyadaniki fork cheyyi
                subTask.fork();
                // Taruvatha join cheyyadaniki list lo pettuko
                subTasks.add(subTask);
            }
        }

        // Anni sub-tasks results kosam wait chesi, vaatini kalupu
        for (FileCounterTask task : subTasks) {
            fileCount += task.join(); // join() anedi blocking call
        }

        return fileCount;
    }
}
```

#### Step 2: Create the `Main` Class

Ee class antha set up chesi, parallel and sequential versions ni run chesi compare chestundi.

```java
// Main.java
import java.io.File;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.TimeUnit;

public class Main {

    public static void main(String[] args) {
        // Nee computer lo oka pedda directory path ivvu
        // Example: Windows lo "C:/Program Files", Mac/Linux lo "/usr/lib"
        String directoryPath = "."; // Current project directory tho start cheddam
        File startDir = new File(directoryPath);

        System.out.println("Counting files in: " + startDir.getAbsolutePath());
        System.out.println("------------------------------------------");

        // --- Parallel Version ---
        ForkJoinPool pool = ForkJoinPool.commonPool();
        FileCounterTask mainTask = new FileCounterTask(startDir);

        long startTime = System.nanoTime();
        int parallelResult = pool.invoke(mainTask);
        long endTime = System.nanoTime();

        long parallelDuration = TimeUnit.NANOSECONDS.toMillis(endTime - startTime);
        System.out.println("Parallel Result: " + parallelResult + " files");
        System.out.println("Time taken (parallel): " + parallelDuration + " ms");

        System.out.println("------------------------------------------");

        // --- Sequential Version ---
        startTime = System.nanoTime();
        int sequentialResult = countFilesSequentially(startDir);
        endTime = System.nanoTime();

        long sequentialDuration = TimeUnit.NANOSECONDS.toMillis(endTime - startTime);
        System.out.println("Sequential Result: " + sequentialResult + " files");
        System.out.println("Time taken (sequential): " + sequentialDuration + " ms");

        pool.shutdown();
    }

    // Simple single-threaded recursive method for comparison
    public static int countFilesSequentially(File directory) {
        int count = 0;
        File[] filesAndDirs = directory.listFiles();
        if (filesAndDirs == null) {
            return 0;
        }
        for (File item : filesAndDirs) {
            if (item.isFile()) {
                count++;
            } else if (item.isDirectory()) {
                count += countFilesSequentially(item);
            }
        }
        return count;
    }
}
```

### 🏃 5. Running the Simulation

Ee program ni run chesinappudu, nuvvu chustav:
1.  Mundu parallel version run ayyi, total file count and tesukunna time ni print chestundi.
2.  Taruvatha, sequential version run ayyi, ade count ni and daani time ni print chestundi.

Okavela nuvvu chala files unna pedda directory (e.g., nee OS installation folder) mida run cheste, parallel version chala fast ga undadam gamanistav. Chinna directories mida aite, task creation overhead valla sequential version fast ga undochu.

**Example Output (oka pedda directory mida):**
```
Counting files in: /usr/lib
------------------------------------------
Parallel Result: 25432 files
Time taken (parallel): 150 ms
------------------------------------------
Sequential Result: 25432 files
Time taken (sequential): 580 ms
```

### 🔑 6. Key Takeaways from the Project

1.  **Natural Fit for Recursion:** Directory traversal lanti recursive problems ki Fork/Join framework chala natural ga fit avutundi. "Divide and conquer" logic ni `compute()` method lo rayadam chala easy ga untundi.
2.  **`fork()` for Parallelism:** `subTask.fork()` anedi magic. Adi pani ni parallel ga cheyyadaniki `ForkJoinPool` ki submit chestundi.
3.  **`join()` for Results:** `task.join()` anedi sub-task result kosam wait cheyyadaniki and daanni tesukovadaniki use avutundi. Idi results ni combine cheyyadaniki chala mukhyam.
4.  **Real Performance Gains:** Ee project correct type of problem (CPU-bound, highly divisible) mida Fork/Join framework entha pedda performance improvement ivvagalado చూపిస్తుంది.
