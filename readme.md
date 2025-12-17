# 深度解析：多级反馈队列 (MLFQ) 调度器实现

## 1. MLFQ 设计目标

在现代操作系统中，调度器需要同时满足两个矛盾的目标：

1. **低延迟（Responsiveness）**：交互式作业（如键盘输入、UI 响应）应能立即获得 CPU。
2. **高吞吐（Throughput）**：计算密集型作业（如科学计算、编译）应能在长的时间片内高效运行，减少上下文切换开销。

MLFQ 通过**观察进程的历史行为**来动态预测其未来需求，从而完美解决了这一问题。

---

## 2. 详细实现机制 (`proc.c`)

### 2.1 队列规则与参数

系统内部逻辑上维护了三个等级（Level 0, 1, 2），其配置如下表：

| 等级 (Level) | 时间片 (Ticks) | 调度算法 | 目标进程类型 |
| --- | --- | --- | --- |
| **0 (最高)** | 2 | 抢占式轮转 | 交互式、短任务 |
| **1 (中等)** | 4 | 抢占式轮转 | 中等长度任务 |
| **2 (最低)** | 8 | 轮转 (RR) | 长耗时计算任务 |

### 2.2 进程状态转移图

MLFQ 的核心在于“奖惩机制”和“保底机制”：

* **初始分配**：所有新进程（`alloc_process`）默认进入 **Level 0**。
* **降级触发**：
* 在 `usertrap()` 或时钟中断中，系统累加 `p->ticks_consumed`。
* 若 `p->ticks_consumed >= mlfq_quantum[p->cur_level]`，且进程状态仍为 `RUNNABLE`（意味着它用完了时间片还没主动放弃 CPU），则执行：
```c
if (p->cur_level < MLFQ_LEVELS - 1) {
    p->cur_level++; // 降级
}
p->ticks_consumed = 0; // 重置计数

```




* **升级触发（优先级提升/Aging）**：
为了解决低优先级进程的“饥饿”问题，`scheduler()` 循环会检查：
* `last_scheduled`：记录进程上一次被选中的时间戳。
* 若 `(current_ticks - p->last_scheduled) > mlfq_aging_threshold`：
```c
p->cur_level = 0; // 提升至最高优先级
p->ticks_consumed = 0;

```





---

## 3. main.c 测试演示解析

`run_priority_mlfq_demo()` 函数通过构造不同特征的“作业流”来验证上述逻辑。

### 3.1 演示案例配置

测试定义了一个 `priority_demo_jobs` 数组：

1. **"Job_Interactive" (Prio 0)**：高初始优先级，较小工作量。预期：始终保持在 Level 0，快速完成。
2. **"Job_Batch" (Prio 20)**：低初始优先级，巨大工作量。预期：迅速降至 Level 2，仅在 Level 0/1 空闲时或被 Aging 提升时运行。
3. **"Job_Mixed" (Prio 10)**：中等优先级。预期：表现介于两者之间。

### 3.2 关键验证点

* **抢占验证**：当 Job_Interactive 变为 `RUNNABLE` 时，它是否能打断正在运行的 Job_Batch？（通过观察 PID 切换频率验证）。
* **降级验证**：通过 `klog` 观察 Job_Batch 的 `level` 字段变化。
* **Aging 验证**：在测试后期，人为阻塞高优先级任务，观察低优先级任务是否因 Aging 重新获得了极短的 Level 0 时间片。

---

## 4. 调试与性能监控

你可以通过 `sys_klog` (系统内核日志) 来观察调度器的实时决策。

### 4.1 如何查看调度日志

内核在每次降级或提升优先级时会记录日志，你可以在用户态运行 `logread` 程序：

```bash
# 在系统 shell 中运行
$ logread

```

**日志示例：**

```text
[DEBUG] MLFQ: pid 5 downgraded to level 1 (consumed 2 ticks)
[INFO] MLFQ: pid 8 aging boost to level 0 (wait time: 17 ticks)

```

### 4.2 核心指标监控

* **Ticks Consumed**: 衡量进程的 CPU 密集程度。
* **Wait Time**: 衡量系统公平性。
* **Context Switch Count**: 衡量 MLFQ 是否有效减少了长任务的无效切换。

---

## 5. 总结：MLFQ 的优越性

1. **无须预知运行时间**：它通过观察过去来预测未来。
2. **自适应性**：一个计算密集型进程如果突然开始等待 I/O（变为交互式），它会在几次调度后通过 Aging 或保持在低层级迅速响应，随后再次降级，保持系统的高效。
3. **参数可调**：可以通过修改 `mlfq_quantum` 和 `mlfq_aging_threshold` 来适配不同的硬件性能。