---
# 🎯 7.1: Callable and Future - Getting Results from Threads

## 🗺️ Learning Path Position

**📍 You Are Here:** 7.1 - Callable and Future

**✅ Prerequisites (Must Complete First):**
- [6.1: Introduction to the Executor Framework](../06-Executor-Framework/01-Introduction-to-Executor-Framework.md) - `ExecutorService` and the `submit()` method gurinchi basic idea undali.

**🔜 Coming After This:**
- [7.2: Introduction to CompletableFuture](./02-Introduction-to-CompletableFuture.md) - `Future` lo unna limitations ni `CompletableFuture` ela solve chestundo chustam.

**⏱️ Estimated Time:** 2.5 hours
**📊 Difficulty Level:** 🟡 Intermediate

---

## 🤔 What & Why

### The Problem: `Runnable` is a One-Way Street  unidirectional 🚚
Mawa, manam ippativaraku `Runnable` use chesam. Adi tasks ni execute cheyadaniki great, kaani daantlo rendu pedda problems unnayi:
1.  **It Can't Return a Result:** `run()` method yokka return type `void`. Ante, oka background thread lo oka calculation chesi (e.g., database nunchi user data fetch cheyadam), aa result ni main thread ki easy ga ivvalem. Maname `volatile` variables or shared objects lanti complex logic rayali.
2.  **It Can't Throw Checked Exceptions:** `run()` method signature lo `throws` clause ledu. Ante, `IOException` lanti checked exceptions ni manam propagate cheyalem, `try-catch` petti handle cheyali anthe. Idi error handling ni chala complicated chestundi.

### The Solution: `Callable` to the Rescue! 🦸‍♂️
`Callable<V>` anedi `Runnable` ki alternative. Idi ee two problems ni solve chestundi:
1.  **Returns a Result:** Deeni `call()` method `void` kadu, adi generic type `V` ni return chestundi. So, `Callable<String>` anedi oka `String` ni, `Callable<Integer>` anedi oka `Integer` ni return chestundi.
2.  **Throws Checked Exceptions:** `call()` method `throws Exception` ani declare chestundi. So, manam checked exceptions ni gracefully handle cheyochu.

### The New Problem: How to Get the Result? 🤔
Super, `Callable` result ni return chestundi. Kaani task anedi vere thread lo, future lo eppudo complete avtundi. Main thread aa result kosam ela wait chestundi? Ekkade `Future<V>` picture loki vastundi.

`Future` anedi oka promise anuko. "Hey, nenu neeku ee task yokka result ni istanu, adi ready ayyaka" ani cheppe oka placeholder object.

### Real-World Analogy: Dominos Pizza Order 🍕
- **You (Main Thread):** Nuvvu Dominos ki velli pizza order chestav.
- **`Callable` Task:** Kitchen lo nee pizza prepare cheyadam anedi `Callable`. Adi complete aite, result (`Pizza`) ready avtundi.
- **`executor.submit(callable)`:** Nuvvu counter lo order place cheyadam.
- **`Future` (The Token/Receipt):** Neeku order place cheyagane oka token or receipt istaru. Adi pizza kadu, kaani aa pizza ki promise. Nuvvu aa token teskuni, waiting area lo kurchoni nee pani (e.g., phone chuskovadam) cheskovachu.
- **`future.isDone()`:** Nuvvu display board meeda "Is my order ready?" ani check chesinattu.
- **`future.get()`:** Nee token number display avvagane, nuvvu counter ki velli, token ichi, pizza teskovadam. Nuvvu pizza (`result`) kosam counter deggara wait cheyali. Kitchen lo inka pizza prepare avthu unte, nuvvu akkade block aypotav.

---

## 📚 Detailed Explanation

### `Callable<V>` Interface
```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```
- Idi generic interface. `V` anedi return type.
- `Runnable` lo `run()` unnatte, deentlo `call()` untundi.

### `Future<V>` Interface
Idi oka task yokka lifecycle ni manage cheyadaniki and result ni teskovadaniki methods istundi.
- `V get()`: **Blocking call.** Task complete ayyevaraku wait chesi, result ni istundi.
- `V get(long timeout, TimeUnit unit)`: Same as `get()`, kaani specified time varaku matrame wait chestundi. Time aypothe `TimeoutException` vastundi.
- `boolean isDone()`: Task complete ayinda leda ani check cheyadaniki. Non-blocking.
- `boolean isCancelled()`: Task cancel ayinda leda.
- `boolean cancel(boolean mayInterruptIfRunning)`: Task ni cancel cheyadaniki try chestundi.

#### 🧠 Mental Model Diagram: The Workflow

```mermaid
graph TD
    A[Main Thread] --> B[Create Callable Task];
    B --> C[ExecutorService.submit(task)];
    C --> D[Returns Future object immediately];
    A --> E[... Main thread continues other work ...];
    C --> F[Worker Thread starts executing call()];
    E --> G[future.get()];
    F -- Computes Result --> H[Result Ready];
    H --> G;
    G -- Returns Result --> A;

    style G fill:#FF6B6B
```

---

## 💻 Code Examples

### Example 1: `Runnable` vs. `Callable`

**Scenario:** Database nunchi user name fetch cheyali. Idi 2 seconds padutundi.

#### ❌ The `Runnable` Way (Complex)
```java
// File: com/example/runnable/RunnableLimitation.java
package com.example.runnable;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class RunnableLimitation {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        // Result ni store cheyadaniki oka temporary, final, mutable container kavali.
        final String[] userName = new String[1];

        Runnable fetchUser = () -> {
            try {
                Thread.sleep(2000); // Simulate DB call
                userName[0] = "Mawa"; // ❌ Clunky way to store result
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        };
        executor.submit(fetchUser);

        System.out.println("Main thread is doing other work...");

        // Maname manually wait cheyali result kosam. Idi reliable kadu.
        // try { Thread.sleep(3000); } catch (InterruptedException e) {}

        executor.shutdown();
        // Result access cheyadam chala complex
        // System.out.println("Fetched user: " + userName[0]);
    }
}
```
**Why This is Bad:** Result ni main thread ki pass cheyadaniki direct way ledu. `final` array or `AtomicReference` lanti workarounds vadali. Checked exceptions ni kuda propagate cheyalem.

#### ✅ The `Callable` Way (Clean & Simple)
```java
// File: com/example/callable/CallableExample.java
package com.example.callable;

import java.util.concurrent.*;

public class CallableExample {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executor = Executors.newSingleThreadExecutor();

        // ✅ Callable<String> returns a String.
        Callable<String> fetchUser = () -> {
            Thread.sleep(2000); // Simulate DB call
            if (Math.random() > 0.5) {
                throw new IOException("DB Connection Failed!"); // ✅ Can throw checked exceptions
            }
            return "Mawa"; // ✅ Directly return the result
        };

        // submit() method ki Callable pass cheste, adi Future<String> return chestundi.
        Future<String> userFuture = executor.submit(fetchUser);
        System.out.println("Task submitted. Main thread is doing other work...");

        try {
            // ✅ future.get() blocks until the result is ready.
            String userName = userFuture.get();
            System.out.println("🎉 Fetched user: " + userName);
        } catch (ExecutionException e) {
            // Exception from the `call()` method will be wrapped in ExecutionException
            System.err.println("❌ Error during task execution: " + e.getCause().getMessage());
        }

        executor.shutdown();
    }
}
```
**Why This is Awesome:** Code chala clean ga undi. Result direct ga vastundi, exceptions kuda proper ga handle avtayi.

---

## 🐛 Debugging Guide

### Common Issue: `future.get()` Blocks Forever
**Symptom:** Nee application `future.get()` deggara hang aypotundi, munduku velladu.
**Meaning:** Idi ante, nee `Callable` task eppatiki complete avvatledu. Daantlo infinite loop undochu, or adi oka lock kosam forever wait chestu undochu (deadlock).
**Solution:**
1.  **Use `get()` with a Timeout:** Production code lo eppudu `future.get(1, TimeUnit.MINUTES)` lanti timeout version vadali. Idi nee application antha hang avvakunda kapadutundi.
2.  **Analyze Thread Dumps:** Application hang ayinappudu, thread dump tesko (`jstack` command). Pool lo unna thread em chestundo chudu. Adi `WAITING` or `BLOCKED` state lo unte, enduku alano investigate chey.

---

## 🧪 Testing Examples

### Unit Test: Testing a `Callable` Task
**Scenario:** Oka `Callable` task correct ga result istundo ledo test cheddam. Exception vasthe, adi `ExecutionException` lo wrap avtundo ledo kuda chuddam.
```java
// File: com/example/testing/DataFetcher.java
package com.example.testing;

import java.util.concurrent.Callable;

public class DataFetcher implements Callable<String> {
    private final boolean shouldThrow;
    public DataFetcher(boolean shouldThrow) { this.shouldThrow = shouldThrow; }

    @Override
    public String call() throws Exception {
        if (shouldThrow) {
            throw new IllegalStateException("Data source is down!");
        }
        return "Real Data";
    }
}

// File: com/example/testing/DataFetcherTest.java
package com.example.testing;

import org.junit.jupiter.api.Test;
import java.util.concurrent.*;
import static org.junit.jupiter.api.Assertions.*;

public class DataFetcherTest {
    @Test
    void testDataFetcherReturnsDataSuccessfully() {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Callable<String> task = new DataFetcher(false);
        Future<String> future = executor.submit(task);

        // Assert
        assertDoesNotThrow(() -> {
            assertEquals("Real Data", future.get(1, TimeUnit.SECONDS));
        });
        executor.shutdown();
    }

    @Test
    void testDataFetcherThrowsException() {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Callable<String> task = new DataFetcher(true);
        Future<String> future = executor.submit(task);

        // Assert that future.get() throws ExecutionException
        ExecutionException thrown = assertThrows(ExecutionException.class, () -> {
            future.get(1, TimeUnit.SECONDS);
        });

        // Assert that the cause of the exception is what our task threw
        assertTrue(thrown.getCause() instanceof IllegalStateException);
        assertEquals("Data source is down!", thrown.getCause().getMessage());

        executor.shutdown();
    }
}
```

---

## ❌ Anti-Patterns

### Anti-Pattern: Blocking the Main Thread Unnecessarily
**What's Wrong:** `submit()` chesina ventane `get()` call cheyadam.
```java
// ❌ BAD CODE
Future<String> future = executor.submit(myCallable);
String result = future.get(); // Why submit to another thread if you're just going to wait immediately?
System.out.println(result);
```
**Why Bad:** Task ni vere thread ki submit cheyadam main purpose ye non-blocking work cheyadaniki. Nuvvu submit chesina ventane `get()` call cheste, nee main thread antha block aypotundi. Idi simple synchronous call laaga aypoindi.
**✅ Correct Way:** `submit()` chesina tarvata, main thread lo inkonni panulu chey. Anni ayyaka, final ga result kosam `get()` call chey.

---

## ✨ Key Takeaways

1.  **`Callable` for Results, `Runnable` for Fire-and-Forget:** Result kavalante `Callable` vadu. Result akkarledu, just task run avvali ante `Runnable` vadu.
2.  **`Future` is a Promise:** Adi result kadu, result ki placeholder matrame.
3.  **`future.get()` is a Blocker 🛑:** Ee method nee thread ni block chestundi. Eppudu timeout version vadadaniki try chey.
4.  **Exceptions are Wrapped:** `Callable` lo vache exception, `future.get()` call chesinappudu `ExecutionException` ga wrap aypoyi vastundi.
5.  **`Future` is Limited:** `Future` tho problem enti ante, manam result kosam poll cheyali (`isDone()`) or block avvali (`get()`). Result vachaka automatically oka pani chey (`callback`) ane option ledu. Ee problem ni `CompletableFuture` solve chestundi.

---

## ✅ Checkpoint: Did You Master This?

- [ ] `Runnable` lo unna rendu main limitations enti? `Callable` vatini ela solve chestundi?
- [ ] `Future` yokka analogy nee own words lo cheppagalava?
- [ ] `future.get()` ki `future.get(timeout)` ki teda enti? Production lo edi vadali?
- [ ] `submit()` chesina ventane `get()` call cheste, adi multithreading use chesinatta? Synchronous call chesinatta?

**✅ Ready?** → [Next: Introduction to CompletableFuture](./02-Introduction-to-CompletableFuture.md)
