-   [Reactor 3 Reference Guide](https://projectreactor.io/docs/core/milestone/reference/aboutDoc.html)
-   [Introduction to Reactive Programming](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html)

[Introduction to Reactive Programming](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html)
====================================

-   [1\. Blocking Can Be Wasteful](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#blocking-can-be-wasteful)
-   [2\. Asynchronicity to the Rescue?](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#asynchronicity-to-the-rescue)
-   [3\. From Imperative to Reactive Programming](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#from-imperative-to-reactive-programming)

Reactor is an implementation of the Reactive Programming paradigm, which can be summed up as follows:

> Reactive programming is an asynchronous programming paradigm concerned with data streams and the propagation of change. This means that it becomes possible to express static (e.g. arrays) or dynamic (e.g. event emitters) data streams with ease via the employed programming language(s).

--- https://en.wikipedia.org/wiki/Reactive_programming

As a first step in the direction of reactive programming, Microsoft created the Reactive Extensions (Rx) library in the .NET ecosystem. Then RxJava implemented reactive programming on the JVM. As time went on, a standardization for Java emerged through the Reactive Streams effort, a specification that defines a set of interfaces and interaction rules for reactive libraries on the JVM. Its interfaces have been integrated into Java 9 under the `Flow` class.

The reactive programming paradigm is often presented in object-oriented languages as an extension of the Observer design pattern. You can also compare the main reactive streams pattern with the familiar Iterator design pattern, as there is a duality to the `Iterable`-`Iterator` pair in all of these libraries. One major difference is that, while an Iterator is pull-based, reactive streams are push-based.

Using an iterator is an imperative programming pattern, even though the method of accessing values is solely the responsibility of the `Iterable`. Indeed, it is up to the developer to choose when to access the `next()` item in the sequence. In reactive streams, the equivalent of the above pair is `Publisher-Subscriber`. But it is the `Publisher` that notifies the Subscriber of newly available values *as they come*, and this push aspect is the key to being reactive. Also, operations applied to pushed values are expressed declaratively rather than imperatively: The programmer expresses the logic of the computation rather than describing its exact control flow.

In addition to pushing values, the error-handling and completion aspects are also covered in a well defined manner. A `Publisher` can push new values to its `Subscriber` (by calling `onNext`) but can also signal an error (by calling `onError`) or completion (by calling `onComplete`). Both errors and completion terminate the sequence. This can be summed up as follows:

```
onNext x 0..N [onError | onComplete]
```

This approach is very flexible. The pattern supports use cases where there is no value, one value, or n values (including an infinite sequence of values, such as the continuing ticks of a clock).

But why do we need such an asynchronous reactive library in the first place?

[1. Blocking Can Be Wasteful](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#blocking-can-be-wasteful)
-----------------------------------------------------------------------------------------------------------------------------------------

Modern applications can reach huge numbers of concurrent users, and, even though the capabilities of modern hardware have continued to improve, performance of modern software is still a key concern.

There are, broadly, two ways one can improve a program's performance:

-   parallelize to use more threads and more hardware resources.

-   seek more efficiency in how current resources are used.

Usually, Java developers write programs by using blocking code. This practice is fine until there is a performance bottleneck. Then it is time to introduce additional threads, running similar blocking code. But this scaling in resource utilization can quickly introduce contention and concurrency problems.

Worse still, blocking wastes resources. If you look closely, as soon as a program involves some latency (notably I/O, such as a database request or a network call), resources are wasted because threads (possibly many threads) now sit idle, waiting for data.

So the parallelization approach is not a silver bullet. It is necessary to access the full power of the hardware, but it is also complex to reason about and susceptible to resource wasting.

[2\. Asynchronicity to the Rescue?](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#asynchronicity-to-the-rescue)
--------------------------------------------------------------------------------------------------------------------------------------------------

The second approach mentioned earlier, seeking more efficiency, can be a solution to the resource wasting problem. By writing asynchronous, non-blocking code, you let the execution switch to another active task that uses the same underlying resources and later comes back to the current process when the asynchronous processing has finished.

But how can you produce asynchronous code on the JVM? Java offers two models of asynchronous programming:

-   Callbacks: Asynchronous methods do not have a return value but take an extra `callback` parameter (a lambda or anonymous class) that gets called when the result is available. A well known example is Swing's `EventListener` hierarchy.

-   Futures: Asynchronous methods *immediately* return a `Future<T>`. The asynchronous process computes a `T` value, but the `Future` object wraps access to it. The value is not immediately available, and the object can be polled until the value is available. For instance, an `ExecutorService` running `Callable<T>` tasks use `Future` objects.

Are these techniques good enough? Not for every use case, and both approaches have limitations.

Callbacks are hard to compose together, quickly leading to code that is difficult to read and maintain (known as "Callback Hell").

Consider an example: showing the top five favorites from a user on the UI or suggestions if she does not have a favorite. This goes through three services (one gives favorite IDs, the second fetches favorite details, and the third offers suggestions with details), as follows:

Example of Callback Hell

```
userService.getFavorites(userId, new Callback<List<String>>() { 1.
  public void onSuccess(List<String> list) { 2.
    if (list.isEmpty()) { 3.
      suggestionService.getSuggestions(new Callback<List<Favorite>>() {
        public void onSuccess(List<Favorite> list) { 4.
          UiUtils.submitOnUiThread(() -> { 5.
            list.stream()
                .limit(5)
                .forEach(uiList::show); 6.
            });
        }

        public void onError(Throwable error) { 7.
          UiUtils.errorPopup(error);
        }
      });
    } else {
      list.stream() 8.
          .limit(5)
          .forEach(favId -> favoriteService.getDetails(favId, 9.
            new Callback<Favorite>() {
              public void onSuccess(Favorite details) {
                UiUtils.submitOnUiThread(() -> uiList.show(details));
              }

              public void onError(Throwable error) {
                UiUtils.errorPopup(error);
              }
            }
          ));
    }
  }

  public void onError(Throwable error) {
    UiUtils.errorPopup(error);
  }
});
```

```
1. We have callback-based services: a `Callback` interface with a method invoked when the asynchronous process was successful and one invoked when an error occurs. |
2. The first service invokes its callback with the list of favorite IDs. |
3. If the list is empty, we must go to the `suggestionService`. |
4. The `suggestionService` gives a `List<Favorite>` to a second callback. |
5. Since we deal with a UI, we need to ensure our consuming code runs in the UI thread. |
6. We use a Java 8 `Stream` to limit the number of suggestions processed to five, and we show them in a graphical list in the UI. |
7. At each level, we deal with errors the same way: We show them in a popup. |
8. Back to the favorite ID level. If the service returned a full list, we need to go to the `favoriteService` to get detailed `Favorite` objects. Since we want only five, we first stream the list of IDs to limit it to five. |
9. Once again, a callback. This time we get a fully-fledged `Favorite` object that we push to the UI inside the UI thread. |
```

That is a lot of code, and it is a bit hard to follow and has repetitive parts. Consider its equivalent in Reactor:

Example of Reactor code equivalent to callback code

```
userService.getFavorites(userId) 1.
           .flatMap(favoriteService::getDetails) 2. 
           .switchIfEmpty(suggestionService.getSuggestions()) 3.
           .take(5) 4.
           .publishOn(UiUtils.uiThreadScheduler()) 5.
           .subscribe(uiList::show, UiUtils::errorPopup); 6.
```
```
1. We start with a flow of favorite IDs. |
2. We *asynchronously transform* these into detailed `Favorite` objects (`flatMap`). We now have a flow of `Favorite`. |
3. If the flow of `Favorite` is empty, we switch to a fallback through the `suggestionService`. |
4. We are only interested in, at most, five elements from the resulting flow. |
5. At the end, we want to process each piece of data in the UI thread. |
6. We trigger the flow by describing what to do with the final form of the data (show it in a UI list) and what to do in case of an error (show a popup). |
```

What if you want to ensure the favorite IDs are retrieved in less than 800ms or, if it takes longer, get them from a cache? In the callback-based code, that is a complicated task. In Reactor it becomes as easy as adding a `timeout` operator in the chain, as follows:

Example of Reactor code with timeout and fallback

```
userService.getFavorites(userId)
           .timeout(Duration.ofMillis(800)) 1.
           .onErrorResume(cacheService.cachedFavoritesFor(userId)) 2. 
           .flatMap(favoriteService::getDetails) 3.
           .switchIfEmpty(suggestionService.getSuggestions())
           .take(5)
           .publishOn(UiUtils.uiThreadScheduler())
           .subscribe(uiList::show, UiUtils::errorPopup);
```
```
1. If the part above emits nothing for more than 800ms, propagate an error. |
2. In case of an error, fall back to the `cacheService`. |
3. The rest of the chain is similar to the previous example. |
```

`Future` objects are a bit better than callbacks, but they still do not do well at composition, despite the improvements brought in Java 8 by `CompletableFuture`. Orchestrating multiple `Future` objects together is doable but not easy. Also, `Future` has other problems:

-   It is easy to end up with another blocking situation with `Future` objects by calling the `get()` method.

-   They do not support lazy computation.

-   They lack support for multiple values and advanced error handling.

Consider another example: We get a list of IDs from which we want to fetch a name and a statistic and combine these pair-wise, all of it asynchronously. The following example does so with a list of type `CompletableFuture`:

Example of `CompletableFuture` combination

```
CompletableFuture<List<String>> ids = ifhIds(); 1.

CompletableFuture<List<String>> result = ids.thenComposeAsync(l -> { 2.
	Stream<CompletableFuture<String>> zip =
			l.stream().map(i -> { 3.
				CompletableFuture<String> nameTask = ifhName(i); 4. 
				CompletableFuture<Integer> statTask = ifhStat(i); 5.

				return nameTask.thenCombineAsync(statTask, (name, stat) -> "Name " + name + " has stats " + stat); 6.
			});
	List<CompletableFuture<String>> combinationList = zip.collect(Collectors.toList()); 7.
	CompletableFuture<String>[] combinationArray = combinationList.toArray(new CompletableFuture[combinationList.size()]);

	CompletableFuture<Void> allDone = CompletableFuture.allOf(combinationArray); 8.
	return allDone.thenApply(v -> combinationList.stream() 
			.map(CompletableFuture::join) 9.
			.collect(Collectors.toList()));
});

List<String> results = result.join(); 10.
assertThat(results).contains(
		"Name NameJoe has stats 103",
		"Name NameBart has stats 104",
		"Name NameHenry has stats 105",
		"Name NameNicole has stats 106",
		"Name NameABSLAJNFOAJNFOANFANSF has stats 121");
```
```
1. We start off with a future that gives us a list of `id` values to process. |
2. We want to start some deeper asynchronous processing once we get the list. |
3. For each element in the list: |
4. Asynchronously get the associated name. |
5. Asynchronously get the associated statistic. |
6. Combine both results. |
7. We now have a list of futures that represent all the combination tasks. To execute these tasks, we need to convert the list to an array. |
8. Pass the array to `CompletableFuture.allOf`, which outputs a `Future` that completes when all tasks have completed. |
9. The tricky bit is that `allOf` returns `CompletableFuture<Void>`, so we reiterate over the list of futures, collecting their results by using `join()` (which, here, does not block, since `allOf` ensures the futures are all done). |
10. Once the whole asynchronous pipeline has been triggered, we wait for it to be processed and return the list of results that we can assert. |
```

Since Reactor has more combination operators out of the box, this process can be simplified, as follows:

Example of Reactor code equivalent to future code

```
Flux<String> ids = ifhrIds(); 1.

Flux<String> combinations =
		ids.flatMap(id -> { 2.
			Mono<String> nameTask = ifhrName(id); 3.
			Mono<Integer> statTask = ifhrStat(id); 4.

			return nameTask.zipWith(statTask, 5.
					(name, stat) -> "Name " + name + " has stats " + stat);
		});

Mono<List<String>> result = combinations.collectList(); 6.

List<String> results = result.block(); 7.
assertThat(results).containsExactly( 8.
		"Name NameJoe has stats 103",
		"Name NameBart has stats 104",
		"Name NameHenry has stats 105",
		"Name NameNicole has stats 106",
		"Name NameABSLAJNFOAJNFOANFANSF has stats 121"
);
```
```
1. This time, we start from an asynchronously provided sequence of `ids` (a `Flux<String>`). |
2. For each element in the sequence, we asynchronously process it (inside the function that is the body `flatMap` call) twice. |
3. Get the associated name. |
4. Get the associated statistic. |
5. Asynchronously combine the two values. |
6. Aggregate the values into a `List` as they become available. |
7. In production, we would continue working with the `Flux` asynchronously by further combining it or subscribing to it. Most probably, we would return the `result` `Mono`. Since we are in a test, we instead block, waiting for the processing to finish, and then directly return the aggregated list of values. |
8. Assert the result. |
```

The perils of using callbacks and `Future` objects are similar and are what reactive programming addresses with the `Publisher-Subscriber` pair.

[3\. From Imperative to Reactive Programming](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#from-imperative-to-reactive-programming)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------

Reactive libraries, such as Reactor, aim to address these drawbacks of "classic" asynchronous approaches on the JVM while also focusing on a few additional aspects:

-   Composability and readability

-   Data as a flow manipulated with a rich vocabulary of operators

-   Nothing happens until you subscribe

-   Backpressure or *the ability for the consumer to signal the producer that the rate of emission is too high*

-   High level but high value abstraction that is *concurrency-agnostic*

### [3.1. Composability and Readability](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#composability-and-readability)

By "composability", we mean the ability to orchestrate multiple asynchronous tasks, in which we use results from previous tasks to feed input to subsequent ones. Alternatively, we can run several tasks in a fork-join style. In addition, we can reuse asynchronous tasks as discrete components in a higher-level system.

The ability to orchestrate tasks is tightly coupled to the readability and maintainability of code. As the layers of asynchronous processes increase in both number and complexity, being able to compose and read code becomes increasingly difficult. As we saw, the callback model is simple, but one of its main drawbacks is that, for complex processes, you need to have a callback executed from a callback, itself nested inside another callback, and so on. That mess is known as "Callback Hell". As you can guess (or know from experience), such code is pretty hard to go back to and reason about.

Reactor offers rich composition options, wherein code mirrors the organization of the abstract process, and everything is generally kept at the same level (nesting is minimized).

### [3.2. The Assembly Line Analogy](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#the-assembly-line-analogy)

You can think of data processed by a reactive application as moving through an assembly line. Reactor is both the conveyor belt and the workstations. The raw material pours from a source (the original `Publisher`) and ends up as a finished product ready to be pushed to the consumer (or `Subscriber`).

The raw material can go through various transformations and other intermediary steps or be part of a larger assembly line that aggregates intermediate pieces together. If there is a glitch or clogging at one point (perhaps boxing the products takes a disproportionately long time), the afflicted workstation can signal upstream to limit the flow of raw material.

### [3.3. Operators](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#operators)

In Reactor, operators are the workstations in our assembly analogy. Each operator adds behavior to a `Publisher` and wraps the previous step's `Publisher` into a new instance. The whole chain is thus linked, such that data originates from the first `Publisher` and moves down the chain, transformed by each link. Eventually, a `Subscriber` finishes the process. Remember that nothing happens until a `Subscriber` subscribes to a `Publisher`, as we will see shortly.

|  | Understanding that operators create new instances can help you avoid a common mistake that would lead you to believe that an operator you used in your chain is not being applied. See this [item](https://projectreactor.io/docs/core/milestone/reference/faq.html#faq.chain) in the FAQ. |

While the Reactive Streams specification does not specify operators at all, one of the best added values of reactive libraries, such as Reactor, is the rich vocabulary of operators that they provide. These cover a lot of ground, from simple transformation and filtering to complex orchestration and error handling.

### [3.4. Nothing Happens Until You `subscribe()`](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#reactive.subscribe)

In Reactor, when you write a `Publisher` chain, data does not start pumping into it by default. Instead, you create an abstract description of your asynchronous process (which can help with reusability and composition).

By the act of subscribing, you tie the `Publisher` to a `Subscriber`, which triggers the flow of data in the whole chain. This is achieved internally by a single `request` signal from the `Subscriber` that is propagated upstream, all the way back to the source `Publisher`.

### [3.5. Backpressure](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#reactive.backpressure)

Propagating signals upstream is also used to implement backpressure, which we described in the assembly line analogy as a feedback signal sent up the line when a workstation processes more slowly than an upstream workstation.

The real mechanism defined by the Reactive Streams specification is pretty close to the analogy: A subscriber can work in *unbounded* mode and let the source push all the data at its fastest achievable rate or it can use the `request` mechanism to signal the source that it is ready to process at most `n` elements.

Intermediate operators can also change the request in-transit. Imagine a `buffer` operator that groups elements in batches of ten. If the subscriber requests one buffer, it is acceptable for the source to produce ten elements. Some operators also implement prefetching strategies, which avoid `request(1)` round-trips and is beneficial if producing the elements before they are requested is not too costly.

This transforms the push model into a push-pull hybrid, where the downstream can pull n elements from upstream if they are readily available. But if the elements are not ready, they get pushed by the upstream whenever they are produced.

### [3.6. Hot vs Cold](https://projectreactor.io/docs/core/milestone/reference/reactiveProgramming.html#reactive.hotCold)

The Rx family of reactive libraries distinguishes two broad categories of reactive sequences: hot and cold. This distinction mainly has to do with how the reactive stream reacts to subscribers:

-   A Cold sequence starts anew for each `Subscriber`, including at the source of data. For example, if the source wraps an HTTP call, a new HTTP request is made for each subscription.

-   A Hot sequence does not start from scratch for each `Subscriber`. Rather, late subscribers receive signals emitted *after* they subscribed. Note, however, that some hot reactive streams can cache or replay the history of emissions totally or partially. From a general perspective, a hot sequence can even emit when no subscriber is listening (an exception to the "nothing happens before you subscribe" rule).

For more information on hot vs cold in the context of Reactor, see [this reactor-specific section](https://projectreactor.io/docs/core/milestone/reference/advancedFeatures/reactor-hotCold.html).

[Getting Started](https://projectreactor.io/docs/core/milestone/reference/gettingStarted.html)

[Reactor Core Features](https://projectreactor.io/docs/core/milestone/reference/coreFeatures.html)