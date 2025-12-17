RISC-V 极简操作系统启动程序文档
1. 项目概述本项目是一个基于 RISC-V 架构的最小操作系统内核原型。它演示了硬件加电后，内核如何从汇编代码开始执行，初始化硬件环境（栈、BSS 段、串口），并最终进入多核 C 语言运行环境。

2. 启动流程 (Boot Flow)
系统的启动遵循以下路径：硬件加电 -> entry.S (汇编入口) -> start.c (硬件初始化) -> main.c (逻辑入口)
2.1 汇编入口 (entry.S)
当 CPU 复位时，跳转至 _entry 符号处开始执行：设置启动栈：为每个 CPU 核心（Hart）分配独立的栈空间。栈顶地址计算公式为：$sp = CPU\_stack + (hartid + 1) \times 4096$。清零 BSS 段：使用链接脚本定义的 edata 和 end 符号，将未初始化数据段清零，确保全局变量初始值为 0。跳转：调用 call start 进入 C 语言环境。
2.2 架构配置 (start.c)
多核识别：通过读取控制状态寄存器 mhartid 获取当前核心编号。初始化环境：主核 (Hart 0)：负责调用 print_init() 初始化串口和打印锁，并输出 "Hello OS"。
从核：目前进入 wfi (Wait For Interrupt) 低功耗等待循环。

3. 核心组件说明
3.1 串口驱动 (uart.c)实现了对 16550a UART 控制器的底层访问：初始化：设置波特率（38.4K）、数据位（8bit）并启用 FIFO。同步输出：uart_putc_sync 采用轮询方式，等待 LSR_TX_IDLE 标志位置位后写入数据。
3.2 打印系统 (print.c)
自旋锁保护：为了防止多核同时打印导致字符错乱，puts 函数受 print_lk 锁保护。Panic 机制：当系统发生不可恢复错误时，panic 会设置全局标志并尝试绕过锁直接输出错误信息，最后挂起系统。
3.3 并发控制 (spinlock.c)
实现了基础的自旋锁：原子性：使用 RISC-V 的原子交换指令 __sync_lock_test_and_set。中断安全：acquire 时会调用 push_off 关闭中断并增加嵌套计数；release 时通过 pop_off 恢复，确保持有锁期间不会被中断打断。
4. 编译与链接配置
4.1 编译选项 (common.mk)
交叉编译：使用 riscv64-linux-gnu- 工具链。内核参数：-mcmodel=medany：寻址模式配置。-ffreestanding：表明程序不依赖标准库。-nostdlib：不链接标准 C 库。
4.2 内存布局 (kernel.ld)
定义了内核在内存中的分布（起始地址通常为 0x80000000）：
| 段名称 | 描述 |
| .text | 代码段，包含 _entry 入口 |
| .rodata | 只读数据段（常量、字符串） |
| .data | 已初始化数据段 |
| .bss | 未初始化数据段（自动清零） |