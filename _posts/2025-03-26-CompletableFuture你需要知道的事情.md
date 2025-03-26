---
layout: post
title: CompletableFuture你需要知道的事情.
subtitle: CompletableFuture你需要知道的事情.
excerpt_image: https://github.com/wtopps/wtopps.github.io/blob/master/images/CompletableFuture.jpeg?raw=true
categories: Java
tags: [Java]
top: 1

---

![banner](https://github.com/wtopps/wtopps.github.io/blob/master/images/CompletableFuture.jpeg?raw=true)

# 问题

面试官：请使用多线程分别打印“Hello World”，当全部线程执行完成后，打印“Finish”。

![please show yourself](https://github.com/wtopps/wtopps.github.io/blob/master/images/please%20show%20yourself.jpeg?raw=true)

这是一道非常经典的面试题，Java的面试中高频出现，主要考察多线程的使用和join()的应用，我们来尝试实现一下：

![露一手](https://github.com/wtopps/wtopps.github.io/blob/master/images/%E9%9C%B2%E4%B8%80%E6%89%8B.jpg?raw=true)

```java
public class ThreadJoinDemo {

    public static void main(String[] args) throws Exception {
        Thread threadA = new MyThread();
        Thread threadB = new MyThread();
        threadA.setName("threadA");
        threadB.setName("threadB");
        threadA.start();
        threadB.start();
        threadA.join();
        threadB.join();
        System.out.println("Finish");
    }
}

class MyThread extends Thread {

    @Override
    public void run() {
        System.out.println("Hello World");
    }
}
```

输出结果：

```
Hello World
Hello World
Finish
```

面试官：小伙子不错，你还有没有更优雅的实现方式？

![这个可以有](https://github.com/wtopps/wtopps.github.io/blob/master/images/%E8%BF%99%E4%B8%AA%E5%8F%AF%E4%BB%A5%E6%9C%89.png?raw=true)

# CompletableFuture

## 一、概述

CompletableFuture 是 Java 8 引入的一个强大的异步编程工具，它实现了 Future 和 CompletionStage 接口，提供了丰富的异步操作和组合能力。相比传统的 Future，CompletableFuture 具有以下优势：

1. 支持显式完成（手动设置结果）
2. 提供丰富的回调机制
3. 支持链式调用和组合操作
4. 内置异常处理机制
5. 支持多个 CompletableFuture 的组合

回到上面的问题，如果使用CompletableFuture如何实现？

```java
List<CompletableFuture<Void>> futures = Arrays.asList(
    CompletableFuture.runAsync(() -> System.out.println("Hello World")),
    CompletableFuture.runAsync(() -> System.out.println("Hello World"))
);
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
System.out.println("Finish"));
```

![优雅](https://github.com/wtopps/wtopps.github.io/blob/master/images/%E4%BC%98%E9%9B%85.gif?raw=true)

下面我们来一起了解一下CompletableFuture的细节。

## 二、核心实现细节

### 1. 基本结构

CompletableFuture 的核心字段包括：

```
volatile Object result;       // 结果或异常的包装
volatile Completion stack;    // 依赖此 Future 的操作栈
```

### 2. 完成机制

CompletableFuture 可以通过以下方式完成：

- `complete(T value)`：正常完成
- `completeExceptionally(Throwable ex)`：异常完成
- `cancel(boolean mayInterruptIfRunning)`：取消任务

### 3. 依赖关系管理

CompletableFuture 使用 `Completion` 链表来管理依赖关系。当一个 CompletableFuture 完成时，它会触发所有依赖它的操作。

### 4. 线程池使用

默认情况下，CompletableFuture 使用 `ForkJoinPool.commonPool()` 作为异步执行的线程池。可以通过提供自定义的 `Executor` 来改变这一行为。

## 三、核心 API 和使用技巧

### 1. 创建 CompletableFuture

```java
// 1. 使用静态工厂方法创建已完成的 Future
CompletableFuture<String> completedFuture = CompletableFuture.completedFuture("Hello");

// 2. 创建异步任务
CompletableFuture<Void> runAsyncFuture = CompletableFuture.runAsync(() -> {
    // 无返回值的异步任务
});

CompletableFuture<String> supplyAsyncFuture = CompletableFuture.supplyAsync(() -> {
    // 有返回值的异步任务
    return "Result";
});

// 3. 使用自定义线程池
ExecutorService executor = Executors.newFixedThreadPool(10);
CompletableFuture<String> customExecutorFuture = CompletableFuture.supplyAsync(() -> {
    return "Result with custom executor";
}, executor);
```

### 2. 结果处理和转换

```java
// thenApply - 同步转换结果
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "Hello")
    .thenApply(s -> s + " World")
    .thenApply(String::toUpperCase);

// thenApplyAsync - 异步转换结果
future.thenApplyAsync(s -> s + "!");

// thenAccept - 消费结果但不返回新值
future.thenAccept(System.out::println);

// thenRun - 不消费结果，只执行操作
future.thenRun(() -> System.out.println("Operation completed"));
```

### 3. 组合多个 Future

```java
// thenCompose - 扁平化 Future
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = future1.thenCompose(s -> 
    CompletableFuture.supplyAsync(() -> s + " World"));

// thenCombine - 合并两个独立 Future 的结果
CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future4 = CompletableFuture.supplyAsync(() -> "World");
CompletableFuture<String> combined = future3.thenCombine(future4, (s1, s2) -> s1 + " " + s2);

// allOf - 等待所有 Future 完成
CompletableFuture<Void> all = CompletableFuture.allOf(future1, future2, future3);

// anyOf - 等待任意一个 Future 完成
CompletableFuture<Object> any = CompletableFuture.anyOf(future1, future2, future3);
```

关于allOf()方法，有一点需要注意，allOf()方法的注释是这样描述的：

```
Returns a new CompletableFuture that is completed when all of the given CompletableFutures complete. If any of the given CompletableFutures complete exceptionally, then the returned CompletableFuture also does so, with a CompletionException holding this exception as its cause. Otherwise, the results, if any, of the given CompletableFutures are not reflected in the returned CompletableFuture, but may be obtained by inspecting them individually. If no CompletableFutures are provided, returns a CompletableFuture completed with the value null.
```

看完第一句话，可能会理解为，哦这个方法是等待所有的Future完成后，才返回结果呀，其实非也。

这句注释需要仔细理解：

**"Returns a new CompletableFuture that is completed when all of the given CompletableFutures complete"**

关键在于：

1. "Returns a new CompletableFuture" - 立即返回一个新的Future对象
2. "is completed when..." - 这个新Future在所有子Future完成时才会完成

可以做一个实验进行验证：

```java
// 创建几个任务
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    sleep(1000);
    return "Task 1";
});
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    sleep(2000);
    return "Task 2";
});

// allOf返回的新Future
CompletableFuture<Void> allFuture = CompletableFuture.allOf(future1, future2);
// 这行代码立即返回，不会等待

// 新Future的状态
System.out.println("Is completed: " + allFuture.isDone());  // false
// 等待2秒后
System.out.println("Is completed: " + allFuture.isDone());  // true
```

如果你希望全部的Future执行完成之前，主线程进行阻塞等待结果，你需要这样：

```java
CompletableFuture<Void> all = CompletableFuture.allOf(future1, future2, future3);
all.join();
```



### 4. 异常处理

```java
CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) {
        throw new RuntimeException("Error");
    }
    return "Success";
})
.exceptionally(ex -> {
    System.out.println("Exception: " + ex.getMessage());
    return "Recovered";
})
.handle((result, ex) -> {
    if (ex != null) {
        return "Handled error: " + ex.getMessage();
    }
    return "Handled result: " + result;
});
```

### 5. 完成控制

```java
CompletableFuture<String> future = new CompletableFuture<>();

// 手动完成
future.complete("Manual completion");

// 手动异常完成
future.completeExceptionally(new RuntimeException("Failed"));

// 超时完成
future.completeOnTimeout("Default", 1, TimeUnit.SECONDS);

// 或超时
future.orTimeout(1, TimeUnit.SECONDS);
```

## 四、高级使用技巧

### 1. 流水线模式

```java
CompletableFuture.supplyAsync(() -> fetchData())
    .thenApply(data -> processData(data))
    .thenApply(processedData -> storeData(processedData))
    .thenAccept(storedId -> logSuccess(storedId))
    .exceptionally(ex -> {
        logError(ex);
        return null;
    });
```

### 2. 异步超时控制

```java
public static <T> CompletableFuture<T> withTimeout(CompletableFuture<T> future, 
        long timeout, TimeUnit unit, T defaultValue) {
    return future.applyToEither(
        CompletableFuture.supplyAsync(() -> defaultValue)
            .completeOnTimeout(defaultValue, timeout, unit),
        Function.identity()
    );
}
```

### 3. 批量操作

```java
List<CompletableFuture<String>> futures = urls.stream()
    .map(url -> CompletableFuture.supplyAsync(() -> fetchUrl(url)))
    .collect(Collectors.toList());

CompletableFuture<Void> allDone = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));

CompletableFuture<List<String>> allResults = allDone.thenApply(v -> 
    futures.stream()
        .map(CompletableFuture::join)
        .collect(Collectors.toList()));
```

### 4. 取消传播

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(1000);
        return "Result";
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
});

CompletableFuture<String> future2 = future1.thenApply(s -> s + " processed");

// 取消第一个 future 会导致依赖它的 future 也被取消
future1.cancel(true);
```

## 五、性能优化和注意事项

1. **线程池选择**：根据任务类型选择合适的线程池
   - CPU密集型：使用有界线程池（如 `Executors.newFixedThreadPool`）
   - IO密集型：使用无界线程池（如 `Executors.newCachedThreadPool`）
2. **避免阻塞**：不要在回调中进行阻塞操作
3. **合理使用异步**：并非所有操作都需要异步，简单的转换可以直接用同步方法
4. **资源清理**：长时间运行的 CompletableFuture 链可能会持有资源，需要适当清理
5. **异常处理**：确保所有可能的异常路径都被处理
6. **避免嵌套过深**：过深的 CompletableFuture 链会影响可读性

## 六、常见问题解决方案

### 1. 如何获取多个 Future 的结果？

```java
List<CompletableFuture<String>> futures = ...;
CompletableFuture<Void> allFutures = CompletableFuture.allOf(
    futures.toArray(new CompletableFuture[0]));

CompletableFuture<List<String>> allResults = allFutures.thenApply(v ->
    futures.stream()
        .map(CompletableFuture::join)
        .collect(Collectors.toList()));


// 使用CompletableFuture并行处理图片上传
List<CompletableFuture<JSONObject>> futures = images.stream()
        .map(imageInfo -> CompletableFuture.supplyAsync(() -> {
            return uploadCorePic();
        }))
        .collect(Collectors.toList());

// 等待所有图片上传完成
JSONArray picArray = new JSONArray();
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
futures.stream()
        .map(CompletableFuture::join)
        .filter(Objects::nonNull)
        .forEach(picArray::add);
```

### 2. 如何实现重试机制？

```java
public static <T> CompletableFuture<T> retry(Supplier<CompletableFuture<T>> task, 
        int maxRetries, long delay, TimeUnit unit) {
    CompletableFuture<T> future = task.get();
    for (int i = 0; i < maxRetries; i++) {
        future = future.exceptionally(ex -> {
            try {
                Thread.sleep(unit.toMillis(delay));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return task.get().join();
        });
    }
    return future;
}
```

### 3. 如何实现优雅的超时回退？

```java
public static <T> CompletableFuture<T> withFallback(
        CompletableFuture<T> future, // 原始任务
        long timeout, TimeUnit unit, // 超时时间及其时间单位
        Supplier<T> fallback) {			 // 降级逻辑
    final CompletableFuture<T> timeoutFuture = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(unit.toMillis(timeout));
            return fallback.get();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return fallback.get();
        }
    });
    return future.applyToEither(timeoutFuture, Function.identity());
}
```

使用示例：

```java
// 创建一个可能耗时的任务
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(5000);  // 模拟耗时操作
        return "任务正常完成";
    } catch (InterruptedException e) {
        return "被中断";
    }
});

// 使用withFallback添加3秒超时
CompletableFuture<String> withTimeout = withFallback(
    future,
    3, TimeUnit.SECONDS,
    () -> "任务超时，返回降级结果"
);

// 获取结果
String result = withTimeout.join();
System.out.println(result);  // 会打印"任务超时，返回降级结果"

```

执行示意图：

```
原始任务 future     ----5秒-----> "任务正常完成"
                            \
                             \-- 采用先完成的结果
                            /
超时任务 timeoutFuture ----3秒-----> "降级结果"
```



## 七、总结

CompletableFuture 是 Java 异步编程的强大工具，通过合理使用可以构建高效、清晰的异步代码。掌握其核心概念、API 和最佳实践，可以帮助开发者编写更健壮、更易维护的并发应用程序。在实际使用中，应根据具体场景选择合适的组合方式，并注意资源管理和异常处理。