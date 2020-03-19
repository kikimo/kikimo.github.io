---
title:  "go Map 并发读写问题处理"
date: 2019-01-17T21:04:40+08:00
draft: false
---

几个月前碰到一起线上 go 应用崩溃问题，
查看日志，注意到其中有一条信息：

```txt
fatal error: concurrent map read and map write
```

回去查了下，代码里面确实有```Map```的并发读写，
但是涉及到```Map```相关的操作都用```sync.RWMutex```做了读写保护，
我把代码拉出来仔细看了一个晚上，愣是没看出啥问题。
这个问题后来交个另外一个同事去跟进，
同事构造了几个单元测试代码，很快就把问题重现出来了。
但是怎么修这个 bug，他说这个问题是由于三方库引起的，
不好修，所以问题后来就这么一直放着。
过了足足两个月，我感觉这问题不解决有点危险，万一哪天线上又复发就尴尬了，
就专门抽个下午的时间去研究这个问题。
代码拉下来，把同事当时写的单元测试跑起来，
很快程序就又爆出```concurrent map read and map write```问题。
正准备深入研究一番，然而一看同事写的测试代码顿时就无语了，
挖槽，测试代码里自身就有一个 map 的并发读写，跑起来不报这个问题才怪。
心里暗暗叫苦，TM 还是要从零开始分析。
那还是从场景复现开始做起，从之前的故障堆栈日志来看，
对于由哪个```Map```数据存在并发访问是比较明显的，
所以准备从代码梳理开始做起，查看哪些地方有涉及到这个```Map```数据的读、写操作，
然后针对这几处访问点构造并发访问场景。
一开始 review 代码的时候，发现这代码有点奇怪。
这些代码都是我几个月前写的，当时是把所有代码中的静态检查告警都搞定了才上线的，
然而今天却又看到了一堆的告警。
一开始是几个```fmt.Printf```的告警，顺手改掉了，
然后看到一个让我虎躯一震的静态检查告警：

```txt
XXX passes lock by value
```

仔细检查代码，发现在那个地方，有个结构体参数传值，
结构体中有一个```sync.RWMutex```字段，
其他地方也有涉及到这个结构体的参数传递，但传的是结构体指针，
那个地方当时眼睛瘸了，没注意到是值传递，
因为不是指针传值，所以和其他地方比它其实是在对不同的```sync.RWMutex```执行操作，
引起```Map```并发访问问题也就不奇怪了。

关于```XXX passes lock by value```问题，
[Detect locks passed by value in Go
](https://medium.com/golangspec/detect-locks-passed-by-value-in-go-efb4ac9a3f2b)
这篇文章有很生动的介绍，其中有段示例代码演示了代码中如何引入这个问题并导致程序死锁的：

```go
package main
import "sync"
type T struct {
    lock sync.Mutex
}
func (t *T) Lock() {
    t.lock.Lock()
}
func (t T) Unlock() {
   t.lock.Unlock()
}
func main() {
    t := T{lock: sync.Mutex{}}
    t.Lock()
    t.Unlock()
    t.Lock()
}
```

这段代码中```Unlock()```方法的```T```是通过值传递的，
不同于```Lock()```方法，```Unlock()```方法中的```T```
参数是```t```的一个拷贝，对应的```lock```字段也是一个拷贝，
这样```t.Lock()```和```t.Unlock()```操作的就是两个不同的锁，
导致最后那行```t.Lock()```代码直接死锁了。

在回到上面的一个问题：为什么原来好好的代码几个月过后突然多了一堆告警？
原来是最近这段之间重装了系统，顺便把 go 环境也重装到最新版了，
最新版的 go 则包含了一个强大的代码检测工具 [vet](https://medium.com/golangspec/detect-locks-passed-by-value-in-go-efb4ac9a3f2b),
那些新出现的告警都是 vet 检测出来的。
所以，这个这是一个升级系统和软件解决疑难问题的故事。
