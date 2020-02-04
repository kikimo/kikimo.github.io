---
title: "Linux 中的 cmpxchg 宏"
date: 2020-02-02T23:51:12+08:00
draft: false
---

cmpxchg 是 intel CPU 指令集中的一条指令，
这条指令经常用来实现原子锁，
我们来看 [intel 文档](https://www.felixcloutier.com/x86/cmpxchg)中对这条指令的介绍：

> Compares the value in the AL, AX, EAX, or RAX register with the first operand (destination operand). If the two values are equal, the second operand (source operand) is loaded into the destination operand. Otherwise, the destination operand is loaded into the AL, AX, EAX or RAX register. RAX register is available only in 64-bit mode.

> This instruction can be used with a LOCK prefix to allow the instruction to be executed atomically. To simplify the interface to the processor’s bus, the destination operand receives a write cycle without regard to the result of the comparison. The destination operand is written back if the comparison fails; otherwise, the source operand is written into the destination. (The processor never produces a locked read without also producing a locked write.)

可以看到 cmpxchg 指令有两个操作数，
同时还使用了 AX 寄存器。
首先，它将第一个操作数（目的操作数）和 AX 寄存器相比较，
如果相同则把第二个操作数（源操作数）赋值给第一个操作数，ZF 寄存器置一，
否则将第一个操作数赋值给 AX 寄存器且 ZF 寄存器置零。
在多核环境中，一般还在指令前加上 LOCK 前缀，来保证指令执行的原子性（LOCK 前缀的主要功能应该是锁内存总线）。

注：AT&T 风格的汇编语法中，第一个操作数是源操作数，第二个操作数是目的操作数。

`cmpxchg` 指令的操作具有原子性，可以用以下伪代码来表示：

```C
if (dst == %ax) {
    dst = src;
    ZF = 1;
} else {
    %ax = dst
    ZF = 0;
}
```

Linux 内核代码中使用宏来封装`cmpxchg`指令操作，相关源码在[这里](https://elixir.bootlin.com/linux/latest/source/tools/arch/x86/include/asm/cmpxchg.h#L86):

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

从源码中可以看到，主体封装代码为宏```__raw_cmpxchg```，
它使用内联汇编来调用```cmpxchg```指令。
代码中包含了对不同字长参数的处理，
只要分析一种情况即可：

```C
__typeof__(*(ptr)) __ret;                   \
__typeof__(*(ptr)) __old = (old);               \
__typeof__(*(ptr)) __new = (new);               \
switch (size) {                         \
...
case __X86_CASE_W:                      \
{                               \
    volatile u16 *__ptr = (volatile u16 *)(ptr);        \
    asm volatile(lock "cmpxchgw %2,%1"          \
             : "=a" (__ret), "+m" (*__ptr)      \
             : "r" (__new), "0" (__old)         \
             : "memory");               \
    break;                          \
}
...
```

这段代码展开后等价于：
```asm
cmpxchg(ptr, old, new)
{
    %ax = __old;
    cmpxchgw __new, *__ptr
    if (*ptr == __old) {
        *__ptr = __new;
        // return __old;
    } else {
        %ax = *__ptr;
        // return *__ptr;
    }

    __ret = %ax;
    return __ret;
}
```

去掉中间寄存器：
```asm
cmpxchg(ptr, old, new)
{
    if (*ptr == __old) {
        *ptr == __new;
        return __old;
    } else {
        return __new;
    }
}
```

现在，`cmpxchg`宏的作用就一目了然了。
如果 `ptr` 值和 `old`相等，
则将`new`赋值给`ptr`且返回`old`，
否则返回`new`。
