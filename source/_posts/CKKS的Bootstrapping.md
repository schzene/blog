---
title: CKKS的Bootstrapping
date: 2025-12-15 22:29:12
categories: 密码学
tags: CKKS
cover: https://image.0xc0de.top/file/1768983377942_sinx.png
mathjax: true 
---

# 前情提要

假设一个明文多项式为\\(\text{p}\\)（已乘缩放因子\\(\Delta\\)），密文\\(\mathbf{ct} = (c\_0, c\_1)\\)，则解密步骤为：

\begin{equation}
\text{dec}\left(\mathbf{ct}\right) = c\_0 + c\_1 \cdot s = p + e \mod q
\end{equation}

其中\\(e\\)为噪声多项式，\\(e \ll \Delta\\)，\\(q = q\_L\cdot q\_{L-1}\cdots q\_0\\)为分层模数链。
同态乘法：假设密文\\(\mathbf{ct}\_1 = (c\_0^1, c\_1^1)\\)，\\(\mathbf{ct}\_2 = (c\_0^2, c\_1^2)\\)，同态乘法过程为：

\begin{equation}
\text{mul}(c\_1,c\_2)=
\begin{pmatrix}
c\_0^1 \cdot c\_0^2\\ c\_0^1 \cdot c\_1^2 + c\_1^1 \cdot c\_0^2 \\ c\_1^1 \cdot c\_1^2
\end{pmatrix}^T=
\begin{pmatrix}
d\_0\\ d\_1\\ d\_2
\end{pmatrix}^T
\end{equation}

验证密文值：

\begin{align}
d\_0 + d\_1 \cdot s + d\_2 \cdot s^2 &= c\_0^1 \cdot c\_0^2 + (c\_0^1 \cdot c\_1^2 + c\_1^1 \cdot c\_0^2) s + c\_1^1 \cdot c\_1^2 \cdot s^2\\\\
&=(c\_0^1 + c\_1^1\cdot s)(c\_0^2 + c\_1^2\cdot s)\\\\
&=(a\_1 + e\_1) (a\_2 + e\_2)\\\\
&=a\_1\cdot a\_2 + e\_1e\_2 + e\_1a\_2 + e\_2a\_1\\\\
&=a\_1\cdot a\_2 + e\_\{mul}
\end{align}


\\(\text{mul}(c\_1,c\_2)\\) 无法直接解密，需要对\\(d\_2 \cdot s^2\\)重线性化，在此不提。重点是解密后的噪音项\\(e\_{\text{mul}} = e\_1e\_2 + e\_1a\_2 + e\_2a\_1\\)，每次乘法后\\(||e\_{new}|| \approx \Delta \cdot ||e\_{old}||\\)，多次乘法后\\(||e|| \approx  e\_0\cdot \Delta^k \\)，当\\(||e|| \gtrsim \Delta\\)时解密失败。

---

# CKKS Regular Bootstrapping

![process](https://image.0xc0de.top/file/1768983377598_process.png)

---

## Step 1: ModRaise（模数提升）
将密文从 \\(R\_{q\_0}\\) 提升到 \\(R\_Q\\)（\\(Q = q\_L \cdot \cdots \cdot q\_0 \cdot P\\)，\\(P\\) 是辅助模数）：

\begin{equation}
\mathbf{ct}' = \mathbf{ct}\_{\text{old}} \cdot \frac{Q}{q\_0} \pmod{Q}
\end{equation}

此时：

\begin{equation}\text{dec}\left(\mathbf{ct}'\right) = \frac{Q}{q\_0} \cdot (p + e) \pmod{Q}\end{equation}

作用：为 CoeffsToSlots 和 SlotsToCoeffs 提升乘法深度、ModDown 泰勒近似需要大模数空间

---

## Step 2: Coefficients-to-Slots (CoeffsToSlots)

对于多项式 $p = \sum\_{i=0}^{N-1} p\_i X^i$，系数值和槽位值（明文向量值）的关系为（快速傅里叶变换FFT）：

\begin{align}
\begin{pmatrix}
m\_0 \\\\m\_1 \\\\ \vdots \\\\ m\_{N/2 - 1}
\end{pmatrix}=
\mathbf{V} \cdot
\begin{pmatrix}
p\_0 \\\\ p\_1 \\\\ \vdots \\\\ p\_{N/2 - 1}
\end{pmatrix}
\end{align}

其中\\(\mathbf{V}\\)为 Vandermonde 矩阵：

\begin{pmatrix}
1&1&1&\cdots&1\\\\
1&\omega^1\_N&(\omega^1\_N)^2&\cdots&(\omega^1\_N)^{N-1}\\\\
1&\omega^2\_N&(\omega^2\_N)^2&\cdots&(\omega^2\_N)^{N-1}\\\\
\vdots&\vdots&\vdots&\ddots&\vdots\\\\
1&\omega^{N-1}\_N&(\omega^{N-1}\_N)^2&\cdots&(\omega^{N-1}\_N)^{N-1}\\\\
\end{pmatrix}


分段优化（`CtoS_piece = 3`）：将 \\(\mathbf{V}\\) 分解为 3 个稀疏矩阵乘法，每个需要 \\(O(\sqrt{N})\\) 次旋转而非 \\(O(N)\\)：

\begin{equation}\mathbf{V} = \mathbf{V}\_1 \cdot \mathbf{V}\_2 \cdot \mathbf{V}\_3\end{equation}

过程：

\begin{align}
\text{dec}(\mathbf{ct}) &= \frac{Q}{q\_0} \cdot (p + e) \\\\
\xrightarrow{\text{CoeffsToSlots}} & \text{ (三阶段同态 FFT)} \\\\
\text{dec}(\mathbf{ct}\_{\text{CtoS}}) &= \frac{Q}{q\_0} \cdot \left(\sum\_{k=0}^{N-1} p\_k \omega^{jk} + e\_j\right), \quad j=0,\ldots,N/2-1 \\\\
&= \frac{Q}{q\_0} \cdot (p(\omega^j) + e(\omega^j))\\\\
&= (s\_1, s\_2, \cdots, s\_N)
\end{align}

---

## Step 3: ModDown（模约化近似）

ModDown对每个槽位同态近似执行 $s\_j \bmod q\_0$（用泰勒/切比雪夫逼近），把值折回到低模数范围，相对 `scale` 的误差显著缩小，是“降噪”的核心数学步骤。由于\\(Q = q\_L\cdot \cdots\cdot q\_0 \cdot P\\)，\\(q\_L, \cdots, q\_0, P\\)互素，令\\(K = q\_L \cdot\cdots\cdot q\_1 \cdot P\\)，则\\(Q = q\_0 \cdot K\\)，\\(Q / q\_0 = K\\)，且\\( K \equiv 1 \bmod q \\)，故有：

\begin{equation}
s\_j' \approx s\_j \mod{q\_0} = m\_j + e\_j' (\mod{q\_0})
\end{equation}

重点是需要同态计算 $s\_j \bmod q\_0$，但 $\bmod$ 是非多项式函数，无法直接同态计算。模运算可以表示为：

\begin{equation}f(x) = x - q\_0 \cdot \left\lfloor \frac{x}{q\_0} \right\rceil\end{equation}

可以直接用$\sin(x)$拟合$f(x)$，之后泰勒展开$\sin(x)$：

![sinx](/images/CKKS的Bootstrapping/sinx.png)

---

## Step 4: Slots-to-Coefficients (SlotsToCoeffs)

实际上这是CoeffsToSlots的逆变换，过程：

\begin{align}
\text{dec}(\mathbf{ct}\_{ModDown}) &= (s\_1', s\_2', \cdots s\_N') \\\\
\xrightarrow{\text{SlotsToCoeffs}} & \text{ (三阶段同态逆 FFT)} \\\\
\text{dec}(\mathbf{ct}\_{\text{StoC}}) &= p + e'\\\\
\end{align}

将槽位域密文转回系数域：

\begin{equation}
\mathbf{p}\_{\text{coeffs}}' = \mathbf{V}^{-1} \cdot \mathbf{m}\_{\text{slots}}'
\end{equation}

其中：

\begin{equation}
\mathbf{V}^{-1} = \mathbf{V}^{-1}\_3 \cdot \mathbf{V}^{-1}\_2 \cdot \mathbf{V}^{-1}\_1
\end{equation}

复杂度与 CoeffsToSlots 对称，约 2 层深度。输出为密文 $\mathbf{ct}\_{\text{new}}$ 在模数 $q\_L$ 下加密 $m$，噪声 $\|e\_{\text{new}}\| \approx \Delta \cdot 10^{-2}$（由泰勒精度决定）。

---

## 完整流程总结

\begin{equation}
\boxed{
\begin{align}
\mathbf{ct}\_{\text{old}} &\xrightarrow{\text{ModRaise}} \mathbf{ct}' \in R\_Q \\\\
&\xrightarrow{\text{CoeffsToSlots (2层)}} \mathbf{ct}\_{\text{slots}} \\\\
&\xrightarrow{\text{ModDown (4层)}} \mathbf{ct}\_{\text{slots}}' \\\\
&\xrightarrow{\text{SlotsToCoeffs (2层)}} \mathbf{ct}\_{\text{new}} \in R\_{q\_L}
\end{align}
}
\end{equation}

总深度消耗：$2 + 4 + 2 = 8$ 层（理论），实际我实验的输出显示消耗 $30 - 25 = 5$ 层（优化后）。