# RISC-V 进程管理与多任务调度说明文档

## 1. 进程控制块 (PCB) 与状态机

系统使用 `struct proc` 记录进程的所有元数据。在内核态，每个进程都拥有独立的**内核栈**，用于处理陷阱和函数调用。

### 1.1 核心字段解析

* **`context`**: 存放调度所需的寄存器快照。
* **`kstack`**: 物理内存分配的 4KB 页面。
* **`chan` (Wait Channel)**: 标识进程正在等待的特定事件地址。
* **`lock`**: 细粒度自旋锁，保护该进程的状态迁移，防止多个 CPU 同时操作同一进程。

### 1.2 状态迁移图

---

## 2. 上下文切换的底层逻辑 (`swtch.S`)

当 `scheduler()` 决定运行某个进程，或者进程调用 `yield()` / `sleep()` 主动让出 CPU 时，会触发 `swtch` 函数。

### 2.1 为什么只保存 Callee-saved 寄存器？

由于 `swtch` 是作为一个标准的 C 函数被调用的，根据 RISC-V 调用约定（Calling Convention）：

* **Caller-saved (`a0-a7`, `t0-t6`)**: 调用者（编译器生成的代码）已经负责在进入 `swtch` 前将其压栈。
* **Callee-saved (`s0-s11`, `ra`, `sp`)**: `swtch` 必须手动保存这些寄存器，以确保切换回原路径后，执行流能感知不到“被中断过”。

### 2.2 踏板函数 (`proc_trampoline`)

新创建的进程 `context.ra` 会指向 `proc_trampoline`。

* 它的作用是作为新进程执行的“第一站”。
* 它负责从 `swtch` 的汇编环境平滑过渡到 C 语言的入口函数（如 `simple_task`）。
* 在函数返回后，它负责调用 `exit()`，确保进程不会因“跑飞”而崩溃。

---

## 3. 陷阱处理中的系统调用 (System Calls)

随着进程的引入，`trap_kernel_handler`（在 `trap_kernel.c` 中）现在具备了处理同步请求的能力：

* **非法地址与写时复制 (COW) 预留**: 虽然目前主要运行内核线程，但异常处理逻辑已预留了对 `Instruction/Store Page Fault` 的拦截，为后续用户态内存管理奠定基础。
* **抢占式调度**: 时钟中断处理函数 `timer_interrupt_handler` 现在会定期调用 `yield()`。这意味着即便一个任务不主动让出 CPU，系统也会通过硬中断强制进行任务轮换。

---

## 4. 复杂同步场景测试案例 (`main.c`)

为了验证锁、调度与进程间通信的正确性，`main.c` 模拟了经典的 **生产者-消费者模型**：

### 4.1 生产者逻辑 (`producer_task`)

1. 获取 `sync_lock`。
2. 检查缓冲区是否已满（`sync_buffer == sync_capacity`）。
3. 如果满，调用 `sleep(&sync_buffer, &sync_lock)`。
4. 生产数据，递增 `produced_total`。
5. 调用 `wakeup(&sync_buffer)` 唤醒等待的消费者。

### 4.2 消费者逻辑 (`consumer_task`)

1. 获取 `sync_lock`。
2. 检查缓冲区是否为空（`sync_buffer == 0`）。
3. 如果空，调用 `sleep(&sync_buffer, &sync_lock)` 挂起。
4. 消费数据，递减 `sync_buffer`。
5. 调用 `wakeup(&sync_buffer)` 提醒生产者缓冲区已有空位。

### 4.3 压力验证

* 通过并发启动多个生产/消费线程，系统验证了 `sleep` 释放锁的原子性，以及 `wakeup` 是否会造成死锁或竞争条件。

---

## 5. 进程回收机制：Wait 与 Exit

1. **`exit(code)`**:
* 将子进程状态改为 `PROC_ZOMBIE`。
* 将其子进程（如果有）托孤给 PID 1 (init)。
* 唤醒正在等待的父进程。


2. **`wait_process(pid)`**:
* 扫描进程表，寻找处于 `ZOMBIE` 状态的指定子进程。
* 提取退出码，并调用 `pmem_free` 释放子进程的内核栈。
* 将 PCB 标记回 `PROC_UNUSED`，完成资源闭环。