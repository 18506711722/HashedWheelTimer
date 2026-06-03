# SimpleHashedWheelTimer

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

### Maven
```xml
<dependency>
    <groupId>top.aion0573.comm</groupId>
    <artifactId>simple-hashed-wheel-timer</artifactId>
    <version>1.0.0</version>
</dependency>
