---
title: "SIGPIPE 引发的悲剧"
date: 2021-02-06T12:47:51+08:00
draft: false
---

快放年假了，昨天公司同事突然要求帮忙紧急看个线上问题。
原来线上告警系统的一个核心服务最近一个礼拜突然频繁挂掉，
系统挂掉时毫无征兆，也没有任何日志或异常堆栈，
每次挂都是所有服务节点一起挂，导致线上告警系统完全不可用,
而且只要手工 kill 掉其中给一个服务节点所有其他服务也跟着挂掉。


没有日志，服务节点的各项基础监控数据看起来也没什么异常，
这种情况下，马上想到的就是 strace attach 上去观察。
让同事 kill 一个其他节点上的服务，
strace 跟踪的进程也挂了，查看 strace 的输出：

```txt
[pid 22772] tgkill(22314, 22772, SIGPIPE <unfinished ...>
[pid 22325] <... write resumed> )       = 42
[pid 22326] <... write resumed> )       = 335
[pid 22772] <... tgkill resumed> )      = 0
[pid 22319] futex(0xc0000924c8, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
[pid 22326] write(183, "\2Y\366\327\35\0\1\274\371\25\0\0\0\0\0\1\0\0\0\1\6offset\10\0\0\0\0"..., 36 <unfinished ...>
[pid 22772] --- SIGPIPE {si_signo=SIGPIPE, si_code=SI_TKILL, si_pid=22314, si_uid=1000} ---
[pid 22325] write(179, "\32\235\313\213\33\0\1\272\371\25\0\0\0\0\0\1\0\0\0\1\4size\10\0\0\0\0\0\0"..., 34 <ptrace(SYSCALL):No such process>
[pid 22323] <... futex resumed> )       = ? <unavailable>
[pid 23419] +++ killed by SIGPIPE +++
[pid 22772] +++ killed by SIGPIPE +++
[pid 22771] +++ killed by SIGPIPE +++
[pid 22770] +++ killed by SIGPIPE +++
[pid 22769] +++ killed by SIGPIPE +++
[pid 22328] +++ killed by SIGPIPE +++
[pid 22326] +++ killed by SIGPIPE +++
[pid 22325] +++ killed by SIGPIPE +++
[pid 22324] +++ killed by SIGPIPE +++
[pid 22323] +++ killed by SIGPIPE +++
[pid 22322] +++ killed by SIGPIPE +++
[pid 22321] +++ killed by SIGPIPE +++
[pid 22319] +++ killed by SIGPIPE +++
[pid 22318] +++ killed by SIGPIPE +++
[pid 22317] +++ killed by SIGPIPE +++
[pid 22316] +++ killed by SIGPIPE +++
[pid 22329] +++ killed by SIGPIPE +++
[pid 22327] +++ killed by SIGPIPE +++
[pid 22320] +++ killed by SIGPIPE +++
+++ killed by SIGPIPE +++
```

看出啥端倪出来了？`+++ killed by SIGPIPE +++`！
`SIGPIPE`信号通常是往 broken pipe 或者关闭的 socket 写数据时触发的。
哪里的操作触发了服务的`SIGPIPE`呢。
我们把线程`22772`的 strace 结果过滤出来做进一步分析：

```txt
[pid 22772] <... close resumed> )       = 0
[pid 22772] write(2, "2021-02-05 20:06:14.345637 W | r"..., 126 <unfinished ...>
[pid 22772] <... write resumed> )       = -1 EPIPE (Broken pipe)
[pid 22772] --- SIGPIPE {si_signo=SIGPIPE, si_code=SI_USER, si_pid=22314, si_uid=1000} ---
[pid 22772] rt_sigreturn({mask=[]} <unfinished ...>
[pid 22772] <... rt_sigreturn resumed> ) = -1 EPIPE (Broken pipe)
[pid 22772] rt_sigprocmask(SIG_UNBLOCK, [PIPE],  <unfinished ...>
[pid 22772] <... rt_sigprocmask resumed> NULL, 8) = 0
[pid 22772] getpid()                    = 22314
[pid 22772] gettid( <unfinished ...>
[pid 22772] <... gettid resumed> )      = 22772
[pid 22772] tgkill(22314, 22772, SIGPIPE <unfinished ...>
[pid 22772] <... tgkill resumed> )      = 0
[pid 22772] --- SIGPIPE {si_signo=SIGPIPE, si_code=SI_TKILL, si_pid=22314, si_uid=1000} ---
[pid 22772] rt_sigreturn({mask=[]} <unfinished ...>
[pid 22772] <... rt_sigreturn resumed> ) = 0
[pid 22772] sched_yield( <unfinished ...>
[pid 22772] <... sched_yield resumed> ) = 0
[pid 22772] sched_yield( <unfinished ...>
[pid 22772] <... sched_yield resumed> ) = 0
[pid 22772] sched_yield( <unfinished ...>
[pid 22772] <... sched_yield resumed> ) = 0
[pid 22772] rt_sigaction(SIGPIPE, {SIG_DFL, ~[], SA_RESTORER|SA_STACK|SA_RESTART|SA_SIGINFO, 0x1606810},  <unfinished ...>
[pid 22772] <... rt_sigaction resumed> NULL, 8) = 0
[pid 22772] getpid( <unfinished ...>
[pid 22772] <... getpid resumed> )      = 22314
[pid 22772] gettid( <unfinished ...>
[pid 22772] <... gettid resumed> )      = 22772
[pid 22772] tgkill(22314, 22772, SIGPIPE <unfinished ...>
[pid 22772] <... tgkill resumed> )      = 0
[pid 22772] --- SIGPIPE {si_signo=SIGPIPE, si_code=SI_TKILL, si_pid=22314, si_uid=1000} ---
[pid 22772] +++ killed by SIGPIPE +++
```

可以看到最近的一个`write()`系统调用是:

```txt
[pid 22772] write(2, "2021-02-05 20:06:14.345637 W | r"..., 126 <unfinished ...>
```

往`stderr`写数据的时候报错了？
不能啊，服务用`nohup`起的，
按理`stderr`和`stdout`都被重定向到`nohup.out`上了。
跑到进程的`proc`目录检查一下，
发现`stderr`和`stdout`真的指向两条管道-_-！：

```txt
...
l-wx------. 1 admin admin 64 Feb  5 19:58 1 -> pipe:[203861356]
...
l-wx------. 1 admin admin 64 Feb  5 19:58 2 -> pipe:[203861357]
...
```

所以 nohup 这是什么情况？翻出[nohup 代码](https://github.com/openbsd/src/blob/master/usr.bin/nohup/nohup.c)来确认下，发现，nohup 还真有特例：

```C
int
main(int argc, char *argv[])
{
...
	if (isatty(STDOUT_FILENO) || errno == EBADF)
		dofile();

	if (pledge("stdio exec", NULL) == -1)
		err(1, "pledge");

	if ((isatty(STDERR_FILENO) || errno == EBADF) &&
	    dup2(STDOUT_FILENO, STDERR_FILENO) == -1) {
		/* may have just closed stderr */
		(void)fprintf(stdin, "nohup: %s\n", strerror(errno));
		exit(EXIT_MISC);
	}

	/*
	 * The nohup utility shall take the standard action for all signals
	 * except that SIGHUP shall be ignored.
	 */
	(void)signal(SIGHUP, SIG_IGN);

	execvp(argv[1], &argv[1]);
	exit_status = (errno == ENOENT) ? EXIT_NOTFOUND : EXIT_NOEXEC;
	err(exit_status, "%s", argv[1]);
}

static void
dofile(void)
{
	int fd;
	const char *p;
	char path[PATH_MAX];

	p = FILENAME;
	if ((fd = open(p, O_RDWR|O_CREAT|O_APPEND, S_IRUSR|S_IWUSR)) >= 0)
		goto dupit;
	if ((p = getenv("HOME")) != NULL && *p != '\0' &&
	    (strlen(p) + strlen(FILENAME) + 1) < sizeof(path)) {
		(void)strlcpy(path, p, sizeof(path));
		(void)strlcat(path, "/", sizeof(path));
		(void)strlcat(path, FILENAME, sizeof(path));
		if ((fd = open(p = path, O_RDWR|O_CREAT|O_APPEND, S_IRUSR|S_IWUSR)) >= 0)
			goto dupit;
	}
	errx(EXIT_MISC, "can't open a nohup.out file");

dupit:
	(void)lseek(fd, (off_t)0, SEEK_END);
	if (dup2(fd, STDOUT_FILENO) == -1)
		err(EXIT_MISC, NULL);
	if (fd > STDERR_FILENO)
		(void)close(fd);
	(void)fprintf(stderr, "sending output to %s\n", p);
}
```

`dofile()`把`stderr`和`stdout`重定向到`nohup.out`，
但是它被执行的前提是`STDOUT_FILENO`关联的是一个终端设备（暨`isatty(STDOUT_FILENO)`）。
所以如何解决当前的问题？在启动脚本里把`stderr`和`stdout`重定向到`/dev/null`或某个文件即可。
还有一个问题是：服务的`stdout`和`sterr`为什么关联到管道上了？
了解到当时服务是通过`salt`远程启动的，估计是`salt`做了管道重定向。
之前是手工在机器上启动，不存在这个问题。
最近一段时间服务都是通过`salt`远程脚本启动的，所以酿成悲剧。



