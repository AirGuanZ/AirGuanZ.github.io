---
title: 取模运算
key: t20210513
tags:
  - Mathematics
---

对常见取模运算规则的总结。

<!--more-->

## Intro

给定整数$a$和$b$，在几乎所有的编程语言中，$r = a \% m$都符合以下条件：

$$
\begin{aligned}
q &\in \mathbb Z \\
a &= qb + r \\
|r| &\le |b|
\end{aligned}
$$

但是，这还不足以严格地定义$r$——其符号仍然是未知的。不同的编程语言中亦有不同的决定符号的方法，本文对这些方法稍做总结。

## Truncated Division

一种常见的方法是通过：

$$
r = a - b \times \mathrm{trunc}\left(\frac a b\right)
$$

来定义$r$，其中$\mathrm{trunc}$把一个数的小数部分直接丢弃掉，即向0取整。注意到：

$$
\left|b \times \mathrm{trunc}\left(\frac a b\right)\right| \le |a|
$$

故$r$若非0，则其符号总是和$a$相同。

## Floored Division

$$
r = a - b \times \left\lfloor \frac a b \right\rfloor
$$

此时容易验证，若$r$非0，则其符号总是和$b$相同。

## Euclidean Division

$$
r = a - |b|\left\lfloor\frac a {|b|} \right\rfloor
$$

此时，$r$总是非负的。这一定义比较符合我们的常识——数论中一般也是这样约定的。

## Implement FD/ED with TD

C/C++等语言提供的%运算符是按Truncated Division定义的，但我们常常需要另外两种取模语义，因此需要对其进行转换：

```cpp
int floor_div_rem(int a, int b)
{
    int r = a % b;
    if((r > 0 && b < 0) || (r < 0 && b > 0))
        r += b;
    return r;
}

int euclidean_div_rem(int a, int b)
{
    int r = a % b;
    if(r < 0)
    {
        if(b > 0)
            r += b;
        else
            r -= b;
    }
    return r;
}
```
