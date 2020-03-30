---
title: "ps 指令 hang 死原因分析（一）"
date: 2020-02-03T14:35:42+08:00
draft: false
---

一台服务器出了问题，
登陆上去执行`ps aux`命令没有响应，
`ctrl+c`也无法退出。
重新登陆机器执行`ps aux`又挂住没法操作了。
只能再登陆机器，
跑`strace ps aux`观察，又卡住了：

```txt
# strace ps aux
stat("/proc/180944", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
open("/proc/180944/stat", O_RDONLY)     = 6
read(6, "180944 (ps) D 1 180804 31034 0 -"..., 2048) = 330
close(6)                                = 0
open("/proc/180944/status", O_RDONLY)   = 6
read(6, "Name:\tps\nUmask:\t0022\nState:\tD (d"..., 2048) = 1205
close(6)                                = 0
open("/proc/180944/cmdline", O_RDONLY)  = 6
read(6, "ps\0-ef\0", 131072)            = 7
read(6, "", 131065)                     = 0
close(6)                                = 0
stat("/etc/localtime", {st_mode=S_IFREG|0644, st_size=388, ...}) = 0
stat("/etc/localtime", {st_mode=S_IFREG|0644, st_size=388, ...}) = 0
write(1, "root     180944  0.0  0.0 155328"..., 73root     180944  0.0  0.0 155328   424 ?        D    Jan29   0:00 ps -ef
) = 73
stat("/proc/181134", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
open("/proc/181134/stat", O_RDONLY)     = 6
read(6, "181134 (java) D 1 177157 31034 0"..., 2048) = 333
close(6)                                = 0
open("/proc/181134/status", O_RDONLY)   = 6
read(6, "Name:\tjava\nUmask:\t0022\nState:\tD "..., 2048) = 1207
close(6)                                = 0
open("/proc/181134/cmdline", O_RDONLY)  = 6
read(6,
```

可以看到`ps aux`似乎是卡在`/proc/181134/cmdline`文件的读取上。
执行`cat /proc/181134/cmdline`，果然又卡住。
再登陆机器，执行`top`指令，谢天谢地，`top`没 hang 死，
搜索`ps`指令，看到`ps`进程处于`D`（uninterruptible sleep）状态，
`cat /proc/181134/cmdline`命令也是：

```txt
...
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 38042 root      20   0  155328   1900   1484 D   0.0  0.0   0:00.09 ps
 38111 root      20   0  154624   5352   4036 S   0.0  0.0   0:00.22 sshd
 38116 tdops     20   0  154624   2876   1548 S   0.0  0.0   0:00.01 sshd
 38118 tdops     20   0  115436   1968   1600 S   0.0  0.0   0:00.02 bash
 38156 root      20   0  241112   4536   3376 S   0.0  0.0   0:00.01 sudo
 38159 root      20   0  193928   2412   1772 S   0.0  0.0   0:00.01 su
 38160 root      20   0  115436   2068   1656 S   0.0  0.0   0:00.05 bash
...
```
**不过这里有一点比较奇怪的是：`ps`和`cat`会进入`D`状态，但是`strace`只是进入`S`（sleep）状态，它还能 kill，why？**

检查`cat` 进程的堆栈，发现进程挂在内核中一个函数`call_rwsem_down_read_failed()`上：

```txt
# cat /proc/38042/stack
[<ffffffff81331ad8>] call_rwsem_down_read_failed+0x18/0x30
[<ffffffff812739d2>] proc_pid_cmdline_read+0xb2/0x560
[<ffffffff81200b9c>] vfs_read+0x9c/0x170
[<ffffffff81201a5f>] SyS_read+0x7f/0xe0
[<ffffffff816b4fc9>] system_call_fastpath+0x16/0x1b
[<ffffffffffffffff>] 0xffffffffffffffff
```

google 搜索 "读取`/proc/pid/cmdline` hang 死"的信息，
看到一个 [2018 年出的内核补丁](https://lkml.org/lkml/2018/2/20/576)，
主题是 **[PATCH] fs: proc: use down_read_killable in proc_pid_cmdline_read()**，
看起来高度相关，

```txt
From	Yang Shi <>
Subject	[PATCH] fs: proc: use down_read_killable in proc_pid_cmdline_read()
Date	Wed, 21 Feb 2018 03:49:29 +0800
share
When running vm-scalability with large memory (> 300GB), the below hung
task issue happens occasionally.

INFO: task ps:14018 blocked for more than 120 seconds.
       Tainted: G            E 4.9.79-009.ali3000.alios7.x86_64 #1
 "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
 ps              D    0 14018      1 0x00000004
  ffff885582f84000 ffff885e8682f000 ffff880972943000 ffff885ebf499bc0
  ffff8828ee120000 ffffc900349bfca8 ffffffff817154d0 0000000000000040
  00ffffff812f872a ffff885ebf499bc0 024000d000948300 ffff880972943000
 Call Trace:
  [<ffffffff817154d0>] ? __schedule+0x250/0x730
  [<ffffffff817159e6>] schedule+0x36/0x80
  [<ffffffff81718560>] rwsem_down_read_failed+0xf0/0x150
  [<ffffffff81390a28>] call_rwsem_down_read_failed+0x18/0x30
  [<ffffffff81717db0>] down_read+0x20/0x40
  [<ffffffff812b9439>] proc_pid_cmdline_read+0xd9/0x4e0
  [<ffffffff81253c95>] ? do_filp_open+0xa5/0x100
  [<ffffffff81241d87>] __vfs_read+0x37/0x150
  [<ffffffff812f824b>] ? security_file_permission+0x9b/0xc0
  [<ffffffff81242266>] vfs_read+0x96/0x130
  [<ffffffff812437b5>] SyS_read+0x55/0xc0
  [<ffffffff8171a6da>] entry_SYSCALL_64_fastpath+0x1a/0xc5

When manipulating a large mapping, the process may hold the mmap_sem for
long time, so reading /proc/<pid>/cmdline may be blocked in
uninterruptible state for long time.

down_read_trylock() sounds too aggressive, and we already have killable
version APIs for semaphore, here use down_read_killable() to improve the
responsiveness.

Signed-off-by: Yang Shi <yang.shi@linux.alibaba.com>
Cc: Alexey Dobriyan <adobriyan@gmail.com>
---
 fs/proc/base.c | 4 +++-
 1 file changed, 3 insertions(+), 1 deletion(-)

diff --git a/fs/proc/base.c b/fs/proc/base.c
index 9298324..44b6f8f 100644
--- a/fs/proc/base.c
+++ b/fs/proc/base.c
@@ -242,7 +242,9 @@ static ssize_t proc_pid_cmdline_read(struct file *file, char __user *buf,
 		goto out_mmput;
 	}

-	down_read(&mm->mmap_sem);
+	rv = down_read_killable(&mm->mmap_sem);
+	if (rv)
+		goto out_mmput;
 	arg_start = mm->arg_start;
 	arg_end = mm->arg_end;
 	env_start = mm->env_start;
--
1.8.3.1
```

看补丁，主要是把`down_read()`函数替换成`down_read_killable()`，
这个函数可以避免读取`/proc/pid/cmdline`文件时发生死锁，
`down_read(&mm->mmap_sem)`看起来应该是和信号量相关的函数操作，
不过为什么`down_read(&mm->mmap_sem)`会死锁呢?
`mm->mmap_sem`应该是个信号量，可能是这个信号量异常导致`ps`，`cat`指令都死锁了。
真正导致死锁的应该不是`ps`, `cat`指令，它们应该是受害者，
查看系统日志，发现有 Java 进程 hang 死的日志：

```txt
{timestamp} {hostname} kernel: INFO: task java:181134 blocked for more than 120 seconds.
{timestamp} {hostname} kernel: "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
{timestamp} {hostname} kernel: java            D ffff881269bd7c00     0 181134      1 0x00000084
{timestamp} {hostname} kernel: ffff881269bd7bd8 0000000000000082 ffff882037e04f10 ffff881269bd7fd8
{timestamp} {hostname} kernel: ffff881269bd7fd8 ffff881269bd7fd8 ffff882037e04f10 ffff882037e04f10
{timestamp} {hostname} kernel: ffff88203c9abef8 ffffffffffffffff ffff88203c9abf00 ffff881269bd7c00
{timestamp} {hostname} kernel: Call Trace:
{timestamp} {hostname} kernel: [<ffffffff816a94c9>] schedule+0x29/0x70
{timestamp} {hostname} kernel: [<ffffffff816aaafd>] rwsem_down_read_failed+0x10d/0x1a0
{timestamp} {hostname} kernel: [<ffffffff81331ad8>] call_rwsem_down_read_failed+0x18/0x30
{timestamp} {hostname} kernel: [<ffffffff816a8760>] down_read+0x20/0x40
{timestamp} {hostname} kernel: [<ffffffff8108d8b9>] do_exit+0x189/0xa40
{timestamp} {hostname} kernel: [<ffffffff81327a16>] ? plist_del+0x46/0x70
{timestamp} {hostname} kernel: [<ffffffff810f49fc>] ? __unqueue_futex+0x2c/0x60
{timestamp} {hostname} kernel: [<ffffffff810f5d8f>] ? futex_wait+0x11f/0x280
{timestamp} {hostname} kernel: [<ffffffff8108e1ef>] do_group_exit+0x3f/0xa0
{timestamp} {hostname} kernel: [<ffffffff8109e31e>] get_signal_to_deliver+0x1ce/0x5e0
{timestamp} {hostname} kernel: [<ffffffff8102a467>] do_signal+0x57/0x6c0
{timestamp} {hostname} kernel: [<ffffffff810c4149>] ? wake_up_new_task+0x119/0x190
{timestamp} {hostname} kernel: [<ffffffff81086ac7>] ? do_fork+0x107/0x320
{timestamp} {hostname} kernel: [<ffffffff8102ab2f>] do_notify_resume+0x5f/0xb0
{timestamp} {hostname} kernel: [<ffffffff816b527d>] int_signal+0x12/0x17
```

可以看到 Java 进程`java:181134` 也卡死在`rwsem_down_read_failed()`函数上，
估计当前版本的内核存在死锁的 bug，
google 发现[centos 官方的 bug 记录](https://bugs.centos.org/view.php?id=15252)，发现问题高度类似：
其中有个回复提及到修复补丁：

```txt
this issue seam is the resem wakeup loss, would be fix by patch below:
Wed Mar 20 2019 Jan Stancek <jstancek@redhat.com> [3.10.0-957.12.1.el7]
- [kernel] locking/rwsem: Fix (possible) missed wakeup (Waiman Long) [1690323 1547078]
- [kernel] futex: Fix (possible) missed wakeup (Waiman Long) [1690323 1547078]
- [kernel] futex: Use smp_store_release() in mark_wake_futex() (Waiman Long) [1690323 1547078]
- [kernel] sched/wake_q: Fix wakeup ordering for wake_q (Waiman Long) [1690323 1547078]
- [kernel] sched/wake_q: Document wake_q_add() (Waiman Long) [1690323 1547078]
```

还看到[内核邮件组对`rwsem`（这个应该就是我们前面提到的信号量）死锁问题的讨论](https://lore.kernel.org/patchwork/patch/1018850/)，
估计就是内核信号量的 bug 了（未完待续）。
