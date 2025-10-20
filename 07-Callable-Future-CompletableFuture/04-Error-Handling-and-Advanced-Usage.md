---
# 🎯 7.4: Error Handling and Advanced Usage

## 🗺️ Learning Path Position

**📍 You Are Here:** 7.4 - Error Handling and Advanced Usage

**✅ Prerequisites (Must Complete First):**
- [7.3: Chaining and Combining CompletableFutures](./03-Chaining-and-Combining-CompletableFutures.md) - `thenApply` and other chaining methods meeda clear idea undali.

**🔜 Coming After This:**
- [Project: Async API Aggregator](./projects/01-Async-API-Aggregator.md) - Ee concepts anni use chesi oka real-world project build cheddam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 🤔 What & Why

### The Problem: An Exception Can Break the Whole Chain! ⛓️💥
Mawa, manam ippativaraku "happy path" scenarios chusam. Kaani real world lo, network calls fail avtayi, databases down untayi, null pointers vastayi. `CompletableFuture` chain lo, madhyalo oka step lo exception vasthe em avtundi?

By default, aa exception anedi chain ni break chestundi. Kindaki propagate avthu, final `get()` or `join()` deggara `ExecutionException` laaga bayataki vastundi. Idi okay, kaani manaki inka better control kavali. Manam chain ni break cheyakunda, exception ni gracefully handle chesi, oka default value or alternative result ni ivvali.

### The Solution: `exceptionally()`, `handle()`, `allOf()`, and `anyOf()`

`CompletableFuture` manaki ee situations kosam special methods istundi:

- **For Error Handling:**
  - **`exceptionally(Function<Throwable, ? extends T> fn)`:**
    - **Purpose:** Chain lo edaina exception vasthe, daanini catch chesi, oka fallback/default value ni return cheyadaniki. Idi `try-catch` block laantidi.
    - **Analogy:** "Pizza order fail aite (e.g., out of stock), naaku బదులుగా garlic bread ivvu."
  - **`handle(BiFunction<? super T, Throwable, ? extends U> fn)`:**
    - **Purpose:** Success or failure, rendu cases lo execute avvadaniki. Result vachina, exception vachina, ee method call avtundi. Idi `try-catch-finally` lo `finally` laantidi.
    - **Analogy:** "Pizza order success ayina, fail ayina, final ga naku oka status update (SMS) pampu."

- **For Coordinating Multiple Futures:**
  - **`allOf(CompletableFuture<?>... cfs)`:**
    - **Purpose:** Anni `CompletableFuture`s complete ayyevaraku wait cheyadaniki.
    - **Returns:** `CompletableFuture<Void>`. **Idi chala important:** `allOf` anedi individual results ni combine cheyadu. Just anni complete ayyaya leda ani cheptundi.
    - **Analogy:** "Nee friends andaru (`CF1`, `CF2`, `CF3`) movie ki ready ayyaka, appudu manam cab book cheddam."
  - **`anyOf(CompletableFuture<?>... cfs)`:**
    - **Purpose:** Multiple `CompletableFuture`s lo, edi **first** complete aite, daani result teskovadaniki.
    - **Analogy:** "Multiple websites (`CF1`, `CF2`) lo flight price check chey. Ye website first result istundo, aa price tesko."

---

## 💻 Code Examples

### Example 1: `exceptionally()` for Fallback Values
**Scenario:** User details fetch chestunnam, kaani network fail aite, oka default guest user profile return cheyali.
```java
// File: com/example/error/ExceptionallyDemo.java
package com.example.error;

import java.util.concurrent.CompletableFuture;

public class ExceptionallyDemo {
    public static void main(String[] args) {
        CompletableFuture.supplyAsync(() -> {
            // Simulate a failure
            if (true) throw new RuntimeException("Network connection lost!");
            return "Real User";
        })
        .exceptionally(ex -> {
            System.err.println("Caught exception: " + ex.getMessage());
            return "Guest User"; // ✅ Return a default value
        })
        .thenAccept(user -> System.out.println("Logged in as: " + user));
    }
}
```

### Example 2: `handle()` for Both Success and Failure
**Scenario:** Oka operation result ni log cheyali, adi success ayina, fail ayina.
```java
// File: com/example/error/HandleDemo.java
package com.example.error;

import java.util.concurrent.CompletableFuture;

public class HandleDemo {
    public static void main(String[] args) {
        CompletableFuture.supplyAsync(() -> {
            if (Math.random() > 0.5) throw new RuntimeException("Something failed!");
            return "Success Result";
        })
        .handle((result, ex) -> {
            if (ex != null) {
                System.err.println("Operation failed with: " + ex.getMessage());
                return "Unknown Status"; // Return a status for the failure case
            }
            System.out.println("Operation succeeded with: " + result);
            return "Success Status"; // Return a status for the success case
        })
        .thenAccept(status -> System.out.println("Final Status: " + status));
    }
}
```

### Example 3: `allOf()` to Wait for All Tasks
**Scenario:** Parallel ga 3 different configuration files load cheyali. Anni load ayyaka, "Configuration loaded" ani print cheyali.
```java
// File: com/example/advanced/AllOfDemo.java
package com.example.advanced;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;
import java.util.stream.Stream;
import java.util.List;
import java.util.stream.Collectors;

public class AllOfDemo {
    public static void main(String[] args) {
        CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
            try { TimeUnit.SECONDS.sleep(1); } catch (Exception e) {}
            System.out.println("Loaded settings.conf");
            return "Settings";
        });
        CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
            try { TimeUnit.SECONDS.sleep(2); } catch (Exception e) {}
            System.out.println("Loaded user.conf");
            return "User";
        });
        CompletableFuture<String> cf3 = CompletableFuture.supplyAsync(() -> {
            try { TimeUnit.SECONDS.sleep(1); } catch (Exception e) {}
            System.out.println("Loaded permissions.conf");
            return "Permissions";
        });

        CompletableFuture<Void> allFutures = CompletableFuture.allOf(cf1, cf2, cf3);

        // allOf complete ayyaka, ee callback trigger avtundi.
        allFutures.thenRun(() -> System.out.println("\n🎉 All configuration files have been loaded."));

        // How to get results from all of them?
        allFutures.thenRun(() -> {
            String combinedResult = Stream.of(cf1, cf2, cf3)
                                          .map(CompletableFuture::join) // join() gets the result (waits if not ready, but here they are)
                                          .collect(Collectors.joining(", "));
            System.out.println("Results: " + combinedResult);
        }).join();
    }
}
```

### Example 4: `anyOf()` to Get the Fastest Result
**Scenario:** Rendu veru veru servers nunchi weather data fetch cheyali. Ye server first response istundo, adi teskovali.
```java
// File: com/example/advanced/AnyOfDemo.java
package com.example.advanced;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class AnyOfDemo {
    public static void main(String[] args) {
        CompletableFuture<String> server1 = CompletableFuture.supplyAsync(() -> {
            try { TimeUnit.SECONDS.sleep(2); } catch (Exception e) {}
            return "Weather from Server 1: 25°C";
        });
        CompletableFuture<String> server2 = CompletableFuture.supplyAsync(() -> {
            try { TimeUnit.SECONDS.sleep(1); } catch (Exception e) {}
            return "Weather from Server 2: 24.5°C";
        });

        CompletableFuture<Object> fastestServer = CompletableFuture.anyOf(server1, server2);

        fastestServer.thenAccept(result -> System.out.println("Fastest result: " + result));

        try { Thread.sleep(3000); } catch (Exception e) {}
    }
}
```

---

## ✅ Checkpoint: Did You Master This?

- [ ] `exceptionally()` ki `handle()` ki main difference enti?
- [ ] `allOf` return chese `CompletableFuture<Void>` nunchi individual results ni ela collect cheyali?
- [ ] `anyOf` anedi fastest result istundi, kaani daani return type `CompletableFuture<Object>` enduku untundi?
- [ ] Oka async chain lo multiple `exceptionally` blocks unte em avtundi?

**✅ Ready?** → [Project: Async API Aggregator](./projects/01-Async-API-Aggregator.md)
