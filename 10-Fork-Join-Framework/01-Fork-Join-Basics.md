<!--
---
title: "Fork/Join Framework: The Basics"
---
-->

> **Learning Path Position**
>
> Phase 10: Fork/Join Framework ➔ **Chunk 1: Fork/Join Basics**

> **Prerequisites**
>
> *   Phase 6: Executor Framework (especially `ThreadPoolExecutor`)
> *   Basic understanding of recursion in programming.

> **Coming After This**
>
> *   Phase 10, Chunk 2: Fork/Join Implementation
> *   Phase 10, Chunk 3: Parallel Streams Internals

---

### 🚀 1. What & Why: Fork/Join Framework?

Mawa, ippativaraku manam `ThreadPoolExecutor` lanti general-purpose tools chusam. Avi chala tasks ki baga paniki vastayi. Kani, okavela nee daggara oka pedda task undi, daanni nuvvu chala chinnachinna pieces ga break cheyyagalanu anuko, malli aa chinna pieces ni inka chinna pieces ga break cheyyochu... ilanti "divide and conquer" problems ki special ga design chesinde ee **Fork/Join Framework**.

Idi Java 7 lo introduce chesaru. Deeni main goal entante, oka pedda recursive task ni parallel ga run chesi, multi-core processors ni full ga utilize chesukovadam.

**Deeni valla use enti? 🤔**

*   **CPU Utilization 💯:** Idle threads ni pani cheyistundi, so CPU cores antha time busy ga untayi, performance penchutundi.
*   **Parallelism for Recursive Tasks 🧑‍💻:** Merge sort lanti recursive algorithms, or oka pedda file system ni scan cheyyadam lanti panulaki parallelism add cheyyadaniki idi perfect.
*   **Foundation for Parallel Streams 🌊:** Java 8 lo vachina Parallel Streams venakala unna engine ide! Deenni ardham chesukunte, parallel streams ela pani chestayo neeku clear ga telustundi.

---

### analogy 2. Real-World Analogy: The Head Chef & Sous Chefs 👨‍🍳

Imagine oka pedda restaurant kitchen lo, Head Chef (nuvvu) ki oka massive order vachindi: "1000 potatoes ni cut cheyyali".

*   **Fork:** Head Chef okkade antha pani cheyyaledu. So, aayana aa 1000 potatoes ni ఇద్దరు Sous Chefs ki 500-500 ga panchi pedatadu. Ee process ni **fork** cheyyadam antaru.
*   **Recursive Decomposition:** Ippudu prathi Sous Chef ki 500 potatoes task undi. Adi kuda ekkuve. So, prathi Sous Chef malli aa pani ni ఇద్దరు Junior Chefs ki 250-250 ga fork chestaru. Idi ilage continue avutundi, oka chef ki cheyyagalige antha chinna task (e.g., 20 potatoes) ayyevaraku.
*   **Join:** Eppaite oka Junior Chef tana 20 potatoes cut cheyyadam purthi chestado, aayana aa result ni tana Sous Chef ki istadu. Sous Chef tana kinda unna Junior Chefs andari results vachesaruku wait chesi, vaatini kaluputadu. Ee process ni **join** cheyyadam antaru. Finally, andari results Head Chef daggiriki cherukuntayi.

---

### 🧠 3. The Secret Sauce: Work-Stealing Algorithm

Regular `ThreadPoolExecutor` lo, anni tasks oka common queue lo untayi. Idi contention ki dari teeyochu. Fork/Join different ga pani chestundi.

Prathi thread ki `ForkJoinPool` lo tana **personal double-ended queue (Deque)** untundi.
1.  Oka thread oka task ni fork chesinappudu (ante, sub-tasks create chesinappudu), aa sub-tasks ni tana Deque *head* ki add chestundi.
2.  Thread tana pani kosam, tana Deque *head* nunchi tasks tesukuni chestu untundi (LIFO - Last-In, First-Out).
3.  **Work-Stealing:** Eppaite oka thread pani aipoyi idle ga untundo, adi velli **inkoka thread Deque *tail*** nunchi pani ni dongilistundi (steals).

Tail nunchi enduku? Endukante tail lo unnavi pedda tasks, avi chala kalam mundu create chesinavi. Vaatini tesukuni chesthe, idle thread ki ekkuva sepu pani dorukutundi. Head lo unnavi chala chinna, recent tasks. Deque owner head lo pani chestunte, vere thread tail lo steal cheyyadam valla contention chala takkuva untundi.

```mermaid
graph TD
    subgraph ForkJoinPool
        direction LR
        subgraph Thread 1
            direction TB
            T1_Deque("Deque for T1<br>[T1a, T1b, T1c]")
        end
        subgraph Thread 2
            direction TB
            T2_Deque("Deque for T2<br>[T2a]")
        end
        subgraph Thread 3 (Idle)
            direction TB
            T3_Deque("Deque for T3<br>[]")
        end
    end

    Thread1 -- Works on T1c (Head) --> T1_Deque
    Thread3 -- STEALS T1a (Tail) --> T1_Deque

    style Thread3 fill:#f9f,stroke:#333
```

---

### ✍️ 4. Detailed Explanation & Key Classes

*   **`ForkJoinPool`**: Idi `ExecutorService` ni implement chestundi. Deenilo work-stealing algorithm untundi. Manam `new ForkJoinPool()` tho create cheyyochu leda common pool (`ForkJoinPool.commonPool()`) ni vaadochu.

*   **`ForkJoinTask<V>`**: `Future` laantidi, kani inkonchem lightweight. Idi fork/join operations ki base class. Manam direct ga deenni use cheyyamu, deeni subclasses ni use chestam.

*   **`RecursiveAction`**: Result return cheyyani task kosam. Idi `Runnable` laantidi. Manam deeni `compute()` method ni override chesi, logic rastam.

    ```java
    // Signature
    protected abstract void compute();
    ```

*   **`RecursiveTask<V>`**: Result return chese task kosam. `V` anedi result type. Idi `Callable` laantidi. Manam deeni `compute()` method ni override chesi, logic rasi, result ni return chestam.

    ```java
    // Signature
    protected abstract V compute();
    ```

#### Key Methods

*   `fork()`: Ee task ni asynchronously execute cheyyi. Ante, task ni pool lo submit chesi, ventane tirigi vachey. Wait cheyyadu.
*   `join()`: Fork chesina task result kosam wait cheyyi. Ee call blocking. Task purthi ayyaka result istundi.
*   `invoke()`: Oka task ni pool ki submit chesi, adi purthi ayye varaku wait cheyyi. Idi `fork()` and `join()` ni kalipina lantiది.

---

### 💻 5. Code Example: Simple `RecursiveAction`

**Scenario:** Manam oka pedda array lo unna prathi element ni double chesi print cheyyali. Ee pani ni manam recursively split cheddam.

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveAction;

public class SimpleRecursiveAction extends RecursiveAction {

    private final int[] data;
    private final int start;
    private final int end;
    // Oka task entha chinna pani cheyyalo anedi ee threshold cheptundi
    private static final int THRESHOLD = 10;

    public SimpleRecursiveAction(int[] data, int start, int end) {
        this.data = data;
        this.start = start;
        this.end = end;
    }

    @Override
    protected void compute() {
        int length = end - start;
        if (length <= THRESHOLD) {
            // Pani chinnadi aite, direct ga chesey
            processDirectly();
        } else {
            // Leda, pani ni renduभाగాలుగా chesi, sub-tasks create cheyyi
            int mid = start + length / 2;
            SimpleRecursiveAction task1 = new SimpleRecursiveAction(data, start, mid);
            SimpleRecursiveAction task2 = new SimpleRecursiveAction(data, mid, end);

            // Rendu tasks ni parallel ga run cheyyadaniki invokeAll use cheddam
            invokeAll(task1, task2);
        }
    }

    private void processDirectly() {
        System.out.println("Processing chunk from " + start + " to " + end + " by " + Thread.currentThread().getName());
        for (int i = start; i < end; i++) {
            data[i] = data[i] * 2;
        }
    }

    public static void main(String[] args) {
        int[] data = new int[100];
        for (int i = 0; i < data.length; i++) {
            data[i] = i + 1;
        }

        // Common pool ni use cheddam
        ForkJoinPool pool = ForkJoinPool.commonPool();
        SimpleRecursiveAction mainTask = new SimpleRecursiveAction(data, 0, data.length);

        System.out.println("Submitting the main task...");
        pool.invoke(mainTask); // Task purthi ayyevaraku wait cheyyi
        System.out.println("Task finished.");

        // Example: First 10 elements chuddam
        for(int i = 0; i < 10; i++) {
            System.out.print(data[i] + " ");
        }
    }
}
```

---

### 🔗 6. Concept Connections

*   **`ExecutorService` (Phase 6):** `ForkJoinPool` anedi `ExecutorService` ki oka specialized implementation.
*   **Divide and Conquer Algorithm:** Ee framework ee algorithmic paradigm ni implement cheyyadaniki design chesaru.
*   **Parallel Streams (Phase 10):** Parallel streams venakala unna main engine ide. `stream.parallel().map(...)` lanti operations `ForkJoinPool` lo ne run avutayi.

### 👎 7. Anti-Patterns & Common Mistakes

*   **I/O Operations:** Fork/Join tasks lo network calls or file I/O lanti blocking operations pettakudadu. Okavela pedithe, aa thread block aipotundi, and pool lo unna migatha threads kuda starve avvochu, endukante work-stealing antha efficient ga pani cheyyadu.
*   **Task Granularity:** Tasks ni మరీ chinnaga or మరీ peddaga cheyyakudadu. Chala chinnaga unte, task creation overhead ekkuva avutundi. Chala peddaga unte, work-stealing ki avakasam undadu. Correct `THRESHOLD` ni choose chesukovadam chala mukhyam.

### 🔑 8. Key Takeaways

1.  **Divide and Conquer:** Fork/Join framework recursive, "divide and conquer" problems kosam design chesaru.
2.  **Work-Stealing:** Deeni secret sauce `work-stealing` algorithm, idi CPU cores ni high utilization lo unchutundi.
3.  **`RecursiveAction` vs `RecursiveTask`:** Result avasaram lekapothe `RecursiveAction`, result kavali ante `RecursiveTask` vaadali.
4.  **`fork()` & `join()`:** `fork()` tho sub-tasks create chestaru, `join()` tho vaati results kosam wait chestaru.
5.  **No Blocking I/O:** Fork/Join tasks lo blocking I/O operations pettakudadu.

---

### ✅ Checkpoint

*   Work-stealing algorithm lo, oka idle thread inkoka thread deque nunchi ekkada (head or tail) nunchi task ni steal chestundi? Enduku?
*   `RecursiveTask` ki and `RecursiveAction` ki madhya unna main difference enti?
*   `fork()` ki and `invoke()` ki madhya unna difference enti?
*   Fork/Join task lo oka network call cheyyadam enduku oka bad idea?

---
