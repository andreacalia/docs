---
title: Introduction to CompletableFuture
slug: introduction-to-completable-future
description: Introduction to Java asynchronous programming using CompletableFuture
categories:
  - Article
  - Java
tags:
  - java
  - programming
  - async
---

# Introduction to `CompletableFuture`

`CompletableFuture` class is introduced in Java 8<sup>[1](#javadoc)</sup> and it is used to make asynchronous computation easier to code.

A `CompletableFuture` object is a "promise" (sometimes this terminology is used in other languages) that a certain result will be available at some point in the future. The generic type specifies the type of the result that will be produced:

```java
CompletableFuture<String> future = doAsyncRead();
String result = future.get(); // Warning: Bad practice
```

In the example above, the `future.get()` method will block the caller Thread until the result of the `CompletableFuture` is available. This behavior make the usage of async computation totally useless! *The caller thread should never be blocked!*

In order to asynchronously obtain the result of a `CompletableFuture`, there are a few options. The most generic one is by attaching a `Biconsumer<T, Throwable>` via the `CompletableFuture#whenComplete()` method:

```java
CompletableFuture<String> future = doAsyncRead();
future.whenComplete((String result, Throwable exception) -> {
    if(exception == null) {
        LOGGER.info("Success!");
    } else {
        LOGGER.error("There was a problem.. ", exception);
    }
});
```

The `CompletableFuture#whenComplete(...)` method forces the user to check whether the computation of the result produced an error or the result. In the successful case, the `Throwable` parameter will be `null`, therefore the result value will be present.

The CompletableFuture API has multiple methods that allows the user to concatenate operations (chaining), handle errors, etc. Some of the most useful methods are:

* `.thenAccept(Consumer<T>)`: consumes the produced value of the chain.
* `.thenApply(Function<T,U>)`: conceptually equivalent to _map_ in stream processing.
* `.thenRun(Runnable)`: simply runs the `Runnable`.
* `.thenCompose(Function<T,CompletionStage<U>>)`: conceptually equivalent to _flatMap_ in stream processing.
* `.whenComplete(Biconsumer<T,Throwable>)`: the specified `Biconsumer` accepts the value that was produced or the (first) exception that was thrown by a previous stage in the chain.

Every method presented above is also available with the _async_ suffix: e.g. `.thenApplyAsync(Function<T,U>)` and `.thenApplyAsync(Function<T,U>, Executor)`. 

The _async_ variants are useful because by default all the subsequent chaining of a `CompletableFuture` are executed on the same Thread that executes the first action. As an example:

```java
CompletableFuture<String> asd = CompletableFuture.supplyAsync(() -> {
    LOGGER.info("Producer");
    return "Hello";
})
.thenApply(previous -> {
    LOGGER.info("Transformation#1");
    return previous + " from";
})
.thenApply(previous -> {
    LOGGER.info("Transformation#2");
    return previous + " the future!";
})
.whenComplete((result, exception) -> {
    if (result != null) {
        LOGGER.info("Result: {}", result);
    } else {
        LOGGER.error("Error from the future!");
    }
});
```

Produces as output:

```
[ForkJoinPool.commonPool-worker-1] - Producer
[ForkJoinPool.commonPool-worker-1] - Transformation#1
[ForkJoinPool.commonPool-worker-1] - Transformation#2
[ForkJoinPool.commonPool-worker-1] - Result: Hello from the future!
```

`ForkJoinPool` is a common Thread pool in which, by default, all the operations are executed on. Each operation specified on a method with suffix _async_ can be executed on a new or re-used Thread of the pool.

In the previous example, the only _async_ method is the `.supplyAsync(...)` and all the subsequent operations are executed on the same Thread that prints _"Producer"_ on the console.

Every _async_ operation switches the Thread that execute the operation (and the subsequent ones until another _async_ is specified).

Depending on the situation, it is necessary to specify the Thread on which operations are executed (Thread bound synchronization, listeners, etc.). In order to achieve this, every _async_ method has an overloaded version that accepts a `Executor` that will be used to execute the operation and the subsequent, chained, ones.

A fully asynchronous version of the example above will look like:

```java
CompletableFuture.supplyAsync(() -> {
    LOGGER.info("Producer");
    return "Hello";
}, producerExecutor)
.thenApplyAsync(previous -> {
    LOGGER.info("Transformation#1");
    return previous + " from";
}, poolAExecutor)
.thenApplyAsync(previous -> {
    LOGGER.info("Transformation#2");
    return previous + " the future!";
}, poolBExecutor)
.whenCompleteAsync((result, exception) -> {
    if (result != null) {
        LOGGER.info("Result: {}", result);
    } else {
        LOGGER.error("Error from the future!");
    }
}, resultExecutor);
```

The execution produces the following:

```
[producer-0] - Producer
[pool-a-0] - Transformation#1
[pool-b-0] - Transformation#2
[result-0] - Result: Hello from the future!
```

In this case, each operation has been executed on a separate `ExecutorService`.

## Case study: `.get()` and `.join()`

The `CompletableFuture` API has two very tempting methods for beginners of asynchronous programming: `.get()` and `.join()`. Both methods, in a slightly different way, block the caller Thread and wait for a result of the `CompletableFuture` to be available.

This is a common anti-pattern since using an API that exposes a `CompletableFuture` usually means that the computation is likely to take some time and/or can timeout. It is not desirable to block and wait, instead it is preferable to use any of the asynchronous methods (e.g. `.whenComplete(...)`) to react when the result, or exception, will be available.

## Case study: designing remote APIs

As explained in this article, `CompletableFuture` is useful when an API need to perform expensive or long operation without blocking the caller Thread and providing, eventually, a result sometime in the future. This matches quite well the design of remote services that an application may need to access. As an example, the developer needs to access some Twitter API and she may decide to create an interface for that:

```java
public interface TwitterService {
    CompletableFuture<List<Tweet>> latestTweetsOf(User user);
    CompletableFuture<List<User>> followersOf(User user);
}
```

Each method of the `TwitterService` need to access the Twitter Rest API and this is not a synchronous operation. The results will be available at some point in the future when the API answer with a result.

When using the API a common pattern is to use the `.whenCompleteAsync(...)` to access the results. In this case it is important to remember to use the _async_ version of the `CompletableFuture` API in order not to block the service's internal Thread pool (in case there is one).

### Services internal Thread pool

Services that express their API using `CompletableFuture` are often remote (e.g. `TwitterService`) and it is a good idea to have a separate Thread pool to handle the connections without potentially overloading the default pool that the `CompletableFuture` implementation uses.

In this case it is possible to use the _async_ methods of the `CompletableFuture` API and specify a custom `ExecutorService`. For example:

```java
public class TwitterServiceImpl implements TwitterService {

    private final Executor executor = Executors.newCachedThreadPool();

    @Override
    public CompletableFuture<List<Tweet>> latestTweetsOf(User user) {
        return CompletableFuture
                .supplyAsync(() -> fetchLatestTweetsOf(user), executor);
    }

    @Override
    public CompletableFuture<List<User>> followersOf(User user) {
        return CompletableFuture
                .supplyAsync(() -> fetchFollowersOf(user), executor);
    }
}
```

## Switch Thread on listeners


<a name="javadoc">1</a>: <https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html>