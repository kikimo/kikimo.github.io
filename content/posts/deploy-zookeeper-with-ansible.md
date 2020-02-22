---
title: "利用 ansible 部署 zookeeper 集群"
date: 2020-02-22T19:30:32+08:00
draft: false
---

本文介绍如何利用 ansible 部署 zookeeper 集群。
zookeeper 的部署可以分为以下几个步骤：

1. 在部署节点创建部署目录
2. 上传并解压 zookeeper 应用包
3. 初始化 zookeeper 配置文件
4. 启动 zookeeper 服务

步骤 1-3 可用 ansible task 来实现，
最后一个步骤使用 handler 来实现，
利用 handler 的好处在于：
只有步骤 1-3 中对节点配置做了变更后才会触发 zookeeper 重启。

需要创建的部署目录有：

1. `/opt/infra`, 这个目录作为 zookeeper 的安装目录
2. `/tmp/zookeeper`, 这个目录作为 zookeeper 的 data 目录

```yaml
  tasks:
    - name: Create zookeeper installation directory
      file:
        path: "{{item}}"
        state: directory
      notify: Restart zookeeper service
      with_items:
          - /opt/infra
          - /tmp/zookeeper
```

这里使用了`with_items`迭代创建目录,
如果文件是第一次创建`notify`指令会调用`Restart zookeeper service handler`来重启 zookeeper 服务。

上传 zookeeper 包：

```yaml
    - name: Upload zookeeper package
      unarchive:
        src: ./infra_pkgs/apache-zookeeper-3.5.7-bin.tar.gz
        dest: /opt/infra
      notify: Restart zookeeper service
```

这里使用`unarchive`模块来传输和解压 zookeeper 包，
很神奇的一点是`unarchive`模块能感知到文件解压后跟之前对比是否有变化，
根据这一点我们我们仍旧可以是用`notify`指令来触发 zookeeper 重启的`handler`。
关于`handler`的知识点：
1. 所有`task`都执行完以后才会执行被触发的`handler`，
2. 如果多个`task`都触发了同一个`handler`，
那么这个`handler`最终也只会被执行一次。

初始化 zookeeper 配置：

```yaml
    - name: Init zookeeper config
      template:
        src: "{{item.src}}"
        dest: "{{item.dest}}"
      with_items: "{{cfg_files}}"
      notify: Restart zookeeper service
```
需要初始化的配置文件我们放在变量`cfg_files`中，
配置文件使用模板来渲染，所以我们用到了`template`模块；
`cfg_files`是一个字典列表，
每个字典中的`src`字段对应本地模板文件，
`dest`对应模板渲染后的上传路径。

```yaml
  vars:
      cfg_files:
        - src: "./infra_pkgs/zoo.cfg.j2"
          dest: "/opt/infra/apache-zookeeper-3.5.7-bin/conf/zoo.cfg"
        - src: "./infra_pkgs/myid.j2"
          dest: "/tmp/zookeeper/myid"
        - src: "./infra_pkgs/zk.sh"
          dest: "/etc/profile.d/zk.sh"
```

一共有三个模板，
`zoo.cfg.j2`是 zookeeper 的配置文件模板：

```j2
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper
dataDir={{zk_data_dir}}
dataLogDir={{zk_datalog_dir}}
clientPort=2181

# server.1=@zknode1:2888:3888
# server.2=@zknode2:2888:3888
# server.3=@zknode3:2888:3888
{% for server in zk_servers %}
server.{{loop.index}}={{server.ip}}:{{zk_communication_port}}:{{zk_election_port}}
{% endfor %}

autopurge.snapRetainCount=5
autopurge.purgeInterval=1
maxClientCnxns=2000
```
`zk_servers`是个字典变量，
里面存放了 zookeeper 的节点配置信息，
这是个字典列表变量，
后面会介绍。
这里我们通过迭代`zk_servers`变量渲染出 zookeeper 服务的节点配置。

`./infra_pkgs/myid.j2`是 zookeeper 的`myid`文件模板：

```j2
{% for server in zk_servers %}
{% if inventory_hostname == server.ip %}
{{server.myid}}
{% endif %}
{% endfor %}
```
节点的`myid`信息存放在变量`zk_servers`变量中，
这里我们通过迭代`zk_servers`列表，
通过和`inventory_hostname`对比（`inventory_hostname`存放当前操作的节点名）获取当前节点的`myid`信息。

`./infra_pkgs/zk.sh`模板用户渲染 zookeeper 的`profile`配置文件：

```shell
export ZK_HOME=/opt/infra/apache-zookeeper-3.5.7-bin
export PATH=$PATH:$ZK_HOME/bin
```

在以上模板中可以看到几个前面没提过的变量：
`zk_data_dir`, `zk_data_dir`, `zk_datalog_dir` 等，
这些变量是 zookeeper 的配置变量，
用于控制诸如：zookeeper 集群都包含哪些机器，zookeeper 的数据目录，服务端口等配置。
我们通过命令行参数传递这些变量：

```shell
$ ansible-playbook zookeeper.yaml \
    -e '{ "zk_servers": [ {"ip": "127.0.0.1", "myid": 1 } ], "zk_election_port": 3888, "zk_communication_port": 2888, "zk_data_dir": "/tmp/zookeeper", "zk_datalog_dir": "/opt/infra/apache-zookeeper-3.5.7-bin/logs" }'
```

服务重启`handler`：

```yaml
  handlers:
    - name: Restart zookeeper service
      shell:
        cmd: /opt/infra/apache-zookeeper-3.5.7-bin/bin/zkServer.sh restart
```

这里通过`shell`模块直接执行指令`/opt/infra/apache-zookeeper-3.5.7-bin/bin/zkServer.sh restart`来重启 zookeeper 服务，
这个做法目前有个问题，
如果服务启动失败了整个部署脚本仍然会显示部署成功，
这是需要优化的一个点。

来看完整的 ansible`playbook`：

```yaml
---
- hosts: "{{ zk_servers | map(attribute='ip') | list }}"
  vars:
      cfg_files:
        - src: "./infra_pkgs/zoo.cfg.j2"
          dest: "/opt/infra/apache-zookeeper-3.5.7-bin/conf/zoo.cfg"
        - src: "./infra_pkgs/myid.j2"
          dest: "/tmp/zookeeper/myid"
        - src: "./infra_pkgs/zk.sh"
          dest: "/etc/profile.d/zk.sh"
  gather_facts: no
  user: root
  tasks:
    - name: Create zookeeper installation directory
      file:
        path: "{{item}}"
        state: directory
      notify: Restart zookeeper service
      with_items:
          - /opt/infra
          - /tmp/zookeeper
    - name: Upload zookeeper package
      unarchive:
        src: ./infra_pkgs/apache-zookeeper-3.5.7-bin.tar.gz
        dest: /opt/infra
      notify: Restart zookeeper service
    - name: Init zookeeper config
      template:
        src: "{{item.src}}"
        dest: "{{item.dest}}"
      with_items: "{{cfg_files}}"
      notify: Restart zookeeper service
  handlers:
    - name: Restart zookeeper service
      shell:
        cmd: /opt/infra/apache-zookeeper-3.5.7-bin/bin/zkServer.sh restart
```

通过管道映射处理，
我们把`zk_servers`变量中的节点 IP 地址提取出来，
这些 IP 是我们要部署的目标机器：`{{ zk_servers | map(attribute='ip') | list }}`。
