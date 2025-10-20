---
# 🎯 7.2: Introduction to CompletableFuture

## 🗺️ Learning Path Position

**📍 You Are Here:** 7.2 - Introduction to CompletableFuture

**✅ Prerequisites (Must Complete First):**
- [7.1: Callable and Future](./01-Callable-and-Future.md) - `Future` lo unna main limitation (`blocking get()`) gurinchi clear ga ardham cheskovali.

**🔜 Coming After This:**
- [7.3: Chaining and Combining CompletableFutures](./03-Chaining-and-Combining-CompletableFutures.md) - `thenApply`, `thenCompose` lanti powerful methods tho async workflows ni ela build cheyalo nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: `Future` is Good, but Not Great! 😕
Mawa, manam chusam `Future` anedi `Callable` nunchi result teskovadaniki help chestundi. Kaani daantlo konni pedda limitations unnayi:
1.  **It's Blocking 🛑:** Result kavali ante, `future.get()` ani call cheyali, adi nee thread ni block chestundi. Asynchronous programming chesi kuda, malli blocking deggarike vacham.
2.  **No Callbacks 📞:** Task complete ayyaka, automatically oka pani chey (`callback`) ane feature ledu. Maname `isDone()` tho poll cheyali or `get()` tho block avvali.
3.  **Can't Be Manually Completed:** Oka `Future` anedi `ExecutorService` matrame complete cheyagaladu. Manam manual ga daaniki oka result or exception set cheyalem.
4.  **No Chaining 🔗:** Oka `Future` result vachaka, daani meeda inko operation cheyali ante, adi inko `Future` tho combine cheyali ante... `Future` API tho chala kastam. Idi "Callback Hell" lanti situation ki lead chestundi.

### The Solution: `CompletableFuture` - The Swiss Army Knife of Async Programming 🇨🇭🔪
Java 8 lo `CompletableFuture<T>` introduce chesaru. Idi `Future` and `CompletionStage` ane rendu interfaces ni implement chestundi. Idi `Future` yokka limitations ni overcome chestundi:

1.  **Non-Blocking by Design:** `get()` undi, kaani manam daanini vadalsina avasaram takkuva. Callbacks use chestam.
2.  **Rich Callback API:** `thenApply`, `thenAccept`, `thenRun` lanti chala methods unnayi. "Task aypogane, ee pani chey" ani manam cheppochu.
3.  **Manually Completable:** Maname oka `CompletableFuture` create chesi, daaniki `complete(result)` or `completeExceptionally(ex)` call cheyochu.
4.  **Easy to Chain and Combine:** Multiple `CompletableFuture`s ni chain cheyadam, combine cheyadam chala easy.

### Real-World Analogy: Cooking Dinner 🍳
- **`Future` (Traditional Cooking):** Nuvvu rice cooker on chesav (`submit task`). Rice ready ayyinda leda ani teluskovadaniki, nuvvu kitchen lo ne wait cheyali (`future.get()`). Ee time lo nuvvu vere pani em cheyalevu.
- **`CompletableFuture` (Modern Smart Cooking):** Nuvvu smart rice cooker on chesav. Adi on chesi, "Rice aypogane, naku phone ki notification pampu" (`thenAccept`) ani cheptav. Nuvvu ee lopu hall lo TV chuskovachu (`main thread is not blocked`). Notification రాగానే, nuvvu curry prepare cheyadam start chestav (`callback function`).

---

## 📚 Detailed Explanation: Creating CompletableFutures

`CompletableFuture` ni create cheyadaniki main ga 2 static factory methods unnayi:

### 1. `runAsync(Runnable runnable)`
- **Purpose:** Result return cheyani task ni asynchronously run cheyadaniki. (`Runnable` teskuntundi).
- **Returns:** `CompletableFuture<Void>`.
- **Thread Pool:** By default, idi global `ForkJoinPool.commonPool()` ni use chestundi. Manam optional ga vere `Executor` ni kuda pass cheyochu.
```java
// Runnable task
Runnable printMessage = () -> System.out.println("Hello from a different thread!");

// Create and run it
CompletableFuture<Void> cf = CompletableFuture.runAsync(printMessage);
```

### 2. `supplyAsync(Supplier<U> supplier)`
- **Purpose:** Result return chese task ni asynchronously run cheyadaniki. (`Supplier<U>` teskuntundi).
- **Returns:** `CompletableFuture<U>`.
- **Thread Pool:** Idi kuda default ga `ForkJoinPool.commonPool()` ni use chestundi.
```java
// Supplier task (like a Callable that doesn't throw checked exceptions)
Supplier<String> fetchMessage = () -> {
    try { Thread.sleep(1000); } catch (Exception e) {}
    return "Hello, Mawa!";
};

// Create and run it
CompletableFuture<String> cf = CompletableFuture.supplyAsync(fetchMessage);
```
**`Supplier<T>` vs `Callable<T>`:** `Supplier` anedi `Callable` laantide, kaani daani `get()` method checked exceptions ni throw cheyadu. `CompletableFuture` design lo idi better fit avtundi.

#### 🧠 Mental Model Diagram: `supplyAsync`

```mermaid
graph TD
    subgraph Main Thread
        A[CompletableFuture.supplyAsync(mySupplier)] --> B[CF Object Returned Immediately];
        A --> C[... Other work ...];
    end

    subgraph ForkJoinPool (or Custom Executor)
        D[Worker Thread] --> E[Executes mySupplier.get()];
        E -- Result Ready --> F[Completes the CF Object];
    end

    A -.-> E;
    B -.-> F;

    style A fill:#90EE90
```

---

## 💻 Code Examples

### Example 1: Basic `runAsync()` - Fire and Forget
**Scenario:** Oka message ni background thread lo log cheyali. Result em avasaram ledu.
```java
// File: com/example/runasync/RunAsyncDemo.java
package com.example.runasync;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class RunAsyncDemo {
    public static void main(String[] args) {
        System.out.println("Main thread started: " + Thread.currentThread().getName());

        Runnable task = () -> {
            try { TimeUnit.SECONDS.sleep(2); } catch (Exception e) {}
            System.out.println("Logging something in background... Thread: " + Thread.currentThread().getName());
        };

        CompletableFuture<Void> cf = CompletableFuture.runAsync(task);

        System.out.println("Main thread continues to run...");

        // Wait for the future to complete to see the output
        cf.join(); // `join()` is similar to `get()`, but throws unchecked exceptions.

        System.out.println("Main thread finished.");
    }
}
```
**Output:**
```
Main thread started: main
Main thread continues to run...
Logging something in background... Thread: ForkJoinPool.commonPool-worker-1
Main thread finished.
```
**Why This Works:** `runAsync` call cheyagane, main thread block avvaledu. Task anedi `ForkJoinPool` lo execute aindi. `join()` just ee example lo output chudadaniki pettam.

### Example 2: Basic `supplyAsync()` - Getting a Result
**Scenario:** Oka user profile ni database (simulated) nunchi fetch cheyali.
```java
// File: com/example/supplyasync/SupplyAsyncDemo.java
package com.example.supplyasync;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutionException;

public class SupplyAsyncDemo {

    // Represents a user profile object
    static class UserProfile {
        String name;
        public UserProfile(String name) { this.name = name; }
        @Override public String toString() { return "UserProfile{name='" + name + "'}"; }
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        System.out.println("Main thread started: " + Thread.currentThread().getName());

        CompletableFuture<UserProfile> userFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println("Fetching user profile... Thread: " + Thread.currentThread().getName());
            try { Thread.sleep(2000); } catch (Exception e) {}
            return new UserProfile("Anand");
        });

        System.out.println("Main thread doing other work while profile is being fetched...");

        // Blocking to get the result (for now!)
        UserProfile userProfile = userFuture.get();

        System.out.println("Got result: " + userProfile);
        System.out.println("Main thread finished.");
    }
}
```
**Output:**
```
Main thread started: main
Main thread doing other work while profile is being fetched...
Fetching user profile... Thread: ForkJoinPool.commonPool-worker-1
Got result: UserProfile{name='Anand'}
Main thread finished.
```
**Why This Works:** Ippatiki, idi `Future` laane anipistundi, endukante manam `get()` tho block chestunnam. Kaani, `CompletableFuture` real power anedi next chunk lo `thenApply` lanti methods use chesinappudu telustundi.

### Example 3: Using a Custom Executor
**Scenario:** `ForkJoinPool` vaddu, manam create chesina custom thread pool vadali.
```java
// ... same as SupplyAsyncDemo.java ...
public static void main(String[] args) throws ExecutionException, InterruptedException {
    ExecutorService customExecutor = Executors.newFixedThreadPool(2);

    CompletableFuture<UserProfile> userFuture = CompletableFuture.supplyAsync(() -> {
        // ... task logic ...
    }, customExecutor); // ✅ Just pass the executor as the second argument

    // ...
    customExecutor.shutdown();
}
```
**Why This is Important:** `ForkJoinPool.commonPool()` anedi JVM-wide shared resource. Long-running or blocking tasks (like I/O) tho daanini block cheste, vere subsystems kuda affect avtayi. So, **blocking I/O operations ki eppudu custom `Executor` vadali.**

---

## ✅ Checkpoint: Did You Master This?

- [ ] `Future` tho compare cheste, `CompletableFuture` solve chese 3 main problems enti?
- [ ] `runAsync` ki `supplyAsync` ki teda enti? Ekkada edi vadali?
- [ ] By default, `CompletableFuture` async methods ఏ thread pool ni use chestayi?
- [ ] Custom `Executor` ni `supplyAsync` ki ela pass cheyali, and enduku adi important?

**✅ Ready?** → [Next: Chaining and Combining CompletableFutures](./03-Chaining-and-Combining-CompletableFutures.md)
