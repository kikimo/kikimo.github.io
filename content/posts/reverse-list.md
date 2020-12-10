---
title: "链表反转"
date: 2020-11-28T19:53:33+08:00
draft: true
---

很久很久很久以前，
在[slashdot 上看到 Linus 关于使用二级指针删除单向链表的分享](https://meta.slashdot.org/story/12/10/11/0030249/linus-torvalds-answers-your-questions)。
同样的二级指针操作技巧，
稍加思考也可以用在链表反转的操作上，可以让代码清晰简洁，具体实现如下：

```golang
package main

import "fmt"

type Node struct {
    val int
    next *Node
}

func reverseList(head *Node) *Node{
    var ptr, next *Node = nil, head

    for next != nil {
        nextp := &next.next
        tmp := next.next
        *nextp = ptr
        ptr = next
        next = tmp
    }

    return ptr
}

func printList(head *Node) {
    for ptr := head; ptr != nil; ptr = ptr.next {
        fmt.Println(ptr.val)
    }
}

func main() {
    // nextNext := Node{val: 3}
    // next := Node{val: 2, next: &nextNext}
    // head := Node{val: 1, next: &next}
    head := Node{val: 1}
    printList(&head)
    ptr := reverseList(&head)
    printList(ptr)
}
```

如果了解过 Linus 关于二级指针的骚操作，
这个思路在正常情况下其实花不到两分钟就能推导出来
（从动笔写伪代码到理清思路确实不用两分钟）。
当然在某些紧张的场景下脑子堵了也可能出现翻车的尴尬，
可以承认心理素质不好，但是要说 coding 水平不行那是一点都不服气的。

