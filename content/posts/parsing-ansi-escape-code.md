---
title: "ANSI 转义码(ANSI escape code)解析"
date: 2020-02-11T22:54:39+08:00
draft: false
---

[ANSI 转义序列](https://en.wikipedia.org/wiki/In-band_signaling)是命令行终端下用来控制光标位置、字体颜色以及其他终端选项的一项 in-bind signaling 标准。
> ANSI escape sequences are a standard for in-band signaling to control the cursor location, color, and other options on video text terminals and terminal emulators. 

> In telecommunications, in-band signaling is the sending of control information within the same band or channel used for data such as voice or video. This is in contrast to out-of-band signaling which is sent over a different channel, or even over a separate network. In-band signals may often be heard by telephony participants, while out-of-band signals are inaccessible to the user.

利用`script`命令，我们可以记录下终端操作的所有输入输入，
记录下来的数据包含着 ANSI 转义码，
这使得它非常不便于阅读，我们可以根据 ANSI 转义码来解析这些数据。
[ANSI 转义码的 wiki](https://en.wikipedia.org/wiki/ANSI_escape_code#SGR)基本覆盖了这个标准的核心内容，
部分`terminal`模拟器可能使用了标准以外的转义序列，可以参考[VT510 Video Terminal Programmer Information](https://vt100.net/docs/vt510-rm/chapter4.html)。

根据 ANSI 转义序列规范实现的 ANSI 转义码解析 demo（该 demo 只用来演示说民工，并未完成所有 ANSI 转义序列的解析）：

```python
# coding: utf-8

import sys

with open('typescript', 'rb') as fp:
    data = fp.read()

ESC = '\x1b'
BEL = '\x07'
STATUS_NORMAL = 'NORMAL'
STATUS_ESC = 'ESC'
STATUS_CSI = 'CSI'
STATUS_QMARK = '?'

status = 'NORMAL'
val = ''
# print(len(data))

class VTParser(object):
    def __init__(self, data):
        self.data = data
        self.sz = len(data)
        self.pos = 0
        self.ch = None
        self.status = STATUS_NORMAL

    def eat_digit(self):
        if self.ch is None:
            raise Exception('ch is None')

        v = ''
        while (self.ch >= '0' and self.ch <= '9'):
            if self.pos >= self.sz:
                raise Exception('malformed data')
            v += self.ch
            self.ch = self.data[self.pos]
            self.pos += 1

        return v

    def eat_ctrl_string(self):
        if self.ch is None:
            raise Exception('ch is None')

        ctrls = ''
        while (self.ch != BEL and not (self.ch == ESC and self.peek_next_ch() == '\\')):
            if self.pos >= self.sz:
                raise Exception('malformed data')
            ctrls += self.ch
            self.ch = self.data[self.pos]
            self.pos += 1

        if self.ch == ESC:
            self.next_ch()

        return ctrls

    def next_ch(self):
        if self.pos >= self.sz:
            return False

        self.ch = self.data[self.pos]
        self.pos += 1
        return True

    def peek_next_ch(self):
        if self.pos >= self.sz:
            return None

        ch = self.data[self.pos]
        # print("peek next: %c" % (ch))
        return ch

    def parse(self):
        while self.next_ch():
            if self.status == 'NORMAL':
                if self.ch == ESC:
                    self.status = STATUS_ESC
                else:
                    sys.stdout.write(self.ch)
            elif self.status == STATUS_ESC:
                if self.ch == '[':
                    self.status = STATUS_CSI
                elif self.ch == '>':
                    self.status = STATUS_NORMAL
                elif self.ch == '=':
                    self.status = STATUS_NORMAL
                elif self.ch == ']':
                    if not self.next_ch():
                        raise Exception('malform data')

                    ctrls = self.eat_ctrl_string()
                    # print('\nOperating System Command: %s' % (ctrls))
                    self.status = STATUS_NORMAL
                else:
                    print('ESC, ch: %c, v: 0x%x' % (self.ch, ord(self.ch)))
                    break
            elif self.status == STATUS_CSI:
                if self.ch == '?':
                    if not self.next_ch():
                        raise Exception('malform data')

                    v = self.eat_digit()
                    o = self.ch
                    self.status = STATUS_NORMAL
                elif self.ch == '>':
                    if not self.next_ch():
                        raise Exception('malform data')

                    v = self.eat_digit()
                    if self.ch != 'c':
                        raise Exception('malform data, CSI > %s %c' % (v, self.ch))

                    self.status = STATUS_NORMAL
                elif self.ch == 's':
                    sef.status = STATUS_NORMAL
                elif self.ch == 'u':
                    self.status = STATUS_NORMAL
                else:
                    v = self.eat_digit()

                    if v == '6' and self.ch == 'n':
                        self.status = STATUS_NORMAL
                    elif (v == '4' or v == '5') and self.ch == 'i':
                        self.status = STATUS_NORMAL
                    elif self.ch == 'K':
                        # erase line
                        self.status = STATUS_NORMAL
                    elif self.ch == 'H':
                        # erase line
                        self.status = STATUS_NORMAL
                    elif self.ch == 'J':
                        self.status = STATUS_NORMAL
                    elif self.ch == 'm':
                        self.status = STATUS_NORMAL
                    elif self.ch == ';':
                        if not self.next_ch():
                            raise Exception('malform data')

                        u = self.eat_digit()
                        o = self.ch
                        self.status = STATUS_NORMAL
                        # print('CSI %s; %s %c' % (v, u, o))
                    else:
                        # TODO
                        print('CSI n: %s, ch: %c, v: 0x%x' % (v, self.ch, ord(self.ch)))
                        break

                    # print('csi ch: %c, v: %s' % (ch, v))
                    # break
            else:
                print('ch: %c, v: 0x%x' % (self.ch, ord(self.ch)))
                break

parser = VTParser(data)
parser.parse()
```

