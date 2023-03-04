---
title: "Ansible vs Shell Script"
date: 2020-02-22T20:23:31+08:00
draft: true
---

ansible 可以通过 playbook 用来完成节点的配置管理工作，如：基建部署、软件安装、服务管理等。
如果只是从完成的任务上看，
ansible 的配置管理功能和 shell 脚本似乎是一样的，
既然如此，为什么还要花时间去部署、学习 ansible 呢？
和 shell 脚本相比 ansible 有什么独特的优势？

## 1. 声明式 vs 指令式

ansible 内置了大量功能模块，
这些模块大部分是声明式（Declarative）的。
声明式的特点是你告诉 ansible 你想要的是一个什么样的结果，
ansible 来帮你达成这个目标。
和声明式相对应的是命令式（imperative）,
shell 脚本代码就是命令式的。
以文件的操作为例，
假设我们要创建一个目录，
使用 shell 指令的操作是：

```shell
$ mkdir /path/to/file
```

用 ansible 则是：

```yaml
  file:
    path: /path/to/file
    state: directory
```

乍看 ansible 这个操作跟 shell 指令也没什么区别，甚至更麻烦。
然而，我们这里用到的 ansible`file`模块可不止能用来创建文件，
你只要把 state 值设置为`file`它就能给你创建文件，
设置为`absent`时，ansible 就会帮你删除文件；
当我们创建多级目录的文件时，
如果忘了给`mkdir`指令加`-p`参数，
脚本可能就执行失败了。
使用 ansible 就不再需要关注如此细节的问题，
对文件的操作统一转化为对目标文件状态的描述，
只要设置好目标文件的状态 ansible 就会自动帮你达成这个目标。

## 2. 清晰的代码结构和结果输出

ansible 是通过定义一系列 task 来最终达成你的目标的。
对于一个复杂的任务，
使用 shell 脚本你的代码中可能会包含复杂的逻辑跳转，
相对而言 ansible 的代码近乎平铺直陈，
你可清晰看到一个 ansible 任务中有多少个 task，
而且每个 task 都有相应的 name 字段用于描述他的作用。
ansible 会按顺序输出每个 task 的执行结果，
并输输出整个任务最总的执行结果，
包括：多少成功、失败的情况，有哪些节点不可达等。
来看一个 ansble 任务执行的例子：

```shell
$ ansible-playbook -i inventory.py zookeeper.yaml -e '{ "zk_servers": [ {"ip": "127.0.0.1", "myid": 1 } ], "zk_election_port": 3888, "zk_communication_port": 2888, "zk_data_dir": "/tmp/zookeeper", "zk_datalog_dir": "/opt/tdinfra/apache-zookeeper-3.5.7-bin/logs" }'

PLAY [[u'127.0.0.1']] **********************************************************************************************************************************************************

TASK [Create zookeeper installation directory] *********************************************************************************************************************************
ok: [127.0.0.1] => (item=/opt/tdinfra)
ok: [127.0.0.1] => (item=/tmp/zookeeper)

TASK [Upload zookeeper package] ************************************************************************************************************************************************
ok: [127.0.0.1]

TASK [Init zookeeper config] ***************************************************************************************************************************************************
ok: [127.0.0.1] => (item={u'dest': u'/opt/tdinfra/apache-zookeeper-3.5.7-bin/conf/zoo.cfg', u'src': u'./infra_pkgs/zoo.cfg.j2'})
ok: [127.0.0.1] => (item={u'dest': u'/tmp/zookeeper/myid', u'src': u'./infra_pkgs/myid.j2'})
ok: [127.0.0.1] => (item={u'dest': u'/etc/profile.d/zk.sh', u'src': u'./infra_pkgs/zk.sh'})

TASK [debug-test] **************************************************************************************************************************************************************
ok: [127.0.0.1] => {
    "msg": "127.0.0.1"
}

PLAY RECAP *********************************************************************************************************************************************************************
127.0.0.1                  : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

当然这些功能理论上也可以通过编写复杂的脚本来实现，
但这不但费时费力，
而且它对脚本编写者有极高的要求。

## 3. 更加强悍的语言表达能力

ansible 具有比脚本强悍得多的表达能力。
我们前面提到的 ansible 的声明式的模块用法就是 ansible 表达能力优越的具体体现之一。
声明式可以让你用更少的代码完成更复杂的操作，
因为具体的脏活累活都被模块承包了。
除此之外，ansible 语言表达能力强悍还体现在它对模板的支持和灵活的变量传递上。

我们可在 ansible 中使用 Jinja 模板，模板的使用有两种形式：

1. 使用`template`模块渲染模板文件
2. 在 ansible playbook 中直接使用 Jinja 模板语法

`template`模块用于渲染本地的模板文件，
并将渲染出来的结果传输到目标节点上。
`template`经常使用在需要配置文件分发的场景中，
例如在部署 zookeeper 集群时，
我们便可以使用`template`来渲染 zookeeper 的`zoo.cfg`和`myid`配置文件。
试想一下如果我们用 shell 脚本来实现 zookeeper 集群中每个节点的配置会是什么样的。
你可能会想到使用 shell 的 here document，
实际上知道 here document 的人估计都不会很多，
更遑论使用。
就算你知道 here docuemnt，你还得小心 here docuemtn 变量替换过程中危险的转义问题，
而一些复杂的场景，如需要使用循环迭代，列表过滤器，单存使用 shell 脚本你几乎是无法完成的。

回到另外一个问题上，也就是 ansible 对变量传递的支持。
ansible 可以帮你自动收集目标节点上的信息，
机器的网口、IP、CPU、内存等信息，
你也可以通过`-e`参数传递参数，
参数除了普通的键值对外，
还可以传递 JSON 对象。
你可以在 ansible 脚本或者 Jinja 模板中直接引用。
你还可以通过`set_fact`模块在原有变量的基础上定义新的变量，
这些场景用 ansible 来处理可能几行代码就能搞定，
用 shell 脚本来做绝对会吐血的。

## 4. 模块幂等性质

> An operation is idempotent if the result of performing it once is exactly the same as the result of performing it repeatedly without any intervening actions.

ansible 自带的模块中有很大一部分比例和 shell 指令存在功能上是有重叠的。
和 shell 指令比，
这些模块除了功能更加强大外，
他们大多实现了幂等性，
由此 ansible 可以告诉你在执行了某个模块后，
系统的配置是否产生了变化，
你可以利用这些信息来触发进一步的操作，如重启服务等。

## 5. 丰富的开源部署模块支持

ansible 提倡编写可复用的部署模块。
ansible 社区维护了丰富的 ansible 开源部署模块， 
这些模块可以在官方维护的[galaxy.ansible.com](https://galaxy.ansible.com/)上找到。
我们可以在上面找到诸如：zookeeper、ES、JDK、kafka 等基建服务的部署模块。
通过 ansible 自带的 ansible-galaxy 指令，可直接从 galaxy.ansible.com 下载和安装部署模块。

