---
title: "获取系统 CPU 数量"
date: 2020-10-12T09:12:05+08:00
draft: false
---

经常碰到需要了解系统 CPU 信息的场景，
例如你登录一台服务器想知道这台服务器大概是什么配置，
或者在应用的场景中我们根据 CPU 数量来配置服务的线程数。
我们可以通过`cat /proc/cpuinfo`指令来获取机器的 CPU 信息。
对于 Java 应用则可以调用`runtime.availableProcessors()`，
Go 则会自动调用`runtime.GOMAXPROCS()`获取可以 CPU 信息，且根据这一信息来设置 GC、MPG 中的 Processor 等配置。
通常情况下这些获取 CPU 等方法不会有问题，但在容器场景，这些办法就不灵了。

首先容器中是没有自己的 proc 文件系统的，所以也就没办法执行`cat /proc/cpuinfo`。
一个解决的办法是通过 lxcfs 挂在一个伪造的`/proc/cpuinfo`，
我们甚至可以让这个伪造的`/proc/cpuinfo`和容器的 CPU 配置一致，
比如容器配置是四核，lxcfs 挂在的`/proc/cpuinfo`也显示四核 CPU。
所以 lxcfs 就能解决所有问题了吗？
显然事情没这么简单，通过简单的测试，可以发现即便配置了伪造的`/proc/cpuinfo`，
Java、Go 依然获取了错误的 CPU 信息。

```Java
public class Docker {

  public static void main(String[] args) {

    Runtime runtime = Runtime.getRuntime();

    int processors = runtime.availableProcessors();
    // long maxMemory = runtime.maxMemory();

    System.out.format("Number of processors: %d\n", processors);
    // System.out.format("Max memory: %d bytes\n", maxMemory);
  }
}
```

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    fmt.Printf("cpus: %d\n", runtime.GOMAXPROCS(0))
}
```

通过 strace，可以发现 Java 和 Go 分别使用了不同的方法来获取 CPU 信息。
先来看看 Java 是如何获取 CPU 信息的：

```txt
# java Docker
Number of processors: 40

# strace -ff java Docker
...
[pid   418] open("/sys/devices/system/cpu/online", O_RDONLY|O_CLOEXEC) = 3
[pid   418] read(3, "0-39\n", 8192)     = 5
...
[pid   418] close(5)                    = 0
[pid   418] sched_getaffinity(418, 32, [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39]) = 8
...
```

strace 结果显示，Java 读取了`/sys/devices/system/cpu/online`文件，
这个文件内容是`0-39\n`也就是机器的 CPU 信息。
另外 Java 还调用了`sched_getaffinity()`这个系统调用，这个调用同样可以获取系统的 CPU 信息。
另外 strace 执行的时候一定要加`-ff`参数，因为这些调用不是在 Java 的祝进程中产生的。

再看 Go 如何获取 CPU 信息：

```txt
# ./ncpu
cpus: 40

# strace -ff ./ncpu 
execve("./ncpu", ["./ncpu"], 0x7ffe90dfaec8 /* 133 vars */) = 0
arch_prctl(ARCH_SET_FS, 0x5579f0)       = 0
sched_getaffinity(0, 8192, [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39]) = 8
...
```

可以看到 Go 也是使用了`sched_getaffinity()`系统调用来获取 CPU 信息。
Go 和 Java 都绕开了`/proc/cpuinfo`来获取信息。
这两种语言一旦获取了错误的 CPU 信息可能会造成一定的负面效果，比如 GC 异常、服务线程数量异常导致 CPU 负载异常等问题。

