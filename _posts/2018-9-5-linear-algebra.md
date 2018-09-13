---
layout: post
title: 线性代数复习
key: t20180905
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

**Definition**. 称两个矩阵是行等价的，当且仅当其中一个矩阵可以经过一系列行初等变换称为另一个矩阵。

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

**Theorem**. 方程$A\boldsymbol x = \boldsymbol b$有解当且仅当$\boldsymbol b$是$A$各列的线性组合。

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

**Theorem**. 矩阵乘法可使用以下公式完成：

$$
(AB)_{ij}
= a_{i1}b_{1j} + a_{i2}b_{2j} + \cdots + a_{in}b_{nj}
$$

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

**Definition**. [向量空间] 一个向量空间是由一些被称为向量的对象构成的非空集合$V$，在$V$上定义了加法和标量乘法运算，且服从以下法则：

1. $\boldsymbol u, \boldsymbol v \in V \Rightarrow \boldsymbol u + \boldsymbol v \in V$；
2. $\boldsymbol u + \boldsymbol v = \boldsymbol v + \boldsymbol u$；
3. $(\boldsymbol u + \boldsymbol v) + \boldsymbol w = \boldsymbol u + (\boldsymbol v + \boldsymbol w)$；
4. $V$中存在一个零向量$\boldsymbol 0$使得$\boldsymbol u + \boldsymbol 0 = \boldsymbol u$；
5. 对每个$\boldsymbol u\in V$，存在$-\boldsymbol u \in V$使得$\boldsymbol u + (-\boldsymbol u) = \boldsymbol 0$；
6. $\boldsymbol u \in V \Rightarrow c\boldsymbol u \in V$；
7. $c(\boldsymbol u + \boldsymbol v) = c\boldsymbol u + c\boldsymbol v$；
8. $(c+d)\boldsymbol u = c\boldsymbol u + d\boldsymbol u$；
9. $c(d\boldsymbol u) = (cd)\boldsymbol u$；
10. $1\boldsymbol u = \boldsymbol u$。

**Definition**. 向量空间$V$的一个子空间是$V$中的一个满足以下两个性质的子集$H$：

1. $V$中的零向量在$H$中；
2. $H$对向量加法和标量乘法封闭。

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

**Theorem**. 若矩阵$A$有$n$列，则$\mathrm{rank}~A + \mathrm{dim}~\mathrm{Nul}~A = n$。

## 行列式

**Definition**. 对任意方阵$A$，令$A_{ij}$表示通过划掉$A$中第$i$行和第$j$列得到的子矩阵。现定义一阶方阵$A$的值为它包含的唯一一个标量值；而当$n \ge 2$时，定义$n$阶方阵$A$的行列式为：

$$
\mathrm{det}~A = \sum_{j=1}^n(-1)^{1+j}a_{1j}\mathrm{det}~A_{1j}
$$

若令$C_{ij} = (-1)^{i+j}\mathrm{det}~A_{ij}$，则：

$$
\mathrm{det}~A = a_{11}C_{11} + a_{12}C_{12} + \cdots + a_{1n}C_{1n}
$$

该公式称为按$A$的第一行的余因子展开式。可以证明，$n$阶方阵$A$的行列式可以按照任意行或任意列的余因子展开式来计算。

**Theorem**. 若矩阵$A$是三角阵，则$\mathrm{det}~A$等于$A$的主对角线上元素的乘积。

**Theorem**. [行变换] 设$A$是一个方阵，则：

1. 若$A$的某一行的倍数加到另一行上得到了$B$，则$A, B$行列式值相同;
2. 若$A$的两行互换得到了$B$，则$A, B$行列式值互为相反数；
3. 若$A$的某一行所有元素乘以$k$倍得到了$B$，则$\mathrm{det}~B = k\mathrm{det}~A$。

**Theorem**. 方阵$A$是可逆的当且仅当$\mathrm{det}~A \ne 0$。

**Theorem**. 转置一个方阵不改变其行列式的值。

**Theorem**. 方阵乘积的行列式等于其各自行列式的乘积。

**Theorem**. [克拉默法则] 设$A$是一个可逆$n$阶方阵，则对任意$\boldsymbol b \in \mathbb R^n$，方程$A\boldsymbol x = \boldsymbol b$的唯一解可由下式给出：

$$
x_i = \frac{\mathrm{det}~A_i(\boldsymbol b)}{\mathrm{det}~A}, i = 1, 2, \ldots, n
$$

其中$A_i(\boldsymbol b)$是将$A$的第$i$列替换为$\boldsymbol b$后得到的矩阵。

注意到$A_{-1}$的第$j$列是一个满足方程$A\boldsymbol x = \boldsymbol e_j$的向量$\boldsymbol x$，故可根据克拉默法则导出一个求逆的一般公式：

$$
(A^{-1})_{ij} = \frac{\mathrm{det}~A_i(\boldsymbol e_j)}{\mathrm{det}~A}
$$

由于$\mathrm{det}~A_i(\boldsymbol e_j) = (-1)^{i + j}\mathrm{det}~A_{ji} = C_{ji}$，故：

$$
A^{-1} = \frac 1 {\mathrm{det}~A} \left[\begin{matrix}
    C_{11} & C_{21} & \cdots & C_{n1} \\
    C_{12} & C_{22} & \cdots & C_{n2} \\
    \vdots & \vdots &        & \vdots \\
    C_{1n} & C_{2n} & \cdots & C_{nn}
\end{matrix}\right]
$$

右边的矩阵称为$A$的伴随矩阵，记作$\mathrm{adj}~A$。

**Theorem**. 二阶方阵的列确定的平行四边形面积为该方阵的行列式；三阶方阵的列确定的平行六面体体积为该方阵的行列式。

## 特征值和特征向量

**Definition**. 设$A$为$n$阶方阵，$\boldsymbol x$为非零向量。若存在$\lambda$使得$A\boldsymbol x = \lambda x$，则称$\lambda$为$A$的特征值，$\boldsymbol x$为对应于$\lambda$的特征向量。

**Theorem**. 三角矩阵的主对角线元素是其特征值。

**Theorem**. $\lambda_1, \ldots, \lambda_r$是矩阵$A$的相异的特征值，则它们各自对应的一个特征向量构成的集合线性无关。

**Theorem**. $\lambda$是$A$的特征值，当且仅当$\mathrm{det}~(A-\lambda I) = 0$。该方程被称作$A$的特征方程。

**Definition**. 设$A$是$n$阶方阵，$\mathrm{det}~(A-\lambda I)$是个$n$阶多项式，称特征值$\lambda$作为特征方程根的重数为$\lambda$的（代数）重数。

**Theorem**. 每个特征值对应的特征空间的维数小于等于该特征值的重数。

**Definition**. 设$A, B$都是$n$阶方阵，若存在可逆矩阵$P$使得$P^{-1}AP = B$，就称$A$相似于$B$；将$A$变成$P^{-1}AP$的变换称为相似变换。

**Theorem**. 若$A$与$B$相似，那么它们的特征多项式相同，从而由相同的特征值和重数。

**Definition**. [相似对角化] 若方阵$A$相似于对角矩阵，即存在可逆矩阵$P$和对角矩阵$D$使得$A = PDP^{-1}$，则称$A$是可对角化的。

**Theorem**. $n$阶方阵$A$可对角化当且仅当$A$有$n$个线性无关的特征向量。事实上，$A = PDP^{-1}$中的$P$就是$A$的$n$个线性无关的特征向量，此时$D$的主对角线上的元素也正是它们所对应的特征值。

**Theorem**. [线性变换的矩阵] 设$T$是从$n$维向量空间$V$到$m$维向量空间$W$的线性变换，$\mathcal B, \mathcal C$分别是$V$和$W$的基。现设$\boldsymbol x \in V$，若$\mathcal B = \{\boldsymbol b_1, \boldsymbol b_2, \ldots, \boldsymbol b_n\}$，$\boldsymbol x = r_1\boldsymbol b_1 + r_2\boldsymbol b_2 + \cdots + r_n\boldsymbol b_n$，那么：

$$
[\boldsymbol x]_\mathcal B = \left[\begin{matrix}
    r_1 \\ r_2 \\ \vdots \\ r_n
\end{matrix}\right]
$$

跟据$T$的线性性，有：

$$
T(\boldsymbol x) = r_1T(\boldsymbol b_1) + r_2T(\boldsymbol b_2) + \cdots + r_nT(\boldsymbol b_n)
$$

把上式用$\mathcal C$中的坐标表达出来，得到：

$$
\begin{aligned}
& [T(\boldsymbol x)]_\mathcal C\\
&= r_1[T(\boldsymbol b_1)]_\mathcal C + \cdots + r_n[T(\boldsymbol b_n)]_\mathcal C \\
&= \left[\begin{matrix} [T(\boldsymbol b_1)] & \cdots & [T(\boldsymbol b_n)] \end{matrix}\right][\boldsymbol x]_\mathcal B \\
&= M[\boldsymbol x]_\mathcal B
\end{aligned}
$$

$M$就是$T$的矩阵表示，称为$T$相对于基$\mathcal B$和$\mathcal C$的矩阵。在$W = V, \mathcal C = \mathcal B$时，也把$M$称为$T$的$\mathcal B$-矩阵。

**Theorem**. 设$A = PDP^{-1}$，其中$D$是$n$阶对角矩阵，若$\mathbb R^n$的基$\mathcal B$由$P$的列向量构成，那么$D$是变换$\boldsymbol x\mapsto A\boldsymbol x$的$\mathcal B$-矩阵。

## 可逆矩阵定理

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
18. $\mathrm{dim}~\mathrm{Nul}~A = 0$；
19. $A$的特征值不包含零。

## 正交和对称

**Definition**. 与子空间$W$中的全据向量都正交的向量构成的子空间称为$W$的正交补，记作$W^\perp$。

**Theorem**. [余弦定理] $\vert\boldsymbol u - \boldsymbol v\vert = \vert\boldsymbol u\vert^2 + \vert\boldsymbol v\vert^2 - 2\boldsymbol u\cdot \boldsymbol v$

**Theorem**. 设$A$是$m \times n$矩阵，$A$的行向量空间的正交补空间是$A$的零空间，$A$的列向量空间的正交补空间是$A^T$的零空间，即：

$$
\begin{aligned}
    (\mathrm{Row}~A)^\perp &= \mathrm{Nul}~A \\
    (\mathrm{Col}~A)^\perp &= \mathrm{Nul}~A^T
\end{aligned}
$$

**Theorem**. 设$U$是一个具有单位正交列的$m \times n$矩阵，那么线性映射$\boldsymbol x \mapsto U\boldsymbol x$保持向量长度和正交性。

**Theorem**. 具有单位正交列的矩阵也具有单位正交行，反之亦然。

**Theorem**. 对称矩阵不同特征空间中的任意两个特征向量是正交的。

**Theorem**. [柯西-施瓦兹不等式] 设$W$是$\boldsymbol u$生成的子空间，则：

$$
\vert\mathrm{Proj}_W\boldsymbol v\vert = \left\vert\frac{\langle\boldsymbol v, \boldsymbol u\rangle}{\langle\boldsymbol u, \boldsymbol u\rangle}\boldsymbol u\right\vert
= \frac{\langle\boldsymbol v, \boldsymbol u\rangle}{\langle\boldsymbol u, \boldsymbol u\rangle}\vert\boldsymbol u\vert
= \frac{\langle\boldsymbol v, \boldsymbol u\rangle}{\vert\boldsymbol u\vert^2}\vert\boldsymbol u\vert
= \frac{\langle\boldsymbol v, \boldsymbol u\rangle}{\vert\boldsymbol u\vert}
$$

由于$\vert\mathrm{Proj}_W\boldsymbol v\vert \le \vert\boldsymbol v\vert$，故有：

$$
\vert\langle \boldsymbol u, \boldsymbol v\rangle\vert \le \vert\boldsymbol u\vert\vert\boldsymbol v\vert
$$

**Theorem**. $n$阶方阵$A$可正交对角化当且仅当$A$是对称的。
