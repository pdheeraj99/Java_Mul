# 🏗️ Mini Project: Asynchronous API Aggregator

## 🤔 Problem Statement
Mawa, idi oka chala common real-world scenario. Manam oka "Product Details" page kosam data ni prepare cheyali. Ee data anedi 3 different microservices nunchi vastundi:
1.  **Product Service:** Product's basic info (name, price) istundi.
2.  **Inventory Service:** Product stock level istundi.
3.  **Review Service:** Product reviews and rating istundi.

Ee 3 network calls independent ga unnayi. Manam vaatini sequentially okati tarvata okati call cheste, total time anedi `(time of call 1 + time of call 2 + time of call 3)` avtundi. Chala slow! Manam `CompletableFuture` use chesi, ee 3 calls ni parallel ga chesi, vaati results ni combine chesi, final `ProductDetails` object ni create cheyali.

**This Project Uses Concepts From:**
- ✅ [7.2: Introduction to CompletableFuture](../02-Introduction-to-CompletableFuture.md) (`supplyAsync`)
- ✅ [7.3: Chaining and Combining CompletableFutures](../03-Chaining-and-Combining-CompletableFutures.md) (`thenCombine`)
- ✅ [6.3: Custom ThreadPoolExecutor](../../06-Executor-Framework/03-ThreadPoolExecutor-Deep-Dive.md) (for I/O tasks)

---

## 🏗️ Architecture
Manam 3 different `CompletableFuture`s ni create chestam, prathi okati oka separate service call ni represent chestundi. Ee 3 futures ni manam `thenCombine` use chesi merge chestam. Rendu futures ni combine chesi, aa result ni third future tho combine chestam. Final ga, anni results vachaka, manam `ProductDetails` DTO (Data Transfer Object) ni build chestam. Network calls blocking operations kabatti, manam custom `ThreadPoolExecutor` vadali.

```mermaid
graph TD
    A[Start] --> B[Get Product ID];

    subgraph Parallel Execution (Custom Executor)
        B --> C[CF_Product: GetProductInfo(id)];
        B --> D[CF_Inventory: GetInventory(id)];
        B --> E[CF_Reviews: GetReviews(id)];
    end

    C --> F[thenCombine];
    D --> F[thenCombine];
    F --> G[CF_Combined: (ProductInfo, Inventory)];

    G --> H[thenCombine];
    E --> H[thenCombine];
    H --> I[CF_Final: (Combined, Reviews)];

    I --> J[thenApply: Build ProductDetails DTO];
    J --> K[Final Result: CompletableFuture<ProductDetails>];
    K --> L[Print Result];

    style Parallel Execution fill:#90EE90
```

---

## 💻 Complete Code

#### File 1: `ApiAggregator.java` (Main Class)
```java
// File: src/com/example/ApiAggregator.java
package com.example;

import com.example.dto.ProductDetails;
import com.example.service.InventoryService;
import com.example.service.ProductService;
import com.example.service.ReviewService;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ApiAggregator {

    public static void main(String[] args) {
        // ✅ I/O operations kosam custom thread pool create cheddam
        ExecutorService executor = Executors.newFixedThreadPool(10);

        ProductService productService = new ProductService();
        InventoryService inventoryService = new InventoryService();
        ReviewService reviewService = new ReviewService();

        int productId = 101;

        // 1. Create 3 independent CompletableFutures, each running on our custom executor
        CompletableFuture<String> productFuture = CompletableFuture.supplyAsync(
            () -> productService.getProductInfo(productId), executor);

        CompletableFuture<Integer> inventoryFuture = CompletableFuture.supplyAsync(
            () -> inventoryService.getStock(productId), executor);

        CompletableFuture<String> reviewFuture = CompletableFuture.supplyAsync(
            () -> reviewService.getReviews(productId), executor);

        System.out.println("🚀 Fired off 3 API calls asynchronously.");

        // 2. Combine product and inventory futures
        CompletableFuture<ProductDetails> combinedFuture = productFuture.thenCombine(inventoryFuture, (productInfo, stock) -> {
            ProductDetails details = new ProductDetails();
            details.setProductInfo(productInfo);
            details.setStock(stock);
            return details;
        });

        // 3. Combine the result with the review future
        CompletableFuture<ProductDetails> finalFuture = combinedFuture.thenCombine(reviewFuture, (details, reviews) -> {
            details.setReviews(reviews);
            return details;
        });

        // 4. Wait for the final result and print it
        ProductDetails finalDetails = finalFuture.join(); // Using join() for simplicity in main method

        System.out.println("\n🎉 Aggregated Product Details:\n" + finalDetails);

        executor.shutdown();
    }
}
```

#### File 2: DTO and Service Classes
Real-world lo ivi separate files lo untayi. Ikkada readability kosam okate chota pedtunna.
```java
// File: src/com/example/dto/ProductDetails.java
package com.example.dto;

public class ProductDetails {
    private String productInfo;
    private int stock;
    private String reviews;

    // Getters and Setters
    public void setProductInfo(String info) { this.productInfo = info; }
    public void setStock(int stock) { this.stock = stock; }
    public void setReviews(String reviews) { this.reviews = reviews; }

    @Override
    public String toString() {
        return "  ProductInfo: '" + productInfo + "'\n" +
               "  Stock: " + stock + "\n" +
               "  Reviews: '" + reviews + "'";
    }
}

// File: src/com/example/service/ProductService.java
package com.example.service;
import java.util.concurrent.TimeUnit;

public class ProductService {
    public String getProductInfo(int id) {
        System.out.println("  -> Calling ProductService...");
        try { TimeUnit.SECONDS.sleep(2); } catch (Exception e) {}
        System.out.println("  <- ProductService responded.");
        return "Product Name: Awesome Gadget, Price: $99.99";
    }
}

// File: src/com/example/service/InventoryService.java
package com.example.service;
import java.util.concurrent.TimeUnit;

public class InventoryService {
    public int getStock(int id) {
        System.out.println("  -> Calling InventoryService...");
        try { TimeUnit.SECONDS.sleep(3); } catch (Exception e) {} // This one is the slowest
        System.out.println("  <- InventoryService responded.");
        return 50;
    }
}

// File: src/com/example/service/ReviewService.java
package com.example.service;
import java.util.concurrent.TimeUnit;

public class ReviewService {
    public String getReviews(int id) {
        System.out.println("  -> Calling ReviewService...");
        try { TimeUnit.SECONDS.sleep(1); } catch (Exception e) {}
        System.out.println("  <- ReviewService responded.");
        return "4.5 Stars, 150 Reviews";
    }
}
```

---

## 🚀 How to Run & Test

### Step 1: Compile the Code
```bash
# Assuming you are in the project root
javac -d out src/com/example/*.java src/com/example/dto/*.java src/com/example/service/*.java
```

### Step 2: Run the Aggregator
```bash
java -cp out com.example.ApiAggregator
```

### Expected Output & Analysis
```
🚀 Fired off 3 API calls asynchronously.
  -> Calling ProductService...
  -> Calling InventoryService...
  -> Calling ReviewService...
  <- ReviewService responded.
  <- ProductService responded.
  <- InventoryService responded.

🎉 Aggregated Product Details:
  ProductInfo: 'Product Name: Awesome Gadget, Price: $99.99'
  Stock: 50
  Reviews: '4.5 Stars, 150 Reviews'
```
**Analysis:**
- Notice chey, anni 3 service calls okate sari start ayyayi ("-> Calling...").
- Slowest service (`InventoryService`, 3 seconds) response vachesariki, migatha rendu services (`ProductService`, `ReviewService`) already complete aypoyayi.
- Total time taken anedi, slowest call time (3 seconds) ki almost equal ga untundi, not the sum of all calls (1+2+3 = 6 seconds). Manam 3 seconds save chesam! Idi `CompletableFuture` yokka magic. ✨

Ee project tho, nuvvu asynchronous calls ni parallel ga run chesi, vaati results ni ela combine cheyalo chusav. Production-level code lo idi chala common pattern. Great job, mawa!
