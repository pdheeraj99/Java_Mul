<!--
---
title: "Structured Concurrency"
---
-->

> **Learning Path Position**
>
> Phase 11: Modern Concurrency ➔ **Chunk 3: Structured Concurrency**

> **Prerequisites**
>
> *   Phase 11, Chunk 2: Virtual Threads (Structured Concurrency vaatitho chala baga pani chestundi)
> *   Phase 6 & 7: `ExecutorService` and `Future` (compare cheyyadaniki)

> **Coming After This**
>
> *   This is the final theory chunk for Phase 11. Next up is the Hands-On Mini-Project!

> **⚠️ Note:** Structured Concurrency anedi Java 19 lo vachina oka **preview feature**. Ante, adi inka final kadu and future versions lo marochu. Daanni use cheyyali ante, nuvvu nee code ni `--enable-preview` flag tho compile and run cheyyali.

---

### 🚀 1. What & Why: Structured Concurrency?

Mawa, ippativaraku manam chusina concurrency model (`ExecutorService`, `Future`) lo oka fundamental problem undi. Manam oka task ni submit chesinappudu, adi oka "fire and forget" laantidi. Aa task parent thread nunchi separate aipotundi.
*   **Error Handling:** Okavela manam rendu tasks ni parallel ga start chesi, okati fail aite? Manam manually inko task ni cancel cheyyali. Ee logic chala complex ga untundi.
*   **Thread Leaks:** Parent thread aagipoina, child tasks background lo run avutune undochu ("orphaned threads").
*   **Readability:** Code chala chidra ga untundi, `Future`s ni manage cheyyadam, exceptions handle cheyyadam antha clean ga undadu.

**Structured Concurrency** ee problem ki solution. Idi concurrent tasks ni oka **single unit of work** la treat chestundi. Main idea entante, **"oka task sub-tasks ga split aite, adi aa sub-tasks anni complete ayye varaku wait cheyyali."**

Deeni valla, concurrent code normal sequential code laaga structured ga, easy to read ga, and robust ga untundi. Oka `try` block lanti scope lo ne anni concurrent tasks jeevitakalam untundi.

---

### analogy 2. Real-World Analogy: Making a Burger 🍔

Imagine nuvvu (main thread) oka burger cheyyali anukuntunnav. Daaniki rendu panulu okesari cheyyali: (A) grill the patty, (B) toast the buns.
*   **Unstructured Concurrency (`ExecutorService`):** Nuvvu oka helper ki patty grill cheyyamani cheppi, inko helper ki buns toast cheyyamani cheppi, nuvvu vere pani chesukuntav. Okavela patty helper daanni madesiste (task fails)? Nuvvu adi telusukuni, buns helper daggiriki velli "Hey, aapey, order cancel aindi" ani cheppali. Marchipothe, aayana buns madesi time waste chestadu.
*   **Structured Concurrency (`StructuredTaskScope`):** Nuvvu oka "Burger Making Station" (`Scope`) open chestav. Ikkada nuvvu iddari helpers ki panulu cheptav. Ee station rule entante: **"Ee station lo panulu anni kalisi success avutayi leda kalisi fail avutayi."**
    *   Patty helper fail aite, station manager ventane buns helper ni aapestadu.
    *   Burger anedi (main task) patty and buns (sub-tasks) rendu ready aite ne "complete" avutundi.

---

### 🧠 3. How It Works: `StructuredTaskScope`

Structured Concurrency ki main entry point `java.util.concurrent.StructuredTaskScope` ane class. Idi `AutoCloseable`, so manam daanni `try-with-resources` block lo use cheyyochu, deeni valla scope chala clear ga define avutundi.

1.  **Scope Create cheyyi:** `try (var scope = new StructuredTaskScope.ShutdownOnFailure()) { ... }` ani oka scope create chestaru.
2.  **Tasks ni Fork cheyyi:** Scope loni `fork()` method use chesi, nuvvu nee `Callable` tasks ni submit chestav. Prathi `fork()` call oka kottha virtual thread ni start chestundi. Idi oka `Future` laantidi (`Supplier<T>`) return chestundi.
3.  **Anni aipoyevaraku Join cheyyi:** `scope.join()` call chestaru. Ee method scope lo unna **anni** tasks purthi ayye varaku block avutundi.
4.  **Results ni tesuko:** `join()` taruvatha, nuvvu `fork()` return chesina `Supplier` mida `.get()` call chesi result tesukovachu.
5.  **Scope Close cheyyi:** `try` block aipogane, scope automatic ga `close()` avutundi. Appudu, running lo unna tasks emaina unte, avi anni interrupt avutayi. Thread leaks undavu!

```mermaid
graph TD
    A[Start `try-with-resources`] --> B(new StructuredTaskScope);
    B --> C1(scope.fork(task1));
    B --> C2(scope.fork(task2));
    C1 --> D{Tasks run in parallel};
    C2 --> D;
    D --> E(scope.join());
    E --> F{Wait for all tasks};
    F --> G(task1.get(), task2.get());
    G --> H(End of `try` block);
    H --> I(scope.close() is called);
    I --> J[All leftover tasks are cancelled];

    style B fill:#cde,stroke:#333
    style I fill:#f99,stroke:#333
```

#### Shutdown Policies

*   `ShutdownOnFailure`: Okate scope lo, okavela edaina task exception tho fail aite, ee policy migatha anni inka running lo unna tasks ni ventane cancel chestundi. `join()` method aa exception ni re-throw chestundi. Ide most common policy.
*   `ShutdownOnSuccess`: Okavela edaina okka task successfully complete aite, ee policy migatha anni tasks ni cancel chestundi. Idi "race" scenarios ki use avutundi - evaru mundu geliste vaalle winner.

---

### 💻 4. Code Example: Concurrent Microservice Calls

**Scenario:** Manam oka user profile ni load cheyyali. Daaniki, manam (A) `fetchUserData()` and (B) `fetchUserPermissions()` ane rendu network calls ni parallel ga cheyyali.

**👎 Failure Case (Unstructured `ExecutorService`):**
Ee code chala verbose ga untundi. `future2` ni cancel cheyyadam, executor ni shutdown cheyyadam lanti logic antha maname rayali.

```java
// Ee method chala complex ga untundi
Future<UserData> future1 = executor.submit(this::fetchUserData);
Future<UserPermissions> future2 = executor.submit(this::fetchUserPermissions);
UserData data = null;
try {
    data = future1.get();
} catch (Exception e) {
    future2.cancel(true); // Manually cancel cheyyali
    throw e;
}
UserPermissions perms = future2.get(); // Main exception aite, idi kuda fail avvochu
return new UserProfile(data, perms);
```

**✅ Success Case (`StructuredTaskScope`):**
Chudu, ee code entha clean ga, readable ga undi. Error handling antha automatic!

```java
import java.time.Duration;
import java.util.concurrent.StructuredTaskScope;
import java.util.concurrent.Future;

// Records for data
record UserData(String name, int id) {}
record UserPermissions(String role) {}
record UserProfile(UserData data, UserPermissions permissions) {}

public class StructuredConcurrencyDemo {

    // Ee method ni --enable-preview flag tho run cheyyali
    public UserProfile fetchUserProfile() throws InterruptedException {
        // ShutdownOnFailure policy tho scope create cheyyi
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

            // 1. Rendu tasks ni fork cheyyi (ivi parallel ga start avutayi)
            Future<UserData> userDataFuture = scope.fork(this::fetchUserData);
            Future<UserPermissions> permissionsFuture = scope.fork(this::fetchUserPermissions);

            // 2. Rendu tasks aipoyevaraku join avvu (or okati fail ayyevaraku)
            scope.join();
            scope.throwIfFailed(); // Okati fail aite, ikkada exception vastundi

            // 3. Rendu success aite, results ni tesuko
            UserData data = userDataFuture.resultNow();
            UserPermissions permissions = permissionsFuture.resultNow();

            return new UserProfile(data, permissions);

        } // Scope ikkada close avutundi, threads anni clean up avutayi
    }

    // Helper methods (network calls ni simulate chestayi)
    private UserData fetchUserData() throws InterruptedException {
        System.out.println("User data fetch start chestunnam...");
        Thread.sleep(Duration.ofSeconds(1));
        // Okavela fail aite: throw new RuntimeException("User Service Down!");
        System.out.println("User data vachesindi.");
        return new UserData("Jules", 123);
    }

    private UserPermissions fetchUserPermissions() throws InterruptedException {
        System.out.println("User permissions fetch start chestunnam...");
        Thread.sleep(Duration.ofSeconds(2));
        System.out.println("User permissions vachesayi.");
        return new UserPermissions("admin");
    }

    public static void main(String[] args) throws InterruptedException {
        StructuredConcurrencyDemo demo = new StructuredConcurrencyDemo();
        UserProfile profile = demo.fetchUserProfile();
        System.out.println("Final Profile: " + profile);
    }
}
```

---

### 🔗 5. Concept Connections

*   **Virtual Threads:** Structured Concurrency anedi Virtual Threads tho kalisi chala baga pani chestundi. `scope.fork()` default ga virtual thread create chestundi.
*   **`ExecutorService.invokeAll()`:** Idi kuda multiple tasks ni run chestundi, kani daani error handling and cancellation antha robust ga undadu. `invokeAll` anni tasks aipoye varaku wait chestundi, fail ayina sare. Structured Concurrency ventane cancel chestundi.

### 🔑 6. Key Takeaways

1.  **Concurrent Code, Sequential Look:** Structured Concurrency tho, manam concurrent code ni simple `try-with-resources` block lo rayochu, adi chala readable ga untundi.
2.  **Clear Lifetime:** Tasks jeevitakalam antha aa `try` block ke పరిమితం. Block aipogane, anni tasks guarantee ga terminate avutayi.
3.  **Robust Error Handling:** `ShutdownOnFailure` lanti policies valla, okati fail aite anni fail avutayi. Error handling chala simple aipotundi.
4.  **No Thread Leaks:** `AutoCloseable` nature valla, "orphaned" threads undavu.

---

### ✅ Checkpoint

*   "Unstructured" concurrency (`ExecutorService`) lo unna main problem enti?
*   `StructuredTaskScope` ni `try-with-resources` lo enduku pedataru? Daani valla vache main benefit enti?
*   `ShutdownOnFailure` policy ela pani chestundi?
*   `scope.join()` call chesinappudu emi jarugutundi?

---
