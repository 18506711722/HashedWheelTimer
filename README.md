# AionHashedWheelTimer

[![Java 21+](https://img.shields.io/badge/Java-21%2B-blue)](https://openjdk.org/projects/jdk/21)
[![License](https://img.shields.io/badge/License-Apache%202.0-green.svg)](LICENSE)

一款高性能、单线程、基于哈希时间轮的定时器，适用于所有高并发定时场景。

## ✨ 特性

- **单线程无锁核心** – 所有桶操作通过任务队列串行化，消除并发竞争。
- **O(1) 桶内删除** – 桶内使用 `LinkedHashSet`，配合 `bucketIndex` 缓存实现瞬间取消/重排。
- **对象池化** – 通过 `ArrayDeque` 复用 `TaskEntry`，高负载下显著降低 GC 压力。
- **非阻塞 API** – `cancelAsync()` 和 `rescheduleAsync()` 返回 `CompletableFuture`，永不阻塞调用线程。
- **原地重排** – 无需先取消再新建，直接更新任务延迟，完美适配高频刷新场景。
- **精确时钟追赶** – `LockSupport.parkNanos` + `maxSkipTicks` 限制，无漂移，GC 暂停后安全恢复。
- **丰富指标** – EMA 调度延迟、平均/最大业务执行耗时、桶深度、异常计数等。
- **虚拟线程友好** – worker 运行于虚拟线程，业务执行器可任意配置。
- **零外部依赖** – 纯 Java 实现，即插即用。

## 🚀 快速开始

> **注意：当前版本暂未上传到 Maven 中央仓库，请直接下载源码或使用本地构建。**

### Maven（待发布）
```xml
<dependency>
    <groupId>top.aion0573.comm</groupId>
    <artifactId>aion-hashed-wheel-timer</artifactId>
    <version>1.0.0</version>
</dependency>
```

### Gradle（待发布）
```gradle
implementation 'top.aion0573.comm:aion-hashed-wheel-timer:1.0.0'
```

### 基本用法

```java
// 创建时间轮：60 个槽，每 tick 100ms（整轮周期 6 秒）
AionHashedWheelTimer timer = new AionHashedWheelTimer(60, 100);

// 一次性任务：500ms 后执行
Timeout t1 = timer.newTimeout(self -> {
    System.out.println("500ms 后执行");
}, 500);

// 固定速率周期任务：首次延迟 100ms，之后每 300ms 执行一次
Timeout t2 = timer.scheduleAtFixedRate(self -> {
    System.out.println("周期任务");
}, 100, 300);

// 取消任务（阻塞式）
t1.cancel();

// 异步取消
t2.cancelAsync().thenAccept(success -> {
    System.out.println("取消结果：" + success);
});

// 重设延迟（原地更新）
t1.reschedule(2000);   // 改为 2 秒后执行

// 异步重设
t1.rescheduleAsync(1000);

// 优雅关闭（等待未完成任务）
timer.shutdownGracefully(5, TimeUnit.SECONDS);
```

### `AionHashedWheelTimer`
- `newTimeout(Consumer<TaskEntry>, long delayMs)` → 注册一次性延时任务
- `scheduleAtFixedRate(Consumer<TaskEntry>, long initialDelayMs, long periodMs)` → 周期任务
- `getMetrics()` → 返回运行时指标快照（待处理任务数、平均执行时间等）
- `shutdown()` / `shutdownGracefully(long, TimeUnit)` → 关闭

### `Timeout`
- `cancel()` / `cancelAsync()` → 取消任务
- `reschedule(long newDelayMs)` / `rescheduleAsync(long newDelayMs)` → 修改延迟
- `isCancelled()`, `isExpired()`, `isDone()` → 状态查询

### `Metrics`
```java
Metrics m = timer.getMetrics();
m.currentTick;          // 当前 tick
m.pendingTasks;         // 待处理任务数
m.avgExecTimeMs;        // 平均业务执行耗时
m.maxExecTimeMs;        // 最大业务执行耗时
m.emaAvgDelayMs;        // 调度延迟的 EMA
m.completedTasks;       // 已完成任务总数
m.exceptionCount;       // 异常计数
// 更多字段...
```

## 📦 环境要求

- Java 21+（内部使用虚拟线程，可简单改为平台线程以降低版本要求）
- 无外部依赖

## 📄 许可证

Apache License 2.0 – 详见 [LICENSE](LICENSE) 文件。
