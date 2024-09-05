How to use Completable Future for Parallelism.
==============================================

[![Vikas Taank](https://miro.medium.com/v2/resize:fill:88:88/1*myL9NCMtAPplh9SxQXZGNA.jpeg)](https://medium.com/@vikas.taank_40391?source=post_page-----dbdacad3c40f--------------------------------)

[Vikas Taank](https://medium.com/@vikas.taank_40391?source=post_page-----dbdacad3c40f--------------------------------)


![](https://miro.medium.com/v2/resize:fit:700/1*XHxQph1FlMJ6Fg_MNVhc_Q.png)

To ensure that two Completable Future operations in Java are executed in parallel rather than sequentially, we can initiate both operations without waiting for the first one to complete before starting the second. After both futures have been started, we need to make sure that the main thread waits for all to complete before proceeding.

> Here's a basic example to demonstrate how to execute two `CompletableFuture` tasks in Sequence and then combine their results:

1.  API Gateway calls API1.
2.  API Gateway Calls API2.
3.  Both the Calls are happening sequentially as the main thread is waiting for each call to get completed.
4.  The time taken to execute both the API's will be T1+T2.

![](https://miro.medium.com/v2/resize:fit:700/1*QSkwKj-_LfFA-TxoPmr1-Q.png)

> Here's a basic example to demonstrate how to execute two `CompletableFuture` tasks in Parallel and then combine their results:

![](https://miro.medium.com/v2/resize:fit:700/1*whmOA4sosjYIQp8Ul8Nc0A.png)

1.  API Gateway calls API1.
2.  API Gateway Calls API2.
3.  Both the Calls are happening in parallel as the main thread waits on both of the completable future to complete.
4.  The time taken to execute both the API's will be larger from T1 and T2.

import java.util.concurrent.CompletableFuture;\
import java.util.concurrent.ExecutionException;

public class CompletableFutureParallelExample {

    public static void main(String[] args) throws ExecutionException, InterruptedException {\
        // Task 1: A CompletableFuture that simulates a long-running operation\
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {\
            try {\
                // Simulate a long-running operation\
                Thread.sleep(2000); // 2 seconds\
            } catch (InterruptedException e) {\
                Thread.currentThread().interrupt();\
            }\
            return "Result of Future 1";\
        });

        // Task 2: Another CompletableFuture that also simulates a long-running operation\
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {\
            try {\
                // Simulate another long-running operation\
                Thread.sleep(1500); // 1.5 seconds\
            } catch (InterruptedException e) {\
                Thread.currentThread().interrupt();\
            }\
            return "Result of Future 2";\
        });

        // Wait for both futures to complete (in parallel)\
        CompletableFuture<Void> combinedFuture = CompletableFuture.allOf(future1, future2);

        // Block and wait for all futures to complete\
        combinedFuture.join();

        // Now that both futures are complete, we can extract their results\
        String result1 = future1.get(); // This won't block since the future is already complete\
        String result2 = future2.get(); // This won't block either

        System.out.println(result1);\
        System.out.println(result2);\
    }\
}

> Lets Understand what is happening in above example.

-   Two `CompletableFuture` tasks (`future1` and `future2`) are initiated asynchronously and execute in parallel. They simulate long-running operations using `Thread.sleep()`.
-   The `CompletableFuture.allOf(future1, future2)` method is used to create a `CompletableFuture<Void>` that is completed when both of the given CompletableFutures complete.
-   `combinedFuture.join()` is called to block the current thread until both `future1` and `future2` are complete. However, `future1` and `future2` are still running in parallel to each other.
-   After both futures are complete, their results are retrieved using `future1.get()` and `future2.get()`.

> These calls do not block because we've already waited for the futures to complete with `combinedFuture.join()`.

Hey But What if these are Reactive API's
========================================

![](https://miro.medium.com/v2/resize:fit:700/1*gBAB23Xp0qANif7NiFheOQ.png)

`Mono.zip` combines several publishers by aggregating their values into a `Tuple`, which can then be mapped or consumed as needed.

import reactor.core.publisher.Mono;

public class MonoParallelExample {\
public static void main(String[] args) {\
Mono<String> mono1 = WebClient.create("http://dumdum.com/api/endpoint1")\
.get()\
.retrieve()\
.bodyToMono(String.class);

        Mono<String> mono2 = WebClient.create("http://dumdum.com/api/endpoint2")\
                .get()\
                .retrieve()\
                .bodyToMono(String.class);

        Mono.zip(mono1, mono2)\
                .subscribe(tuple -> {\
                    String result1 = tuple.getT1();\
                    String result2 = tuple.getT2();\
                    System.out.println("Result 1: " + result1);\
                    System.out.println("Result 2: " + result2);\
                    // Further processing with result1 and result2\
                });\
    }\
}

Using `Mono.when` for Side Effects Only
=======================================

If you're only interested in knowing when both operations have completed, without necessarily combining their results, you can use Mono.when()

import reactor.core.publisher.Mono;

public class MonoParallelExample {\
public static void main(String[] args) {\
Mono<String> mono1 = WebClient.create("http://example.com/api/endpoint1")\
.get()\
.retrieve()\
.bodyToMono(String.class);

        Mono<String> mono2 = WebClient.create("http://example.com/api/endpoint2")\
                .get()\
                .retrieve()\
                .bodyToMono(String.class);

        Mono.when(mono1, mono2)\
                .subscribe(aVoid -> {\
                    // Both Monos have completed\
                    System.out.println("Both API calls completed");\
                    // Execute any follow-up action here\
                });\
    }\
}

Considerations for Parallel Execution
=====================================

-   Schedulers: By default, `Mono` operations execute on the same thread (reactor-http-nio thread when using WebClient).

> To truly execute tasks in parallel, especially for blocking or CPU-intensive tasks, you might want to subscribe to different schedulers using `.subscribeOn(Schedulers.boundedElastic())` or another appropriate scheduler.

-   Error Handling: Consider how errors from each `Mono` should be handled, especially in parallel scenarios. Using operators like `onErrorResume` can help manage errors in individual streams.