---
title: Paillier半同态加密算法和优化加速
date: 2025-10-31 22:07:47
categories: 密码学
tags: paillier
cover: /imgs/default-cover.jpeg
mathjax: true 
---

Paillier 算法是一种经典的半同态加密算法，其密文具有加法同态性。本文介绍Paillier算法的密钥生成、加解密、同态加、同态乘以及算法的优化加速。

# Pailler 算法具体内容

## 密钥生成

1. 随机选取两个大素数\\( p \\)和\\( q \\)，取\\\( n=pq \\\)，\\( \Phi(n)=(p-1)(q-1) \\)，满足\\(\left|p\right|=\left|q\right|=k\\)（\\(p\\)和\\(q\\)长度相等），且\\(gcd(n,\Phi(n))=1\\)（最大公约数为1）；

2. 计算\\(n=pq\\)以及\\(\lambda=lcm(p-1,q-1)\\)（最小公倍数）；

3. 随机选取整数\\(g\in\mathbb Z^*_{n^2}\\)，满足\\(\mu=(L(g^\lambda\mod n^2))^{-1}\mod n\\)存在，其中\\(L(x)=(x-1)/n\\)；

4. 公钥为：\\((n,g)\\)，私钥为：\\((\lambda,\mu)\\)。


## 加密 \\(Encrypt(m)\\)

1. 输入明文\\(m\\)，满足\\(0\le m \le n\\)；

2. 选择随机数\\(r\in \mathbb Z^*_n\\)，且\\(gcd(r,n)=1\\)；

3. 计算密文\\(c=g^mr^n\mod n^2\\)；

   若\\(g=n+1\\)，则\\(g^m\mod n^2=(mn+1)\mod n^2\\)。

## 解密 \\(Decrypt(c)\\)

1. 输入密文c，满足\\(c\in\mathbb Z^*_{n^2}\\)；

2. 计算明文\\(m=\frac{L(c^\lambda\mod n^2)}{L(g^\lambda\mod n^2)}\mod n=L(c^\lambda\mod n^2)\cdot\mu\mod n\\)。

## 密文-密文同态加法

返回 \\(c=c_1\cdot c_2\mod n^2\\)。

## 密文-明文同态乘法

返回 \\(c=c_1^{m_2} \mod n^2\\)。

## 加解密正确性

\begin{align}
&\lambda=l_1(p-1)=l_2(q-1)\\\\
&g^\lambda=g^{l_1(p-1)}=(g^{p-1})^{l_1}\equiv1\mod p（费马小定理）\\\\
&同理，g^\lambda\equiv1\mod q\\\\
&故，g^\lambda\equiv1\mod(pq)\equiv1\mod n，即g^\lambda\mod n=(1+l\cdot n)\mod n\\\\
&也即，g^{n\lambda}\equiv1\mod n^2\\\\\\\\
Decrypt(Encrypt(m))&=Decrypt(g^mr^n\mod n^2)\\\\
&=L((g^mr^n)^\lambda\mod n^2)\cdot\mu\mod n\\\\
&=\frac{L((g^\lambda)^m\mod n^2)}{L(g^\lambda\mod n^2)}\\\\
&=\frac{L((1+lm\cdot n)\mod n^2)}{L(1+l\cdot n)\mod n^2}\\\\
&=\frac{lm\cdot n}{l\cdot n}=m
\end{align}

## 密文-密文同态加正确性

\begin{align}
Decrypt((c_1\cdot c_2)\mod n^2)&=Decrypt(g^{m_1}r_1^ng^{m^2}r_2^n\mod n^2)\\\\
&=Decrypt(g^{(m_1+m_2)}r_1^nr_2^n\mod n^2)\\\\
&=m_1+m_2
\end{align}

## 密文-明文同态乘正确性

\begin{align}
Decrypt((c_1^{m_2})\mod n^2)&=Decrypt((g^{m_1}r^n)^{m^2}\mod n^2)\\\\
&=Decrypt(g^{m_1m_2}r^{nm_2}\mod n^2)\\\\
&=m_1m_2
\end{align}

## 安全性

方案安全性可以归约到判定性合数剩余假设（Decisional Composite Residuosity Assumption, DCRA），即给定一个合数\\(n\\)和整数\\(z\\)，很难判定\\(z\\) 在模\\(n^2\\)下是否是\\(n\\)次剩余（目前为止还没有多项式时间的算法可以攻破），即是否存在\\(y\\)满足\\(z\equiv y^n(\mod n^2)\\)。

# Paillier 算法优化

## 快速密钥生成

1. 令\\(g=n+1\\)，\\(\lambda=\Phi(n)\\)，\\(\mu=(\lambda^{-1})\mod n\\)；

2. 其余和密钥生成内容相同。

## 加密优化

若\\(g = n + 1\\)，则

## 解密优化

若有素数\\(p\\)和\\(q\\)，\\(n=pq\\)，计算模指数\\(a^b\mod n\\)时，可采用中国剩余定理（Chinese remainder theorem， CRT）优化，即先把\\(a^b\\)映射到\\(\mathbb Z_p\\)、\\(\mathbb Z_q\\)上计算，再将结果聚合到\\(\mathbb Z_n\\)，可得最终结果。

1. （映射到\\(\mathbb Z_p\\)）令\\(x_p=a^{b_p}_p\\)，其中\\(a_p=a\mod p\\)，通过欧拉定理，\\(a_p^{p-1}=1\mod n\\)，故\\(b_p=b\mod(p-1)\\)

2. （映射到\\(\mathbb Z_q\\)）令\\(x_q=a^{b_q}_q\\)，其中\\(a_q=a\mod q\\)，\\(b_q=b\mod(q-1)\\)

3. 使用\\(CRT\\)通项公式：\\(x=x_p\cdot q^{-1}(\mod p)\cdot q+x_q\cdot p^{-1}(\mod q)\cdot p\\)，根据裴蜀定理，由于\\(p\\)、\\(q\\)互素，则\\(q^{-1}(\mod p)\cdot q+p^{-1}(\mod q)\cdot p=1\\)，故：

\begin{align}
x&=x_p\cdot q^{-1}(\mod p)\cdot q+x_q\cdot p^{-1}(\mod q)\cdot p\\\\
&=x_p\cdot(1-p^{-1}(\mod q)\cdot p)+x_q\cdot p^{-1}(\mod q)\cdot p\\\\
&=x_p+(x_q-x_p)\cdot p^{-1}(\mod q)\cdot p
\end{align}

应用到解密中，令\\(L_p(x)=\frac{x-1}{p}\\)，\\(L_q(x)=\frac{x-1}{q}\\)，计算：

1. \\(h_p=L_p(g^{p-1}\mod p^2)^{-1}\mod p\\)，\\(h_q=L_q(g^{q-1}\mod q^2)^{-1}\mod q\\)

2. \\(m_p=L_p(c^{p-1}\mod  p^2)\cdot h_p\mod p\\)，\\(m_q=L_q(c^{q-1}\mod  q^2)\cdot h_q\mod q\\)

3. \\(m=CRT(m_p,m_q)\mod n=m_p+(m_q-m_p)\cdot p^{-1}(\mod q)\cdot p\\)
