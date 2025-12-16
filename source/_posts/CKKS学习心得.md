---
title: CKKS学习心得
date: 2025-12-15 21:42:00
categories: 密码学
tags: CKKS
cover: /images/CKKS学习心得/ckks-process.svg
mathjax: true 
---

# CKKS全过程

<img src="/images/CKKS学习心得/ckks-process.svg" style="zoom:90%;" />

## 一些背景知识

### 分圆多项式

在复数集内有\\(n\\)个单位根，分别是\\(\omega^k\_n=\cos{\frac{2k\pi}{n} }+j\sin{\frac{2k\pi}{n} }=e^{\frac{2k\pi}{n}j}\\)，其中\\(k=0,1,\dots,n-1\\)\\((\omega^0\_n=\omega^n\_n=1)\\)，如果\\(k\\)与\\(n\\)互质，那么称\\(\omega^k\_n\\)为本原单位根。如下图的\\(\omega^k\_1\\)，\\(\omega^k\_3\\)，\\(\omega^k\_5\\)，\\(\omega^k\_7\\)。

<img src="/images/CKKS学习心得/cyclotomic_polynomial.png" style="zoom:50%;" />

若一个整系数多项式的根都是n次本原单位根，这个多项式就是分圆多项式。它是一个不可约多项式，即：

\begin{equation}
\Phi\_n(X) =\prod\_{\begin{subarray}{}1\le k\le n\\gcd(k, n)=1\end{subarray} }{(X-e^{\frac{2k\pi}{n}j})}
\end{equation}

如上图的\\(\Phi\_8 (X)=(X-\omega\_1)(X-\omega\_3)(X-\omega\_5)(X-\omega\_7)=1+X^4\\)。

性质：\\(\Phi\_{2^n} (X)=X^{2^{n-1} }+1\\)，即若\\(N=2^{n-1}\\)，则\\(\Phi\_{2N}(X)=X^N+1\\)

### 离散傅里叶变换：

傅里叶变换(FT)：对于函数$f(t)$，定义其傅里叶变换:

\begin{equation}
F(\omega)=\int^{+\infty}\_{-\infty}f(t)e^{-j\omega t}dt
\end{equation}

离散傅里叶变换(DFT)：对于$N$个离散点序列\\(\{x\_n\}, {0\le n \le N-1}\\)，记$W^k\_n=e^{-j\frac{2\pi}{N}nk}$，定义其DFT：

\begin{equation}
X[K]=\sum^{N-1}\_{n=0}{x\_nW^k\_n},0\le k \le N-1
\end{equation}

快速傅里叶变换(FFT)：对于以上DFT，将后面的和式分为偶数部分和奇数部分：

\begin{align}
X[K] &= \sum^{N/2-1}\_{n=0}x\_{2n}W^{k}\_{2n} + \sum^{N/2-1}\_{n=0}x\_{2n+1}W^{k}\_{2n+1}\\\\
&= \sum^{N/2-1}\_{n=0}{x\_{2n}W^{k}\_{2n}}+W^k\_1\sum^{N/2-1}\_{n=0}{x\_{2n+1}W^{k}\_{2n}}\\\\
&= E[K]+W^k\_1O[K]
\end{align}

以上把一个规模为$N$的DFT分解为$2$个规模为$N/2$的DFT，又$e^{-j\frac{2\pi}{N}2nk}=e^{-j\frac{2\pi}{N/2}nk}=e^{-j\frac{2\pi}{N/2}n(k+N/2)}$，则$W^k\_{2n}=W^{k+N∕2}\_{2n}$；则有：

\begin{align}
E[K]&=E[K+N/2]\\\\
O[K]&=O[K+N/2]\\\\
W^k\_1(N)&=-W^{k+N/2}\_1(N)\\\\
X[K+N/2]&=E[K]-W^{k}\_1(N)O[K]
\end{align}

设$N=2^s$，$s\in\mathbb{N}$，对于多项式$f(x)=a\_0+a\_1x+\cdots+a\_{N-1}x^{N-1}$，令:

$f\_1(x)=a\_0+a\_2x+\cdots+a\_{N-2}x^{N/2-1}$, $f\_2(x)=f(x)=a\_1x+a\_3x^2+\cdots+a\_{N-1}x^{N/2-1}$

则有$f(x)=f\_1(x^2)+xf\_2(x^2)$，若令$x=\omega^k\_N=e^{\frac{2k\pi}{n}j}$，则有：

\begin{align}
f(\omega^k\_N)&=f\_1(\omega^k\_{N/2})+\omega^k\_Nf\_2(\omega^k\_{N/2})\\\\
f(\omega^{k+N/2}\_N)&=f\_1(\omega^k\_{N/2})-\omega^k\_Nf\_2(\omega^k\_{N/2})
\end{align}

只需求出$f\_1(\omega^k\_{N/2})$和$f\_2(\omega^k\_{N/2})$，便可求出$f(\omega\_N^k )$和$f(\omega^{k+N/2}\_N)$，这样就把问题转化为一个递归问题，时间复杂度为$O(log\_2 N)$。

现有：

\begin{align}
\begin{pmatrix}
f(\omega^{0}\_N)\\\\
f(\omega^{1}\_N)\\\\
f(\omega^{2}\_N)\\\\
\vdots\\\\
f(\omega^{N-1}\_N)\\\\
\end{pmatrix}=
\begin{pmatrix}
1&1&1&\cdots&1\\\\
1&\omega^1\_N&(\omega^1\_N)^2&\cdots&(\omega^1\_N)^{N-1}\\\\
1&\omega^2\_N&(\omega^2\_N)^2&\cdots&(\omega^2\_N)^{N-1}\\\\
\vdots&\vdots&\vdots&\ddots&\vdots\\\\
1&\omega^{N-1}\_N&(\omega^{N-1}\_N)^2&\cdots&(\omega^{N-1}\_N)^{N-1}\\\\
\end{pmatrix}\cdot
\begin{pmatrix}
a\_0\\\\
a\_1\\\\
a\_2\\\\
\vdots\\\\
a\_{N-1}\\\\
\end{pmatrix}
\end{align}

根据其共轭性，可知$((\omega\_N^{i-1})^{j-1})\_{N×N}$的逆矩阵为$\frac{1}{N}((\omega\_N^{-i+1})^{j-1})\_{N×N}$，即

\begin{align}
\begin{pmatrix}
a\_0\\\\
a\_1\\\\
a\_2\\\\
\vdots\\\\
a\_{N-1}
\end{pmatrix}=\frac{1}{N}\begin{pmatrix}
1&1&1&\cdots&1\\\\
1&\omega^{-1}\_N&(\omega^{-1}\_N)^2&\cdots&(\omega^{-1}\_N)^{N-1}\\\\
1&\omega^{-2}\_N&(\omega^{-2}\_N)^2&\cdots&(\omega^{-2}\_N)^{N-1}\\\\
\vdots&\vdots&\vdots&\ddots&\vdots\\\\
1&\omega^{1-N}\_N&(\omega^{1-N}\_N)^2&\cdots&(\omega^{1-N}\_N)^{N-1}\\\\
\end{pmatrix}\cdot
\begin{pmatrix}
f(\omega^{0}\_N)\\\\
f(\omega^{1}\_N)\\\\
f(\omega^{2}\_N)\\\\
\vdots\\\\
f(\omega^{N-1}\_N)\\\\
\end{pmatrix}
\end{align}

即可求得其系数。通过$FFT$，时间复杂度为$O(Nlog\_2 N)$

# CKKS的编码解码

## 两个算子：

$\sigma$：将复多项式解码为向量
	假设$N$是$2$的整数次幂，明文$m\in\mathbb{C}^{N/2}$，$q(X)\in\mathbb{C}[x]/(X^N+1)$，分圆多项式$\Phi\_{2N}(X)=1+X^N$的解为$\omega\_1,\omega\_3,\cdots,\omega\_{2N-1}$。
	$\sigma$：$m=\sigma(q(X))=(q(\omega\_1),q(\omega\_3),\cdots,q(\omega\_{2N-1}))$；
	$\sigma^{-1}$：快速傅里叶逆变换，即通过$m=(q(\omega\_1),q(\omega\_3),\cdots,q(\omega\_{2N-1}))$求出$q(X)$。

$\pi$：将$\mathbb{C}^{N}$缩减到$\mathbb{C}^{N/2}$。即设$z\in\mathbb{C}^{N}$，$z'\in\mathbb{C}^{N/2}$：
	$z'=\pi(z)=(z\_0, z\_1,\cdots,z\_{N/2-1})$，
	$z=\pi^{-1}(z')=(z\_0',z\_1',\cdots,z\_{N/2-1}',\overline{z\_{N/2-1}'},\overline{z\_{N/2-2}'},\cdots,\overline{z\_{0}'})$
	注：如果我们在$CKKS$中编码向量时使用大小为$N/2$的复数向量，我们需要通过复制其共轭根的来扩展出它的另一半。

## 编码解码：

编码：

1. 取$m\in\mathbb{C}^{N/2}$
2. 用$\pi^{-1}$将$m$扩展到$\mathbb{C}^N$
3. 计算$\lfloor\Delta\pi^{-1}(m)\rfloor$，其中$\Delta$为精度控制。
4. 编码：$p(X)=\sigma^{-1}(\lfloor\Delta\pi^{-1}(m)\rfloor)$

注：取整“$\lfloor\cdot\rfloor$”使用coordinate-wise random rounding(坐标随机舍入)，取正交基$(1,X,\cdots,X^{N-1})$，令$b=(\sigma(1),\sigma(X),\cdots,\sigma(X^{N-1}))$，$\lfloor z\rfloor=\left[\left(\frac{\langle z, b\_i\rangle}{||b\_i||^2}\right)\_{0\le i\le N-1}\right]$。

解码：$m=\pi(\sigma(\Delta^{-1}p(X))$

以上算法可以看出，$CKKS$使用近似算法而不是精确算法。

# 加密解密

$CKKS$使用$RLWE$来编码和解码。取商环$\mathbb{Z}\_q[x]/\varphi(x)$(通常是由$\mathbb{Z}\_q[x]$中的所有多项式对不可约多项式 $\varphi(x)$求模而形成的有限商（因子）环)。给定多项式对列表$(a\_i(x), b\_i(x))$，搜索未知项式$s(x)$。其中：

• $a\_i(x)$是一组来自$\mathbb{Z}\_q[x]/\varphi(x)$的随机已知多项式；

• $e\_i(x)$是一组环$\mathbb{Z}\_q[x]/\varphi(x)$中关于界t的随机未知小多项式；

• $s(x)$是环$\mathbb{Z}\_q[x]/\varphi(x)$中关于界t的随机未知小多项式；

• $b\_i(x)$=$[a\_i(x)⋅s(x)+ e\_i(x)]\_q$。

此处定义私钥是一对小多项式组$(s(x),e\_i(x))$，对应的公钥是一对多项式组$(a\_i(x),b\_i(x))$，给定$a\_i(x)$，$b\_i(x)$，恢复多项式$s(x)$在计算上是不可实现的。

## 加密enc

设$p\in\mathbb{Z}\_q[X]/(X^N+1)$为编码后的明文多项式，设加密消息为$ct$，有：

\begin{align}
ct=\text{enc}(pt)&=([au+e\_1]\_q,[bu+e\_2+qp/t]\_q)\\\\
&=([au+e\_1]\_q,[asu+eu+e\_2+qp/t]\_q)
\end{align}

其中$e\_1,e\_2$是一组$Z\_q[x]/(X^N+1)$中关于界𝑡的随机小多项式，$u$是系数为$\{-1,0,1\}$的多项式。

## 解密dec

计算：

\begin{align}
pt=\text{dec}(ct)&=\left[asu+eu+e\_2+qp/t-(au+e\_1)s\right]\_q\\\\
&=\left[qp/t+eu+e\_2-e\_1s\right]\_q
\end{align}

$pt$中系数小$q/t$的舍去，得到的结果乘以$t/q$即可得到$p$。

# CKKS的同态运算(同态加和同态乘)

设$p\_0,p\_1\in\mathbb{Z}\_q[X]/(X^N+1)$为编码后的明文多项式，加密后为$c\_0=(c^0\_0,c^1\_0),c\_1=(c^0\_1,c^1\_1)$。

## 同态加：

1. 明文和密文加：

\begin{align}
\text{add}(p\_0, c\_1)&=(qp\_0/t+c^0\_1,c^1\_1)\\\\
\text{dec}(\text{add}(p\_0, c\_1))&=t/q\cdot[qp\_0/t+c^1\_1-sc^0\_1)]\\\\
&=p\_0+t/q(c^1\_1-sc^0\_1)\\\\
&\sim p\_0+p\_1\\\\
\end{align}

2. 密文和密文加：

\begin{align}
\text{add}(c\_0, c\_1)&=(c^0\_0+c^0\_1,c^1\_0+c^1\_1)\\\\
\text{dec}(\text{add}(c\_0, c\_1))&=t/q\cdot[c^1\_0+c^1\_1-s(c^0\_0+c^0\_1)]\\\\
&=t/q\cdot[(c^1\_0-sc^0\_0)+(c^1\_1-sc^0\_1)]\\\\
&\sim p\_0+p\_1
\end{align}


## 同态乘：

1. 明文和密文乘：

\begin{align}
\text{mul}(p\_0,c\_1)&=(p\_0c^0\_1,p\_0c^1\_1)\\\\
\text{dec}(\text{mul}(p\_0,c\_1))&=t/q\cdot(p\_0c^1\_1-p\_0sc^0\_1)\\\\
&=p\_0[t/q\cdot(c^1\_1-sc^0\_1)]\\\\
&\sim p\_0p\_1
\end{align}


2. 密文和密文乘：

\begin{align}
\text{mul}(c\_0,c\_1)=(c^0\_0 \cdot c^1\_0, c^0\_0 \cdot c^1\_1+c^0\_1,c^0\_1 \cdot c^1\_1)=(d\_0, d\_1, d\_2)\\\\
\text{relin}((d\_0, d\_1, d\_2), evk)=(d\_0,d\_1)+\lfloor p^{-1}d\_2 \cdot evk \rfloor=(d'\_0,d'\_1)\\\\
\text{dec}((d\_0,d\_1)+\lfloor p^{-1}d\_2 \cdot evk \rfloor)\sim p\_0p\_1
\end{align}

其中$evk=[-a\_0s+e\_0+ps^2, a\_0]\_{pq}$，$e\_0$是一个小的随机多项式，$p$是一个大整数，$a\_0$从$\mathbb{R}\_{pq}$中随机采样。$\text{relin}$的作用是将三维的密文调整为二维密文，同时可以进行$\text{dec}$计算。