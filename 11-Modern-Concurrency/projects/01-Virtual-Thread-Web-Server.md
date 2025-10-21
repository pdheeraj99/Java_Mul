<!--
---
title: "Mini-Project: High-Scalability Virtual Thread Web Server"
---
-->

> **Learning Path Position**
>
> Phase 11: Modern Concurrency ➔ **Hands-On Mini-Project: Virtual Thread Web Server**

> **Project Goal**
>
> Manam oka simple web server ni build cheddam, adi chala concurrent requests ni handle cheyyagaladu. Ee project lo, manam virtual threads valla vache dramatic scalability improvement ni chustam. Mundu, manam server ni traditional platform threads tho ("thread-per-request" model) build chesi, adi ekkuva load vachinappudu ela fail avutundo chustam. Taruvatha, okate okka line code marchi, daanni virtual threads tho run chesi, adi thousands of concurrent requests ni entha easy ga handle chestundo చూస్తాం.

---

### 🚀 1. Project Overview

Mawa, web servers, microservices lanti network applications ki virtual threads anevi oka game-changer. Endukante, prathi request lo edo oka I/O operation untundi (database query, another service call, etc.). Traditional platform threads ee I/O kosam wait chestu block aipotayi.

Ee project lo manam Java lo unna built-in `HttpServer` use chesi, rendu versions of a server build cheddam:
1.  **`PlatformThreadWebServer`**: Fixed thread pool use chestundi. Prathi request ki oka platform thread assign avutundi.
2.  **`VirtualThreadWebServer`**: Prathi request ki oka kottha virtual thread ni create chestundi.

Manam renditini load test chesi, performance difference entha peddado chuddam.

###🎯 2. Learning Objectives

*   Platform thread pools scalability limitations ni practical ga chudatam.
*   I/O-bound server applications lo virtual threads valla vache massive benefits ni ardham chesukovadam.
*   `Executors.newVirtualThreadPerTaskExecutor()` ni oka real-world scenario lo use cheyyadam.
*   Blocking code ni virtual threads tho migrate cheyyadam entha easy o telusukovadam.

### 🛠️ 3. Project Setup

*   **Java 21+:** Virtual threads final aindi Java 21 lo, so manaki adi kavali.
*   **`HttpServer`:** Java tho paatu vastundi, external libraries em avasaram ledu.
*   **Load Tester:** Manam `curl` ane command-line tool ni use cheddam. Prathi OS lo untundi. Manam oka simple shell script tho server mida load generate cheddam.

---

### PART 1: The Platform Thread Server (The Old Way)

Ee model lo, manam 100 platform threads unna oka fixed pool create cheddam. Ante, mana server okesari 100 concurrent requests ni matrame handle cheyyagaladu. 101st request vachinappudu, adi pool lo oka thread free ayye varaku wait cheyyali.

#### Step 1: Write the Server Code

Ee code oka server ni port 8080 lo start chestundi. `/test` ane endpoint ki request vachinappudu, adi 2 seconds aagi, "Hello" ani response istundi. Aa 2 seconds sleep anedi oka slow database call ni simulate chestundi.

```java
// PlatformThreadWebServer.java
import com.sun.net.httpserver.HttpServer;
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.concurrent.Executors;
import java.util.concurrent.ExecutorService;

public class PlatformThreadWebServer {
    public static void main(String[] args) throws IOException {
        System.out.println("Platform Thread Server start avutondi...");

        // 100 platform threads unna oka fixed pool. Idi mana bottleneck.
        ExecutorService executor = Executors.newFixedThreadPool(100);

        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);

        server.createContext("/test", exchange -> {
            // Prathi request ee executor lo run avutundi
            byte[] response = "Hello from Platform Thread Server!\n".getBytes(StandardCharsets.UTF_8);
            try {
                // DB call ni simulate cheddam
                Thread.sleep(Duration.ofSeconds(2));
            } catch (InterruptedException e) {
                // ...
            }
            exchange.sendResponseHeaders(200, response.length);
            try (OutputStream os = exchange.getResponseBody()) {
                os.write(response);
            }
        });

        // Server ki mana custom executor ni set cheddam
        server.setExecutor(executor);
        server.start();
        System.out.println("Server port 8080 lo listen chestondi.");
    }
}
```

#### Step 2: Compile and Run

```bash
javac PlatformThreadWebServer.java
java PlatformThreadWebServer
```

#### Step 3: Load Test It!

Inko terminal open chesi, ee shell script run cheyyi. Idi 200 requests ni okesari (in background `&`) pampistundi.

```bash
#!/bin/bash
echo "Sending 200 concurrent requests to the Platform Thread Server..."
for i in {1..200}
do
   # time command valla entha time pattindo chudochu
   # -s flag valla progress meter kanipinchadu
   time curl -s http://localhost:8080/test > /dev/null &
done
wait # Anni background jobs aipoyevaraku wait cheyyi
echo "All requests finished."
```

#### Step 4: Analyze the Results

Nuvvu chustav:
*   First 100 requests ventane start avutayi.
*   Aa 100 requests aipodaniki ~2 seconds padutundi.
*   **Migatha 100 requests appativaraku hang avutayi!** Avi pool lo threads free avvadam kosam wait chestayi.
*   Total time anedi ~4 seconds (2 rounds of 100) avutundi. Server okesari 100 requests kanna ekkuva handle cheyyalekapoindi.

---

### PART 2: The Virtual Thread Server (The New Way)

Ippudu, manam okate okka line marchi, server ni virtual threads tho run cheddam.

#### Step 1: Write the Server Code

Chudu, code antha same. Okate okka line marindi: `Executors.newFixedThreadPool(100)` badulu `Executors.newVirtualThreadPerTaskExecutor()`.

```java
// VirtualThreadWebServer.java
import com.sun.net.httpserver.HttpServer;
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.concurrent.Executors;
import java.util.concurrent.ExecutorService;

public class VirtualThreadWebServer {
    public static void main(String[] args) throws IOException {
        System.out.println("Virtual Thread Server start avutondi...");

        // Ee okka line marindi! Prathi task ki oka kottha virtual thread create cheyyi.
        ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

        HttpServer server = HttpServer.create(new InetSocketAddress(8080), 0);

        server.createContext("/test", exchange -> {
            byte[] response = "Hello from Virtual Thread Server!\n".getBytes(StandardCharsets.UTF_8);
            try {
                // Ee sleep call OS thread ni block cheyyadu
                Thread.sleep(Duration.ofSeconds(2));
            } catch (InterruptedException e) {
                // ...
            }
            exchange.sendResponseHeaders(200, response.length);
            try (OutputStream os = exchange.getResponseBody()) {
                os.write(response);
            }
        });

        server.setExecutor(executor);
        server.start();
        System.out.println("Server port 8080 lo listen chestondi.");
    }
}
```

#### Step 2: Compile and Run

```bash
javac VirtualThreadWebServer.java
java VirtualThreadWebServer
```

#### Step 3: Load Test It!

Malli ade script run cheyyi.

```bash
#!/bin/bash
echo "Sending 200 concurrent requests to the Virtual Thread Server..."
for i in {1..200}
do
   time curl -s http://localhost:8080/test > /dev/null &
done
wait
echo "All requests finished."
```

#### Step 4: Analyze the Results

Ippudu nuvvu chuse results chala different ga untayi:
*   Anni 200 requests okesari start avutayi. Edi hang avvadu.
*   Total time antha antha ~2 seconds matrame untundi. Endukante, anni 200 "sleep" operations parallel ga jarugutunnayi, OS threads ni block cheyyakunda.
*   Ee server 1000s of concurrent requests ni kuda ilage handle cheyyagaladu.

---

### 🔑 6. Key Takeaways from the Project

1.  **Bottleneck Exposed:** Traditional thread-per-request model lo, thread pool anedi scalability ki pedda bottleneck.
2.  **Virtual Threads for I/O:** I/O-bound tasks (DB calls, network requests) unna server applications kosam virtual threads perfect solution. Avi OS threads ni block cheyyavu, deeni valla massive scalability vastundi.
3.  **Easy Migration:** Manam chusam kada, mana application logic (`Handler`) lo okka line kuda marchalsina avasaram raledu. Just `ExecutorService` implementation ni swap cheste chalu. Existing blocking code ni virtual threads tho run cheyyadam chala easy.
4.  **No More Pooling (for Virtual Threads):** Virtual threads chala cheap kabatti, vaatini pool cheyyalsina avasaram ledu. "Prathi task ki oka kottha thread" anedi virtual threads tho best practice.
