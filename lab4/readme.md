# RISC-V 异常与中断系统说明文档

## 1. 陷阱（Trap）处理模型

在 RISC-V 中，所有的控制流非正常转移统称为 **Trap**。本项目实现了 **Supervisor Mode (S-Mode)** 下的 Trap 处理，通过 `stvec` 寄存器将硬件事件重定向到软件代码。

### 1.1 分类

* **Interrupt (中断)**：由硬件外设触发，如时钟过期、串口收到数据。
* **Exception (异常)**：由指令执行触发，如非法操作码、页表缺失、地址未对齐。

---

## 2. 核心执行流 (The Trap Path)

### 第一阶段：硬件自动处理

当 Trap 发生时，处理器硬件会自动完成：

1. 保存当前 PC 到 `sepc`。
2. 保存当前特权级到 `sstatus` 的 SPP 位。
3. 关闭 S-Mode 响应中断 (`sstatus.SIE = 0`)。
4. 根据 `stvec` 的地址跳转到 `trap.S`。

### 第二阶段：汇编上下文保存 (`trap.S`)

由于 C 语言运行需要寄存器环境，汇编入口 `kernel_vector` 执行以下操作：

* **栈空间分配**：在当前内核栈开辟 256 字节。
* **寄存器压栈**：保存 `ra`, `gp`, `tp` 以及所有 `a0-a7`, `t0-t6`, `s0-s11`。
* **进入 C 空间**：调用 `trap_kernel_handler()`。

### 第三阶段：C 语言分发 (`trap_kernel.c`)

`trap_kernel_handler` 负责解析 `scause` 寄存器：

* **中断 (Highest bit = 1)**：
* **Case 1 (SSIP)**: 来自 M-Mode 转发的时钟中断，调用 `timer_interrupt_handler`。
* **Case 9 (SEIP)**: 外部硬件中断，调用 `plic_claim()` 获取 IRQ 编号并查找注册的回调函数。


* **异常 (Highest bit = 0)**：
* 通过 `exception_info` 数组匹配错误类型。
* 调用 `panic` 输出寄存器快照，帮助开发者快速定位代码中的 Bug。



---

## 3. 时钟与外设管理

### 3.1 时钟中断 (Timer)

系统利用 **Sstc 扩展**（Supervisor-mode Timer Compare）：

* **stimecmp**: 这是一个 64 位寄存器。当硬件计数器 `time` 的值达到 `stimecmp` 时，触发时钟中断。
* **周期性实现**: 每次中断处理函数中，执行 `w_stimecmp(r_time() + INTERVAL)` 预设下一次中断，从而形成稳定的 Ticks 流。

### 3.2 PLIC 中断控制 (`plic.c`)

* **Priority**: 为每个 IRQ（如 UART 的 IRQ 10）设置优先级。
* **Threshold**: 设置当前核心的中断门槛，低于此优先级的信号被屏蔽。
* **Claim/Complete**: 典型的原子确认机制。处理前“声明”获取 IRQ，处理后“完成”释放信号。

---

## 4. 自动化测试套件详解

我们在 `main.c` 中设计了针对性的测试函数，确保系统在复杂情况下依然稳定：

### 4.1 异常隔离测试 (`test_exception_handling`)

1. **非法指令测试**：构造一段包含不支持操作码的内存并执行。
* *验证点*：`scause` 是否等于 `2` (Illegal instruction)，`sepc` 是否指向该指令地址。


2. **非法地址访问测试**：尝试向物理内存之外的地址（如 `0xFFFFFFFF...`）写入数据。
* *验证点*：`scause` 是否触发 `15` (Store/AMO page fault)，`stval` 是否包含报错的虚拟地址。



### 4.2 实时性测试 (`test_interrupt_overhead`)

通过对比**逻辑时间**与**物理周期**来评估：


* **意义**：该测试能反映出内核进入/退出 Trap 路径的机器周期损耗，是评估内核响应延迟的重要指标。

---

## 5. 关键状态寄存器总结 (CSRs)

| 寄存器 | 作用 |
| --- | --- |
| `stvec` | 陷阱向量基地址，存放 `kernel_vector` 的位置。 |
| `sepc` | 记录触发 Trap 的那条指令地址。 |
| `scause` | 记录 Trap 的具体原因（中断或异常码）。 |
| `stval` | 存放与 Trap 相关的附加值（如 Page Fault 时的地址）。 |
| `sstatus` | 控制中断使能（SIE 位）和之前的特权模式。 |

---

