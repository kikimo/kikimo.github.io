---
title: "Ubuntu 20.04 中文输入法配置"
date: 2020-10-17T19:30:34+08:00
draft: true
---

给新机器安装了 Ubuntu 20.04，安装了语言支持、选择 ibus 输入法并安装了 ibus-pinyin，
然而发现无论是`im-config`还是`ibus-setup`都无法把中文输入法调出来。
最后发现输入法配置在 ubuntu 的 settings 里，配置过程如下：

1. 进入 settings

![进入 settings](/images/uim/goto-settings.jpg)

2. 进入`Region & Language`配置

![进入 Region & Language](/images/uim/goto-rl.jpg)

3. 添加输入源

![添加输入源](/images/uim/add-is.jpg)

4. 添加中文拼音输入法

![添加中文拼音输入法](/images/uim/add-pinyin.jpg)

5. 修改输入法切换快捷键

![修改输入法切换快捷键](/images/uim/update-shortcut.jpg)
