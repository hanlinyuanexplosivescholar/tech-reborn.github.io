第一步、使用top命令，找出占用 CPU 最高的进程 ID (PID)
第二步、确定是哪个 Java 进程（PID）占用 CPU 最高后，需要定位是该进程内部的哪个线程出了问题。

| 工具 | 命令 | 目的 |
| :--- | :--- | :--- |
| `top -H -p <PID>` | `top -H -p 12345` | **线程级别**查看指定进程内的 CPU 消耗。按 `P` 键排序，找出占用 CPU 最高的**线程 ID (TID)**。 |
| `printf '%x\n' <TID>` | `printf '%x\n' 54321` | 将高 CPU 线程的十进制 ID (TID) **转换为十六进制**。这是因为 Java 线程 Dump 中线程 ID 是十六进制的。 |
| `jstack <PID>` | `jstack 12345 > thread_dump.td` | 生成 Java 进程的**线程堆栈快照 (Thread Dump)**。必须在 CPU 飙高时立刻执行，至少连续执行 3 次，每次间隔 5-10 秒。 |

第三步、分析线程 Dump (Thread Dump)

利用生成的 `thread_dump.td` 文件，结合十六进制的 TID 进行分析。

1.  **搜索高 CPU 线程：**
    在 `thread_dump.td` 文件中搜索之前转换得到的**十六进制 TID**。

    ```bash
    # 在文件中搜索十六进制TID
    # 查找 "nid=0x<十六进制TID>"
    grep -A 20 "nid=0x54321" thread_dump.td
    ```

2.  **分析线程状态和调用栈：**

      * **状态：** 查看线程状态（如 `RUNNABLE`）。高 CPU 的线程通常处于 `RUNNABLE` 状态。
      * **调用栈 (Stack Trace)：** 仔细阅读该线程的调用栈。调用栈会清晰显示该线程正在执行的类和方法。

3.  **确定瓶颈类型：**

      * **如果栈中看到大量的业务代码：** 可能是代码中存在**无限循环、正则匹配效率低下、或大量无谓的 CPU 密集型计算**。
      * **如果栈中看到 `java.util.concurrent` 或锁相关代码：** 可能存在**死锁或锁竞争激烈**，导致大量线程被阻塞，CPU 资源浪费在上下文切换上（但这通常表现为 `sy` 偏高）。
      * **如果栈中看到 GC 相关代码：** 可能是频繁进行 Full GC 或 Young GC 耗费了大量 CPU。需要进行下一步内存分析。


第四步：内存和 GC 检查（如果怀疑是 GC 问题）

如果线程 Dump 或初步观察发现 CPU 高负载与 GC 相关，则需要检查内存和垃圾回收情况。

| 工具 | 命令 | 目的 |
| :--- | :--- | :--- |
| `jstat -gc <PID> 1000` | `jstat -gc 12345 1000` | 持续监控 GC 统计数据（每 1000ms 输出一次）。 |
| **关注指标：** | `YGC` (Young GC 次数)、`FGC` (Full GC 次数)、`FGCT` (Full GC 耗时)。 | 如果 `FGC` 频繁且 `FGCT` 很高，说明 GC 成为瓶颈。 |
| `jmap -histo:live <PID>` | `jmap -histo:live 12345` | 查看堆中存活的对象统计，判断是否有内存泄漏或大量临时对象。 |


第五步：使用性能分析工具（Profiling）
  * **Arthas (阿里巴巴开源)：** 使用 `thread -n 3 --state RUNNABLE` 快速定位最忙的线程，并可使用 `trace` 或 `monitor` 命令观察方法耗时。
