---
title: "扩展欧几里德算法及其应用"
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
我们记录 d, x', y' = egcd(b, c)，其中 x'b + y'c = d。
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

扩展欧几里德算法可用来求解以下问题：
设 a, m 为整数且 m 为素数，求某个整数 x 使得 ax=1(mod m)。
这个问题其实也可以理解为求 a 在取模运算下的倒数，既 x=a^-1(mod m)。
由于 m 为素数，所以必然有 gcd(a, m) = 1，
又通过扩展欧几里德算法我们可以计算出 x, y 使得 ax + my = 1，
可知 x 即是我们要的解，因为 (ax + my)=1(mod m)，my=0(mod m)，所以 ax=1(mod m)。

我们继续看这个算法的一个应用场景：椭圆曲线数字签名。
椭圆曲线加密算法首先会确定一条椭圆曲线，
再选择曲线上的一个点`G`，和一个素数`p`。
算法先利用随机算法生成一个私钥 dA（dA 是一个大整数），
通过计算`Qa = dA*G (mod p)`，得道椭圆曲线上的另一个点`Qa`
这个点就是算法的公钥。
签名时还是利用随机算法生成一个临时私钥`k`，
再利用`k`生成临时公钥`P = k*G = (px, py)`，记`R = px`。
通过以下公式计算`S`，
`S = k^-1 (Hash(m) + dA * R) mod p`，这个过程中所得的 `(R, S)`即为椭圆曲线的数字签名。
其中`Hash(m)`是消息`m`的哈希，
算法通过以下等式来验证一个签名：
```txgt
P = S^-1 * Hash(m) * G + S^-1 * R * Qa
```
这个公示利用公开的消息签名`（R，S）`和公钥`Qa`计算出椭圆曲线上的一个点`P`，如果`P`点的 x 轴坐标等于`R`，那么这个签名就是有效的。
如何验证这个等式呢？
由`S = k^-1 (Hash(m) + dA * R) `可导出

```txt
SP = (k^-1 (Hash(m) + dA * R))P => 
P = (k^-1 (Hash(m) + dA * R)) * P *S^-1
  = k^-1 * Hash(m) * P * S^-1 + Qa * R * S^-1
  = Hash(m) * G * S^-1 + Qa * R * S^-1
```

这就证明了上面的签名验证等式，这个算法里面我们看到的一系列倒数如`k^-1`,`S^-1` 都可以用扩展欧几里德算法来求解。

