---
title:  "ptrace 如何实现单步跟进"
date: 2017-09-18T21:57:46+08:00
draft: false
---

### 1. `ptrace()` 单步跟进（PTRACE_SINGLESTEP）源码分析

`ptrace()` 是一个重要的 Linux 系统调用，它的代码在 kernel/ptrace.c 文件中。
`ptrace()` 可以实现诸如暂停进程、观察进程内存数据、进程指令单步执行等功能，
它可以用来实现调试器，例如 gdb 调试器便是基于 `ptrace()` 实现的。
以 Linux 4.10.17 版本的代码为例，我们来分析 `ptrace()` 函数单步跟进功能的实现。
`ptrace()` 函数的定义如下：

```c
1114 SYSCALL_DEFINE4(ptrace, long, request, long, pid, unsigned long, addr,
1115                 unsigned long, data)
1116 {
1117         struct task_struct *child;
1118         long ret;
...
1149         ret = arch_ptrace(child, request, addr, data);
1150         if (ret || request != PTRACE_DETACH)
1151                 ptrace_unfreeze_traced(child);
1152 
1153  out_put_task_struct:
1154         put_task_struct(child);
1155  out:
1156         return ret;
1157 }
```

`ptrace()` 函数的一些操作是平台相关的，
这些平台相关的操作被放置到 `arch_ptrace()` 函数中，
对于不同的 CPU，`arch_ptrace()` 函数有不一样的实现。
我们要分析的单步执行功能就是一个平台相关的操作，
所以接下来要去看 x86 平台上 `arch_ptrace()` 函数的实现。
x86 的 `arch_ptrace()` 函数在文件 arch/x86/kernel/ptrace.c 中：

```c
 766 long arch_ptrace(struct task_struct *child, long request,
 767                  unsigned long addr, unsigned long data)
 768 {
 769         int ret;
 770         unsigned long __user *datap = (unsigned long __user *)data;
 771 
...
 875         default:
 876                 ret = ptrace_request(child, request, addr, data);
 877                 break;
 878         }
 879 
 880         return ret;
 881 }
```

这个文件中大部分代码和我们要分析单步跟进功能没关系，
第 876 行对`ptrace_request()` 函数的调用才是我们要关注的，
`ptrace_request()` 函数这文件 kernel/ptrace.c 中：

```c
 876 int ptrace_request(struct task_struct *child, long request,
 877                    unsigned long addr, unsigned long data)
 878 {
 879         bool seized = child->ptrace & PT_SEIZED;
 880         int ret = -EIO;
 881         siginfo_t siginfo, *si;
 882         void __user *datavp = (void __user *) data;
 883         unsigned long __user *datalp = datavp;
 884         unsigned long flags;
...
1045 #ifdef PTRACE_SINGLESTEP
1046         case PTRACE_SINGLESTEP:
1047 #endif
1048 #ifdef PTRACE_SINGLEBLOCK
1049         case PTRACE_SINGLEBLOCK:
1050 #endif
1051 #ifdef PTRACE_SYSEMU
1052         case PTRACE_SYSEMU:
1053         case PTRACE_SYSEMU_SINGLESTEP:
1054 #endif
1055         case PTRACE_SYSCALL:
1056         case PTRACE_CONT:
1057                 return ptrace_resume(child, request, data);
...
1088         default:
1089                 break;
1090         }
1091 
1092         return ret;
1093 }
```

我们可以看到, 单步跟进请求会触发 `ptrace_resume()` 函数的执行，
跳过去看同一文件下的 `ptrace_resume()` 函数：

```c
 773 static int ptrace_resume(struct task_struct *child, long request,
 774                          unsigned long data)
 775 {
 776         bool need_siglock;
 777 
 778         if (!valid_signal(data))
 779                 return -EIO;
 780 
 781         if (request == PTRACE_SYSCALL)
 782                 set_tsk_thread_flag(child, TIF_SYSCALL_TRACE);
 783         else
 784                 clear_tsk_thread_flag(child, TIF_SYSCALL_TRACE);
 785 
 786 #ifdef TIF_SYSCALL_EMU
 787         if (request == PTRACE_SYSEMU || request == PTRACE_SYSEMU_SINGLESTEP)
 788                 set_tsk_thread_flag(child, TIF_SYSCALL_EMU);
 789         else
 790                 clear_tsk_thread_flag(child, TIF_SYSCALL_EMU);
 791 #endif
 792 
 793         if (is_singleblock(request)) {
 794                 if (unlikely(!arch_has_block_step()))
 795                         return -EIO;
 796                 user_enable_block_step(child);
 797         } else if (is_singlestep(request) || is_sysemu_singlestep(request)) {
 798                 if (unlikely(!arch_has_single_step()))
 799                         return -EIO;
 800                 user_enable_single_step(child);
 801         } else {
 802                 user_disable_single_step(child);
 803         }
 ...
 826         return 0;
 827 }
```

`is_singlestep()` 看起来像是用来判断是否为单步执行的测试，
来看 kernel/ptrace.c 中的宏定义 `is_singlestep()`：

```C
 755 #ifdef PTRACE_SINGLESTEP
 756 #define is_singlestep(request)          ((request) == PTRACE_SINGLESTEP)
 757 #else
 758 #define is_singlestep(request)          0
 759 #endif
 ```

`is_singlestep()` 用来判断是不是 `PTRACE_SINGLESTEP` 请求，
如果是则进入 ./arch/x86/kernel/step.c 文件下的 `user_enable_single_step()` 函数中：

```c
211 void user_enable_single_step(struct task_struct *child)
212 {
213         enable_step(child, 0);
214 }
```

`enable_step()` 函数：

```c
193 /*
194  * Enable single or block step.
195  */
196 static void enable_step(struct task_struct *child, bool block)
197 {
198         /*
199          * Make sure block stepping (BTF) is not enabled unless it should be.
200          * Note that we don't try to worry about any is_setting_trap_flag()
201          * instructions after the first when using block stepping.
202          * So no one should try to use debugger block stepping in a program
203          * that uses user-mode single stepping itself.
204          */
205         if (enable_single_step(child) && block)
206                 set_task_blockstep(child, true);
207         else if (test_tsk_thread_flag(child, TIF_BLOCKSTEP))
208                 set_task_blockstep(child, false);
209 }
```

`enable_single_step()` 函数:

```c
109 static int enable_single_step(struct task_struct *child)
110 {
111         struct pt_regs *regs = task_pt_regs(child);
112         unsigned long oflags;
113 
114         /*
115          * If we stepped into a sysenter/syscall insn, it trapped in
116          * kernel mode; do_debug() cleared TF and set TIF_SINGLESTEP.
117          * If user-mode had set TF itself, then it's still clear from
118          * do_debug() and we need to set it again to restore the user
119          * state so we don't wrongly set TIF_FORCED_TF below.
120          * If enable_single_step() was used last and that is what
121          * set TIF_SINGLESTEP, then both TF and TIF_FORCED_TF are
122          * already set and our bookkeeping is fine.
123          */
124         if (unlikely(test_tsk_thread_flag(child, TIF_SINGLESTEP)))
125                 regs->flags |= X86_EFLAGS_TF;
126 
127         /*
128          * Always set TIF_SINGLESTEP - this guarantees that
129          * we single-step system calls etc..  This will also
130          * cause us to set TF when returning to user mode.
131          */
132         set_tsk_thread_flag(child, TIF_SINGLESTEP);
133 
134         oflags = regs->flags;
135 
136         /* Set TF on the kernel stack.. */
137         regs->flags |= X86_EFLAGS_TF;
138 
139         /*
140          * ..but if TF is changed by the instruction we will trace,
141          * don't mark it as being "us" that set it, so that we
142          * won't clear it by hand later.
143          *
144          * Note that if we don't actually execute the popf because
145          * of a signal arriving right now or suchlike, we will lose
146          * track of the fact that it really was "us" that set it.
147          */
148         if (is_setting_trap_flag(child, regs)) {
149                 clear_tsk_thread_flag(child, TIF_FORCED_TF);
150                 return 0;
151         }
152 
153         /*
154          * If TF was already set, check whether it was us who set it.
155          * If not, we should never attempt a block step.
156          */
157         if (oflags & X86_EFLAGS_TF)
158                 return test_tsk_thread_flag(child, TIF_FORCED_TF);
159 
160         set_tsk_thread_flag(child, TIF_FORCED_TF);
161 
162         return 1;
163 }
```

这个函数最重要的操作就是设置 `X86_EFLAGS_TF` 标志位，也就是 x86 的 Trap 标志位。
对于 x86 CPU，Trap 标志为将使 CPU 进入单步调试模式。
到这里，我们已经能明白 `ptrace()` 单步执行的实现原理了。
`ptrace()` 在单步执行结束后会回复原来的 Trap 标志位，
具体操作是在 ./arch/x86/kernel/step.c 中的 `handle_signal()` 函数中：

```C
701 static void
702 handle_signal(struct ksignal *ksig, struct pt_regs *regs)
703 {
704         bool stepping, failed;
705         struct fpu *fpu = &current->thread.fpu;
...
732         /*
733          * If TF is set due to a debugger (TIF_FORCED_TF), clear TF now
734          * so that register information in the sigcontext is correct and
735          * then notify the tracer before entering the signal handler.
736          */
737         stepping = test_thread_flag(TIF_SINGLESTEP);
738         if (stepping)
739                 user_disable_single_step(current);
740 
741         failed = (setup_rt_frame(ksig, regs) < 0);
742         if (!failed) {
743                 /*
744                  * Clear the direction flag as per the ABI for function entry.
745                  *
746                  * Clear RF when entering the signal handler, because
747                  * it might disable possible debug exception from the
748                  * signal handler.
749                  *
750                  * Clear TF for the case when it wasn't set by debugger to
751                  * avoid the recursive send_sigtrap() in SIGTRAP handler.
752                  */
753                 regs->flags &= ~(X86_EFLAGS_DF|X86_EFLAGS_RF|X86_EFLAGS_TF);
754                 /*
755                  * Ensure the signal handler starts with the new fpu state.
756                  */
757                 if (fpu->fpstate_active)
758                         fpu__clear(fpu);
759         }
760         signal_setup_done(failed, ksig, stepping);
761 }
```
这个函数调用 `user_disable_single_step()` 将 TF 标志为清零。


### 2. 判断程序是否执行了 ptrace 单步跟进

我们知道 x86 的 `int3` 指令可以用来实现软中断，
如何区分 `int3` 和 `ptrace()` 产生的程序中断？
一开始我以来可以通过 TF 标志位去判断，
单后面发现 `ptrace()` 的单步跟进执行完后 TF 标志为都已经被清零的，
这应该是系统调用结束前调用 `handle_signal()` 清除掉的。
通过 TF 标志为判断的方法行不通，后来在翻内核代码的过程中发现
./arch/x86/include/uapi/asm/debugreg.h 文件中有一个 `DR_STEP` 宏定义:

```C
 20 #define DR_TRAP0        (0x1)           /* db0 */
 21 #define DR_TRAP1        (0x2)           /* db1 */
 22 #define DR_TRAP2        (0x4)           /* db2 */
 23 #define DR_TRAP3        (0x8)           /* db3 */
 24 #define DR_TRAP_BITS    (DR_TRAP0|DR_TRAP1|DR_TRAP2|DR_TRAP3)
 25 
 26 #define DR_STEP         (0x4000)        /* single-step */
```

看着 `DR_STEP` 宏周围的代码和注释，很像是 x86 的调试寄存器相关的。
`DR_STEP` 的注释 "single-step" 似乎表明这个宏和 `ptrace()`单步执行有关。
之前倒是没注意到 x86 的调试寄存器和指令单步执行可能有关，
继续搜了 kernel 代码中对 `DR_STEP` 的引用：

```bash
C symbol: DR_STEP

  File            Function              Line
0 debugreg.h      <global>               26 #define DR_STEP (0x4000)
1 traps.h         get_si_code            99 if (condition & DR_STEP)
2 hw_breakpoint.c hw_breakpoint_handler 457 if (dr6 & DR_STEP)
3 kgdb.c          single_step_cont      510 (*(unsigned long *)ERR_PTR(args->err)) &= ~DR_STEP;
4 traps.c         do_debug              623 if ((dr6 & DR_STEP) && kmemcheck_trap(regs))
5 traps.c         do_debug              670 if ((dr6 & DR_STEP) && !user_mode(regs)) {
6 traps.c         do_debug              671 tsk->thread.debugreg6 &= ~DR_STEP;
7 traps.c         do_debug              676 if (tsk->thread.debugreg6 & (DR_STEP | DR_TRAP_BITS) || user_icebp)
8 kmmio.c         kmmio_die_notifier    583 if (val == DIE_DEBUG && (*dr6_p & DR_STEP))
9 kmmio.c         kmmio_die_notifier    589 *dr6_p &= ~DR_STEP;
```

从输出结果上看，
"4 traps.c         do_debug              623 if ((dr6 & DR_STEP) && kmemcheck_trap(regs))" 显示
`DR_STEP` 应该和 dr6 调试寄存器有关。
在 x86 CPU 中 dr6 是几个调试寄存器中作为状态寄存器使用的，
[wiki 上关于 dr6 寄存器的描述](https://en.wikipedia.org/wiki/X86_debug_register) 
只提到它的标志位和 dr0、dr1、dr2、dr3 寄存器所设置的硬件断点的触发有关。
但我猜测 dr6 还有更丰富的功能，于是把 [x86 系统编程手册](https://www.intel.com/content/www/us/en/architecture-and-technology/64-ia-32-architectures-software-developer-manual-325462.html)
翻出来，
在 **17.2.3 Debug Status Register (DR6)** 这一节中找到了 dr6 寄存器的完整说明，
其中有一段和单步执行有关的描述：

>**BS (single step) flag (bit 14)** — Indicates (when set) that the debug exception
was triggered by the single-step execution mode (enabled with the TF flag in the
EFLAGS register). The single-step mode is the highest-priority debug exception.
When the BS flag is set, any of the other debug status bits also may be set.

bit 14 也就是定义 `DR_STEP` 宏的 0x4000，
根据文档，那么当程序执行完指令单步跟进后这个标志位会被设置，
写了段[测试代码](https://github.com/kikimo/pygdb/blob/master/demos/hw_bp.c)
验证了下，果不其然，
所以，通过 dr6 寄存器就可以直接判断是否利用 `ptrace()` 执行了指令单步跟进。
 
