# 🏗️ Mini Project 1: Concurrent Web Server

## 🤔 Problem Statement
Mawa, manam oka basic HTTP web server ni build cheddam. Ee server okate sari multiple client requests ni handle cheyagalali. Oka client request slow ga unte, adi vere clients ni block cheyakudadu.

**Requirements:**
1.  Server port 8080 meeda listen cheyali.
2.  Prathi client connection ki, adi oka separate thread lo handle avvali.
3.  Server anedi fixed number of threads (e.g., 10) unna pool ni use cheyali to prevent resource exhaustion.
4.  Client ki "Hello from the Concurrent Server, mawa!" ane simple response ivvali.

**This Project Uses Concepts From:**
- ✅ [1.2: Creating & Managing Threads](../01-Introduction-to-Executor-Framework.md) (as a comparison)
- ✅ [6.2: Understanding Thread Pools](../02-Understanding-Thread-Pools.md) (`FixedThreadPool`)
- ✅ Basic Java Sockets (I/O)

---

## 🏗️ Architecture

Manam oka `ServerSocket` create chestam. Main thread anedi continuously incoming connections kosam wait chestu untundi (`serverSocket.accept()`). Oka connection రాగానే, adi aa client `Socket` object ni oka `RequestHandler` task ki ichi, daanini `FixedThreadPool` ki submit chestundi. Pool lo unna worker thread aa request ni handle chestundi.

```mermaid
graph TD
    subgraph Main Thread
        A[Start ServerSocket on Port 8080] --> B{Loop forever};
        B --> C[ServerSocket.accept()];
        C --> D[Client Socket created];
        D --> E[Create RequestHandler(socket)];
        E --> F[Submit task to ThreadPool];
        F --> B;
    end

    subgraph Thread Pool (10 Threads)
        G[Worker Thread 1]
        H[Worker Thread 2]
        I[...]
        J[Worker Thread 10]
    end

    F ==> G;
    G --> K[Handle HTTP Request & Send Response];

    style Main Thread fill:#87CEEB
    style Thread Pool fill:#90EE90
```

---

## 📁 Folder Structure
Ee project ki external dependencies em levu, so manaki simple structure saripotundi.

```
concurrent-server/
└── src/
    └── com/
        └── example/
            ├── ConcurrentWebServer.java
            └── RequestHandler.java
```

---

## 💻 Complete Code

#### File 1: `ConcurrentWebServer.java`
Idi mana main class. Server ni start chesi, thread pool ni create chesi, incoming connections ni accept chestundi.
```java
// File: src/com/example/ConcurrentWebServer.java
package com.example;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ConcurrentWebServer {

    private static final int PORT = 8080;
    private static final int THREAD_POOL_SIZE = 10;

    public static void main(String[] args) {
        // 1. ✅ Create a fixed thread pool to handle client requests.
        // Idi mana server ni overload kakunda kapadutundi.
        ExecutorService executor = Executors.newFixedThreadPool(THREAD_POOL_SIZE);

        try (ServerSocket serverSocket = new ServerSocket(PORT)) {
            System.out.println("🚀 Server is listening on port " + PORT);

            // 2. ✅ Continuously wait for client connections.
            while (true) {
                // 3. 🤝 Accept a new client connection. Idi blocking call.
                Socket clientSocket = serverSocket.accept();
                System.out.println("✅ Accepted connection from " + clientSocket.getInetAddress());

                // 4. 🎯 Create a task for the new client and submit it to the pool.
                Runnable requestHandler = new RequestHandler(clientSocket);
                executor.execute(requestHandler);
            }
        } catch (IOException e) {
            System.err.println("❌ Server exception: " + e.getMessage());
            // Production lo, ikkada proper logging undali.
            e.printStackTrace();
        } finally {
            // 5. 🛑 Real application lo, server ni stop cheyadaniki oka mechanism undali
            //    (e.g., another thread listening for a 'shutdown' command),
            //    appudu manam executor ni shutdown cheyali.
            //    Ee simple example lo, server eppudu run avtundi kabatti shutdown logic chupinchatledu.
            executor.shutdown();
        }
    }
}
```

#### File 2: `RequestHandler.java`
Idi `Runnable` task. Prathi client connection ni ide handle chestundi.
```java
// File: src/com/example/RequestHandler.java
package com.example;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.Socket;
import java.io.IOException;

public class RequestHandler implements Runnable {

    private final Socket clientSocket;

    public RequestHandler(Socket socket) {
        this.clientSocket = socket;
    }

    @Override
    public void run() {
        // Thread.currentThread().getName() use chesi, ఏ thread ee request handle
        // chestundo chudochu. Threads reuse avvadam manam gamaninchavachu.
        System.out.println("... Handling request on thread: " + Thread.currentThread().getName());

        try (
            // ✅ Try-with-resources automatically closes these resources.
            BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
            PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true);
        ) {
            // Read the HTTP request line (optional for this simple server)
            String requestLine = in.readLine();
            System.out.println("   -> Received: " + requestLine);

            // Simulate some work, like a database call
            Thread.sleep(2000); // 2 seconds delay

            // ✅ Send the HTTP response
            String httpResponse = "HTTP/1.1 200 OK\r\n\r\n" + "Hello from the Concurrent Server, mawa!";
            out.println(httpResponse);

        } catch (IOException e) {
            System.err.println("❌ I/O error on thread " + Thread.currentThread().getName() + ": " + e.getMessage());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // Restore the interrupted status
            System.err.println("❌ Task interrupted on thread " + Thread.currentThread().getName());
        } finally {
            try {
                clientSocket.close(); // Make sure the socket is always closed.
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        System.out.println("... Finished handling request on thread: " + Thread.currentThread().getName());
    }
}
```

---

## 🚀 How to Run & Test

### Step 1: Compile the Code
Command line lo, `concurrent-server` directory ki velli, compile chey.
```bash
# Navigate to the project root
cd concurrent-server

# Compile all Java files into the 'out' directory
javac -d out src/com/example/*.java
```

### Step 2: Run the Server
Compile ayyaka, server ni run chey.
```bash
# Run from the 'out' directory
java -cp out com.example.ConcurrentWebServer
```
Server start ayyi, "🚀 Server is listening on port 8080" ani message chupistundi.

### Step 3: Test the Server
#### Option A: Using a Web Browser
Nee browser open chesi, ee address ki vellu: `http://localhost:8080`
Page lo "Hello from the Concurrent Server, mawa!" ani kanipistundi. Notice chey, page load avvadaniki 2 seconds padutundi, endukante manam `Thread.sleep(2000)` pettam.

#### Option B: Using `curl` (for concurrent testing)
`curl` anedi command-line tool. Manam deenitho multiple requests ni parallel ga pampochu. Rendu separate terminal windows open chesi, okate sari ee command ni run chey:
```bash
# In Terminal 1
curl http://localhost:8080

# In Terminal 2 (run at the same time)
curl http://localhost:8080
```
Rendu terminals lo kuda 2 seconds tarvata response vastundi. Server anedi requests ni parallel ga handle chesindi. Nuvvu server console lo chuste, ee requests veru veru threads (`pool-1-thread-1`, `pool-1-thread-2`) meeda run avvadam chudochu.

**Congratulations, mawa!** Nuvvu nee first concurrent server ni successfully build chesav! 🎉
