ThreadPoolExecutor 是 Java 并发编程中一个强大且灵活的工具，它能帮助我们有效地管理线程资源，避免因频繁创建和销毁线程带来的开销，同时控制并发任务的数量。然而，不恰当地配置其参数，轻则导致性能不佳，重则可能耗尽系统资源，甚至引发死锁。因此，深入理解并实践 `ThreadPoolExecutor` 的参数调优至关重要。

---

### 为什么需要自定义 `ThreadPoolExecutor` 参数？

Java 提供了 `Executors` 工厂类来快速创建一些预设的线程池，例如 `FixedThreadPool`、`CachedThreadPool` 等。这些便捷方法在很多简单场景下足够用，但它们内部的 `ThreadPoolExecutor` 参数是固定的。例如，`FixedThreadPool` 使用无界队列，当任务量巨大时可能导致内存溢出；`CachedThreadPool` 的 `maximumPoolSize` 为 `Integer.MAX_VALUE`，在高并发下可能创建过多线程，耗尽系统资源。

因此，为了更好地控制线程行为、避免资源耗尽并优化性能，我们通常需要根据实际应用场景，自定义 `ThreadPoolExecutor` 的各个参数。

---

### `ThreadPoolExecutor` 的核心参数解析

`ThreadPoolExecutor` 的构造方法通常有以下几个关键参数：

**Java**

```
public ThreadPoolExecutor(
    int corePoolSize,
    int maximumPoolSize,
    long keepAliveTime,
    TimeUnit unit,
    BlockingQueue<Runnable> workQueue,
    ThreadFactory threadFactory,
    RejectedExecutionHandler handler
)
```

1. **`corePoolSize` (核心线程数)**
   * **定义：** 线程池中保持活动状态的最小线程数。即使这些线程处于空闲状态，也不会被销毁。
   * **作用：** 线程池创建后，只有当任务提交时，才会创建线程。当线程数小于 `corePoolSize` 时，会优先创建核心线程来执行任务。
   * **实践：** 根据系统CPU核数和任务类型（CPU密集型或I/O密集型）来设置。
2. **`maximumPoolSize` (最大线程数)**
   * **定义：** 线程池允许创建的最大线程数。当工作队列已满，且当前线程数小于 `maximumPoolSize` 时，线程池会创建新的非核心线程来执行任务。
   * **作用：** 限制线程池能达到的最大并发度，防止创建过多线程导致系统资源耗尽。
   * **实践：** 同样受CPU核数和任务类型影响。
3. **`keepAliveTime` (空闲线程存活时间)**
   * **定义：** 当线程池中的线程数量超过 `corePoolSize` 时，多余的空闲线程在终止前等待新任务的最长时间。
   * **作用：** 允许非核心线程在空闲一段时间后被销毁，释放资源。
   * **实践：** 对于任务峰谷明显的应用，可以设置较短时间；对于持续高负载的应用，可以设置较长时间甚至0（表示不销毁）。
4. **`unit` (空闲线程存活时间单位)**
   * **定义：**`keepAliveTime` 参数的时间单位，例如 `TimeUnit.SECONDS`、`TimeUnit.MILLISECONDS`。
5. **`workQueue` (工作队列)**
   * **定义：** 用于保存等待执行的任务的阻塞队列。
   * **作用：** 当线程池中的核心线程都在忙碌时，新提交的任务会进入此队列等待。
   * **实践：** 选择合适的队列类型是关键。
6. **`threadFactory` (线程工厂)**
   * **定义：** 用于创建新线程的工厂。
   * **作用：** 可以自定义线程的命名、优先级、是否为守护线程等。
   * **实践：** 强烈建议自定义，方便问题排查和日志分析。
7. **`RejectedExecutionHandler` (拒绝策略)**
   * **定义：** 当线程池和工作队列都已满，无法接收新任务时的处理策略。
   * **作用：** 定义了线程池过载时的行为。
   * **实践：** 选择适合业务需求的拒绝策略。

---

### `ThreadPoolExecutor` 的任务执行流程

理解参数之间的相互作用，关键在于了解任务的执行流程：

1. 当一个任务提交给线程池时，如果当前运行的线程数少于 `corePoolSize`，即使有空闲线程，线程池也会优先创建新的核心线程来执行任务。
2. 如果当前运行的线程数已经达到或超过 `corePoolSize`，新提交的任务会被添加到 `workQueue` 中等待执行。
3. 如果 `workQueue` 已满，且当前运行的线程数少于 `maximumPoolSize`，线程池会创建新的非核心线程来执行队列中的任务。
4. 如果 `workQueue` 已满，并且当前运行的线程数已经达到 `maximumPoolSize`，线程池会根据 `RejectedExecutionHandler` 定义的拒绝策略来处理新提交的任务。

---

### 自定义参数的最佳实践

#### 1. 根据任务类型选择线程数

这是最重要的决策点之一。

* **CPU 密集型任务：**
  * **特点：** 任务执行主要消耗CPU资源，计算量大，I/O操作少。
  * **核心思想：** 线程数过多会导致频繁的上下文切换，降低效率。
  * **建议：**`corePoolSize` 和 `maximumPoolSize` 通常设置为 **CPU核心数 + 1** 或 ​**CPU核心数**​。
    * 公式：`N_threads = N_CPU` 或 `N_CPU + 1`
    * 例如：四核CPU，可设为 4 或 5。
* **I/O 密集型任务：**
  * **特点：** 任务执行主要等待I/O操作（如网络请求、数据库查询、文件读写），CPU利用率相对较低。
  * **核心思想：** 在等待I/O时，CPU可以切换到其他线程，因此可以创建更多线程来提高整体吞吐量。
  * **建议：**`corePoolSize` 和 `maximumPoolSize` 通常设置为 **2倍CPU核心数 + 1** 或更高。
    * 公式：`N_threads = N_CPU * (1 + (I/O耗时 / CPU耗时))` 或 `2 * N_CPU`
    * 例如：四核CPU，可设为 8 或 10。但具体数值需要通过压测确定。

#### 2. 选择合适的工作队列 `BlockingQueue`

* **`ArrayBlockingQueue` (有界队列)：**
  * **特点：** 基于数组实现，需要指定容量。
  * **优点：** 可以有效防止内存溢出，提供更可控的队列行为。
  * **适用场景：** 任务提交速率可能远超处理速率，需要防止任务无限堆积的场景。当队列满时，会触发拒绝策略。
* **`LinkedBlockingQueue` (无界队列)：**
  * **特点：** 基于链表实现，默认容量为 `Integer.MAX_VALUE`，可以看作是无界队列。
  * **优点：** 任务可以无限加入队列，不会触发 `maximumPoolSize` 和拒绝策略（除非系统内存耗尽）。
  * **适用场景：** 对任务吞吐量要求高，且任务生产速度和处理速度相对平衡的场景。**注意：** 如果任务生产速度远大于处理速度，容易造成内存溢出。
* **`SynchronousQueue` (同步移交队列)：**
  * **特点：** 不存储任务，每个提交的任务都必须立即被一个工作线程消费，否则就拒绝。
  * **优点：** 任务提交后直接由线程处理，适用于即时处理的场景，且可以完全由 `corePoolSize` 和 `maximumPoolSize` 控制线程数。
  * **适用场景：** 对任务提交和执行的同步性要求高，或需要精确控制并发线程数的场景。

#### 3. 选择合适的拒绝策略 `RejectedExecutionHandler`

* **`ThreadPoolExecutor.AbortPolicy` (默认策略)：**
  * **行为：** 抛出 `RejectedExecutionException` 异常。
  * **适用场景：** 对任务丢失零容忍，希望立即知道任务提交失败的场景。调用者需要处理异常。
* **`ThreadPoolExecutor.CallerRunsPolicy`：**
  * **行为：** 不抛出异常，而是由提交任务的线程（调用者线程）来执行这个任务。
  * **适用场景：** 希望任务不丢失，并且通过减缓任务提交速度来“自适应”负载的场景。
* **`ThreadPoolExecutor.DiscardOldestPolicy`：**
  * **行为：** 丢弃队列中最老的任务，然后尝试提交当前任务（如果队列仍满则继续丢弃）。
  * **适用场景：** 对任务时效性有要求，可以接受少量任务丢失，但希望新任务能被尽快处理的场景。
* **`ThreadPoolExecutor.DiscardPolicy`：**
  * **行为：** 直接丢弃新提交的任务，不抛出异常。
  * **适用场景：** 对任务丢失可以接受，且不希望影响提交者线程的场景。
* **自定义拒绝策略：**
  * 可以实现 `RejectedExecutionHandler` 接口，自定义更复杂的拒绝逻辑，例如记录日志、持久化到消息队列、发送告警等。

#### 4. 自定义 `ThreadFactory`

* **目的：** 为线程池中的线程设置有意义的名称，这在进行线程栈分析、日志排查和监控时非常有用。
* **建议：** 使用如 Google Guava 的 `ThreadFactoryBuilder` 或自定义实现。
  **Java**
  
  ```
  new ThreadFactoryBuilder().setNameFormat("My-Service-Thread-%d").build();
  ```

#### 5. 合理设置 `keepAliveTime`

* **短时间（如几秒）：** 适用于任务处理有明显波峰波谷的场景，可以及时回收空闲线程，节省资源。
* **长时间或0：** 适用于线程池需要长期保持较高活跃度，或者不希望频繁创建/销毁线程的场景。如果设置为0，非核心线程在空闲时不会被销毁。

#### 6. 优雅关闭线程池

* **`shutdown()`：** 不再接受新任务，但会等待已提交的任务（包括队列中的任务）执行完成。
* **`shutdownNow()`：** 尝试停止所有正在执行的任务，并清空队列中的任务。它会中断正在运行的线程。
* **配合使用 `awaitTermination()`：** 在调用 `shutdown()` 后，可以使用 `awaitTermination()` 等待所有任务在指定时间内完成，否则可以考虑采取更强制的措施。

---

### 常见陷阱与反模式

1. **盲目使用无界队列 (`LinkedBlockingQueue`)：** 最常见的陷阱。可能导致内存溢出，即使 `maximumPoolSize` 设定了上限，也无法生效。
2. **`maximumPoolSize` 设置过大：** 导致创建过多线程，频繁上下文切换，耗尽CPU和内存资源，性能反而下降。
3. **未处理拒绝策略：** 默认的 `AbortPolicy` 可能会导致任务丢失且程序崩溃。
4. **未进行线程命名：** 导致在 `jstack` 或日志中难以区分不同线程池的线程，增加问题排查难度。
5. **忽略优雅关闭：** 导致应用关闭时，正在运行的任务被粗暴中断或任务丢失。
6. **混淆CPU密集型和I/O密集型任务的配置：** 导致CPU密集型任务因为线程过多而性能下降，或I/O密集型任务因为线程过少而吞吐量不足。

---

### 总结

自定义 `ThreadPoolExecutor` 参数是 Java 并发编程中的一项关键技能。没有一劳永逸的“最佳”配置，所有参数都应根据​**任务类型**​、**系统资源**和**业务需求**进行权衡和调整。

* **理解原理：** 掌握线程池的执行流程是正确配置的基础。
* **区分任务类型：** CPU密集型和I/O密集型任务的线程池配置策略截然不同。
* **谨慎选择队列：** 尤其是有界队列和无界队列的选择，关系到系统的稳定性和任务处理能力。
* **完善拒绝策略：** 定义线程池过载时的行为，确保系统的健壮性。
* **加强可观测性：** 通过自定义 `ThreadFactory` 和适当的监控，提升线程池的可观测性，便于问题定位和性能调优。

通过精细化地配置 `ThreadPoolExecutor`，我们可以更好地管理并发任务，优化系统性能，提升应用的稳定性和响应速度。



附1：生产环境配置示例
```
        /**
         * @description:  冷热数据分离专用线程池
         * @author: PengHL
         * @date: 2023/6/15 14:39
         * @param:
         * @return:
         **/
        dataSeparationExecutor = new ThreadPoolExecutor(
                8,//核心线程数
                8,//最大线程数
                keepAliveTime,//存活时间
                TimeUnit.SECONDS,//存活单位(秒
                new LinkedBlockingDeque<>(800000),//阻塞队列（指定容量
                new CustomizableThreadFactory("dataSeparationExecutor-"),//线程工厂
                new ThreadPoolExecutor.CallerRunsPolicy()//拒绝策略
        );

```
