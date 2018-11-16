# 密码学数学基础知识

## 基本模运算

$$
(a+b)\bmod n = ((a\bmod n)+(b\bmod n)) \bmod n\\
(a-b)\bmod n = ((a\bmod n)-(b\bmod n)) \bmod n\\
(a\times b)\bmod n = ((a\bmod n)\times(b\bmod n)) \bmod n
$$

如果有$a_1\equiv b_1\pmod n,a_2\equiv b_2\pmod n$， 那么就有：
$$
a_1+a_2\equiv b_1+b_2\pmod n\\\\
a_1-a_2\equiv b_1-b_2\pmod n\\\\
a_1\times a_2\equiv b_1\times b_2\pmod n
$$


## 拓展GCD

`(g,s,k)=ex_gcd(a, b)`

其中`g=gcd(a,b)`，`sa+kb=g`

当ab互质时，`g=1`，存在`(s,k)`可以线性表出`sa+kb=1`

## 逆元

$a$在$\bmod n$下的逆元记为$a^{-1}\pmod n$，如果$a^{-1}\pmod n$为$a\pmod n$的逆元，那么有$a\times a^{-1}\equiv 1\pmod n$.

如果要求$a^{-1}\pmod n$，有$a\times a^{-1}\pmod n$

进一步$a\times a^{-1}+kn=1$，所以要求$a,n$线性表出1，要求$a,n$互质。

所以存在$a^{-1}\pmod n$的充要条件是$gcd(a, n)=1$，然后用拓展GCD算法求算$a^{-1}\pmod n$

## 中国剩余定理 CRT

问题：

> 有物不知其数，三三数之剩二，五五数之剩三，七七数之剩二。问物几何?

形式化问题：
$$
\begin{cases}
x\equiv q_1\pmod {p_1}\\\\
x\equiv q_2\pmod {p_2}\\\\
\dots\\\\
x\equiv q_n\pmod {p_n}
\end{cases}
$$
解法：先定义一个$w_1=5\times7\times[(5\times7)^{-1}\pmod {3}]$

$w_1$有这样几点性质：
$$
w_1\equiv1\pmod 3\\\\
w_1\equiv0\pmod 5\\\\
w_1\equiv0\pmod 7
$$
我们构造类似的$w_2,w_3$

$w_2=3\times7\times[(3\times7)^{-1}\pmod 5]$

$w_3=3\times5\times[(3\times 5)^{-1}\pmod 7]$

就有$x\equiv w_1q_1+w_2q_2+w_3q_3\pmod{p_1p_2p_3}$满足x的要求

我们给出更形式化的解题过程：
$$
w_k=\prod_{i\neq k}p_i[(\prod_{i\neq k}p_i)^{-1}\pmod {p_k}]
$$
$s_n$有这样的性质：
$$
\begin{cases}
s_k\equiv 1\pmod {p_k}\\\\
s_k\equiv 0\pmod {p_i, i\neq k}
\end{cases}
$$

$$
x\equiv \displaystyle \sum_{k=1}^n\{q_k\displaystyle\prod_{i\neq k}p_i[(\displaystyle\prod_{i\neq k}p_i)^{-1}\pmod {p_k}]\}\pmod {\displaystyle\prod_{i=1}^kp_i}
$$



## 欧拉定理

如果$a$与$n$互质，那么：
$$
a^{\phi(n)}\equiv 1\pmod n
$$

## 费马小定理

欧拉定理的特殊形式， 当$n$为质数时：
$$
a^{n-1}\equiv1\pmod n
$$
