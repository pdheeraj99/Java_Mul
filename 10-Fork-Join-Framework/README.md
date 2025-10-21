# Phase 10: Fork/Join Framework 🍴

Mawa, welcome to Phase 10! Ee phase lo, manam Java lo unna aalinganamaina parallel execution engines lo okati ayna **Fork/Join Framework** gurinchi deep ga dive cheddam. "Divide and Conquer" problems ni solve cheyyadaniki idi oka super powerful tool.

Oka pedda task ni chinnachinna sub-tasks ga ela break cheyyali (`fork`), and aa sub-task results ni malli ela kalapali (`join`) anedi ikkada nerchukuntam. Ee framework multi-core CPUs ni maximum efficiency tho use chesukovadaniki design chesaru, special ga deeni `work-stealing` algorithm valla. Idi Java 8 Parallel Streams ki foundation kuda, so deenni ardham chesukovadam chala important.

---

### 📚 Learning Chunks

Ee phase lo unna learning chunks ive. Order lo follow aite, neeku ee powerful framework mida complete clarity vastundi.

*   **[Chunk 1: Fork/Join Basics](./01-Fork-Join-Basics.md)**
    *   **Description:** Core concepts tho start cheddam. `Work-stealing` algorithm ante enti, `ForkJoinPool` ela pani chestundi, and result return chese `RecursiveTask` ki, cheyyani `RecursiveAction` ki madhya unna difference gurinchi telusuko.
    *   **Key Concepts:** `ForkJoinPool`, `RecursiveTask`, `RecursiveAction`, `fork()`, `join()`.

*   **[Chunk 2: Fork/Join Implementation](./02-Fork-Join-Implementation.md)**
    *   **Description:** Theory nunchi practice loki. Oka `RecursiveTask` ni step-by-step ga ela implement cheyyalo nerchuko. Correct `THRESHOLD` ni ela set cheyyali, `compute()` method lo logic ela rayali, and results ni ela combine cheyyalo chuddam.
    *   **Key Concepts:** `compute()` template, thresholding, `fork()`-`compute()`-`join()` pattern.

*   **[Chunk 3: Parallel Streams & Fork/Join Internals](./03-Parallel-Streams-Internals.md)**
    *   **Description:** Java 8 Parallel Streams venakala unna magic ni chuddam. Avi venakala Fork/Join framework ni, special ga shared `commonPool()` ni, ela use chesukuntayo ardham chesuko. Common pool ni block cheyyadam valla vache risks gurinchi kuda telusuko.
    *   **Key Concepts:** `parallelStream()`, `ForkJoinPool.commonPool()`, pool starvation, custom pools.

---

### 🛠️ Hands-On Mini-Project

*   **[Project 1: Parallel File System Counter](./projects/01-Parallel-File-Counter.md)**
    *   **Goal:** Oka classic Fork/Join problem ni solve cheddam: oka pedda directory structure lo unna total files ni parallel ga count cheyyadam. Ee project lo, manam `RecursiveTask` ni implement chesi, sequential version tho performance ni compare chesi, Fork/Join framework entha powerful o chustam.

---

Ee phase aipoyesariki, nuvvu complex recursive problems ni tesukuni, vaatini efficient parallel solutions ga marchagalav. Let's get forking!
