---
layout: post
title: xv6系列：Lab2核心流程分析
date: 2025-06-26
categories: blog
tags: [OS]
description: 初步全面认知操作系统
---


`Lab2`的核心是了解整体了解操作系统并学会创建`syscall`。
## xv6读书笔记
> 笔记范围：Chapter-2，Section 4.3 & 4.4

**操作系统的3大核心目标：**
- 多工（multiplexing）：不同进程可以时分复用资源
- 隔离（isolation）：进程之间的隔离，避免错误进程干扰无关进程
- 交互（interaction）：进程之间也不能完全隔离，需要进行信息交互，例如`pipeline`

**用户态，内核态，系统调用**
能够实现用户态与内核态隔离性的硬件保证是`CPU`提供的隔离（[[cpu mode]]）
`RISC-V`提供了3中模式：
- 机器模式（machine mode）：拥有全部权限，CPU初启动时是机器模式，用于系统配置
- 内核态（supervisor mode）：可以执行特权指令（privileged instructions），例如修改页表寄存器，开关中断等
- 用户态（user mode）
用户态的程序执行系统调用时，`CPU`会使用特殊的指令[[ecall]]去切换模式，并且进入指定的`entry point`

### xv6启动流程
1. `Makefile`中的关键参数
	1. `-bios none`：这个标志告诉 QEMU **不要加载** 任何默认的 BIOS 或固件（比如 OpenSBI）。
	2. **`-kernel kernel/kernel`**: 这个标志告诉 QEMU 直接将 `kernel/kernel` 这个二进制文件加载到内存的起始地址（`0x80000000`,`0x80000000` 是 QEMU `virt` 平台为操作系统内核设定的一个事实上的标准加载地址），然后把程序计数器（PC）指向这里，直接开始执行。
```Makefile
QEMUOPTS = -machine virt -bios none -kernel kernel/kernel ...
```

2. `0x80000000`对应`entry.S`：所有核心都从 `entry.S` 开始执行
	1. **为当前核心设置栈**：它计算出当前核心应该使用的栈顶地址，并将其加载到 `sp` 寄存器。公式为 `sp = stack0_base_address + (hartid + 1) * 4096`。这确保了每个核心都有自己独立的、不冲突的栈空间。
	2. **调用 C 函数**：一旦栈准备就绪，它就执行 `call start`，将控制权交给 `kernel/start.c` 中的 `start()` C 函数。

> 即使 RISC-V 的启动过程比 x86 简单，但 C 语言的运行有一个最基本的前提：**必须有一个合法的栈（Stack）**。任何 C 函数的调用（包括参数传递、局部变量存储）都依赖于栈指针寄存器 `sp` 被正确设置。
> 而 CPU 刚上电时，`sp` 寄存器里的值是未定义的。因此，在调用任何 C 函数（即便是 `start()` 函数）之前，必须有一段汇编代码来完成这项最原始的设置工作。
> 这个工作就是由 `entry.S` 完成的。

3. `start.c`执行（M-Mode）：继续执行 M-Mode 初始化工作：
	1. 设置 `mstatus` 和 `mepc`，为 `mret` 到 S-Mode 的 `main` 函数做准备。
	2. 委托所有中断和异常给 S-Mode。
	3. 初始化 M-Mode 的定时器 (`timerinit()`)。
4. **`main.c` 执行 (S-Mode)** `start()` 函数最后执行 `mret`，CPU 切换到 S-Mode，并跳转到 `main()` 函数。核心 0 进行初始化，其他核心等待。
```c
// kernel/main.c
void main() {
  if(cpuid() == 0){
    // 我是启动核心 (hart 0)
    // 执行所有主要的初始化...
    // ...
    // 唤醒其他核心（通过发送 IPI 或其他机制）
  } else {
    // 我是其他核心
    // 在这里等待，直到启动核心完成初始化...
    // ...
    // 完成自己的每核心初始化，然后进入调度循环
  }
  scheduler();
}
```
## 实现过程中关键操作
### 用户态与内核态的信息交互
> 参数返回值的传递主要依赖`struct proc`中的`trapframe`

**系统调用**
- 当 `ecall` 发生时，会将用户态的所有寄存器值自动保存在一个地方，这个地方就是当前进程的**陷阱帧 (`trapframe`)**，它正是 `struct proc` 的一个重要成员 (`p->trapframe`)。  
- 内核函数通过访问当前进程的 `p->trapframe` 来获取用户传递的参数。
**内核态向用户态返回数据**
- 通过`trapframe`返回简单值
	- 内核函数（如 `sys_fork`）执行完毕后，会将返回值直接写入当前进程陷阱帧的 `a0` 寄存器中 (`p->trapframe->a0 = ...`)。当内核执行 `sret` 指令返回用户态时，`a0` 寄存器的值就被恢复了，用户程序可以直接读取 `a0` 寄存器获得返回值。
	- `trapframe` 作为 `struct proc` 的一部分，是内核向用户态传递返回值的桥梁。 
- 向用户地址空间拷贝数据[[todo]]
	- 当需要返回大量数据时（例如 `read` 系统调用读取文件内容），不能只通过一个寄存器。

