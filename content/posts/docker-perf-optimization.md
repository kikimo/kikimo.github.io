---
title: "Docker 性能优化"
date: 2020-10-13T13:31:07+08:00
draft: false
---

Docker 在生产环境跑了一段时间后我们发现 dockerd 进程占用了不少 CPU 资源：

```txt
# pidstat  -u -p 4265 1
Linux *.x86_64 (*) 	*/*/2020 	_x86_64_	(* CPU)

07:05:44 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
07:05:45 PM     0      4265  100.00   16.00    0.00  100.00    38  dockerd
07:05:46 PM     0      4265   87.00   14.00    0.00  100.00    38  dockerd
07:05:47 PM     0      4265   17.00    4.00    0.00   21.00    38  dockerd
07:05:48 PM     0      4265   36.00    7.00    0.00   43.00    38  dockerd
07:05:49 PM     0      4265  100.00   17.00    0.00  100.00    38  dockerd
07:05:50 PM     0      4265  100.00   23.00    0.00  100.00    38  dockerd
07:05:51 PM     0      4265  100.00   22.00    0.00  100.00    38  dockerd
07:05:52 PM     0      4265   25.00    6.00    0.00   31.00    38  dockerd
07:05:53 PM     0      4265   21.00    5.00    0.00   26.00    38  dockerd
...
```

通过 perf 我们观察到 dockerd 的 CPU 大部分消耗在 sha256 哈希值计算、json 解码以及 gc 操作上：

```txt
$ perf top -p 4265
...
  14.22%  dockerd-ce  [.] crypto/sha256.block
   7.24%  dockerd-ce  [.] encoding/json.stateInString
   6.15%  dockerd-ce  [.] runtime.scanobject
   5.24%  dockerd-ce  [.] encoding/json.(*decodeState).scanWhile
   4.16%  dockerd-ce  [.] encoding/json.checkValid
   3.60%  dockerd-ce  [.] runtime.mallocgc
   3.59%  dockerd-ce  [.] runtime.heapBitsForObject
   3.21%  dockerd-ce  [.] encoding/json.unquoteBytes
   2.83%  dockerd-ce  [.] runtime.greyobject
   1.50%  dockerd-ce  [.] encoding/json.(*decodeState).object
   1.17%  dockerd-ce  [.] runtime.heapBitsSetType
   1.13%  dockerd-ce  [.] runtime.memclrNoHeapPointers
   1.10%  dockerd-ce  [.] runtime.memmove
   0.97%  dockerd-ce  [.] time.parse
...
```

利用 pprof 我们对 dockerd 做进一步细致的观察，我们看到 dockerd 的 CPU 都花镜像层`chain id`的计算上，
而计算`chain id`主要是为了给`getImagesJSON()`接口返回镜像基本信息，
镜像今本信息还有一部分是以`json`形式存储在磁盘上的，dockerd 在读取这些 json 信息的同时还对它作了`sha256`哈希校验，
所以这部分操作在作`json`解码操作的同时也耗费了一部分 CPU 在`sha256`哈希值的计算上。

![docker flame](/images/docker-flame.png)

`getImagesJSON()`是谁调用的呢？通过`ss`查看读写`docker.sock`的进程，我们看到主要的嫌疑在`kubelet`上。
查看`kubelet`我们发现至少有两处地方在定时轮询 docker 的`getImagesJSON()`接口。
一处在`kubelet`定时更新节点状态的代码中，频率默认为 10s，由`--node-status-update-frequency`参数控制

```go
kubelet/kubelet.go:Run(): `go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)`
```

另一处在`kubelet`的镜像 GC 代码中，频率是写死的每 30s 一次：

```go
pkg/kubelet/images/image_gc_manager.go:realImageGCManager.Start()

go wait.Until(func() {
	images, err := im.runtime.ListImages()
	if err != nil {
		klog.Warningf("[imageGCManager] Failed to update image list: %v", err)
	} else {
		im.imageCache.set(images)
	}
}, 30*time.Second, wait.NeverStop)
```

如何优化？首先 docker 镜像的基础信息在 dockerd 首次加载镜像时已经包含了`getImagesJSON()`接口需要返回的大部分信息，
我们只需要把这部分信息缓存下来就行，后续 dockerd 对镜像进行增删改的时候我们相应的更新缓存中的信息就行了。
除此之外还有一个 CPU 消耗非常的的就是 docker 镜像层`chain id`的计算，真对这一问题我们同样可以用缓存来解决，
dockerd 镜像在加载的时候会从镜像层存储组件中查询镜像层的数据数据结构，这个数据结构中就包含这镜像层的`chain id`信息，
同样我们只要把这个结果缓存缓存下来就可以避免大量重复的哈希计算了。调整过后在同样场景下，dockerd 的 CPU 基本下降到 0%-2% 这个范围波动了。
具体修改可以参考 docker 的这个[Add image cache for image store #41361](https://github.com/moby/moby/pull/41361)。

