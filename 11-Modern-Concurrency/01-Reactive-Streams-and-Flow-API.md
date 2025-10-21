---
# 🎯 11.1: Reactive Streams & The Flow API

## 🗺️ Learning Path Position

**📍 You Are Here:** 11.1 - Reactive Streams & The Flow API

**✅ Prerequisites (Must Complete First):**
- [8.3: Blocking Queues for Producer-Consumer](../08-Concurrent-Collections/03-Blocking-Queues-for-Producer-Consumer.md) - Producer-Consumer pattern and the problem of fast producers gurinchi ardham cheskovali.

**🔜 Coming After This:**
- [11.2: Virtual Threads - Project Loom](./02-Virtual-Threads-Project-Loom.md) - Modern concurrency lo inko revolutionary idea gurinchi nerchukuntam.

**⏱️ Estimated Time:** 3 hours
**📊 Difficulty Level:** 🔴 Advanced

---

## 🤔 What & Why

### The Problem: Data Overload! 🌊
Mawa, manam Producer-Consumer pattern chusam. Producer fast ga unte, consumer slow ga unte, `BlockingQueue` lanti bounded buffer use chesi manam system crash avvakunda (`OutOfMemoryError`) kapadatham. Ee mechanism ni manam **back-pressure** antam.

Kaani, idi "blocking" back-pressure. Producer antha block aypotadu. Inka better approach unda? Asynchronous, non-blocking streams of data ni ela handle cheyali? E.g., Twitter firehose nunchi vache infinite stream of tweets, or a stock market feed. Ee data ni manam request cheyam, adi push avtundi. Consumer anedi overwhelmed avvakunda ee "push" stream ni ela control cheyagaladu?

### The Solution: Reactive Streams & The Publisher-Subscriber Model
Reactive Streams anedi oka specification (a set of rules and interfaces). Ee rules ni follow ayye libraries anni (like RxJava, Project Reactor) asynchronous streams ni back-pressure tho handle cheyagalavu.

**Core Idea:** The consumer is in control. "Push" model ni "push-pull hybrid" model ga marchadam.
- **Publisher:** Data source. Adi data ni emit chestundi.
- **Subscriber:** Data consumer. Adi data ni receive cheskuntundi.
- **Subscription:** Publisher ki Subscriber ki madhyalo unna connection.
- **Back-Pressure in Action:** Subscriber anedi subscription start ayinappudu, publisher ki cheptundi: "Naku ippudu `n` items matrame pampu" (`subscription.request(n)`). Publisher `n` items pampina tarvata aagipotadu. Subscriber aa `n` items ni process chesaka, malli publisher ni inkonni items kosam request chestadu. Ee "pull" mechanism valla, consumer eppudu overwhelmed avvadu.

### Real-World Analogy: Newspaper Subscription 📰
- **Publisher:** Newspaper company (e.g., Eenadu).
- **Subscriber:** Nuvvu.
- **`onSubscribe`:** Nuvvu subscription a form fill chesi istav. Company antha accept chesi, neeku oka subscription ID istundi (`Subscription` object).
- **`request(n)`:** Nuvvu company ki call chesi, "Next 7 days ki matrame papers pampandi, nenu vacation lo unta" ani cheptav.
- **`onNext`:** Daily paper nee intiki vastundi.
- **`onError`:** Oka roju printing press lo problem vachi, paper publish avvakapothe, neeku oka sorry message vastundi.
- **`onComplete`:** Nee subscription aypoyaka, final "Thank you" letter vastundi.

---

## 📚 Detailed Explanation: The `java.util.concurrent.Flow` API
Java 9 lo, ee Reactive Streams specification ni standard library lo `java.util.concurrent.Flow` ane nested interface laaga add chesaru. Deentlo 4 core interfaces untayi:

1.  **`Flow.Publisher<T>`:**
    - `void subscribe(Flow.Subscriber<? super T> subscriber);` - Okate method. Subscriber ni register cheskuntundi.

2.  **`Flow.Subscriber<T>`:**
    - `void onSubscribe(Flow.Subscription subscription);` - Subscription start ayyaka, publisher ee method ni call chestundi. Ikkade manam `subscription.request()` call cheyali.
    - `void onNext(T item);` - Publisher nunchi kotha data item vachinappudu ee method call avtundi.
    - `void onError(Throwable throwable);` - Edaina error vasthe call avtundi. Terminal state.
    - `void onComplete();` - Data stream end aypothe call avtundi. Terminal state.

3.  **`Flow.Subscription`:**
    - `void request(long n);` - Inka enni items kavalo publisher ki cheptundi.
    - `void cancel();` - Subscription ni cancel cheyadaniki.

4.  **`Flow.Processor<T, R>`:**
    - Publisher and Subscriber rendu laaga pani chestundi. Oka stream nunchi data teskuni, daanini transform chesi, vere stream ki publish chestundi. (e.g., `map` operator).

---

## 💻 Code Example: A Simple Publisher-Subscriber

Manam `SubmissionPublisher` (Java 9+ lo unna built-in `Flow.Publisher` implementation) use chesi, oka simple publisher-subscriber ni create cheddam.

```java
// File: com/example/flow/FlowApiDemo.java
package com.example.flow;

import java.util.concurrent.Flow;
import java.util.concurrent.SubmissionPublisher;
import java.util.stream.IntStream;

// Our custom subscriber
class MySubscriber implements Flow.Subscriber<Integer> {
    private Flow.Subscription subscription;
    private final int demand;
    private int count = 0;

    MySubscriber(int demand) { this.demand = demand; }

    @Override
    public void onSubscribe(Flow.Subscription subscription) {
        System.out.println("SUBSCRIBER: onSubscribe - Subscription received!");
        this.subscription = subscription;
        // ✅ Initial request for items
        System.out.println("SUBSCRIBER: Requesting " + demand + " items.");
        subscription.request(demand);
    }

    @Override
    public void onNext(Integer item) {
        count++;
        System.out.println("SUBSCRIBER: onNext - Got item: " + item);
        // After processing 'demand' items, request more
        if (count % demand == 0) {
            System.out.println("SUBSCRIBER: Processed a batch. Requesting " + demand + " more items.");
            subscription.request(demand);
        }
    }

    @Override
    public void onError(Throwable throwable) {
        System.err.println("SUBSCRIBER: onError - " + throwable.getMessage());
    }

    @Override
    public void onComplete() {
        System.out.println("SUBSCRIBER: onComplete - All done!");
    }
}

public class FlowApiDemo {
    public static void main(String[] args) throws InterruptedException {
        // 1. Create a Publisher
        SubmissionPublisher<Integer> publisher = new SubmissionPublisher<>();

        // 2. Create a Subscriber (with a demand of 3)
        MySubscriber subscriber = new MySubscriber(3);

        // 3. Subscribe
        publisher.subscribe(subscriber);

        // 4. Publish items
        System.out.println("PUBLISHER: Publishing items...");
        IntStream.range(1, 10).forEach(publisher::submit);

        // Let it run for a bit
        Thread.sleep(1000);

        // 5. Close the publisher (which will call onComplete)
        publisher.close();

        System.out.println("PUBLISHER: Closed.");
    }
}
```

---

## ✅ Checkpoint: Did You Master This?

- [ ] Reactive Streams lo, "back-pressure" anedi ela pani chestundi? Evaru control lo untaru?
- [ ] `Flow.Subscriber` lo `onSubscribe` method yokka main purpose enti?
- [ ] `onNext`, `onError`, `onComplete` - ee events lo, evi "terminal" events?
- [ ] Java lo direct ga `Flow` API enduku vadakunda, RxJava or Project Reactor lanti libraries vadataru? (Hint: `Flow` API is just the foundation).

**✅ Ready?** → [Next: Virtual Threads - Project Loom](./02-Virtual-Threads-Project-Loom.md)
