---
title: "k8s GPU 加载失败问题排查"
date: 2021-02-04T20:29:25+08:00
draft: false
---

最近接连收到了几起线上 GPU 容器实例性能低下的问题反馈。经排查发现出问题的实例并没有成功挂载 GPU 设备。进一步的调查发现**这是一个和时间赛跑的随机概率问题，且几乎当前所有线上的 GPU 机器都受该问题影响**。下面我们具体描述这个问题。

首先应用都是可以成功发布的，但是发布上去的应用会有一定的概率无法挂载 GPU 设备，
一旦出现这种情况就会导致实例的性能低下（因为实例用 CPU 而不是 GPU 做计算）。
应用只有在服务启动的前几秒钟才能成功挂载 GPU 设备，
如果服务启动较慢，
就会出现无法挂载的情况。
错过这几秒的时间窗口过去，
实例将无法再挂载 GPU。
这个现象（超过特定时间窗口，GPU 设备无法加载）在我们线上的 k8s 实例上几乎 100% 重现。

根据我们的测试，
**一种可能的问题解决方法是重启 nvidia 的 k8s device plugin 插件**，
我们在新添加的机器上测试发现启动可以解决问题，
针对现有的集群我们还需要做进一步验证。
下面我们具体介绍问题的分析定位。

首先我们通过以下代码来获取实例上的 GPU 信息：

```C
#include <cuda_runtime.h>
#include <stdio.h>

int main(int argc, char **argv) {
        printf("%s Starting...\n", argv[0]);
        int deviceCount = 0;
        cudaError_t error_id = cudaGetDeviceCount(&deviceCount);
        if (error_id != cudaSuccess) {
                printf("cudaGetDeviceCount returned %d\n-> %s\n",
                                (int)error_id, cudaGetErrorString(error_id));
                printf("Result = FAIL\n");
                exit(EXIT_FAILURE);
        }
        if (deviceCount == 0) {
                printf("There are no available device(s) that support CUDA\n");
        } else {
                printf("Detected %d CUDA Capable device(s)\n", deviceCount);
        }
        int dev, driverVersion = 0, runtimeVersion = 0;
        dev =0;
        cudaSetDevice(dev);
        cudaDeviceProp deviceProp;
        cudaGetDeviceProperties(&deviceProp, dev);
        printf("Device %d: \"%s\"\n", dev, deviceProp.name);
        cudaDriverGetVersion(&driverVersion);
        cudaRuntimeGetVersion(&runtimeVersion);
        printf(" CUDA Driver Version / Runtime Version %d.%d / %d.%d\n",
                        driverVersion/1000, (driverVersion%100)/10,
                        runtimeVersion/1000, (runtimeVersion%100)/10);
        printf(" CUDA Capability Major/Minor version number: %d.%d\n",
                        deviceProp.major, deviceProp.minor);
        printf(" Total amount of global memory: %.2f MBytes (%llu bytes)\n",
                        (float)deviceProp.totalGlobalMem/(pow(1024.0,3)),
                        (unsigned long long) deviceProp.totalGlobalMem);
        printf(" GPU Clock rate: %.0f MHz (%0.2f GHz)\n",
                        deviceProp.clockRate * 1e-3f, deviceProp.clockRate * 1e-6f);
        printf(" Memory Clock rate: %.0f Mhz\n",
                        deviceProp.memoryClockRate * 1e-3f);
        printf(" Memory Bus Width: %d-bit\n",
                        deviceProp.memoryBusWidth);
        if (deviceProp.l2CacheSize) {
                printf(" L2 Cache Size: %d bytes\n",
                                deviceProp.l2CacheSize);
        }
        printf(" Max Texture Dimension Size (x,y,z) "
                        " 1D=(%d), 2D=(%d,%d), 3D=(%d,%d,%d)\n",
                        deviceProp.maxTexture1D , deviceProp.maxTexture2D[0],
                        deviceProp.maxTexture2D[1],
                        deviceProp.maxTexture3D[0], deviceProp.maxTexture3D[1],
                        deviceProp.maxTexture3D[2]);
        printf(" Max Layered Texture Size (dim) x layers 1D=(%d) x %d, 2D=(%d,%d) x %d\n",
                        deviceProp.maxTexture1DLayered[0], deviceProp.maxTexture1DLayered[1],
                        deviceProp.maxTexture2DLayered[0], deviceProp.maxTexture2DLayered[1],
                        deviceProp.maxTexture2DLayered[2]);
        printf(" Total amount of constant memory: %lu bytes\n",
                        deviceProp.totalConstMem);
        printf(" Total amount of shared memory per block: %lu bytes\n",
                        deviceProp.sharedMemPerBlock);
        printf(" Total number of registers available per block: %d\n",
                        deviceProp.regsPerBlock);
        printf(" Warp size: %d\n", deviceProp.warpSize);
        printf(" Maximum number of threads per multiprocessor: %d\n",
                        deviceProp.maxThreadsPerMultiProcessor);
        printf(" Maximum number of threads per block: %d\n",
                        deviceProp.maxThreadsPerBlock);
        printf(" Maximum sizes of each dimension of a block: %d x %d x %d\n",
                        deviceProp.maxThreadsDim[0],
                        deviceProp.maxThreadsDim[1],
                        deviceProp.maxThreadsDim[2]);
        printf(" Maximum sizes of each dimension of a grid: %d x %d x %d\n",
                        deviceProp.maxGridSize[0],
                        deviceProp.maxGridSize[1],
                        deviceProp.maxGridSize[2]);
        printf(" Maximum memory pitch: %lu bytes\n", deviceProp.
                        memPitch);
        exit(EXIT_SUCCESS);
}
```

我们在 GPU 挂载失败的实例上运行以上代码，报 unkonw error（顺便提一嘴，nvidia 的报错信息实在太特么坑爹了）：

```shell
# nvcc dev.cu -o dev
# ./dev
./dev Starting...
cudaGetDeviceCount returned 30
-> unknown error
Result = FAIL
```

开 strace 观测系统调用情况：

```shell
# strace ./dev
...
openat(AT_FDCWD, "/proc/devices", O_RDONLY) = 4
fstat(4, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
read(4, "Character devices:\n  1 mem\n  4 /"..., 1024) = 609
close(4)                                = 0
stat("/dev/nvidia-uvm", {st_mode=S_IFCHR|0666, st_rdev=makedev(237, 0), ...}) = 0
stat("/dev/nvidia-uvm-tools", {st_mode=S_IFCHR|0666, st_rdev=makedev(237, 1), ...}) = 0
openat(AT_FDCWD, "/dev/nvidia-uvm", O_RDWR|O_CLOEXEC) = -1 EPERM (Operation not permitted)
openat(AT_FDCWD, "/dev/nvidia-uvm", O_RDWR) = -1 EPERM (Operation not permitted)
ioctl(-1, _IOC(0, 0, 0x2, 0x3000), 0)   = -1 EBADF (Bad file descriptor)
ioctl(3, _IOC(_IOC_READ|_IOC_WRITE, 0x46, 0x29, 0x10), 0x7ffdf9339bd0) = 0
close(3)                                = 0
munmap(0x7f039e10b000, 18310792)        = 0
munmap(0x7f039debd000, 2415200)         = 0
futex(0x55615a8702b0, FUTEX_WAKE_PRIVATE, 2147483647) = 0
write(1, "cudaGetDeviceCount returned 30\n", 31cudaGetDeviceCount returned 30
) = 31
write(1, "-> unknown error\n", 17-> unknown error
)      = 17
write(1, "Result = FAIL\n", 14Result = FAIL
)         = 14
exit_group(1)                           = ?
+++ exited with 1 +++
```

注意到几个失败系统调用：

```txt
openat(AT_FDCWD, "/dev/nvidia-uvm", O_RDWR|O_CLOEXEC) = -1 EPERM (Operation not permitted)
openat(AT_FDCWD, "/dev/nvidia-uvm", O_RDWR) = -1 EPERM (Operation not permitted)
ioctl(-1, _IOC(0, 0, 0x2, 0x3000), 0) = -1 EBADF (Bad file descriptor)
```

`/dev/nvidia-uvm`设备打开失败，这应该是问题的根本原因，nvidia 坑爹的地方就在于它甚至都没判断返回值就继续撸 ioctl 调用，上层的错误信息也让人不明所以。我们接下来观察 docker 实例的 spec：

```shell
$ docker inspect 90d6d4dbe64f
...
            "Devices": [
                {
                    "PathOnHost": "/dev/nvidiactl",
                    "PathInContainer": "/dev/nvidiactl",
                    "CgroupPermissions": "rw"
                },
                {
                    "PathOnHost": "/dev/nvidia1",
                    "PathInContainer": "/dev/nvidia1",
                    "CgroupPermissions": "rw"
                }
            ],
...
```

注意到 GPU 实例的 docker spec 里面指定了设备列表，
但是并没有`nvidia-uvm`设备， 为啥啊？
检查线上的其他机器，
发现抽查的几个实例都是这个情况，
包括服务已经成功挂载到 GPU 上的实例。
问题一下子就变得有意思起来了。
查看实例对应 cgroup 的 device 子系统：

```shell
$ ls -l /dev/nvidia-uvm
crw-rw-rw-. 1 root root 237, 0 Jan 14 18:53 /dev/nvidia-uvm

$ cat devices.list
c 1:5 rwm
c 1:3 rwm
c 1:9 rwm
c 1:8 rwm
c 5:0 rwm
c 5:1 rwm
c 195:255 rw
c 195:1 rw
c *:* m
b *:* m
c 1:7 rwm
c 136:* rwm
c 5:2 rwm
c 10:200 rwm
```

我们发现`nvidia-uvm`不在 GPU 实例可访问的设备列表上。
手工加上去呢？
我们发现一个很有意思的情况：确实可以通过 device.allow 手工把设备权限加上去，
但是没过一会儿，设备又从列表上消失了：

```shell
# echo 'c 237:0 rw' > devices.allow  && cat devices.list  && echo && sleep 5  &&  cat devices.list
c 1:5 rwm
c 1:3 rwm
c 1:9 rwm
c 1:8 rwm
c 5:0 rwm
c 5:1 rwm
c 195:255 rw
c 195:1 rw
c *:* m
b *:* m
c 1:7 rwm
c 136:* rwm
c 5:2 rwm
c 10:200 rwm
c 237:0 rw

c 1:5 rwm
c 1:3 rwm
c 1:9 rwm
c 1:8 rwm
c 5:0 rwm
c 5:1 rwm
c 195:255 rw
c 195:1 rw
c *:* m
b *:* m
c 1:7 rwm
c 136:* rwm
c 5:2 rwm
c 10:200 rwm
```

在`/dev/nvidia-uvm`设备的短暂可用期间，
dev.cu 代码是可以正常执行的。
nvidia-uvm 设备添加后被不明实体清除了，谁干的？
嫌疑最大的可能就是 containerd-shim，
通过 strace 跟踪 containerd-shim，我们发现：

```shell
# strace -ff -p 154308 2>&1 | tee s.txt
...
[pid 30755] openat(AT_FDCWD, "/sys/fs/cgroup/devices/kubepods/burstable/pod0dbd65af-6693-11eb-994b-246e9634b5c8/90d6d4dbe64f821d43e4ef954808089d3f2f63af3fc9824d2c1e719768cd7d18/devices.deny", O_WRONLY|O_CREAT|O_TRUNC|O_CLOEXEC, 0700) = 5
[pid 30755] epoll_ctl(4, EPOLL_CTL_ADD, 5, {EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, {u32=669191840, u64=139870574153376}}) = 0
[pid 30755] fcntl(5, F_GETFL)           = 0x8001 (flags O_WRONLY|O_LARGEFILE)
[pid 30755] fcntl(5, F_SETFL, O_WRONLY|O_NONBLOCK|O_LARGEFILE) = 0
[pid 30755] write(5, "a *:* rwm", 9)    = 9
[pid 30755] epoll_ctl(4, EPOLL_CTL_DEL, 5, 0xc42013e84c) = 0
[pid 30755] close(5)                    = 0
[pid 30755] openat(AT_FDCWD, "/sys/fs/cgroup/devices/kubepods/burstable/pod0dbd65af-6693-11eb-994b-246e9634b5c8/90d6d4dbe64f821d43e4ef954808089d3f2f63af3fc9824d2c1e719768cd7d18/devices.allow", O_WRONLY|O_CREAT|O_TRUNC|O_CLOEXEC, 0700) = 5
[pid 30755] epoll_ctl(4, EPOLL_CTL_ADD, 5, {EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, {u32=669191840, u64=139870574153376}}) = 0
[pid 30755] fcntl(5, F_GETFL)           = 0x8001 (flags O_WRONLY|O_LARGEFILE)
[pid 30755] fcntl(5, F_SETFL, O_WRONLY|O_NONBLOCK|O_LARGEFILE) = 0
[pid 30755] write(5, "c 1:5 rwm", 9)    = 9
...
```

从 strace 结果我们可以看出来，
它先往 devices.deny 写了一条`"a *:* rwm"`从而禁用了所有设备，
然后往 devices.allow 添加设备。
所以估计 containerd-shim 是根据 spec 中 devices 设置设备的白名单，
这一点可以从 kubelet 的代码中得到验证，
`kubernetes/vendor/github.com/opencontainers/runc/libcontainer/cgroups/fs/devices.go`：

```go
...
func (s *DevicesGroup) Set(path string, cgroup *configs.Cgroup) error {
	if system.RunningInUserNS() {
		return nil
	}

	devices := cgroup.Resources.Devices
	if len(devices) > 0 {
		for _, dev := range devices {
			file := "devices.deny"
			if dev.Allow {
				file = "devices.allow"
			}
			if err := writeFile(path, file, dev.CgroupString()); err != nil {
				return err
			}
		}
		return nil
	}
	if cgroup.Resources.AllowAllDevices != nil {
		if *cgroup.Resources.AllowAllDevices == false {
			if err := writeFile(path, "devices.deny", "a"); err != nil {
				return err
			}

			for _, dev := range cgroup.Resources.AllowedDevices {
				if err := writeFile(path, "devices.allow", dev.CgroupString()); err != nil {
					return err
				}
			}
			return nil
		}

		if err := writeFile(path, "devices.allow", "a"); err != nil {
			return err
		}
	}

	for _, dev := range cgroup.Resources.DeniedDevices {
		if err := writeFile(path, "devices.deny", dev.CgroupString()); err != nil {
			return err
		}
	}

	return nil
}
...
```

所以，为什么 GPU 实例的 spec 中没有`nvidia-uvm`设备？
这个设备应该是实例启动前由 nvidia 的 k8s device plugin 添加到 spec 里的，
翻出代码来验证猜想`k8s-device-plugin-master/server.go`：

```go
...
func (m *NvidiaDevicePlugin) apiDeviceSpecs(filter []string) []*pluginapi.DeviceSpec {
    var specs []*pluginapi.DeviceSpec

    paths := []string{
        "/dev/nvidiactl",
        "/dev/nvidia-uvm",
        "/dev/nvidia-uvm-tools",
        "/dev/nvidia-modeset",
    }

    for _, p := range paths {
        if _, err := os.Stat(p); err == nil {
            spec := &pluginapi.DeviceSpec{
                ContainerPath: p,
                HostPath:      p,
                Permissions:   "rw",
            }
            specs = append(specs, spec)
        }
    }

    for _, d := range m.cachedDevices {
        for _, id := range filter {
            if d.ID == id {
                spec := &pluginapi.DeviceSpec{
                    ContainerPath: d.Path,
                    HostPath:      d.Path,
                    Permissions:   "rw",
                }
                specs = append(specs, spec)
            }
        }
    }

    return specs
}
...
```

nvidia device plugin 会在 spec 里面添加`nvidia-uvm`设备，
但前提是它能在系统中找到这个设备。
系统里明明有这个设备为啥 device plugin 没把它加进去？
莫非 device plugin 抽了？
docker exec 进入 device plugin 里，果然，没这个设备：

```shell
# ls -l /dev/nvidia*
crw-rw-rw-. 1 root root 195,   0 Jan 11 05:32 /dev/nvidia0
crw-rw-rw-. 1 root root 195,   1 Jan 11 05:33 /dev/nvidia1
crw-rw-rw-. 1 root root 195,   2 Jan 11 05:33 /dev/nvidia2
crw-rw-rw-. 1 root root 195,   3 Jan 11 05:33 /dev/nvidia3
crw-rw-rw-. 1 root root 195, 255 Jan 11 05:32 /dev/nvidiactl
```

靠，找不到 uvm 设备为什么 device plugin 还要继续执行不报错啊？
为什么 device plugin 找不到`nvidia-uvm`设备？
目前还不知道是什么原因，
估计是 device plugin 启动的时候 cuda 驱动还没完全搞好或者是像同事说的"nvidia-docker 还没装好"，
这个问题暂时放着不去验证。
考虑修复方案，抽调一台新机器过来，
让 device plugin 调度上去运行，
确定 device plugin 挂载了`nvidia-uvm`设备后节点上的实例就不再发生 GPU 设备挂载的问题。
预计后续可以通过重启 device plugin 的方式解决 这个问题。

