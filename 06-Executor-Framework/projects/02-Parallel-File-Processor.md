# 🏗️ Mini Project 2: Parallel File Processor

## 🤔 Problem Statement
Mawa, manam ee project lo, oka directory lo unna multiple text files ni concurrently process cheddam. Prathi file lo enni lines unnayo count chesi, total number of lines ni calculate cheyali.

**Requirements:**
1.  Program oka directory path ni input ga teskovali.
2.  Aa directory lo unna `.txt` files anni process cheyali.
3.  File line counting anedi I/O-bound operation kabatti, manam custom `ThreadPoolExecutor` create chesi, daanni ee task kosam tune cheddam.
4.  Prathi file processing result ni collect chesi, final ga total line count ni print cheyali. Ee result collection anedi thread-safe ga undali.
5.  Tasks nunchi results (`Integer` line count) return avtayi kabatti, `Runnable` badulu `Callable` vadali.

**This Project Uses Concepts From:**
- ✅ [6.3: ThreadPoolExecutor Deep Dive](../03-ThreadPoolExecutor-Deep-Dive.md)
- ✅ [5.1: Atomic Variables](../../05-Atomic-Operations/01-Atomic-Variables.md) (`AtomicInteger` for thread-safe counting)
- ✅ [7.1: Callable and Future](../..//07-Callable-Future-CompletableFuture/01-Callable-and-Future.md) (briefly, as a preview)

---

## 🏗️ Architecture
Main thread anedi input directory ni scan chesi, `.txt` files list ni prepare chestundi. Tarvata, adi oka custom `ThreadPoolExecutor` ni create chestundi. Prathi file ki, oka `LineCounterTask` (`Callable`) create chesi, pool ki submit chestundi. Ee `submit` method oka `Future` object ni return chestundi. Manam ee `Future` objects ni oka list lo store cheskuntam. Anni tasks submit chesina tarvata, manam aa `Future` objects meeda loop chestu, `future.get()` call chesi, prathi task result ni collect chesi, total sum calculate chestam.

```mermaid
graph TD
    subgraph Main Thread
        A[Get Directory Path] --> B[Scan for .txt files];
        B --> C[Create Custom ThreadPoolExecutor];
        C --> D{Loop through each file};
        D --> E[Create LineCounterTask(file)];
        E --> F[Future<Integer> f = executor.submit(task)];
        F --> G[Add f to List<Future>];
        G --> D;
        D -- Loop End --> H[Shutdown Executor];
        H --> I{Loop through List<Future>};
        I --> J[Integer count = future.get()];
        J --> K[Add count to total];
        K --> I;
        I -- Loop End --> L[Print Total];
    end

    subgraph Thread Pool
        Worker1[Worker Thread 1]
        Worker2[...]
    end

    F ==> Worker1;

    style Main Thread fill:#87CEEB
    style Thread Pool fill:#90EE90
```

---

## 📁 Folder Structure
```
file-processor/
├── input-files/
│   ├── file1.txt
│   ├── file2.txt
│   └── non-text-file.tmp
└── src/
    └── com/
        └── example/
            ├── ParallelFileProcessor.java
            └── LineCounterTask.java
```

---

## 💻 Complete Code

#### File 1: `ParallelFileProcessor.java`
Main class. Directory ni scan chestundi, thread pool ni setup chestundi, tasks ni submit chesi, results ni aggregate chestundi.
```java
// File: src/com/example/ParallelFileProcessor.java
package com.example;

import java.io.File;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

public class ParallelFileProcessor {

    public static void main(String[] args) {
        // Step 1: Setup - Create a directory and some sample files
        setupSampleFiles();

        String directoryPath = "input-files";
        File directory = new File(directoryPath);
        File[] files = directory.listFiles((dir, name) -> name.endsWith(".txt"));

        if (files == null || files.length == 0) {
            System.out.println("No .txt files found in the directory.");
            return;
        }

        // Step 2: ✅ Create a custom ThreadPoolExecutor for I/O tasks
        // corePoolSize = 2, maxPoolSize = 4. Bounded queue. CallerRunsPolicy for back-pressure.
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                2, 4, 60L, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(10),
                Executors.defaultThreadFactory(), // Simple factory for this example
                new ThreadPoolExecutor.CallerRunsPolicy()
        );

        // Step 3: 🎯 Submit tasks and collect Futures
        List<Future<Integer>> results = new ArrayList<>();
        for (File file : files) {
            Callable<Integer> task = new LineCounterTask(file);
            Future<Integer> future = executor.submit(task);
            results.add(future);
        }

        // Step 4: 🛑 Shutdown the executor
        executor.shutdown();
        try {
            // Wait for all tasks to complete
            if (!executor.awaitTermination(5, TimeUnit.MINUTES)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }

        // Step 5: 📊 Aggregate results
        int totalLines = 0;
        for (Future<Integer> future : results) {
            try {
                // future.get() is a blocking call. It waits for the task to complete.
                totalLines += future.get();
            } catch (InterruptedException | ExecutionException e) {
                System.err.println("❌ Error retrieving result from a task: " + e.getMessage());
            }
        }

        System.out.println("\n🎉 Total number of lines in all files: " + totalLines);
    }

    // Helper method to create dummy files for the project
    private static void setupSampleFiles() {
        File dir = new File("input-files");
        if (!dir.exists()) dir.mkdirs();
        try {
            new File("input-files/file1.txt").createNewFile();
            java.nio.file.Files.write(new File("input-files/file1.txt").toPath(), "line1\nline2\nline3".getBytes());
            new File("input-files/file2.txt").createNewFile();
            java.nio.file.Files.write(new File("input-files/file2.txt").toPath(), "lineA\nlineB\nlineC\nlineD".getBytes());
            new File("input-files/non-text-file.tmp").createNewFile();
        } catch (Exception e) { e.printStackTrace(); }
    }
}
```

#### File 2: `LineCounterTask.java`
Idi `Callable` task. Oka file teskuni, daantlo enni lines unnayo count chesi, aa number ni return chestundi.
```java
// File: src/com/example/LineCounterTask.java
package com.example;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileReader;
import java.io.IOException;
import java.util.concurrent.Callable;

// ✅ Implement Callable<Integer> because we want to return an integer (the line count).
public class LineCounterTask implements Callable<Integer> {

    private final File file;

    public LineCounterTask(File file) {
        this.file = file;
    }

    @Override
    public Integer call() throws Exception {
        System.out.println("... Processing file '" + file.getName() + "' on thread: " + Thread.currentThread().getName());
        int lineCount = 0;
        try (BufferedReader reader = new BufferedReader(new FileReader(file))) {
            while (reader.readLine() != null) {
                lineCount++;
            }
        } catch (IOException e) {
            System.err.println("❌ Could not read file " + file.getName());
            // Re-throw exception to be caught by Future.get()
            throw new Exception("Error processing file: " + file.getName(), e);
        }

        // Simulate some I/O delay
        Thread.sleep(500);

        System.out.println("... Finished file '" + file.getName() + "', found " + lineCount + " lines.");
        return lineCount;
    }
}
```

---

## 🚀 How to Run & Test

### Step 1: Compile the Code
`file-processor` root directory nunchi compile chey.
```bash
# Navigate to the project root
cd file-processor

# Compile all Java files into the 'out' directory
javac -d out src/com/example/*.java
```

### Step 2: Run the Processor
```bash
# Run from the 'out' directory
java -cp out com.example.ParallelFileProcessor
```

### Expected Output
```
... Processing file 'file1.txt' on thread: pool-1-thread-1
... Processing file 'file2.txt' on thread: pool-1-thread-2
... Finished file 'file1.txt', found 3 lines.
... Finished file 'file2.txt', found 4 lines.

🎉 Total number of lines in all files: 7
```
Nee output lo thread names veru undochu, kaani concept adhe: rendu files ni veru veru threads parallel ga process chesayi.

Ee project tho, nuvvu custom `ThreadPoolExecutor` ni, `Callable` tasks ni, and `Future` results ni ela use cheyalo nerchukunnav. Super, mawa! 🔥
