---
layout: post
title: xv6系列：Pagetable与Meltdown攻击
date: 2023-03-29
categories: blog
tags: [OS,Pagetable,Meltdown]
description: Pagetable与Meltdown攻击
---

# Pagetable与Meltdown攻击

在[pgtal](https://pdos.csail.mit.edu/6.828/2020/labs/pgtbl.html)实验的`copyin/copyinstr`部分提到让进程的页表既映射用户态地址，又映射内核态地址，这样可以减少进程陷入内核态时切换页表造成的开销，但是会造成诸如`Meltdown`这类的侧信道攻击(side-chanel attacks)。

**一段Meltdown程序**
```c
uint8_t probe_array[256][4096];
uint_t *p = 0x8f000000;

__try {
    *(uint32_t*)NULL = 0;
    probe_array[*p][0]++;
}
__except (EXCEPTION_EXECUTE_HANDLER) {}
```


由于CPU采用了乱序执行和分支预测的策略，因此CPU会预先执行`probe_array[*p][0]++;`这句话，而后因为`*(uint32_t*)NULL = 0;`造成了异常，便丢弃执行结果。但是在执行过程中，会预先将`probe_array[*p][0]`加载从内存到cache中，因此再次访问时`probe_array[*p]`处的访问速度要比其它位置更快，这便可以推断出`*p`的值。

而如果`kernel`和`user`采用两个页表，则CPU在乱序执行`probe_array[*p][0]++;`这句话时，根据`user`的页表根本无法找到`*p`对应的物理地址，更不用谈将其缓存到cache中了，这就可以防止侧信道攻击。