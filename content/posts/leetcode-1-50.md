---
title: "Leetcode 题解——1-50"
date: 2020-12-16T19:55:11+08:00
draft: true
---

## 1. Two Sum

在给定的数组中寻找两个数使其和等于给定的某数，题目保证只有且只有一个解。

题解：

1. 使用字典存放数字出现的此时，便利数组中的每个元素，寻找对应的元素时期和等于给定的数。
2. *上述方法使用了两趟便利，也可以使用一趟便利——即一遍存放字典一遍查找互补原数，效果是一样的*。

## 2. Add Two Numbers

用链表反序存储两非负数字，输出两数之和的结果链表。

题解：

链表加法，需要注意处理长度不一致的情况。

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func addTwoNumbers(l1 *ListNode, l2 *ListNode) *ListNode {
    head := &ListNode{}
    ptr := &head
    p1, p2 := l1, l2

    for p1 != nil && p2 != nil; p1, p2 = p1.Next, p2.Next {
        e := &ListNode{Val: p1.Val + p2.Val}
        ptr.Next = e
        ptr = e
    }

    left := p1
    if left == nil {
        left = p2
    }

    for left != nil; ptr = ptr.Next {
        e := &ListNode{Val: left.Val}
        ptr.Next = e
        ptr = e
    }

    return head.Next
}
```

## 3. Longest Substring Without Repeating Characters

寻找子串中最长的无重复子串。

题解：

1. 用 Set + 滑动窗口，O(2n) 时间复杂度，实现的技巧中，左右的移动可以合并到一个循环中，嵌套循环的形式代码比较丑。
2. 用 Map + 滑动窗口的优化形式，时间复杂度可以优化到 O(n)，实现的技巧在于 Map 存储字符在串中出现的位置，Map 中字符不在滑动窗口 [i, j) 之间的的可以直接覆盖

## 4. Median of Two Sorted Arrays

求两个有序数组的有序数组的中位数。

题解：

使用类似快排的分区算法，具体思路 TODO

*important*

## 5. Longest Palindromic Substring

求最长回文子串。

题解：

TODO（当初用动态规划做的，思路都忘了）

important

## 6. ZigZag Conversion

ZigZag 顺序转化。

题解：

TODO

*very important*

## 7. Reverse Integer

整数顺序翻转：123 -> 321, -123 -> -321。

题解：

模十取数，整数、整数转换。

TODO: 熟悉整数、字符串之间相互转换的函数。

*easy very important*

## 8. String to Integer (atoi)

字串 -> 整数。

题解：

有限状态机器。

*easy very important*
TODO: 字符(rune) -> 数字的转换

## 9. Palindrome Number

判断回文数。

题解：

转化为字串判断回文。

## 10. Regular Expression Matching

正则匹配：支持 ./* 操作符。

题解：

TODO：dfs 搜索

*very important*

## 11. Container With Most Water

最多雨水收集。

题解：

TODO

## 12. Integer to Roman

整数 -> 罗马数。

题解：

*TODO important*

## 12. Roman to Integer

罗马数 -> 整数

题解：

*TODO important*

## 14. Longest Common Prefix

最常公共前缀。

题解：

遍历字串。

## 15. 3Sum

寻找数组中所有符合 a + b + c = 0 的组合。

题解：

*TODO important*

## 16. 3Sum Closest

寻找数组中三个数之和使其最接近给定的整数 target。

题解：

*TODO important*

## 17. Letter Combinations of a Phone Number

根据给定的输入数字，找出手机键盘上的所有可能的字串组合。

题解：

循环遍历生成所有字串组合即可。

## 18. 4Sum

寻找数组中的所有的 a + b + c + d = target。

题解：

*TODO important*

## 19. Remove Nth Node From End of List

从链表中删除倒数第 N 个元素。

题解：

（1）使用二级指针技巧用于元素删除；
（2）使用双指针技巧，第一个指针先向前移动 N 个位置，第二指针开始同步移动。

*TODO important*

## 20. Valid Parentheses

合法的括号。

题解：

堆栈遍历。

*TODO importnat*

## 21. Merge Two Sorted Lists

合并有序链表。

题解：

简单的链表操作。

*TODO*

## 22. Generate Parentheses

给定 n 对括号，生成所有合法的括号配对。

题解：

1. 一种错误的解法：括号生成规则：p(n) = set({ (e), ()e, e()| for e in p(n-1)})，循环迭代生成所有括号集合。
为什么是错的：以上规则无法生成字串 "(())(())"
2. 一种正确的解法：根据括号生成规则深度遍历生成括号字串。

*TODO very important*

## 23. Merge k Sorted Lists

合并 k 个有序链表。

题解：

使用最小堆。
[Package heap](https://golang.org/pkg/container/heap/)

*TODO very important*

## 24. Swap Nodes in Pairs

交换链表中每对相邻的两个元素。

题解：

二级指针骚操作：

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func swapPairs(head *ListNode) *ListNode {
    p = &head
    
    // TODO
    for *p != nil && *p.Next != nil {
        next := *p.Next
        nextNext := next.Next
        
        next.Next = *p
        *p.Next = nextNext
        *p = next
    }

    return head
}
```

*TODO very important*

## 25. Reverse Nodes in k-Group

链表 k 翻转。

题解：

先解决链表翻转的问题，然后再利用二级指针的技巧处理链表 k 翻转。

## 26. Remove Duplicates from Sorted Array

删除有序数组中的重复元素。

题解：

双指针移动。

*TODO very important*

## 27. Remove Element

原地删除数组中的指定元素。

题解：

双指针技巧。i, j: i 表示当前元素会被移动进去的位置，j 表示当前带测试的元素的位置，初始状态 i, j := 0, 0

## 28. Implement strStr()

字符串子串查找。

题解：

1. KMP：现场敲时间不够
2. 保险一点用 rabin-karpa 算法 TODO

*TODO very import*

## 29. Divide Two Integers

求两数相除的商，要求不能使用乘法、出发、模运算。

题解：

TODO

*TODO very important*

## 30. Substring with Concatenation of All Words

题目没看懂!!!

*TODO important*

## 31. Next Permutation

给定一个数组，求这些数字组成的下一个排列组合数。

题解：

交换两个数字形成一个逆序，从右往左找到第一个逆序数对，交换两个数字。

*TODO very important*

## 32. Longest Valid Parentheses

最常可行的括号匹配

题解：

动态规划
*TODO very important*

## 33. Search in Rotated Sorted Array

旋转有序数组查找

题解：
*TODO very important*

## 34. Find First and Last Position of Element in Sorted Array

寻找有序数组中元素起始和结束的位置。

题解：

二分法

*TODO very important*

## 35. Search Insert Position

寻找数组中元素的插入位置。

题解：

二分法变种。

*TODO very important*

## 36. Valid Sudoku

验证速度的合法性

题解：

简单模拟

## 37. Sudoku Solver

数独求解

题解：

深度搜索，剪纸策略：可以算出每个格子可行的元素是什么（行、列、方形中可行元素的交集）

## 38. Count and Say

TODO

## 39. Combination Sum

TODO

## 40. Combination Sum II

TODO

## 41. First Missing Positive

TODO

## 42. Trapping Rain Water

TODO

## 43. Multiply Strings

数乘模拟。

题解：

数乘模拟。

*TODO very important*

## 44. Wildcard Matching

通配符匹配，支持符号：?, *

题解：

深度遍历
*TODO very important*

## 45. Jump Game II

TODO

## 46. Permutations

给定一个数组，数字都不相同，返回所有全排列。

题解：

递归函数。

*TODO very important*

## 47. Permutations II

含重复数的全排列。

题解：

*TODO very important*

## 48. Rotate Image

矩阵九十度翻转，不能分配额外的内存空间。

题解：寻找规律，由外二内翻转。

*TODO very important*

## 49. Group Anagrams

组合字符集相同的单词。

题解：

给单词的字母排序，然后组合集合。

*TODO very important*

## 50. Pow(x, n)

计算 x^n

题解：

连续模平方。

*TODO very important*





 
 


