---
title: "RPC Pipeline"
date: 2022-01-21T21:27:19+08:00
draft: false
---

在使用 RPC 通信时通常我们会等待一次请求结束后再发起下一个请求。
也就等待上一个请求受到回应后才发起下一个请求开始。
但是很多时候，其实并不需要等收到回应后才开始下一个请求。
比如通过 RPC 向服务端发送大量数据，在网络稳定的情况下，服务一般不会出什么错，
这时候如果不等服务端发来回应就继续发起下一个请求的话可以大幅提升数据传输性能。
RPC 回应可以放到异步回调里处理。

以 Go RPC 为例，我们来开如何实现一个 RPC Pipeline。
首先需要一个客户端，就以最简单的 echo server 当例子：

```go
package server

type HelloService struct {}

func (p *HelloService) Hello(request string, reply *string) error {
    *reply = "hello:" + request
    return nil 
}

func runServer() {
    rpc.RegisterName("HelloService", new(server.HelloService))
    listerner, err := net.Listen("tcp", ":8848")
    if err != nil {
        glog.Fatalf("error starting rpc server: %+v", err)
    }   

    for {
        conn, err := listerner.Accept()
        if err != nil {
            glog.Fatalf("error accepting connection: %+v", err)
        }

        go func(conn net.Conn) {
            glog.Infof("serving conn: %+v", conn)
            rpc.ServeConn(conn)
        }(conn)
    }   
}
```

这就是一个最简单的 Go RPC echo server 实现。
我们将在客户端实现 RPC Pipeline。
在实现 RPC Pipeline 之前，
我们先实现常见的 RPC 同步客户端，也就是需要等待服务端回应再发起下个请求：

```go
func runSyncClient(loop int) {
    client, err := rpc.Dial("tcp", "localhost:8848")
    if err != nil {
        glog.Fatalf("dialing: %+v", err)
    }

    for i := 0; i < loop; i++ {
        var reply string
        err = client.Call("HelloService.Hello", fmt.Sprintf("client %d", i), &reply)
        if err != nil {
            glog.Fatal(err)
        }

        glog.V(1).Infof("reply: %+v", reply)
    }
}
```

`client.Call()`一直阻塞指导服务端返回结国。
要实现 Pipeline 只需要把`client.Call()`替换成`cleint.Go()`，
然后在启一个 goroutine 去异步处理`client.Go()`返回的结果：


```go
func runPiplineClient(loop int) {
    client, err := rpc.Dial("tcp", "localhost:8848")
    if err != nil {
        glog.Fatalf("dialing: %+v", err)
    }

    var wg sync.WaitGroup
    wg.Add(loop)

    for i := 0; i < loop; i++ {
        var reply string
        helloCall := client.Go("HelloService.Hello", fmt.Sprintf("client %d", i), &reply, nil)
        go func(call *rpc.Call, replyp *string) {
            <-call.Done
            glog.V(1).Infof("reply: %+v", reply)
            wg.Done()
        }(helloCall, &reply)
    }
    wg.Wait()
}
```

对比两种实现的执行效率，执行 524288 次 RPC 调用的时间消耗分别如下：

```txt
// pipeline
real    0m22.410s
user    0m10.519s
sys     0m44.574s

// sync clients:
real    1m4.834s
user    0m12.472s
sys     0m47.499s
```

可以看到，pipeline 的性能提升非常明显，实际运行时间几乎比同步客户端快了三倍！
而且，cpu 的适用时间也更低，非常优秀有没有。

注意到我们 Pipeline 发送数据用的是同一个 goroutine，
如果启动多个 goroutine 并法调 RPC 呢？
再改一版客户端代码：

```go
// not as good as pipeline and for rpcs like apache Thrift concurrent
// write to client might not work beacause theire clients are not thread
// safe
func runConncurrentClient() {
    client, err := rpc.Dial("tcp", "localhost:8848")
    if err != nil {
        glog.Fatalf("dialing: %+v", err)
    }   

    var wg sync.WaitGroup
    wg.Add(concurrentClientOpts.clients)
    for i := 0; i < concurrentClientOpts.clients; i++ {
        go func(id int) {
            var reply string
            err = client.Call("HelloService.Hello", fmt.Sprintf("client %d", id), &reply)
            if err != nil {
                glog.Fatal(err)
            }

            glog.V(1).Infof("reply: %+v", reply)
            wg.Done()
        }(i)
    }   

    wg.Wait()
}
```

同样还是测试 524288 次 RPC 调用，我们发现对比单个 Goroutine 的 Pipeline 实现，
并发请求的效率反而各方面之指标都明显变差了：

```txt
// and for concurrent clients:
real    0m28.981s
user    0m14.819s
sys     1m17.591s
and sync client takes:
```

估计太多的的并发 goroutine 同时间争用通信信道反而恶化了调度延迟了。
除了性能变差意外，并发 RPC 调用还有一个严重的问题是——很多 RPC 的客户端实现不是线程安全的。
例如 Thrift RPC，如果多线程调用同一个 RPC client 并发请求的话很快它就会报错。

RPC Pipeline 一个经典的应用场景是 Raft 中的日志复制。
Raft Leader 负责日志复制，
可以在 Raft leder 中启动一个线程专门负责日志。
这个线程跑一个循环只要本地还有没发送到客户端的 log。
这个循环就会调用异步 AppendEntries RPC 把日志复制过去，
RPC 的回调丢给一个专门的线程池来处理。
