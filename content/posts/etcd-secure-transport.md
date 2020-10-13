---
title: "etcd 安全通信"
date: 2020-10-13T12:59:53+08:00
draft: false
---

etcd 中的安全通信包括两中场景，客户端和 etcd 服务端之间的安全通信，etcd 集群节点之间的安全通信。
这两种场景使用的都是 ssl 来实现安全通信。
etcd 中的 ssl 可以设置为服务器单向安全通信也可以设置服务器、客户端双向安全通信。

### 1. 设置服务端安全通信

```txt
$ etcd --name infra0 --data-dir infra0 \
  --cert-file=/path/to/server.crt --key-file=/path/to/server.key \
  --advertise-client-urls=https://127.0.0.1:2379 --listen-client-urls=https://127.0.0.1:2379
```

客户端链接服务端时需要指定 CA 公钥证书以验证服务端的公钥证书：

```txt
$ curl --cacert /path/to/ca.crt https://127.0.0.1:2379/v2/keys/foo -XPUT -d value=bar -v
```

或者也可用`-k`参数让 curl 跳过证书验证环节：

```txt
$ curl -k https://127.0.0.1:2379/v2/keys/foo -XPUT -d value=bar -v
```

### 2. 验证客户端

服务端配置：

```txt
etcd --name infra0 --data-dir infra0 \
  --client-cert-auth --trusted-ca-file=/path/to/ca.crt --cert-file=/path/to/server.crt --key-file=/path/to/server.key \
  --advertise-client-urls https://127.0.0.1:2379 --listen-client-urls https://127.0.0.1:2379
```

`--client-cert-auth`开启客户端请求验证，
客户请求，需要指定客户端公钥证书、私钥这两参数，服务端会用它的 CA 证书验证客户端的公钥证书：

```txt
curl --cacert /path/to/ca.crt --cert /path/to/client.crt --key /path/to/client.key \
  -L https://127.0.0.1:2379/v2/keys/foo -XPUT -d value=bar -v
```

### 3. 集群间的安全通信

etcd 集群间的安全通信和上面的例子类似，只是参数名称不一样：

```txt
# member1
$ etcd --name infra1 --data-dir infra1 \
  --peer-client-cert-auth --peer-trusted-ca-file=/path/to/ca.crt --peer-cert-file=/path/to/member1.crt --peer-key-file=/path/to/member1.key \
  --initial-advertise-peer-urls=https://10.0.1.10:2380 --listen-peer-urls=https://10.0.1.10:2380 \
  --discovery ${DISCOVERY_URL}

# member2
$ etcd --name infra2 --data-dir infra2 \
  --peer-client-cert-auth --peer-trusted-ca-file=/path/to/ca.crt --peer-cert-file=/path/to/member2.crt --peer-key-file=/path/to/member2.key \
  --initial-advertise-peer-urls=https://10.0.1.11:2380 --listen-peer-urls=https://10.0.1.11:2380 \
  --discovery ${DISCOVERY_URL}
```

`--peer-client-cert-auth`参数开启集群节点间的安全通信，
`--peer-trusted-ca-file`, `--peer-cert-file`, `--peer-key-file` 和前面例子中的`--trusted-ca-file`, `--cert-file`, `--key-file`类似。
