---
title: "编写 Ansible Connection 插件"
date: 2020-03-26T09:21:41+08:00
draft: true
---

## 1. Ansible Connection 插件

Ansible Connection 插件负责和受控主机之间的连接通信。
根据 Ansible 的设计，Connection 插件的核心功能包括三个方面：

1. 执行指令
2. 上传文件
3. 下载文件

执行指令接口可用于执行命令、脚本，所执行的脚本可通过上传指令接口传输到目标机上。
通过这几个接口的配合即可覆盖所有和远程主机通信的需求。
Ansible 的发行版自带丰富的连接插件，如默认使用的 ssh 连接插件，local 本地连接插件。
用户也可以根据需要自行编写连接插件。
所有的连接插件都需要继承基类 ansible.plugins.connection.ConnectionBase，
然后实现这个基类中定义的抽象方法，其中包括我们上面提到的三个核心通信接口，
我们来看 ConnectionBase 中的抽象方法定义：

```python

def ensure_connect(func):
    @wraps(func)
    def wrapped(self, *args, **kwargs):
        if not self._connected:
            self._connect()
        return func(self, *args, **kwargs)
    return wrapped

class ConnectionBase(AnsiblePlugin):
    '''
    A base class for connections to contain common code.
    '''
    ...
    @abstractproperty
    def transport(self):
        """String used to identify this Connection class from other classes"""
        pass

    @abstractmethod
    def _connect(self):
        """Connect to the host we've been initialized with"""

    @ensure_connect
    @abstractmethod
    def exec_command(self, cmd, in_data=None, sudoable=True):
        """Run a command on the remote host.

        :arg cmd: byte string containing the command
        :kwarg in_data: If set, this data is passed to the command's stdin.
            This is used to implement pipelining.  Currently not all
            connection plugins implement pipelining.
        :kwarg sudoable: Tell the connection plugin if we're executing
            a command via a privilege escalation mechanism.  This may affect
            how the connection plugin returns data.  Note that not all
            connections can handle privilege escalation.
        :returns: a tuple of (return code, stdout, stderr)  The return code is
            an int while stdout and stderr are both byte strings.

        When a command is executed, it goes through multiple commands to get
        there.  It looks approximately like this::

            [LocalShell] ConnectionCommand [UsersLoginShell (*)] ANSIBLE_SHELL_EXECUTABLE [(BecomeCommand ANSIBLE_SHELL_EXECUTABLE)] Command
        :LocalShell: Is optional.  It is run locally to invoke the
            ``Connection Command``.  In most instances, the
            ``ConnectionCommand`` can be invoked directly instead.  The ssh
            connection plugin which can have values that need expanding
            locally specified via ssh_args is the sole known exception to
            this.  Shell metacharacters in the command itself should be
            processed on the remote machine, not on the local machine so no
            shell is needed on the local machine.  (Example, ``/bin/sh``)
        :ConnectionCommand: This is the command that connects us to the remote
            machine to run the rest of the command.  ``ansible_user``,
            ``ansible_ssh_host`` and so forth are fed to this piece of the
            command to connect to the correct host (Examples ``ssh``,
            ``chroot``)
        :UsersLoginShell: This shell may or may not be created depending on
            the ConnectionCommand used by the connection plugin.  This is the
            shell that the ``ansible_user`` has configured as their login
            shell.  In traditional UNIX parlance, this is the last field of
            a user's ``/etc/passwd`` entry   We do not specifically try to run
            the ``UsersLoginShell`` when we connect.  Instead it is implicit
            in the actions that the ``ConnectionCommand`` takes when it
            connects to a remote machine.  ``ansible_shell_type`` may be set
            to inform ansible of differences in how the ``UsersLoginShell``
            handles things like quoting if a shell has different semantics
            than the Bourne shell.
        :ANSIBLE_SHELL_EXECUTABLE: This is the shell set via the inventory var
            ``ansible_shell_executable`` or via
            ``constants.DEFAULT_EXECUTABLE`` if the inventory var is not set.
            We explicitly invoke this shell so that we have predictable
            quoting rules at this point.  ``ANSIBLE_SHELL_EXECUTABLE`` is only
            settable by the user because some sudo setups may only allow
            invoking a specific shell.  (For instance, ``/bin/bash`` may be
            allowed but ``/bin/sh``, our default, may not).  We invoke this
            twice, once after the ``ConnectionCommand`` and once after the
            ``BecomeCommand``.  After the ConnectionCommand, this is run by
            the ``UsersLoginShell``.  After the ``BecomeCommand`` we specify
            that the ``ANSIBLE_SHELL_EXECUTABLE`` is being invoked directly.
        :BecomeComand ANSIBLE_SHELL_EXECUTABLE: Is the command that performs
            privilege escalation.  Setting this up is performed by the action
            plugin prior to running ``exec_command``. So we just get passed
            :param:`cmd` which has the BecomeCommand already added.
            (Examples: sudo, su)  If we have a BecomeCommand then we will
            invoke a ANSIBLE_SHELL_EXECUTABLE shell inside of it so that we
            have a consistent view of quoting.
        :Command: Is the command we're actually trying to run remotely.
            (Examples: mkdir -p $HOME/.ansible, python $HOME/.ansible/tmp-script-file)
        """
        pass

    ...

    @ensure_connect
    @abstractmethod
    def put_file(self, in_path, out_path):
        """Transfer a file from local to remote"""
        pass

    @ensure_connect
    @abstractmethod
    def fetch_file(self, in_path, out_path):
        """Fetch a file from remote to local; callers are expected to have pre-created the directory chain for out_path"""
        pass

    @abstractmethod
    def close(self):
        """Terminate the connection"""
        pass

```

1. `_connect()`方法用于建立和目标机的连接
2.  `ensure_connect()`装饰器作用于类方法，用来保证放掉调用前和目标机已经建立连接
3. `exec_command(self, cmd, in_data=None, sudoable=True)`方法即为我们上面介绍的指令执行方法，`cmd`参数表示待执行指令，`in_data`参数表示标准输入信息，可以理解为 stdin

## 2. 实现自定义 Ansible 连接插件

这一节我们来是实现一个具体的 Ansible 连接插件。
我们将在[jsternberg/ansible-agent](https://github.com/jsternberg/ansible-agent)基础上完成我们的插件。
jsternberg/ansible-agent 是一个带 agent 的连接插件的实现。
agent 使用 Go 实现，它本质上是一个 HTTP 服务，提供`/exec`、`/upload`接口，连接插件调用者两个接受实现指令执行和文件上传接口。
agent 的代码本身比较简单，我们就不深入了。
但是 jsternberg/ansible-agent 的 Connection 插件因为 Ansbile 版本更新的原因，目前已经无法运行了，
通过简单的修改，我们让它可以运行起来：

```python
from __future__ import (absolute_import, division, print_function)

from StringIO import StringIO
import requests, json
__metaclass__ = type

import ansible.constants as C
from ansible.errors import AnsibleError, AnsibleConnectionFailure
from ansible.module_utils.six import text_type, binary_type
from ansible.module_utils._text import to_bytes, to_native, to_text
from ansible.plugins.connection import ConnectionBase
from ansible.utils.display import Display

display = Display()

class Connection(ConnectionBase):
    ''' Local based connections '''

    transport = 'local'
    has_pipelining = True

    def __init__(self, *args, **kwargs):
        super(Connection, self).__init__(*args, **kwargs)
        self.cwd = None

    def _connect(self):
        ''' connect to the local host; nothing to do here '''
        return self

    def exec_command(self, cmd, in_data=None, sudoable=True):
        addr = self._play_context.remote_addr
        if isinstance(cmd, (text_type, binary_type)):
            cmd = to_bytes(cmd)
        else:
            cmd = map(to_bytes, cmd)

        files = {}
        if in_data:
            files['stdin'] = StringIO(in_data)

        resp = requests.post('http://{}:8700/exec'.format(addr), data={'command': cmd}, files=files)
        if not resp.ok:
            raise AnsibleConnectionFailure('Failed to exec command on {}: {}'.format(addr, resp.reason))

        data = resp.json()
        return data['status'], data['stdout'], data['stderr']

    def put_file(self, in_path, out_path):
        with open(in_path) as fp:
            remote_addr = self._play_context.remote_addr
            resp = requests.put('http://{}:8700/upload'.format(remote_addr), data={'dest': out_path}, files={'src': fp})

            if not resp.ok:
                raise AnsibleConnectionFailure('Failed to upload file: {}'.format(resp.reason))

    def fetch_file(self, in_path, out_path):
        ''' fetch a file from local to local -- for compatibility '''
        super(Connection, self).fetch_file(in_path, out_path)

        display.vvv(u"FETCH {0} TO {1}".format(in_path, out_path), host=self._play_context.remote_addr)
        self.put_file(in_path, out_path)

    def close(self):
        ''' terminate the connection; nothing to do here '''
        self._connected = False

    def fetch_file(self, in_path, out_path):
        raise AnsibleError("not unimplemented")

    def close(self):
        pass
```

我们使用 requests 库直接调用 agent 的 HTTP 接口，这在 Mac 下可能会产生如下错误：

```shell
TASK [Test credstash lookup plugin -- get the password with a context defined here] ***********************************************************************************************************************************************************************************************
objc[6763]: +[__NSPlaceholderDate initialize] may have been in progress in another thread when fork() was called.
objc[6763]: +[__NSPlaceholderDate initialize] may have been in progress in another thread when fork() was called. We cannot safely call it or ignore it in the fork() child process. Crashing instead. Set a breakpoint on objc_initializeAfterForkError to debug.
```

通过设置环境变量可以解决这一问题：

```shell
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
```

具体可以看这个[Ansible issue](https://github.com/ansible/ansible/issues/31869)


