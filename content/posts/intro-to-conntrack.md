---
title: "conntrack 模块简介"
date: 2020-11-01T09:31:55+08:00
draft: true
---

conntrack 是 Linux 下的一个内核模块，这个名字是 connection track 的缩写，
顾名思义，这个模块就是用来做连接跟踪的。
然而这里的链接需要同 TCP 协议中的连接区分开来，
它指的是通信的两个端点之间用于传输数据的连接，
因此它不止可以用来跟踪 TCP 的连接，还可以跟踪 UDP、ICMP 协议保报文这样”连接“。
conntrack 维护一张连接表——conntrack table。
通过 netfilter 的 hook 机制，conntrack 模块可以检查系统中进出的每个网络数据包。
当发现一条新的连接建立时，比如检查到一个 TCP SYNC 数据包，
conntrack 就在 conntrack table 中添加一条连接记录。
可以用 Linux 中的 conntrack 指令查看 conntrack table，
conntrack table 表项中记录的格式如下：

```txt
tcp      6 48 SYN_SENT src=192.168.199.132 dst=172.217.24.14 sport=58074 dport=443 [UNREPLIED] src=172.217.24.14 dst=192.168.199.132 sport=443 dport=58074 mark=0 use=1
```
表项字段说明：

| 序号   | 说明                                                                |
|--------|:-------------------------------------------------------------------:|
| 1      | 协议名称                                                          |
| 2      | 协议号                                                            |
| 3      | 连接 ttl（单位秒）                                                |
| 4      | 连接状态（ESTABLISHED、TIME_WAIT、SYN_SENT...）                   |
| 5      | 发送端的源 IP 地址、目的 IP 地址、源端口、目的端口                |
| 6      | [UNREPLIED] 告诉我们还未受到回应报文，当连接建立后该字段会被删除  |
| 7      | 期望收到的回应报文中的源 IP 地址、目的 IP 地址、源端口、目的端口  |


conntrack 是一个非常重要的模块，它是 ipvs 负载均衡、NAT 的核心依赖模块。
以 SNAT 为例我们来说明 conntrack 的一个应用场景。
当 NAT 模块收到一个新的报文时，它先查看 conntrack table，（需要先检查 NAT 规则吗？）
如果其中存在相关连接信息的表项，它会根据这个表项修改报文的 ip 和 端口信息。
当相关表项不存在时，NAT 想根据 NAT 规则判断是否需要对报文做地址转换，
如果符合 NAT 地址转换规则，那么 NAT 会在 conntrack talbe 中写入转换后的地址，
后续地址转换直接查 conntrack table 即可完成。

我来看一个 NAT 修改 conntrack table 的实例。
我们在一个 Linux Host 节点中启动一个 Linux Guest 虚拟机，
Guest 通过 Host 节点上的网桥以 NAT 的方式实现对外的网络通信，
其中 Guest 节点的 IP 地址为：192.168.199.132，
Host 节点访问 Internet 的 IP 地址为：192.168.199.132。
我们在 Guest 节点上访问百度的主页：

```txt
curl https://www.baidu.com
```

然后在 Host 节点上查看 conntrack table：

```txt
tcp      6 8 CLOSE src=192.168.122.148 dst=180.101.49.12 sport=40212 dport=443 src=180.101.49.12 dst=192.168.199.132 sport=443 dport=40212 [ASSURED] mark=0 use=1
```

可以看到被 NAT 修改后的 conntrack 表项，
其中返回报文的目的地址被修改为 192.168.199.132 也就是运行了NAT 协议的 Host IP 地址。

---
参考：

1. [连接跟踪（conntrack）：原理、应用及 Linux 内核实现](http://arthurchiao.art/blog/conntrack-design-and-implementation-zh/)
2. [Iptables Tutorial 1.2.1 - 7.2. The conntrack entries](https://www.frozentux.net/iptables-tutorial/chunkyhtml/x1309.html)

