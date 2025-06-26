---
layout: post
title: xv6系列：Lab1核心流程分析
date: 2025-06-26
categories: blog
tags: [OS,Syscall]
description: 初识系统调用
---

# Lab1: 初识系统调用

`Lab1`的核心主要是熟悉使用`system call`，使用系统调用完成用户的程序，包括`sleep`，`fork`等操作。
## 核心流程

**完成Lab时可以感知的流程**
1. 在`user`文件夹下创立用户程序
2. 用户程序使用`system call`完成系统调用

**内部`syscall`隐藏流程**
1.  用户程序使用`system call`完成系统调用
```c
// user/sleep.c
sleep(seconds);
```
2. 对应的`system call`在`user/usys.S`中定义，通过[[ecall]]触发系统调用
```nasm
;user.usys.
#include "kernel/syscall.h"
.global sleep
sleep:
li a7, SYS_sleep
ecall
ret
```
3. `ecall`后最终会在内核态调用`syscall`函数，`syscall`函数将根据传入的参数（`SYS_sleep`）调用指定的系统调用。参数定义在`kernel/syscall.h`中
```c
// kernel/syscall.h
#define SYS_sleep 13
```

```c
// kernel/syscall.c
static uint64 (*syscalls[])(void) = {
	[SYS_sleep] sys_sleep
};

void
syscall(void)
{
// ...
	if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
		p->trapframe->a0 = syscalls[num]();
	}
// ...
}
```
4. 最终还是通过`user/usys.S`中的`ret`返回用户态