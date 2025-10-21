# 🏗️ Mini Project: High-Throughput Virtual Thread Server

## 🤔 Problem Statement
Mawa, manam `ConcurrentWebServer` project ni Phase 6 lo build chesam. Adi `FixedThreadPool` use chesindi. Adi okay, kaani adi kevalam konni hundreds or thousands of concurrent connections matrame handle cheyagaladu, endukante adi platform threads meeda depend avtundi.

Ee project lo, manam aa server ni virtual threads use chesi rewrite cheddam. Manam chustam, almost same code tho, mana server **hundreds of thousands** of concurrent connections ni easy ga handle cheyagaladu. Idi Project Loom yokka real power ni demonstrate chestundi.

**Requirements:**
1.  Server port 8080 meeda listen cheyali.
2.  Prathi client connection ki, oka **kotha virtual thread** ni start cheyali.
3.  Manam `Executors.newVirtualThreadPerTaskExecutor()` use chesi, ee process ni manage cheddam.
4.  Server "Hello, you are client #X" ani response ivvali.

**This Project Uses Concepts From:**
- ✅ [11.2: Virtual Threads - Project Loom](../02-Virtual-Threads-Project-Loom.md)
- ✅ Classic "Thread-per-request" server architecture.

---

## 🏗️ Architecture
Architecture anedi Phase 6 project laage untundi. Kaani, main difference enti ante, manam `newFixedThreadPool` badulu, `newVirtualThreadPerTaskExecutor` use chestam. Deenivalla, prathi `socket.accept()` ki, manam without hesitation oka kotha thread ni start cheyochu, endukante virtual threads anevi chala cheap.

```mermaid
graph TD
    subgraph Main Thread
        A[Start ServerSocket] --> B{Loop forever};
        B --> C[ServerSocket.accept()];
        C --> D[Client Socket created];
        D --> E[Create RequestHandler(socket)];
        E --> F[executor.submit(task)];
        F --> B;
    end

    subgraph Executor (Virtual Threads)
        F ==> G[Starts New Virtual Thread 1];
        F ==> H[Starts New Virtual Thread 2];
        F ==> I[... up to millions ...];
    end

    G --> K[Handle Request];
    H --> L[Handle Request];

    style Executor fill:#90EE90
```

---

## 💻 Complete Code

#### File 1: `VirtualThreadServer.java`
```java
// File: src/com/example/server/VirtualThreadServer.java
// NOTE: To run this, you need JDK 19+ with preview features enabled.
// javac --release 19 --enable-preview src/com/example/server/*.java -d out
// java --enable-preview -cp out com.example.server.VirtualThreadServer
package com.example.server;

import java.io.IOException;
import java.io.PrintWriter;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicLong;

public class VirtualThreadServer {

    private static final int PORT = 8080;
    private static final AtomicLong clientCounter = new AtomicLong(0);

    public static void main(String[] args) {
        // ✅ The magic is here: an executor that creates a new virtual thread for each task.
        //    try-with-resources ensures the executor is closed properly on exit.
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {

            ServerSocket serverSocket = new ServerSocket(PORT);
            System.out.println("🚀 Virtual Thread Server is listening on port " + PORT);

            while (true) {
                // Accept a new connection
                Socket clientSocket = serverSocket.accept();

                // For each connection, submit a task to the executor.
                // It will be handled by a new virtual thread.
                executor.submit(() -> handleClient(clientSocket));
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static void handleClient(Socket clientSocket) {
        long clientId = clientCounter.incrementAndGet();
        System.out.println("Handling client #" + clientId + " on thread: " + Thread.currentThread());

        try (PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true)) {

            // Simulate a blocking I/O call (e.g., a database query)
            Thread.sleep(1000);

            String response = "HTTP/1.1 200 OK\r\n\r\n" + "Hello, you are client #" + clientId;
            out.println(response);

        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        } finally {
            try {
                clientSocket.close();
            } catch (IOException e) {
                // ignore
            }
        }
        System.out.println("Finished client #" + clientId);
    }
}
```

---

## 🚀 How to Run & Test

### Step 1: Compile the Code (with Preview Features)
```bash
# Make sure you are using JDK 19+
javac --release 19 --enable-preview src/com/example/server/*.java -d out
```

### Step 2: Run the Server (with Preview Features)
```bash
java --enable-preview -cp out com.example.server.VirtualThreadServer
```

### Step 3: Test with a Load Testing Tool
Ee server yokka real power chudali ante, `curl` or browser saripodu. Manaki thousands of concurrent connections create chese tool kavali. `apache-bench` (`ab`) or `hey` lanti tools perfect.

`hey` anedi oka modern, easy-to-use tool. Let's use that.
First, install `hey` (platform batti instructions marachu, e.g., on macOS: `brew install hey`).

Now, run this command to send **20,000 requests** with a concurrency of **5,000** (ante, at any time 5,000 connections open ga untayi):
```bash
hey -n 20000 -c 5000 http://localhost:8080
```

### Expected Output & Analysis
Nee server console lo, nuvvu thousands of "Handling client #..." messages chustav, prathi okati oka kotha virtual thread meeda run avtundi.
```
Handling client #4998 on thread: VirtualThread[#21451]/runnable@ForkJoinPool-1-worker-1
Handling client #4999 on thread: VirtualThread[#21453]/runnable@ForkJoinPool-1-worker-2
Handling client #5000 on thread: VirtualThread[#21455]/runnable@ForkJoinPool-1-worker-3
...
```
`hey` tool yokka output ilantidi untundi:
```
Summary:
  Total:	10.51 s
  Slowest:	2.52 s
  Fastest:	1.01 s
  Average:	1.53 s
  Requests/sec:	1902.13

  ...
```
**Analysis:**
- **Incredible Scalability:** Manam konni platform threads tho (`ForkJoinPool-1-worker-X`), 5000 concurrent connections ni handle chesam. `FixedThreadPool` tho idi cheyali ante, manaki 5000 platform threads kavali, adi system ni crash chestundi.
- **Simple Code:** Manam complex, asynchronous, non-blocking code rayaledu (like Netty or reactive frameworks). Manam simple, old-school, blocking "thread-per-request" code rasam. Kaani, virtual threads valla, adi automatically high-performance, non-blocking laaga behave chestundi. **Idi Project Loom yokka main selling point.**

Ee project tho, nuvvu Java lo server-side programming ni virtual threads ela completely change chestayo live ga chusav. Congratulations, mawa! 🚀
