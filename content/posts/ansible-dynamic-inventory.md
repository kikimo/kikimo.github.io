---
title: "Ansible Dynamic Inventory"
date: 2020-02-22T14:50:45+08:00
draft: false
---

ansible 默认从静态 inventory 文件`/etc/ansible/hosts`读取节点信息。
在构建自动化运维系统时，
一般我们的系统会动态的添加或者删除节点的操作，
这个时候使用 ansible 的静态 inventory 文件存放节点信息就难以满足我们的需求。
所幸 ansible 提供了 dynamic inventory 机制，让我们可以通过脚本的形式来提供节点信息。
ansible 通过`-i`参数指定 inventory 脚本：

```shell
$ ansible all -i inventory.py --list-hosts
  hosts (3):
    127.0.0.1
    10.58.10.209
    10.57.33.40
```

`inventory.py`就是我们编写的 dynamic inventory 脚本，
这个脚本需要提供两个命令行选项（可以理解为 dynamic inventory 需要满足的接口规范）：

```shell
$ ./inventory.py
usage: inventory.py [-h] [--list] [--host HOST]

optional arguments:
  -h, --help   show this help message and exit
  --list       Get all hosts.
  --host HOST  Get host vars.
```

`--list`选项让脚本按照节点的分组，输出所有的节点信息：

```shell
$ ./inventory.py --list
{
    "local": [
        "127.0.0.1"
    ],
    "zk1": [
        "10.58.10.209",
        "10.57.33.40"
    ]
}
```

`--host`选线让脚本输出某个节点的 ansible 变量信息：

```shell
$ ./inventory.py --host 127.0.0.1
{
    "ansible_ssh_user": "root"
}
```
测试 `inventory.py`：

```shell
$ ansible all  -i inventory.py -m ping
10.57.33.40 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
10.58.10.209 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
127.0.0.1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

**注意事项**

`inventory.py`必须是一个可执行文件，否则 ansible 会把它当做一个静态 inventory 文件并尝试使用 `ini` 插件去解析它，这事后我们会得类似一下的报错：

```shell
$ ansible all  -i inventory.py -m ping
[WARNING]:  * Failed to parse inventory.py with script plugin: problem running ansible/inventory.py --list ([Errno 13]
Permission denied)

[WARNING]:  * Failed to parse inventory.py with ini plugin: inventory.py:3: Expected key=value host variable
assignment, got: sys,

[WARNING]: Unable to parse inventory.py as an inventory source

[WARNING]: No inventory was parsed, only implicit localhost is available

[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'
```
