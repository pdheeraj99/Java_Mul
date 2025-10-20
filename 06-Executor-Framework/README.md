# Phase 6: Executor Framework 🎯

Welcome to Phase 6, mawa! Ikkada manam Java Concurrency lo oka chala powerful and essential topic gurinchi nerchukuntam: **The Executor Framework**. Manual ga `new Thread()` create cheyadam lo unna kashtalu tesesi, mana life ni easy chese manager idi.

Ee phase complete ayye sariki, nuvvu production-ready thread pools ni create cheyagalav, tasks ni schedule cheyagalav, and nee application resources ni efficiently manage cheyagalav.

## Learning Path for this Phase

### 📚 Core Concepts

1.  **[6.1: Introduction to the Executor Framework](./01-Introduction-to-Executor-Framework.md)**
    - **What:** Manual thread management nunchi Executor Framework ki enduku shift avvalo nerchukuntam.
    - **Difficulty:** 🟢 Beginner
    - **Time:** 2 hours

2.  **[6.2: Understanding Thread Pools](./02-Understanding-Thread-Pools.md)**
    - **What:** `newFixedThreadPool`, `newCachedThreadPool` lanti common thread pools gurinchi and vaati use cases gurinchi telusukuntam.
    - **Difficulty:** 🟡 Intermediate
    - **Time:** 2.5 hours

3.  **[6.3: ThreadPoolExecutor Deep Dive](./03-ThreadPoolExecutor-Deep-Dive.md)**
    - **What:** Production-grade, custom thread pools ni `ThreadPoolExecutor` class tho ela build cheyalo deep dive chestam. Idi ee phase lo most important topic!
    - **Difficulty:** 🔴 Advanced
    - **Time:** 3.5 hours

4.  **[6.4 & 6.5: ScheduledExecutors & Lifecycle](./04-ScheduledExecutors-and-Lifecycle.md)**
    - **What:** Tasks ni delay tho or periodic ga ela run cheyalo, and executors ni graceful ga ela shutdown cheyalo nerchukuntam.
    - **Difficulty:** 🟡 Intermediate
    - **Time:** 3 hours

Ready ah? Let's dive in! 🚀

---

### 🏗️ Hands-On Mini Projects

Theory nerchukunnaka, practice chala important mawa. Ee projects tho, nuvvu concepts ni real-world lo ela apply cheyalo chustav.

1.  **[Project 1: Concurrent Web Server](./projects/01-Concurrent-Web-Server.md)**
    - **Goal:** `FixedThreadPool` use chesi, multiple client requests ni handle chese simple web server ni build chey.
    - **Concepts Used:** `FixedThreadPool`, `ServerSocket`, `Runnable`.

2.  **[Project 2: Parallel File Processor](./projects/02-Parallel-File-Processor.md)**
    - **Goal:** Custom `ThreadPoolExecutor` and `Callable` use chesi, oka directory lo unna files ni concurrently process chey.
    - **Concepts Used:** `ThreadPoolExecutor`, `Callable`, `Future`.
