---
title:  "cron 僵尸进程问题分析"
date:   2018-04-14T21:57:46+08:00
draft: false
---

前段时间遇到一个比较诡异的问题, 一台服务器上突然出现了许多僵尸进程。
多数时候，这些僵尸进程出现在早上，大概到了下午又自动消失。
第二天又重复这样的情况。

查了下，这些僵尸进程的父进程都是一些挂在 cron 下的 bash 进程。

![僵尸进程](/images/cron_zombie_01_pub.png)

猜测是 cron 每天早上定时执行了某些任务留下了这些僵尸进程．
为什么会出现僵尸进程，以及，为什么这些僵尸进程到了下午又自己消失了？
一开始希望直接从 cron 配置定位产生僵尸进程的任务，
检查发现这台机器上配置了相当数量的 cron 任务，逐一排查过去不现实，只能寻找其他思路．

僵尸进程意味着该进程已经执行结束，
但是父进程还没有调用 
```wait()``` 获取它的返回值，
从而导致已结束的进程进程处于僵尸(zombie)进程状态。
正常而言，一个进程结束后通常会在短暂的时间内处于僵尸进程状态．
长时间处于僵尸进程状态，
意味着父进程一直都没有调用 ```wait()```，
所以我们首先检查下这些僵尸进程的父进程也就是 cron 进程在干什么。

![父进程状态](/images/cron_zombie_02_pub.png)

可以看到 2584 这个僵尸进程的父进程 2582 处于睡眠状态。
strace 显示 2582 进程在等待在 ```read()``` 系统调用。
```read()```系统调用读取的文件描述符是 6，
利用 [lsof](http://man7.org/linux/man-pages/man8/lsof.8.html) 指令来查看这个文件描述符的详细信息。

![描述符 6 的详细信息](/images/cron_zombie_03_pub.png)

可以看到这是个管道文件， 对应 inode 是 13865。
进程 2582 处于管道读的一端，
根据这个 inode 信息，
我们可以利用 [lsof](http://man7.org/linux/man-pages/man8/lsof.8.html) 进一步找出管道写的一端关联的进程。

![管道写端的进程](/images/cron_zombie_04_pub.png)

根据 lsof 的输出，大致可以断定 2586 和 18826 这两个进程把标准输出和错误输入都重定向到管道 6 写的一端了。
经过检查发现 2586 进程执行的 sleep.sh 脚本就是 cron 下的定时任务之一。
但是 ps 指令却显示，2586 任务进程的父进程是 init 进程。
![任务进程的父进程](/images/cron_zombie_05_pub.png)

根据目前的分析，大概可以猜测会出现 bash 僵尸进程的直接原因－－父进程卡在管道读的一端上而没法继续调用 [wait()函数]．
但是我们还需要搞清楚：

1. bash 僵尸进程的父进程为什么会卡在管道读的一端上.
2. 为什么任务进程的父进程会变成 1。

要继续分析下去只能去看 cron 的代码了，先确定当前系统上的 cron 版本。

![cron 版本](/images/cron_version_pub.png) 

找到[这个版本 cron 的代码](http://rpm.pbone.net/index.php3/stat/26/dist/79/size/230920/name/cronie-1.4.4-12.el6.src.rpm)，以 pipe 为关键字搜索代码：

![cron pipe](/images/cron_pipe_pub.png)

可以看到 do_command.c 这个源码文件中有大量管道相关的操作，所以重点来分析这个文件。
do_command.c 文件总共 571 行代码，一共就三个函数。
其中重要的两个函数是

```c
void do_command(entry * e, user * u);
static void child_process(entry * e, user * u);
```

分析这两个函数代码的，可以看到
```do_command()``` 函数先从主 cron 进程 ```fork()``` 出一个 cron 子进程，
我们把这个 cron 子进程称为 cron worker 进程，
cron worker 接下来会执行 ```child_process()``` 函数。
在 ```child_process()``` 函数中
cron worker 通过显式的 ```wait()``` 调用来回收它子进程的返回值，
这个可以从 93 - 98 和 535 - 553 行处的代码看出来：

```c
 93         /* our parent is watching for our death by catching SIGCHLD.  we
 94          * do not care to watch for our children's deaths this way -- we
 95          * use wait() explicitly.  so we have to reset the signal (which
 96          * was inherited from the parent).
 97          */
 98         (void) signal(SIGCHLD, SIG_DFL);
...
535         for (; children > 0; children--) {
536             WAIT_T waiter;
537             PID_T pid;
538 
539             Debug(DPROC, ("[%ld] waiting for grandchild #%d to finish\n",
540                     (long) getpid(), children))
541                 while ((pid = wait(&waiter)) < OK && errno == EINTR) ;
542             if (pid < OK) {
543                 Debug(DPROC,
544                     ("[%ld] no more grandchildren--mail written?\n",
545                         (long) getpid()))
546                     break;
547             }
548             Debug(DPROC, ("[%ld] grandchild #%ld finished, status=%04x",
549                     (long) getpid(), (long) pid, WEXITSTATUS(waiter)))
550                 if (WIFSIGNALED(waiter) && WCOREDUMP(waiter))
551                 Debug(DPROC, (", dumped core")) 
552                     Debug(DPROC, ("\n"))
553         }  
```

cron woker 会进一步 ```fork()``` 出一个任务进程，这个任务进程通过 ```exec()``` 调用来执行任务程序。
cron worker 在 ```fork()``` 任务进程之前，会建立两条管道：

1. 管道一用于读取任务进程的输出结果，任务进程则会将标准输出和错误输出重定向到这条管道写的一端。
2. 管道二用于向任务进程提供输入，任务进程将标准输入重定向到这个管道读的一端。

cron worker 进程会继续 ```fork()``` 出一个 stdin_writer 进程，这个进程往管道一写入数据．
几个父子进程间的关系可以用下图来描述：

![cron woker 和任务进程间的通信](/cron_task_talk_pub.jpg)

到这里，我们可以完全确定为什么会出现 bash 僵尸进程了：
调用 ```wait()``` 函数之前
cron worker 卡在和任务进程通信的管道一上了，
bash 僵尸进程会一直存在直到子进程执行结束将关闭管道，
这之后 cron worker 才从管道一端的等待中返回然后调用 ```wait()``` 函数，
然后 bash 僵尸进程就消失了。

但是还有一个问题没解决，任务进程的父进程为什么会变成 init 进程？
通过检查 cron 中的定时任务指令，发现有如下形式的 cron 任务:

```bash
/root/sleep.sh &
```

怀疑跟 & 符号有关，& 符号会使相关指令进入后台执行，查看 bash 的文档，看到以下关于 & 符号的详细描述：

>If a command is terminated by the control operator &, the shell executes the command in  the  background  in  a subshell. The  shell does not wait for the command to finish, and the return status is 0. 

bash 遇到 & 符号会启动一个后台运行的 subshell，
更重要的一点是，
bash 不会等待这个 subshell 执行结束。
这就导致一方面 shell 进程先退出执行，但是因为 cron worker 进程没有调用 ```wait()``` 而变成僵尸进程，
另一方面 subshell 中执行的任务进程因为父进程变成僵尸进程，所以 init 进程就收养了它，它的父进程 id 变成 1。

通过一个简单的 demo 我们来验证一下:

```c
#include <stdio.h>
#include <unistd.h>

int main()
{
    int pid;

    if ((pid = fork()) == 0) {  // child process
        execle("/bin/bash", "/bin/bash", "-c", "/root/sleep.sh&", (char *) 0, NULL);
    }

    while (1) {
       sleep(1);
    }
}
```

编译执行以上代码：

![bash zombie test](/images/bash_zombie_demo_pub.png)

可以看看到，exec.c 的代码成功的复现出 bash 僵尸进程产生的场景了。

现在，cron 任务产生僵尸进程的原因算是分析得比较清楚了，那要如何避免这个问题呢？
解决方法就是 cron 任务指令末尾不要加 & 符号，这样做完全没必要，而且是产生僵尸进程的直接原因。
高版本的 cron 不再使用管道和任务进程通信，& 符号不会导致僵尸进程的产生，
但是 & 符号依然会使任务进程的父进程变成 init 进程，
要检查当前的 cron 运行的具体任务就变得麻烦了，
所以，最好也不要加 & 符号。
