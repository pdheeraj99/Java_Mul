---
# 🎯 11.3: Structured Concurrency

## 🗺️ Learning Path Position

**📍 You Are Here:** 11.3 - Structured Concurrency

**✅ Prerequisites (Must Complete First):**
- [11.2: Virtual Threads - Project Loom](./02-Virtual-Threads-Project-Loom.md) - Structured concurrency anedi virtual threads tho kalisi chala powerful ga untundi.

**🔜 Coming After This:**
- [Project: High-Throughput Virtual Thread Server](./projects/01-Virtual-Thread-Server.md) - Ee modern concepts ni use chesi, oka hands-on project build cheddam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 🤔 What & Why

### The Problem: Unstructured Concurrency is Chaos! 🤯
Mawa, manam `ExecutorService` or `CompletableFuture` use chesinappudu, manam tasks ni submit chestam, avi background lo run avtayi. Kaani, main thread ki and child threads ki madhyalo unna relationship chala loose ga untundi.
- **Error Handling is Hard:** Oka child thread fail aite, main thread ki direct ga teliyadu. Maname `Future.get()` tho check cheyali or `CompletableFuture` lo `exceptionally()` lanti handlers pettali. Multiple threads unte, idi inka complex avtundi.
- **Cancellation is Messy:** Oka task ni cancel chesina, daani child tasks ni kuda maname manually cancel cheyali (`shutdownNow`, `Future.cancel()`). Ee "thread leakage" anedi common problem.
- **Reasoning is Difficult:** Code chusthe, thread boundaries clear ga teliyavu. Oka method lo start ayina thread, aa method aypoyaka kuda run avtune undochu.

### The Solution: Structured Concurrency - Treating Threads like a Family 👨‍👩‍👧‍👦
Structured Concurrency (a preview feature in Java 19+) ee problem ni solve chestundi. Idi simple `try-catch` block laanti structure ni concurrency ki testundi.
- **Core Idea:** "If a task splits into concurrent subtasks, they all return to the same place." Child tasks yokka lifetime anedi parent task yokka scope lo ne untundi.
- **How it Works:** Manam `StructuredTaskScope` ni `try-with-resources` block lo create chestam.
  - Ee scope loపల, manam sub-tasks ni `fork()` chestam.
  - `scope.join()` call chesi, anni sub-tasks complete ayyevaraku wait chestam.
  - `try` block exit avvagane, ee scope automatically `close` avtundi. Close ayye mundu, inka running lo unna child threads anni automatically interrupt avtayi. No thread leakage!

### Real-World Analogy: Head Chef and Junior Chefs 👨‍🍳
- **Unstructured Concurrency:** Head chef junior chefs ki panulu cheppi, intiki vellipotadu. Oka junior chef ki ingredient dorakka pani aapesthe, head chef ki teliyadu. Inko junior chef pani aypoyaka kuda, kitchen lo ne tirugutu untadu. Chaos.
- **Structured Concurrency:** Head chef (`StructuredTaskScope`) junior chefs (`forked tasks`) ki panulu cheptadu. "Andari panulu ayyevarku nenu ikkade unta" ani `scope.join()` deggara wait chestadu.
  - **Success:** Andaru panulu complete cheste, head chef "Good job" cheppi, andarni intiki pampistadu.
  - **Failure:** Oka junior chef ("Naaku salt dorakatledu!") ani fail aite, head chef ventane "EVERYONE, STOP!" ani migatha chefs andari panulu aapestadu (`scope.close()` on failure). Antha clean and predictable.

---

## 📚 Detailed Explanation: `StructuredTaskScope`

`StructuredTaskScope` anedi an incubator module lo undi, so nuvvu command line lo `--enable-preview --add-modules jdk.incubator.concurrent` lanti flags add cheyali.

### Key Policies:

1.  **`ShutdownOnFailure`:**
    - Idi default policy. Ee scope lo, edoka child task fail aite, scope antha shutdown aypotundi and inka running lo unna migatha child tasks anni cancel avtayi.
    - **Use Case:** "Andaru success aite ne naku final result kavali." (e.g., API aggregator lo, oka API fail aite, antha operation fail).

2.  **`ShutdownOnSuccess`:**
    - Ee scope lo, edoka child task successfully complete aite, scope antha shutdown aypotundi and migatha child tasks anni cancel avtayi.
    - **Use Case:** "Evaru first result istaro, adi teskuni munduku vellipotha." (e.g., multiple servers nunchi okate data kosam query cheyadam).

---

## 💻 Code Example

### Example 1: `ShutdownOnFailure` for an "All or Nothing" Task
**Scenario:** Rendu critical, independent operations (`operationA`, `operationB`) unnayi. Rendu success aite ne, manam proceed avvali. Okati fail aite, inkoti kuda aagipovali.
```java
// File: com/example/structured/ShutdownOnFailureDemo.java
// NOTE: To run this, you need JDK 19+ with preview features enabled.
// javac --release 19 --enable-preview --add-modules jdk.incubator.concurrent ...
// java --enable-preview --add-modules jdk.incubator.concurrent ...
package com.example.structured;

import java.util.concurrent.StructuredTaskScope;
import java.util.concurrent.Future;

public class ShutdownOnFailureDemo {

    public static void main(String[] args) throws InterruptedException {
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            System.out.println("Forking tasks...");

            Future<String> futureA = scope.fork(() -> operationA());
            Future<Integer> futureB = scope.fork(() -> operationB());

            System.out.println("Joining tasks...");
            scope.join(); // Wait for both to complete or one to fail

            // If one failed, this line throws an exception
            scope.throwIfFailed();

            // If we are here, both succeeded
            System.out.println("Both tasks succeeded!");
            System.out.println("Result A: " + futureA.resultNow());
            System.out.println("Result B: " + futureB.resultNow());

        } catch (Exception e) {
            System.err.println("One of the tasks failed: " + e.getMessage());
        }
    }

    static String operationA() throws InterruptedException {
        System.out.println("  -> Starting op A...");
        Thread.sleep(1000);
        System.out.println("  <- Finished op A.");
        return "ResultA";
    }

    static Integer operationB() throws InterruptedException {
        System.out.println("  -> Starting op B (will fail)...");
        Thread.sleep(500);
        throw new RuntimeException("Operation B failed!");
    }
}
```

### Example 2: `ShutdownOnSuccess` to Find the First Result
**Scenario:** Rendu servers nunchi weather data fetch cheyali. Evaru first istaro, aa data teskovali.
```java
// File: com/example/structured/ShutdownOnSuccessDemo.java
package com.example.structured;

import java.util.concurrent.StructuredTaskScope;
import java.util.function.Supplier;

public class ShutdownOnSuccessDemo {
    public static void main(String[] args) throws InterruptedException {
        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {

            scope.fork(() -> fetchFromServer("ServerA", 2000));
            scope.fork(() -> fetchFromServer("ServerB", 1000));

            scope.join(); // Wait for the first one to succeed

            // Get the result of the first successful task
            System.out.println("Fastest server responded: " + scope.result());
        } catch (Exception e) {
            System.err.println("All tasks failed: " + e.getMessage());
        }
    }

    static String fetchFromServer(String name, long delay) throws InterruptedException {
        System.out.println("  -> Querying " + name);
        Thread.sleep(delay);
        System.out.println("  <- " + name + " responded!");
        return name + " Data";
    }
}
```

---

## ✅ Checkpoint: Did You Master This?
- [ ] Unstructured concurrency lo unna 3 main problems enti?
- [ ] `StructuredTaskScope` anedi thread leakage ni ela prevent chestundi?
- [ ] `ShutdownOnFailure` ki `ShutdownOnSuccess` ki teda enti? Oka real-world example cheppu.
- [ ] `scope.join()` call chesina tarvata, `scope.throwIfFailed()` enduku call cheyali?

**✅ Ready?** → [Project: High-Throughput Virtual Thread Server](./projects/01-Virtual-Thread-Server.md)
