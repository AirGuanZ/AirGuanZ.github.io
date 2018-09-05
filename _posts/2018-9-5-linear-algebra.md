---
layout: post
title: 线性代数复习
key: 20180905
tags:
  - Mathematics
---

参考《Linear Algebra and Its Applications》 by David C. Lay。

<!--more-->

## 矩阵和线性方程组

**Definition**. 行初等变换：

1. [倍加变换] 把某一行换成它本身与另一行的倍数的和；
2. [对换变换] 把两行对换；
3. [倍乘变换] 把某一行所有元素乘以同一个非零数。

**Definition**. 称两个矩阵时行等价的，当且仅当其中一个矩阵可以经过一系列行初等变换称为另一个矩阵。

**Theorem**. 若两个线性方程组的增广矩阵是行等价的，则它们具有相同的解集。

**Definition**. 一个矩阵被称为阶梯型（或行阶梯型），当且仅当它满足：

1. 每一非零行都在零行之上；
2. 某一行的先导元素（最左边的非零元素）所在列位于前一行先导元素之后；
3. 先导元素所在列下方元素都是零。

特别地，若一阶梯型矩阵还满足下面两个性质，就称它是简化阶梯型（或简化行阶梯型）：

4. 每一非零行的先导元素均为一；
5. 每一先导元素一时该元素所在列的唯一非零元素。

**Theorem**. 每个矩阵行等价于唯一的简化阶梯型矩阵。

**Definition**. 矩阵$A$中的主元位置是$A$中对应于它的阶梯型中先导元素的位置；主元列则是$A$中包含主元位置的列。若$A$是某线性方程组的增广矩阵，则对应主元列的变量称为基本变量，其他变量称为自由变量。

**Theorem**. [存在与唯一性定理] 线性方程组相容的充要条件是增广矩阵的最右列不是主元列。对一个相容的线性方程组，若无自由变量，则有唯一解，否则有无穷多解。

**Definition**. 若$A$是$m\times n$矩阵，$\boldsymbol x \in \mathbb R^n$，则$A$与$\boldsymbol x$的积$A\boldsymbol x$为：

$$
A\boldsymbol x =
\left[\begin{matrix}
    \boldsymbol a_1 & \boldsymbol a_2 & \cdots & \boldsymbol a_n
\end{matrix}\right]
\left[\begin{matrix}
    x_1 \\ x_2 \\ \vdots \\ x_n
\end{matrix}\right]
= x_1\boldsymbol a_1 + x_2\boldsymbol a_2 + \cdots + x_n\boldsymbol a_n
$$

**Theorem**. 方程$A\boldsymbol x = \boldsymbol x$有解当且仅当$\boldsymbol b$是$A$各列的线性组合。

**Theorem**. 设$A$是$m\times n$矩阵，下列命题等价：

1. $\forall \boldsymbol b \in \mathbb R^m$，$A\boldsymbol x = \boldsymbol b$有解。
2. $\forall \boldsymbol b \in \mathbb R^m$，$\boldsymbol b$是$A$各列的一个线性组合。
3. $A$的各列生成$\mathbb R^m$。
4. $A$在每一行都有一个主元位置。

**Theorem**. 齐次方程$A\boldsymbol x = \boldsymbol 0$有非平凡解，当且仅当方程至少含有一个自由变量。

**Theorem**. 设$A\boldsymbol x = \boldsymbol b$相容，$\boldsymbol p$是一个特解，则$A\boldsymbol x = \boldsymbol b$的解集是：

$$
\{\boldsymbol w = \boldsymbol p + \boldsymbol v_h \mid A\boldsymbol v_h = \boldsymbol 0\}
$$

**Theorem**. 称一组向量$\{\boldsymbol v_1, \ldots, \boldsymbol v_p\}$是线性无关的，当且仅当向量方程

$$
x_1\boldsymbol v_1 + x_2\boldsymbol v_2 + \cdots + x_p\boldsymbol v_p = \boldsymbol 0
$$

仅有平凡解。反之，称该向量组是线性相关的。

**Theorem**. 矩阵$A$各列线性无关，当且仅当$A\boldsymbol x = \boldsymbol 0$仅有平凡解。

**Definitioni**. [变换] 由$\mathbb R^n$到$\mathbb R^m$的一个变换（或称函数、映射）$T$是一个规则，它把$\mathbb R^n$中的每个向量$\boldsymbol x$对应以$\mathbb R^m$中的一个向量$T(\boldsymbol x)$。$\mathbb R^n$称为定义域，$\mathbb R^m$称为余定义域（或称取值空间）。$T(\boldsymbol x)$称为$\boldsymbol x$的像，所有像$T(\boldsymbol x)$的集合称为$T$的值域。

**Definition**. 称变换$T$是线性的，当且仅当：

1. $\forall \boldsymbol u, \boldsymbol v \in \mathrm{Dom}(T)$，有$T(\boldsymbol u + \boldsymbol v) = T(\boldsymbol u) + T(\boldsymbol v)$。
2. $\forall u \in \mathrm{Dom}(T) \forall c \in \mathbb R$，有$T(c\boldsymbol u) = cT(\boldsymbol u)$。

**Theorem**. 设$T: \mathbb R^n \to \mathbb R^m$是线性变换，则存在唯一矩阵$A$使得：

$$
\forall \boldsymbol x \in \mathbb R^n, T(\boldsymbol x) = A\boldsymbol x
$$

事实上可以证明：

$$
A =
\left[\begin{matrix}
    T(\boldsymbol e_1) & T(\boldsymbol e_2) & \cdots & T(\boldsymbol e_n)
\end{matrix}\right]
$$

在这里，$A$被称作线性变换$T$的标准矩阵。

## 矩阵代数

**Definition**. 若$A$是$m\times n$矩阵，$B$是$n \times p$矩阵，$B$的列是$\boldsymbol b_1, \boldsymbol b_2, \ldots, \boldsymbol b_p$，则乘积$AB$是$m\times p$矩阵：

$$
AB = \left[\begin{matrix}
    A\boldsymbol b_1 & A\boldsymbol b_2 & \cdots & A\boldsymbol b_p
\end{matrix}\right]
$$

**Theorem**. $(AB)_{ij} = a_{i1}b_{1j} + a_{i2}b_{2j} + \cdots + a_{in}b_{nj}$。

**Theorem**. 设$A$是$m \times n$矩阵，其相关乘法满足以下性质：

1. $A(BC) = (AB)C$；
2. $A(B + C) = AB + AC$；
3. $(B + C)A = BA + CA$；
4. $r(AB) = (rA)B = A(rB)$；
5. $I_mA = A = AI_n$。

**Definition**. $m \times n$矩阵$A$的转置是一个$n \times m$矩阵，其列由$A$的相关行构成。

**Theorem**. 转置满足以下性质：

1. $(A^T)^T = A$；
2. $(A + B)^T = A^T + B^T$；
3. $(rA)^T = rA^T$；
4. $(AB)^T = B^TA^T$。

**Definition**. 称$n$阶方阵$A$是可逆的，当且仅当存在一个$n$阶方阵$C$满足：

$$
CA = AC = I
$$

称$C$为$A$的逆矩阵，记作$C = A^{-1}$。容易证明可逆矩阵的逆是唯一的。

**Theorem**. 逆矩阵具有以下性质：

1. $(A^{-1})^{-1} = A$；
2. $(AB)^{-1} = B^{-1}A^{-1}$；
3. $(A^T)^{-1} = (A^{-1})^T$。

**Algorithm**. [求$A^{-1}$] 将$A$与$I$排在一起构成增广矩阵$[A~I]$，然后用一系列行初等变换将其中的$A$变为$I$，此时原本的$I$将被变为$A^{-1}$。若变换过程不成功，则$A$不可逆。

**Theorem**. 若$A, B​$都是$n​$阶方阵且$AB = I​$，则$A = B^{-1}, B = A^{-1}​$。

**Definition**. $\mathbb R^n$中的一个子空间是某个集合$H \subseteq \mathbb R^n$，它满足：

1. $\boldsymbol 0 \in H$；
2. $\forall \boldsymbol u, \boldsymbol v \in H$，均有$\boldsymbol u + \boldsymbol v \in H$；
3. $\forall \boldsymbol u \in H$以及标量$c$，均有$c\boldsymbol u \in H$。

**Definition**. 矩阵$A$的列空间$\mathrm{Col}~A$是$A$的各列在加法和标量乘法下的闭包；$A$的零空间$\mathrm{Nul}~A$是$A\boldsymbol x = \boldsymbol 0$的所有解的集合。

**Definition**. 称向量集$A$生成向量集$B$当且仅当$B$中的每个向量都是$A$中向量的线性组合，且不存在$B$以外的向量是$A$中向量的线性组合。若$A$是生成$B$的极小向量集，就称$A$是一个生成$B$的线性无关集。

**Definition**. $\mathbb R^n$中子空间$H$的一组基是$H$中的一个生成$H$的线性无关集。

**Theorem**. 矩阵$A$的主元列构成$\mathrm{Col}~A$的基。

**Definition**. 设$\mathcal B = \{\boldsymbol b_1, \ldots, \boldsymbol b_p\}$是$H$的一组基，对每个$\boldsymbol x \in H$，$\boldsymbol x$相对于基$\mathcal B$的坐标是使得：

$$
\boldsymbol x = c_1\boldsymbol b_1 + \cdots + c_p\boldsymbol b_p
$$

成立的权值$c_1, \ldots, c_p$。称向量：

$$
[\boldsymbol x]_{\mathcal B} = \left[\begin{matrix}
    c_1 \\ c_2 \\ \vdots \\ c_p
\end{matrix}\right]
$$

为$\boldsymbol x$相对于$\mathcal B$的坐标向量。

**Definition**. 非零子空间$H$的维数$\mathrm{dim}~H$是$H$的任意一组基的向量个数；零子空间的维数被特别定义为零。

**Definition**. 矩阵$A$的秩$\mathrm{rank}~A$是$A$的列空间的维数。

**Theorem**. 若矩阵$A$有$n$l列，则$\mathrm{rank}~A + \mathrm{dim}~\mathrm{Nul}~A = n$。

**Theorem**. 设$A$是$n$阶方阵，则下列命题等价：

1. $A$可逆；
2. $A$等价于$I_n$；
3. $A$有$n$个主元位置；
4. $A\boldsymbol x = \boldsymbol 0$仅有平凡解；
5. $A​$各列线性无关；
6. $\boldsymbol x \mapsto A\boldsymbol x$是单射；
7. $\forall \boldsymbol b \in \mathbb R^n$，方程$A\boldsymbol x = \boldsymbol b$至少有一个解；
8. $A$的各列生成$\mathbb R^n$；
9. $\boldsymbol x \mapsto A\boldsymbol x$是满射；
10. 存在$n$阶方阵$C$使得$CA = I$；
11. 存在$n$阶方阵$D$使得$AD = I$；
12. $A^T$可逆；
13. $A$的列向量构成$\mathbb R^n$的一个基；
14. $\mathrm{Col}~A = \mathbb R^n$；
15. $\mathrm{dim}~\mathrm{Col}~A = n$；
16. $\mathrm{rank}~A = n$；
17. $\mathrm{Nul}~A = \{\boldsymbol 0\}$；
18. $\mathrm{dim}~\mathrm{Nul}~A = 0$。

## 行列式

