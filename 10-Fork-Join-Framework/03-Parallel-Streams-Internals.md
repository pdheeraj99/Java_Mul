<!--
---
title: "Parallel Streams & Fork/Join Internals"
---
-->

> **Learning Path Position**
>
> Phase 10: Fork/Join Framework ➔ **Chunk 3: Parallel Streams Internals**

> **Prerequisites**
>
> *   Phase 10, Chunk 1 & 2: Fork/Join Framework (Absolutely essential)
> *   Java 8 Streams API gurinchi basic knowledge.

> **Coming After This**
>
> *   This is the final theory chunk for Phase 10. Next up is the Hands-On Mini-Project!

---

### 🚀 1. What & Why: Parallel Streams?

Mawa, Java 8 lo vachina okka super cool feature **Streams API**. Manam collections mida operations (`filter`, `map`, `reduce`) chala easy ga, declarative ga rayochu. For example:

```java
List<Integer> numbers = ...;
int sum = numbers.stream()
                 .filter(n -> n % 2 == 0)
                 .mapToInt(n -> n * 2)
                 .sum();
```

Idi antha sequential ga, okate thread lo run avutundi. Kani, okavela nee daggara chala pedda list and multi-core CPU unte? Appude **Parallel Streams** picture loki vastayi. Just `.stream()` badulu `.parallelStream()` ani call cheste chalu, Java aa pani ni automatic ga multiple threads ki panchi pedutundi.

```java
int sum = numbers.parallelStream() // Ide magic!
                 .filter(n -> n % 2 == 0)
                 .mapToInt(n -> n * 2)
                 .sum();
```

Kani... ee magic venakala em undi? Adi ela pani chestundi? The answer is... **the Fork/Join Framework!**

**Why is this important? 🤔**

*   **Performance Boost ⚡:** Correct ga use cheste, parallel streams nee code performance ni dramatically improve cheyyagalavu.
*   **Simplicity ✨:** Fork/Join tasks manually rayadam kanna, parallel stream rayadam chala simple ga untundi.
*   **Understanding the Risks ⚠️:** Ee magic venakala unna mechanism teliyakapothe, nuvvu performance problems and subtle bugs create chese risk undi. "Common pool" gurinchi telusukovadam chala mukhyam.

---

### analogy 2. Real-World Analogy: Pizza Restaurant Chain 🍕

Imagine chesuko, Domino's Pizza undi.
*   **The Task (Stream Operation):** Nuvvu 100 pizzas order chesav.
*   **`stream()` (Single Restaurant):** Nuvvu sequential stream use cheste, aa 100 pizzas order antha okate Domino's branch ki veltundi. Aa branch lo unna staff antha aa pani chestu untaru.
*   **`parallelStream()` (All Branches):** Nuvvu parallel stream use cheste, Domino's Head Office (`ForkJoinPool`) aa 100 pizza order ni tesukuni, daanni chinnachinna parts ga (e.g., 10 pizzas per branch) break chesi, city lo unna anni branches ki (`Worker Threads`) panchi pedutundi.
*   **The Common Pool:** Domino's city lo unna anni branches ni tana "common pool" of resources la vaadukuntundi. Anni normal orders (`parallelStream` calls) ide pool ni share chesukuntayi.

Ippudu, okavela oka branch lo power cut (oka thread lo blocking I/O) aite? Aa branch pani aagipotundi, and Head Office daggara aa branch mida depend ayina migatha panulu kuda aagipotayi. Ide parallel streams lo unna main risk.

---

### 🧠 3. How It Works: The `commonPool()`

Java lo anni parallel stream operations default ga oka **single, static, shared `ForkJoinPool`** ni use chesukuntayi. Ee pool ni `ForkJoinPool.commonPool()` antaru.
*   **Size:** Ee pool size default ga nee system lo unna number of CPU cores minus one (`Runtime.getRuntime().availableProcessors() - 1`) untundi. (Minimum 1).
*   **Shared Resource:** JVM lo run ayye nee application lo unna **anni** parallel stream calls ide pool ni share chesukuntayi.

`parallelStream()` call chesinappudu, venakala idi jarugutundi:
1.  Stream source (e.g., `ArrayList`) ni chinnachinna chunks ga `spliterator()` ane mechanism dwara break chestaru.
2.  Prathi chunk processing kosam oka `ForkJoinTask` (internal ga) create chestaru.
3.  Ee tasks anni `ForkJoinPool.commonPool()` ki submit chestaru.
4.  Pool lo unna worker threads work-stealing algorithm use chesi aa tasks ni execute chestayi.

```mermaid
graph TD
    A[myList.parallelStream()] --> B{Spliterator breaks list into chunks};
    B --> C1(Chunk 1 Task);
    B --> C2(Chunk 2 Task);
    B --> C3(Chunk 3 Task);
    B --> C4(Chunk 4 Task);

    subgraph ForkJoinPool.commonPool()
        direction LR
        W1(Worker 1)
        W2(Worker 2)
        W3(Worker 3)
        W4(Worker 4)
    end

    C1 --> W1;
    C2 --> W2;
    C3 --> W3;
    C4 --> W4;

    W1 -- Steals work if idle --> W2;
```

---

### ⚠️ 4. The Dangers of the Common Pool

Mawa, idi chala jagrattaga vinu. `commonPool()` anedi chala convenient, kani oka double-edged sword laantidi.

**The Problem: Blocking Tasks**
Okavela nuvvu parallel stream lo blocking operation (network call, file I/O, `Thread.sleep()`) pedithe emavutundi?
1.  Oka worker thread aa blocking task ni tesukuntundi.
2.  Aa thread block aipotundi. Adi I/O complete ayye varaku wait chestu untundi.
3.  Ippudu, `commonPool()` lo oka thread takkuva aindi.
4.  Okavela nuvvu ilanti chala blocking tasks ni submit cheste, `commonPool()` lo unna **anni** worker threads block aipovachu.

Ee situation ni **pool starvation** antaru. Appudu, nee application lo unna migatha parallel stream operations kuda aagipotayi, endukante vaatiki pani cheyyadaniki threads levu! Ee shared pool antha oka pedda traffic jam aipotundi.

**👎 Failure Case Code Example:**

```java
import java.util.List;
import java.util.stream.IntStream;

public class CommonPoolStarvation {

    public static void main(String[] args) {
        List<Integer> numbers = IntStream.range(0, 20).boxed().toList();

        System.out.println("Submitting a BAD parallel task with blocking I/O...");

        // Ee task common pool ni block chestundi
        numbers.parallelStream().forEach(i -> {
            try {
                System.out.println("Processing " + i + " on " + Thread.currentThread().getName());
                // DON'T DO THIS! I/O operation in parallel stream
                Thread.sleep(1000);
            } catch (InterruptedException e) {}
        });

        // Okavela vere edaina important parallel task run avvali anukunte, daaniki threads dorakavu.
        System.out.println("Bad task finished.");
    }
}
```
Ee code run chestunnapudu, nee CPU usage chala takkuva untundi, kani program chala slow ga untundi. Endukante, threads antha time sleep lo ne unnayi, pani cheyyatledu.

---

### ✅ 5. The Solution: Custom Fork/Join Pool

Blocking tasks ni parallel ga run cheyyali ante, `commonPool()` ni use cheyyakudadu. Daani badulu, manam oka **custom `ForkJoinPool`** create chesukuni, mana task ni daaniki submit cheyyali. Deeni valla, mana "slow" task kosam separate ga oka pool untundi, adi `commonPool()` ni disturb cheyyadu.

**✅ Success Case Code Example:**

```java
import java.util.List;
import java.util.concurrent.ForkJoinPool;
import java.util.stream.IntStream;

public class CustomPoolSolution {

    public static void main(String[] args) throws Exception {
        List<Integer> numbers = IntStream.range(0, 20).boxed().toList();

        // 1. Manam oka custom pool create cheddam
        ForkJoinPool customPool = new ForkJoinPool(4); // Example: 4 threads

        System.out.println("Submitting a blocking task to a CUSTOM pool...");

        // 2. Mana parallel stream logic ni oka task la submit cheyyi
        //    customPool.submit() ki oka lambda pass cheddam
        customPool.submit(() -> {
            numbers.parallelStream().forEach(i -> {
                try {
                    System.out.println("Processing " + i + " on " + Thread.currentThread().getName());
                    Thread.sleep(1000);
                } catch (InterruptedException e) {}
            });
        }).get(); // .get() anedi task purthi ayye varaku wait chestundi

        System.out.println("Custom pool task finished.");

        // Ippudu, commonPool() anedi free ga undi, vere panulaki ready ga undi.

        customPool.shutdown();
    }
}
```
Ee approach tho, manam blocking operations ni `commonPool()` ni jam cheyyakunda safe ga handle cheyyochu.

---

### 🔑 6. Key Takeaways

1.  **Parallel Streams = Fork/Join:** Parallel streams venakala unna engine Fork/Join Framework.
2.  **The `commonPool()`:** Default ga, anni parallel streams oka shared, static `ForkJoinPool` (`commonPool()`) ni use chesukuntayi.
3.  **No Blocking I/O in Parallel Streams:** `commonPool()` ni use chestunnapudu, parallel streams lo eppudu blocking operations (I/O, `sleep`) pettakudadu. Idi pool starvation ki dari teestundi.
4.  **Isolate Blocking Tasks:** Okavela neeku blocking operations ni parallel ga run cheyyali anipiste, eppudu oka **custom `ForkJoinPool`** create chesi, nee task ni daaniki submit cheyyi.

### ✅ Checkpoint

*   `myList.stream()` ki and `myList.parallelStream()` ki madhya unna main difference enti?
*   `ForkJoinPool.commonPool()` size default ga entha untundi?
*   "Pool starvation" ante enti and adi parallel streams lo ela jarugutundi?
*   Parallel stream lo oka network call cheyyali, `commonPool()` ni disturb cheyyakunda. Ela chestav?

---
