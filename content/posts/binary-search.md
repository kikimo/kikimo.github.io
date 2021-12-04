---
title: "二分搜索"
date: 2021-12-04T04:24:48+08:00
draft: false
---

二分搜索指在有序列表中通过折半查找的方式寻找目标允许。
对比顺序检索，二分搜索可以吧时间复杂度从 O(n) 优化到 O(nlogn)。
来看二分搜索的示例代码：

```c++
int bsearch(std::vector<int> &a, int target) {
  int s = 0, e = a.size();
  while (s < e) {
    int m = (s + e) / 2;
    if (a[m] == target) {
      return m;
    } else if (a[m] > target) {
      e = m;
    } else { // a[m] < target
      s = m + 1;
    }
  }

  return -1;
}
```

注意到变量`s`和`e`，它们构成一个左闭右开的搜索空间`[s, e)`，
每次迭代取中点`m = floor((s + e) / 2)`，
它满足`s <= m < e`。
*每次迭代如果没有发现目标元素那么算法或者更新`s = m + 1`，或者更新`e = m`，
这可以保证我们的搜索范围是严格收敛的（因为`s <= m < e`）*。

以上是二分搜索的实现方式。
现在如果我们把需求寻找目标元值改成寻找一个目标元素的插入位置，
使得目标元素大于所有它之前的元素。
我们依然可以用二分的思路来处理这个问题。
要保证搜索空间一直是收敛的，
*我们保持每次迭代中`s`和`e`的更新方式必不变，
暨如果`s`和`e`的更新都是`s = m + 1`和`e = m`，
当`a[m] >= target`时我们更新`e = m`*。
这个操作实际上相当于一直把大于等于`target`的元素截断，实现代码如下：

```c++
int bisect_left(std::vector<int> &a, int target) {
  int s = 0, e = a.size();
  while (s < e) {
    if (a[m] >= target) {
      e = m;
    } else {  // a[m] < target
      s = m + 1;
    }
  }

  return s;
}
```

如果我们的需求在改下：寻找数组中的插入位置，使得目标元素大于等于所有它之前的元素。
我们依然可以用二分法，
而实现方式只要改变上面一段代码中的判断符号即可，暨当`a[m] > target`时我们设置`e = m`，

```c++
int bisect_right(std::vector<int> &a, int target) {
  int s = 0, e = a.size();
  while (s < e) {
    if (a[m] > target) {
      e = m;
    } else {  // a[m] <= target
      s = m + 1;
    }
  }

  return s;
}
```

`bisect_left()`和`bisect_right()`两个方法联合起来可以用来检索数组中一个左闭右开的空间，
这个空间中的元素等于目标元素。
