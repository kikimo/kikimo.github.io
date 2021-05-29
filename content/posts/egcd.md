---
title: "扩展欧几里德算法"
date: 2021-05-29T18:56:21+08:00
draft: false
---

利用辗转相除法可以计算两个整数的最大公约数。
我们记两个整数 a, b 的最大公约数为 gcd(a, b)，假设 gcd(a, b) = d，
那么必定存在两个整数 x, y 使得 ax + by = d。
x， y 可以用扩展欧几里德算法来计算。
我们记 egcd(a, b) 为扩展欧几里德算法，且 d, x, y = egcd(a, b)
其中 d = gcd(a, b)，且 ax + by = d。

令 c = a % b，当 c = 0 时，易知 egcd(a, b) = b, 0, 1。
当 c != 0，那么此时有 gcd(a, b) = gcd(b, c) = d。
我们记录 d, x', y1 = egcd(b, c)，其中 x'b + y'c = d。
记录 k = [a / b]，那么有 c = a - kb，所以有
x'b + y'(a - kb) = d，整理后可得 y'a + (x' - ky')b = d，
可知有 x = y', y = x' - ky' 使 ax + by = d。
算法的实现如下：

```go
package main

import "fmt"

func egcd(a, b int) (d, x, y int) {
    if a > b {
        d, x, y = doEgcd(a, b)
    } else {
        d, y, x = doEgcd(b, a)
    }

    return
}

func doEgcd(a, b int) (d, x, y int) {
    // assume a >= b
    c := a % b
    if c == 0 {
        d, x, y = b, 0, 1
        return
    }

    d, xp, yp := doEgcd(b, c)
    x, y = yp, xp - (a / b) * yp
    return
}

func main() {
    d, x, y := egcd(5, 3)
    fmt.Printf("egcd %d, %d: %d, x: %d, y: %d\n", 5, 3, d, x, y)
}
```

扩展欧激励得算法可用来求解以下问题：
设 a, m 为整数且 m 为素数，求某个整数 x 使得 ax=1(mod m)。
这个问题其实也可以理解为求 a 在取模运算下的倒数，既 x=a^-1(mod m)。
由于 m 为素数，所以必然有 gcd(a, m) = 1，
又通过扩展欧几里德算法我们可以计算出 x, y 使得 ax + my = 1，
可知 x 即是我们要的解，因为 (ax + my)=1(mod m)，my=0(mod m)，所以 ax=1(mod m)。
