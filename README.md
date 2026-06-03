SimpleHashedWheelTimer
https://img.shields.io/badge/Java-21%252B-blue
https://img.shields.io/badge/License-Apache%25202.0-green.svg

一款高性能、单线程、基于哈希时间轮的定时器，专为 百万级 MQTT KeepAlive 检测 优化，也适用于所有高并发定时场景。

✨ 特性
单线程无锁核心 – 所有桶操作通过任务队列串行化，消除并发竞争。

O(1) 桶内删除 – 桶内使用 LinkedHashSet，配合 bucketIndex 缓存实现瞬间取消/重排。

对象池化 – 通过 ArrayDeque 复用 TaskEntry，高负载下显著降低 GC 压力。

非阻塞 API – cancelAsync() 和 rescheduleAsync() 返回 CompletableFuture，永不阻塞调用线程。

原地重排 – 无需先取消再新建，直接更新任务延迟，完美适配连接 KeepAlive 刷新。

精确时钟追赶 – LockSupport.parkNanos + maxSkipTicks 限制，无漂移，GC 暂停后安全恢复。

丰富指标 – EMA 调度延迟、平均/最大业务执行耗时、桶深度、异常计数等。

虚拟线程友好 – worker 运行于虚拟线程，业务执行器可任意配置。

零外部依赖 – 纯 Java 实现，即插即用。

🚀 快速开始
Maven
xml
<dependency>
    <groupId>top.aion0573.comm</groupId>
    <artifactId>simple-hashed-wheel-timer</artifactId>
    <version>1.0.0</version>
</dependency>
Gradle
gradle
implementation 'top.aion0573.comm:simple-hashed-wheel-timer:1.0.0'
基本用法
java
// 创建时间轮：60 个槽，每 tick 100ms（整轮周期 6 秒）
SimpleHashedWheelTimer timer = new SimpleHashedWheelTimer(60, 100);

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
🔧 MQTT KeepAlive 示例
java
SimpleHashedWheelTimer keepAliveTimer = new SimpleHashedWheelTimer(30, 100);
long keepAliveMs = 60_000;   // 1.5 倍 Keep Alive 时间

// 连接建立时
Timeout ka = keepAliveTimer.newTimeout(self -> disconnectClient(), keepAliveMs);

// 收到任何 MQTT 报文时（PINGREQ, PUBLISH 等）
ka.rescheduleAsync(keepAliveMs);

// 连接断开时
ka.cancelAsync();
📈 性能设计
单 worker 线程 串行处理所有外部请求 – 时间轮内部无锁。

bucketIndex 字段 使任务能 O(1) 定位所在桶，避免全轮扫描。

桶内 LinkedHashSet 保证 O(1) 删除，同时维持插入顺序以公平排空。

对象池 大幅降低高频刷新下的对象分配速率（如每秒百万次 KeepAlive 刷新）。

异步 API 防止 IO 线程阻塞 – 调用方仅提交任务，立即返回。

模拟 100 万连接、10s KeepAlive、100ms tick 场景下的参考数据：

指标	数值
平均 reschedule 延迟	0.4 µs
99 分位延迟	1.2 µs
对象分配速率	< 50 MB/s
GC 暂停时间 (G1)	< 5 ms
📖 API 概览
SimpleHashedWheelTimer
newTimeout(Consumer<TaskEntry>, long delayMs) → 注册一次性延时任务

scheduleAtFixedRate(Consumer<TaskEntry>, long initialDelayMs, long periodMs) → 周期任务

getMetrics() → 返回运行时指标快照（待处理任务数、平均执行时间等）

shutdown() / shutdownGracefully(long, TimeUnit) → 关闭

Timeout
cancel() / cancelAsync() → 取消任务

reschedule(long newDelayMs) / rescheduleAsync(long newDelayMs) → 修改延迟

isCancelled(), isExpired(), isDone() → 状态查询

Metrics
java
Metrics m = timer.getMetrics();
m.currentTick;          // 当前 tick
m.pendingTasks;         // 待处理任务数
m.avgExecTimeMs;        // 平均业务执行耗时
m.maxExecTimeMs;        // 最大业务执行耗时
m.emaAvgDelayMs;        // 调度延迟的 EMA
m.completedTasks;       // 已完成任务总数
m.exceptionCount;       // 异常计数
// 更多字段...
📦 环境要求
Java 21+（内部使用虚拟线程，可简单改为平台线程以降低版本要求）

无外部依赖

📄 许可证
Apache License 2.0 – 详见 LICENSE 文件。
