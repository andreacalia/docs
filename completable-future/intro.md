# CompletableFuture

CompletableFuture class is introduced in Java 8[^javadoc-cf] and it is used to make asynchronous computation easier to code.

A CompletableFuture object is a "promise" (sometimes this terminology is used in other languages) that a certain result will be available at some point in the future. The generic type specifies the type of the result that will be produced:

```java
CompletableFuture<String> asyncResult = doAsyncRead();
String result = asyncResult.get();
```

In the example above, the `asyncResult.get()` method will block the caller Thread until the result of the CompletableFuture is available. This behavior make the usage of async computation totally useless! *The caller thread should never be blocked!*

In order to asynchronously obtain the result of a CompletableFuture, there are a few options. The most generic one is by attaching a `Biconsumer<T, Throwable>` via the `CompletableFuture#whenComplete()` method:

```java
CompletableFuture<String> asyncResult = doAsyncRead();
asyncResult.whenComplete((String result, Throwable exception) -> {
    if(exception == null) {
        System.out.println("Success!");
    } else {
        System.err.println("There was a problem.. " + exception.getMessage());
    }
});
```

The `CompletableFuture#whenComplete()` method forces the user to check whether the computation of the result produced an error or the result. In the successful case, the `Throwable` parameter will be null, therefore the result value will be present.

The CompletableFuture API has multiple methods that allows the user to concatenate operations, handle errors, etc. Some of the most useful methods are:
* 

# Avoid .get() and .join()

# Switch Thread on listeners!


[^javadoc-cf]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html