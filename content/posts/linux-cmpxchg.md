---
title: "Linux 中的 cmpxchg"
date: 2020-02-02T23:51:12+08:00
draft: true
---

cmpxchg 是在 intel CPU 指令集中的一条指令，
这条指令经常用来实现原子锁，
我们来看 intel 文档中对这条指令的介绍：

> Compares the value in the AL, AX, EAX, or RAX register with the first operand (destination operand). If the two values are equal, the second operand (source operand) is loaded into the destination operand. Otherwise, the destination operand is loaded into the AL, AX, EAX or RAX register. RAX register is available only in 64-bit mode.

> This instruction can be used with a LOCK prefix to allow the instruction to be executed atomically. To simplify the interface to the processor’s bus, the destination operand receives a write cycle without regard to the result of the comparison. The destination operand is written back if the comparison fails; otherwise, the source operand is written into the destination. (The processor never produces a locked read without also producing a locked write.)

可以看到 cmpxchg 指令有两个操作数，
同时还使用了 AX 寄存器。
首先，它将第一个操作数（目的操作数）和 AX 寄存器相比较，
如果相同则把第二个操作数（源操作数）的值加载到第一个操作数上，
否则将第一个操作数的值加载到 AX 寄存器上。
在多核环境中，一般还在指令前加上 LOCK 前缀，来保证指令执行的原子性（LOCK 前缀的主要功能应该是锁内存总线）。

Linux 内核代码中使用宏来封装 cmpxchg 的操作，相关源码在[这里](https://elixir.bootlin.com/linux/latest/source/tools/arch/x86/include/asm/cmpxchg.h#L86):

```C
/* SPDX-License-Identifier: GPL-2.0 */
#ifndef TOOLS_ASM_X86_CMPXCHG_H
#define TOOLS_ASM_X86_CMPXCHG_H

#include <linux/compiler.h>

/*
 * Non-existant functions to indicate usage errors at link time
 * (or compile-time if the compiler implements __compiletime_error().
 */
extern void __cmpxchg_wrong_size(void)
	__compiletime_error("Bad argument size for cmpxchg");

/*
 * Constants for operation sizes. On 32-bit, the 64-bit size it set to
 * -1 because sizeof will never return -1, thereby making those switch
 * case statements guaranteeed dead code which the compiler will
 * eliminate, and allowing the "missing symbol in the default case" to
 * indicate a usage error.
 */
#define __X86_CASE_B	1
#define __X86_CASE_W	2
#define __X86_CASE_L	4
#ifdef __x86_64__
#define __X86_CASE_Q	8
#else
#define	__X86_CASE_Q	-1		/* sizeof will never return -1 */
#endif

/*
 * Atomic compare and exchange.  Compare OLD with MEM, if identical,
 * store NEW in MEM.  Return the initial value in MEM.  Success is
 * indicated by comparing RETURN with OLD.
 */
#define __raw_cmpxchg(ptr, old, new, size, lock)			\
({									\
	__typeof__(*(ptr)) __ret;					\
	__typeof__(*(ptr)) __old = (old);				\
	__typeof__(*(ptr)) __new = (new);				\
	switch (size) {							\
	case __X86_CASE_B:						\
	{								\
		volatile u8 *__ptr = (volatile u8 *)(ptr);		\
		asm volatile(lock "cmpxchgb %2,%1"			\
			     : "=a" (__ret), "+m" (*__ptr)		\
			     : "q" (__new), "0" (__old)			\
			     : "memory");				\
		break;							\
	}								\
	case __X86_CASE_W:						\
	{								\
		volatile u16 *__ptr = (volatile u16 *)(ptr);		\
		asm volatile(lock "cmpxchgw %2,%1"			\
			     : "=a" (__ret), "+m" (*__ptr)		\
			     : "r" (__new), "0" (__old)			\
			     : "memory");				\
		break;							\
	}								\
	case __X86_CASE_L:						\
	{								\
		volatile u32 *__ptr = (volatile u32 *)(ptr);		\
		asm volatile(lock "cmpxchgl %2,%1"			\
			     : "=a" (__ret), "+m" (*__ptr)		\
			     : "r" (__new), "0" (__old)			\
			     : "memory");				\
		break;							\
	}								\
	case __X86_CASE_Q:						\
	{								\
		volatile u64 *__ptr = (volatile u64 *)(ptr);		\
		asm volatile(lock "cmpxchgq %2,%1"			\
			     : "=a" (__ret), "+m" (*__ptr)		\
			     : "r" (__new), "0" (__old)			\
			     : "memory");				\
		break;							\
	}								\
	default:							\
		__cmpxchg_wrong_size();					\
	}								\
	__ret;								\
})

#define __cmpxchg(ptr, old, new, size)					\
	__raw_cmpxchg((ptr), (old), (new), (size), LOCK_PREFIX)

#define cmpxchg(ptr, old, new)						\
	__cmpxchg(ptr, old, new, sizeof(*(ptr)))


#endif	/* TOOLS_ASM_X86_CMPXCHG_H */
```

从源码中可以看到，主体代码为宏```__raw_cmpxchg```，
它使用内联汇编来调用```cmpxchg```指令。
Linux C 的内联汇编格式如下：

```C
asm ( assembler template
    : output operands                   (optional)
    : input operands                    (optional)
    : clobbered registers list          (optional)
    );
```
output operands 和inpupt operands 指定输入和输出参数，
参数从做到右排列，可以使用数字编号来引用，编号从 0 开始。
例如在```__raw_cmpxchg```宏中，%0 对应 ```__ret``` 变量，%1 对应 ```*__ptr```。
```m```表示操作数在内存中。

```=a```表示把结果写入到寄存器中，```a```指代```AX```寄存器。
```+m```表示TODO；
```
