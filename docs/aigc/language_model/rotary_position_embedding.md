# 旋转式位置编码(RoPE)

旋转式位置编码（RoPE）最早是论文`[1]`提出的一种能够将相对位置信息依赖集成到 self-attention 中并提升 transformer 架构性能的位置编码方式。而目前很火的 LLaMA 模型也是采用该位置编码方式。

## **基本概念**

首先论文中定义一个长度为 `N` 的输入序列为：
$$
S_{N}=\{ {token}_{i} \}_{i=1}^{N} \\
$$
其中 ${token}_i$  表示输入序列中第 $i$ 个 token，而输入序列 `SN` 对应的 embedding 表示为：
$$
E_{N}=\{ x_i \}_{i=1}^N\\
$$
其中 $x_i$ 表示第 $i$ 个 token `wi` 对应的 `d` 维词嵌入向量。

接着在做 self-attention 之前，会用词嵌入向量计算 `q, k, v` 向量同时加入位置信息，函数公式表达如下：
$$
q_m=f_q(x_m,m) \\ k_n=f_k(x_n,n) \\ v_n=f_v(x_n,n) \\
$$
其中 $q_m$ 表示第 `m` 个 token 对应的词向量 $x_m$  集成位置信息 `m` 之后的 query 向量。而 $k_n$ 和 $v_n$ 则表示第 `n` 个 token 对应的词向量 $x_n$ 集成位置信息 `n` 之后的 key 和 value 向量。

基于 transformer 的位置编码方法都是着重于构造一个合适的 `f{q,k,v}` 函数形式。而计算第 m 个词嵌入向量 $x_m$ 对应的 self-attention 输出结果，就是 $q_m$ 和其他 $k_n$ 都计算一个 $attention score$ ，然后再将 $attention score$ 乘以对应的 $v_n$ 再求和得到输出向量 $o_m$：

$$
a_{m,n}=\frac{exp(\frac{q_m^Tk_n}{\sqrt{d}})}{\sum_{j=1}^Nexp(\frac{q_m^Tk_j}{\sqrt{d}})} \\ o_m=\sum_{n=1}^Na_{m,n}v_n \\
$$

## **绝对位置编码**

对于位置编码，常规的做法是在计算 query, key 和 value 向量之前，会计算一个位置编码向量 $Pi$ 加到词嵌入 $x_i$ 上，位置编码向量 $Pi$ 同样也是 `d` 维向量，然后再乘以对应的变换矩阵 `W{q,k,v}`：
$$
f_{\{q,k,v\}}(x_i,i)=W_{\{q,k,v\}}(x_i+p_i) \\
$$
而经典的位置编码向量 $Pi$ 的计算方式是：
$$
p_{i,2t}=sin(\frac{i}{10000^{\frac{2t}{d}}}) \\ p_{i,2t+1}=cos(\frac{i}{10000^{\frac{2t}{d}}})\\
$$
其中 $p_{i,2t}$ 表示位置 `d` 维度向量 $Pi$ 中的第 `2t` 位置分量也就是偶数索引位置的计算公式，而 $p_{i,2t+1}$ 就对应第 `2t+1` 位置分量也就是奇数索引位置的计算公式。

## **旋转式位置编码**

接下来介绍 Rotary Transformer（RoFormer）模型，它的主要改动是应用“旋转式位置编码 Rotary Position Embedding, RoPE) ”, 这是一种配合Attention机制能达到“绝对位置编码的方式实现相对位置编码”的设计。而也正因为这种设计, 它还是目前唯一一种可用于线性Attention的相对位置编码。

### 基本思路

在RoPE中，我们的出发点就是“通过绝对位置编码的方式实现相对位置编码”，这样做既有理论上的优雅之处，也有实践的实用之处，比如它可以拓展到线性Attention中就是主要因为这一点。

在机器学习中，我们通常只关注实数，但对于旋转嵌入来说，使用复数作为空间的基域在数学上更为方便。我们先考虑二维情形, 然后借助复数来求解。将query向量和key向量的元素视为单个复数，我们将使用 $\mathbb{C}^{d/2}$而不是通常的 $\mathbb{R}^{d}$空间来表示。具体来说，我们不再把 $\mathbf{q}=(q_1,q_2,q_3,q_4,\ldots,q_{d})$ 视为 d 维实数向量，而是把它视为 $\mathbf{q}=(q_1+iq_2, q_3+iq_4,\ldots q_{d-1} + iq_{d})\in\mathbb{C}^{d/2}$。如果 d 是奇数，我们可以用一个虚坐标来填充它，以确保对齐。

$q$ 和 $k$ 分别为query 向量和key向量，$m$ 和$n$分别为相应标记的绝对位置。假设 $f(x,ℓ)$ 是一个函数，它接收位于位置 $ℓ$ 的嵌入$ x$，并输出一个包含相对位置信息的新嵌入。我们假设通过下述运算来给 $q, k$ 添加绝对位置信息：
$$
\tilde{\boldsymbol{q}}_{m}=\boldsymbol{f}(\boldsymbol{q}, m), \quad \tilde{\boldsymbol{k}}_{n}=\boldsymbol{f}(\boldsymbol{k}, n)
$$

也就是说, 我们分别为 $\boldsymbol{q}, \boldsymbol{k}$ 设计操作 $f(\cdot, m), \boldsymbol{f}(\cdot, n)$, 使得经过该操作后, $\tilde{\boldsymbol{q}}_{m}, \tilde{\boldsymbol{k}}_{n}$ 就带有了位置 $m, n$ 的绝对位置信息。Attention的核心运算是内积, 所以我们希望的内积的结果带有相对位置信息, 因此假设存在恒等关系:

$$
\langle\boldsymbol{f}(\boldsymbol{q}, m), \boldsymbol{f}(\boldsymbol{k}, n)\rangle=g(\boldsymbol{q}, \boldsymbol{k}, m-n)
$$

所以我们要求给出该恒等式的一个（尽可能简单的）解。求解过程还需要一些初始条件, 显然我们可以合理地设 $f(\boldsymbol{q}, 0)=\boldsymbol{q}$ 和 $f(k, 0)=k$ 。

### 求解过程

在复数中有 $\langle\boldsymbol{q}, \boldsymbol{k}\rangle=\operatorname{Re}\left[\boldsymbol{q} \boldsymbol{k}^{*}\right], \operatorname{Re}[]$ 代表复数的实部, 所以我们有
$$
\operatorname{Re}\left[\boldsymbol{f}(\boldsymbol{q}, m) \boldsymbol{f}^{*}(\boldsymbol{k}, n)\right]=g(\boldsymbol{q}, \boldsymbol{k}, m-n)
$$

简单起见, 我们假设存在复数 $\boldsymbol{g}(\boldsymbol{q}, \boldsymbol{k}, m-n)$, 使得 $f(\boldsymbol{q}, m) \boldsymbol{f}^{*}(\boldsymbol{k}, n)=\boldsymbol{g}(\boldsymbol{q}, \boldsymbol{k}, m-n)$, 然后我们用复数的指数形式, 设

$$
\begin{aligned}
\boldsymbol{f}(\boldsymbol{q}, m) & =R_{f}(\boldsymbol{q}, m) e^{\mathrm{i} \Theta_{f}(\boldsymbol{q}, m)} \\
\boldsymbol{f}(\boldsymbol{k}, n) & =R_{f}(\boldsymbol{k}, n) e^{\mathrm{i} \Theta_{f}(\boldsymbol{k}, n)} \\
\boldsymbol{g}(\boldsymbol{q}, \boldsymbol{k}, m-n) & =R_{g}(\boldsymbol{q}, \boldsymbol{k}, m-n) e^{\mathrm{i} \Theta_{g}(\boldsymbol{q}, \boldsymbol{k}, m-n)}
\end{aligned}
$$

$$
\begin{aligned}
\boldsymbol{R}_{f}(\boldsymbol{q}, m) R_{f}(\boldsymbol{k}, n) & =R_{g}(\boldsymbol{q}, \boldsymbol{k}, m-n) \\
\Theta_{f}(\boldsymbol{q}, m)-\Theta_{f}(\boldsymbol{k}, n) & =\Theta_{g}(\boldsymbol{q}, \boldsymbol{k}, m-n)
\end{aligned}
$$

对于第一个方程, 代入 $m=n$ 得到

$$
R_{f}(\boldsymbol{q}, m) R_{f}(\boldsymbol{k}, m)=R_{g}(\boldsymbol{q}, \boldsymbol{k}, 0)=R_{f}(\boldsymbol{q}, 0) R_{f}(\boldsymbol{k}, 0)=\|\boldsymbol{q}\|\|\boldsymbol{k}\|
$$

最后一个等号源于初始条件 $f(\boldsymbol{q}, 0)=\boldsymbol{q}$ 和 $f(\boldsymbol{k}, 0)=\boldsymbol{k}$ 。所以现在我们可以很简单地设 $R_{f}(\boldsymbol{q}, m)=\|\boldsymbol{q}\|, R_{f}(\boldsymbol{k}, m)=\|\boldsymbol{k}\|$, 即它不依赖于 $m$ 。至于第二个方程, 同样代入 $m=n$ 得到

$$
\Theta_{f}(\boldsymbol{q}, m)-\Theta_{f}(\boldsymbol{k}, m)=\Theta_{g}(\boldsymbol{q}, \boldsymbol{k}, 0)=\Theta_{f}(\boldsymbol{q}, 0)-\Theta_{f}(\boldsymbol{k}, 0)=\Theta(\boldsymbol{q})-\Theta(\boldsymbol{k})
$$

这里的 $\Theta(\boldsymbol{q}), \Theta(\boldsymbol{k})$ 是 $\boldsymbol{q}, \boldsymbol{k}$ 本身的幅角, 最后一个等号同样源于初始条件。根据上式得到

$\Theta_{f}(\boldsymbol{q}, m)-\Theta(\boldsymbol{q})=\Theta_{f}(\boldsymbol{k}, m)-\Theta(\boldsymbol{k})$, 所以 $\Theta_{f}(\boldsymbol{q}, m)-\Theta(\boldsymbol{q})$ 应该是一个只与 $m$ 相关、跟 $\boldsymbol{q}$ 无关的函数, 记为 $\varphi(m)$, 即 $\Theta_{f}(\boldsymbol{q}, m)=\Theta(\boldsymbol{q})+\varphi(m)$ 。接着代入 $n=m-1$, 整理得到

$$
\varphi(m)-\varphi(m-1)=\Theta_{g}(\boldsymbol{q}, \boldsymbol{k}, 1)+\Theta(\boldsymbol{k})-\Theta(\boldsymbol{q})
$$

即 $\{\varphi(m)\}$ 是等差数列, 代入初始值$\varphi(0) = 0,  \varphi(1) = \theta $ , 解得 $\varphi(m)=m \theta$ 。

把前面所有的公式推导放在一起， 就可以得到 Rotery Position Embedding 的最终表达式：
$$
f(\mathbf{q}, m) = R_f(\mathbf{q}, m)e^{i\Theta_f(\mathbf{q}, m)}=\mathbf{q}e^{i(\Theta(\mathbf{q})+m\mathbf{\theta})} = \sum_{j=1}^{d/2} q_je^{im\theta_j} \vec{e_j}
$$
因此，对于任意的 $0 < \varepsilon \leq \frac \pi {2N}$， 其中$N$是最大序列长度。当按元素计算$q$ 和 $k$ 时， 以$j$作为元素索引，RoPE可以表示如下：
$$
\begin{align}
\mathrm{RoPE}(x, m) &= xe^{mi\varepsilon} \\
\langle \mathrm{RoPE}(q_j, m), \mathrm{RoPE}(k_j, n)\rangle &= \langle q_j e^{mi\varepsilon}, k_j e^{ni\varepsilon} \rangle \\
&= q_j k_j e^{mi\varepsilon} \overline{e^{ni\varepsilon}} \\
&= q_j k_j e^{(m - n)i\varepsilon} \\
&= \mathrm{RoPE}(q_j k_j, m - n)
\end{align}
$$
由于与复数相比，计算机更喜欢实数和矩阵，因此将此表达式转换为矩阵方程很方便。
$$
f(\mathbf{q}, m) =
\begin{pmatrix}
M_1 & & & \\
& M_2 & & \\
& & \ddots & \\
& & & M_{d/2}
\end{pmatrix}
\begin{pmatrix}
q_1\\
q_2\\
\vdots\\
q_d
\end{pmatrix} = \mathbf{\Theta_m Q_m} = \mathbf{\Theta_m W_q X_m}
$$
其中， $M_j=\begin{pmatrix}\cos m\theta_j & -\sin m\theta_j \\sin m\theta_j & \cos m\theta_j\end{pmatrix}$ ， $\mathbf{\Theta_m}$  为块对角矩阵， $\mathbf{\Theta_m}$为可学习的 query 权重。 $\mathbf{X_m}$  为 m 处的嵌入。

### 编码形式

综上, 我们得到二维情况下用复数表示的RoPE：

$$
\boldsymbol{f}(\boldsymbol{q}, m)=R_{f}(\boldsymbol{q}, m) e^{\mathrm{i} \Theta_{f}(\boldsymbol{q}, m)}=\|q\| e^{\mathrm{i}(\Theta(\boldsymbol{q})+m \theta)}=\boldsymbol{q} e^{\mathrm{i} m \theta}
$$

根据复数乘法的几何意义, 该变换实际上对应着向量的旋转, 所以我们称之为“旋转式位置编码”, 它还可以写成矩阵形式:

$$
\boldsymbol{f}(\boldsymbol{q}, m)=\left(\begin{array}{cc}
\cos m \theta & -\sin m \theta \\
\sin m \theta & \cos m \theta
\end{array}\right)\left(\begin{array}{l}
q_{0} \\
q_{1}
\end{array}\right)
$$

由于内积满足线性叠加性, 因此任意偶数维的RoPE, 我们都可以表示为二维情形的拼接, 即

$$
\underbrace{\left(\begin{array}{ccccccc}
\cos m \theta_{0} & -\sin m \theta_{0} & 0 & 0 & \cdots & 0 & 0 \\
\sin m \theta_{0} & \cos m \theta_{0} & 0 & 0 & \cdots & 0 & 0 \\
0 & 0 & \cos m \theta_{1} & -\sin m \theta_{1} & \cdots & 0 & 0 \\
0 & 0 & \sin m \theta_{1} & \cos m \theta_{1} & \cdots & 0 & 0 \\
\vdots & \vdots & \vdots & \vdots & \ddots & \vdots & \vdots \\
0 & 0 & 0 & 0 & \cdots & \cos m \theta_{d / 2-1} & -\sin m \theta_{d / 2-1} \\
0 & 0 & 0 & 0 & \cdots & \sin m \theta_{d / 2-1} & \cos m \theta_{d / 2-1}
\end{array}\right)}\left(\begin{array}{c}
q_{0} \\
q_{1} \\
q_{2} \\
q_{3} \\
\vdots \\
q_{d-2} \\
q_{d-1}
\end{array}\right)
$$

也就是说, 给位置为 $m$ 的向量 $q$ 乘上矩阵 $\boldsymbol{R}_{m}$ 、位置为 $n$ 的向量 $k$ 乘上矩阵 $\boldsymbol{R}_{n}$, 用变换后的 $Q, K$ 序列做Attention, 那么Atten tion就自动包含相对位置信息了, 因为成立恒等式:

$$
\left(\boldsymbol{\mathcal { R }}_{m} \boldsymbol{q}\right)^{\top}\left(\boldsymbol{\mathcal { R }}_{n} \boldsymbol{k}\right)=\boldsymbol{q}^{\top} \boldsymbol{\mathcal { R }}_{m}^{\top} \boldsymbol{\mathcal { R }}_{n} \boldsymbol{k}=\boldsymbol{q}^{\top} \boldsymbol{\mathcal { R }}_{n-m} \boldsymbol{k}
$$

值得指出的是, $\boldsymbol{R}_{m}$ 是一个正交矩阵, 它不会改变向量的模长, 因此通常来说它不会改变原模型的稳定性。
由于 $\boldsymbol{R}_{m}$ 的稀疏性，所以直接用矩阵乘法来实现会很浪费算力，推荐通过下述方式来实现RoPE：

$$
\left(\begin{array}{c}
q_{0} \\
q_{1} \\
q_{2} \\
q_{3} \\
\vdots \\
q_{d-2} \\
q_{d-1}
\end{array}\right) \otimes\left(\begin{array}{c}
\cos m \theta_{0} \\
\cos m \theta_{0} \\
\cos m \theta_{1} \\
\cos m \theta_{1} \\
\vdots \\
\cos m \theta_{d / 2-1} \\
\cos m \theta_{d / 2-1}
\end{array}\right)+\left(\begin{array}{c}
-q_{1} \\
q_{0} \\
-q_{3} \\
q_{2} \\
\vdots \\
-q_{d-1} \\
q_{d-2}
\end{array}\right) \otimes\left(\begin{array}{c}
\sin m \theta_{0} \\
\sin m \theta_{0} \\
\sin m \theta_{1} \\
\sin m \theta_{1} \\
\vdots \\
\sin m \theta_{d / 2-1} \\
\sin m \theta_{d / 2-1}
\end{array}\right)
$$

其中 $\otimes$ 是逐位对应相乘，即Numpy、Tensorflow等计算框架中的 $*$ 运算。从这个实现也可以看到，RoPE可以视为是言概括置编码的变体。

### LLama模型中的 RoPE

LLama模型使用了 Rotary Position Embedding， 对于 Q的第m个位置向量 q, 通过以下方式注入位置编码。

#### Step1:初始化 $\theta$ 矩阵

$$
\left(\begin{array}{c}
\theta_{0}  & \theta_{1}   &  \cdots & \theta_{d/2-1}   &
\theta_{0}  & \theta_{1}   &  \cdots & \theta_{d/2-1}   \\

\theta_{0}  & \theta_{1}   &  \cdots & \theta_{d/2-1}    &
\theta_{0}  & \theta_{1}   &  \cdots & \theta_{d/2-1}     \\

2\theta_{0} & 2\theta_{1}  &  \cdots & 2\theta_{d/2-1}   &
2\theta_{0} & 2\theta_{1}  &  \cdots & 2\theta_{d/2-1}   \\

 \vdots     & \vdots       &  \ddots &  \vdots            &
 \vdots     & \vdots       &  \ddots &  \vdots            \\

m\theta_{0} & m\theta_{1}  &  \cdots & m\theta_{d/2-1}   &
m\theta_{0} & m\theta_{1}  &  \cdots & m\theta_{d/2-1}   \\
\end{array}\right)
$$

#### Step2:计算 $cos$ 矩阵和 $sin$ 矩阵

$$
\left(\begin{array}{c}

\cos\theta_{0}  & \cos\theta_{1}   & \cdots & \cos\theta_{d/2-1}   &
\cos\theta_{0}  & \cos\theta_{1}   &  \cdots & \cos\theta_{d/2-1}   \\

\cos\theta_{0}  & \cos\theta_{1}   &  \cdots & \cos\theta_{d/2-1}   &
\cos\theta_{0}  & \cos\theta_{1}   &  \cdots & \cos\theta_{d/2-1}   \\

\cos2\theta_{0} & \cos2\theta_{1}  &  \cdots & \cos2\theta_{d/2-1}  &
\cos2\theta_{0} & \cos2\theta_{1}  &  \cdots & \cos2\theta_{d/2-1}  \\

 \vdots     & \vdots       &  \ddots &  \vdots     &  \vdots
 \vdots     & \vdots       &  \ddots &  \vdots                 \\

\cos m\theta_{0} & \cos  m\theta_{1}  &  \cdots &\cos  m\theta_{d/2-1}  &
\cos m\theta_{0} & \cos  m\theta_{1}  &  \cdots & \cos  m\theta_{d/2-1}   \\

\end{array}\right)
$$

#### Step3:计算 Qury 向量

```python
q_embed = (q * cos) + (rotate_half(q) * sin)
k_embed = (k * cos) + (rotate_half(k) * sin)
```

$$
\left(\begin{array}{c}
q_{0} \\
q_{1} \\
\vdots \\
q_{d / 2-1} \\
q_{d /2} \\
\vdots \\
q_{d-2}\\
q_{d-1}
\end{array}\right) \otimes

\left(\begin{array}{c}
\cos m \theta_{0} \\
\cos m \theta_{1} \\
\vdots \\
\cos m \theta_{d / 2-1} \\
\cos m \theta_{0} \\
\cos m \theta_{1} \\
\vdots \\
\cos m \theta_{d / 2-1} \\
\end{array}\right)

+\left(\begin{array}{c}
-q_{d/2} \\
-q_{d/2+1} \\
\vdots \\
-q_{d-1} \\
q_{0} \\
q_{1} \\
\vdots \\
q_{d/2-1}
\end{array}\right) \otimes\left(\begin{array}{c}
\sin m \theta_{0} \\
\sin m \theta_{1} \\
\vdots \\
\sin m \theta_{d / 2-1} \\
\sin m \theta_{0} \\
\sin m \theta_{1} \\
\vdots \\
\sin m \theta_{d / 2-1}
\end{array}\right)
$$



## RoPE证明过程

#### 简单证明

简单起见, 我们先假设 $\boldsymbol{q}_{m}, \boldsymbol{k}_{n}$ 是所在位置分别为 $m, n$ 的二维行向量, 既然是二维, 那么我们可以将它当作复数来运算。我们知道, Attention关键之处在于向量的内积, 用复数表示为

$$
\left\langle\boldsymbol{q}_{m}, \boldsymbol{k}_{n}\right\rangle=\operatorname{Re}\left[\boldsymbol{q}_{m} \boldsymbol{k}_{n}^{*}\right]
$$

其中* 是共轭复数, 右端的乘法是普通的复数乘法, $\operatorname{Re}[]$ 表示取结果的实部。上式也就是说：如果我们将 $\boldsymbol{q}_{m}, \boldsymbol{k}_{n}$ 分别乘以 $e^{\mathrm{i} i \theta}, e^{\mathrm{i} n \theta}$ 变成 $\boldsymbol{q}_{m} e^{\mathrm{i} i n}, \boldsymbol{k}_{n} e^{\mathrm{i} n \theta}$, 那么就相当于给它们配上了绝对位置编码了（因为显式地依赖绝对位置 $m, n)$, 然后放到内积里边, 我们有

$$
\left\langle\boldsymbol{q}_{m} e^{\mathrm{i} i \theta}, \boldsymbol{k}_{n} e^{\mathrm{i} n \theta}\right\rangle=\operatorname{Re}\left[\left(\boldsymbol{q}_{m} e^{\mathrm{i} m \theta}\right)\left(\boldsymbol{k}_{n} e^{\mathrm{i} n \theta}\right)^{*}\right]=\operatorname{Re}\left[\boldsymbol{q}_{m} \boldsymbol{k}_{n}^{*} e^{\mathrm{i}(m-n) \theta}\right]
$$

相当有意思的是，内积只依赖于相对位置 $m-n$ ! 这就巧妙地将绝对位置与相对位置融合在一起了。

由上述结果可知, 对于位置为 $n$ 的二维实数向量 $[x, y]$, 我们当它复数来运算, 乘以 $e^{\mathrm{i} n \theta}$, 得到恒等式
$$
(x+y \mathrm{i}) e^{\mathrm{i} n \theta}=(x \cos n \theta-y \sin n \theta)+\mathrm{i}(x \sin n \theta+y \cos n \theta)
$$

这也就是意味着，通过

$$
\left(\begin{array}{l}
x \\
y
\end{array}\right) \rightarrow\left(\begin{array}{c}
x \cos n \theta-y \sin n \theta \\
x \sin n \theta+y \cos n \theta
\end{array}\right)=\left(\begin{array}{l}
x \\
y
\end{array}\right) \cos n \theta+\left(\begin{array}{c}
-y \\
x
\end{array}\right) \sin n \theta
$$

来赋予 $[x, y]$ 绝对位置信息, 那么在Attention运算的时候也等价于相对位置编码。如果是多于二维的向量, 可以考虑每两维为一组进行同样的运算, 每一组的 $\theta$ 可以不一样。

这样一来, 我们得到了一种融绝对位置与相对位置于一体的位置编码方案, 从形式上看它有点像乘性的绝对位置编码, 通过在 $q, k$ 中施行该位置编码, 那么效果就等价于相对位置编码, 而如果还需要显式的绝对位置信息, 则可以同时在 $v$ 上施行这种位置编码。

#### 完整证明

假定 query 向量 $q_m$ 和 key 向量 $k_n$ 之间的内积操作可以被一个函数 $g$ 表示，该函数 $g$ 的输入是词嵌入向量 $x_m$ ， $x_n$ 和它们之间的相对位置 $m-n$：
$$
<f_q(x_m,m),f_k(x_n,n)>=g(x_m,x_n,m-n) \\
$$
我们的目标就是找到一个等价的位置编码方式，从而使得上述关系成立。即构造出函数$f$ 和 $g$, 使得上述等式成立。

假定现在词嵌入向量的维度是两维 `d=2`，这样就可以利用上2维度平面上的向量的几何性质，然后论文中提出了一个满足上述关系的 $f$ 和 $g$ 的形式如下：
$$
f_q(x_m,m)=(W_qx_m)e^{im\theta} \\ f_k(x_n,n)=(W_kx_n)e^{in\theta} \\ g(x_m,x_n,m-n)=Re[(W_qx_m)(W_kx_n)^{*}e^{i(m-n)\theta}] \\
$$
这里面 Re 表示复数的实部。

首先看到上述 $f$ 和 $g$ 公式中有个指数函数： $$e^{ix} $$

这个其实是欧拉公式 `[2]`，其中 $x$ 表示任意实数， $e$ 是自然对数的底数，$i$ 是复数中的虚数单位，则根据欧拉公式有：
$$
e^{ix} = \cos x + i\sin x \\
$$
则上述指数函数可以表示为实部为 $cosx$，虚部为 $sinx$ 的一个复数，欧拉公式 `[2] `建立了指数函数、三角函数和复数之间的桥梁。

则上述 $f$ 和 $g$ 公式中的
$$
e^{im\theta}=\cos (m\theta) + i\sin (m\theta) \\ e^{in\theta}=\cos (n\theta) + i\sin (n\theta) \\ e^{i(m-n)\theta}=\cos ((m-n)\theta) + i\sin ((m-n)\theta) \\
$$
然后我们看回公式：
$$
f_q(x_m,m)=(W_qx_m)e^{im\theta} \\
$$
其中 `Wq` 是个二维矩阵，$x_m$ 是个二维向量，相乘的结果也是一个二维向量，这里用 $q_m$ 表示：
$$
q_m= \begin{pmatrix} q_m^{(1)} \\ q_m^{(2)} \end{pmatrix} = W_qx_m =\begin{pmatrix} W_q^{(11)} & W_q^{(12)} \\ W_q^{(21)} & W_q^{(22)} \end{pmatrix} \begin{pmatrix} x_m^{(1)} \\ x_m^{(2)} \end{pmatrix} \\
$$
然后首先将 $q_m$ 表示成复数形式：
$$
q_m = [q_m^{(1)}, q_m^{(2)}] = [q_m^{(1)} + iq_m^{(2)}] \\
$$
接着
$$
f_q(x_m,m)=(W_qx_m)e^{im\theta}=q_me^{im\theta} \\
$$
其实就是两个复数相乘：
$$
q_me^{im\theta}=(q_m^{(1)} + iq_m^{(2)}) * (\cos (m\theta) + i\sin (m\theta)) \\
$$
我们首先来复习一下复数乘法的性质：
$$
(a+ib) \cdot (c+id) = ac + ibc + iad + i^2bd=(ac-bd)+i(bc+ad) \\
$$
可以看到，复数乘法也是用的分配律，还有用到了复数的一个性质： $i^2=-1$, 然后就有：

$$
q_me^{im\theta}=(q_m^{(1)} + iq_m^{(2)}) * (\cos (m\theta) + i\sin (m\theta)) \\ =(q_m^{(1)}cos (m\theta) - q_m^{(2)} \sin (m\theta) ) + i(q_m^{(2)}\cos (m\theta) + q_m^{(1)}\sin (m\theta)) \\
$$
将结果重新表达成实数向量形式就是：
$$
q_me^{im\theta}=[q_m^{(1)} \cos (m\theta) - q_m^{(2)} \sin (m\theta), q_m^{(2)}\cos (m\theta) + q_m^{(1)}\sin (m\theta)] \\
$$

$$
f_q(x_m,m)=(W_qx_m)e^{im\theta}=q_me^{im\theta}\\ =[q_m^{(1)} \cos (m\theta) - q_m^{(2)} \sin (m\theta), q_m^{(2)}\cos (m\theta) + q_m^{(1)}\sin (m\theta)] \\ = \begin{pmatrix} \cos (m\theta) & -\sin (m\theta) \\ \sin (m\theta) & \cos (m\theta) \end{pmatrix} \begin{pmatrix} q_m^{(1)} \\ q_m^{(2)} \end{pmatrix} \\
$$

看到这里会发现，这不就是 query 向量乘以了一个旋转矩阵吗？这就是为什么叫做旋转位置编码的原因。

同理，$f_k$  可以表示成下面的式子：
$$
\begin{align} f_k\left( {x}_m,m \right) &= \begin{pmatrix} \cos m\theta & -\sin m\theta) \\ \sin m \theta & \cos m \theta \end{pmatrix} \begin{pmatrix} W^{(1,1)}_{k} & W^{(1,2)}_{k} \\ W^{(2,1)}_{k} & W^{(2,2)}_{k} \end{pmatrix} \begin{pmatrix} x_m^{(1)} \\ x_m^{(2)} \end{pmatrix} \\ &= \begin{pmatrix} \cos m\theta & -\sin m\theta) \\ \sin m \theta & \cos m \theta \end{pmatrix}\begin{pmatrix} k_m^{(1)} \\ k_m^{(2)} \end{pmatrix} \end{align}
$$
同理可得 key 向量 $k_n$ ：
$$
f_k(x_n,n)=(W_kx_n)e^{in\theta}=k_ne^{in\theta}\\ =[k_n^{(1)} \cos (n\theta) - k_n^{(2)} \sin (n\theta), k_n^{(2)}\cos (n\theta) + k_n^{(1)}\sin (n\theta)] \\ = \begin{pmatrix} \cos (n\theta) & -\sin (n\theta) \\ \sin (n\theta) & \cos (n\theta) \end{pmatrix} \begin{pmatrix} k_n^{(1)} \\ k_n^{(2)} \end{pmatrix} \\
$$
最后还有个函数 $g$：
$$
g(x_m,x_n,m-n)=Re[(W_qx_m)(W_kx_n)^{*}e^{i(m-n)\theta}] \\
$$
其中 `Re[x]` 表示一个复数 $x$ 的实部部分，而 $(W_kx_n)^{*} $则表示复数 $W_kx_n$ 的共轭。

复习一下共轭复数的定义：
$$
z=a+ib\\ z^*=a-ib \\
$$
所以可得：
$$
W_qx_m = q_m = q_m^{(1)} + iq_m^{(2)} \\ W_kx_n=k_n= k_n^{(1)} + ik_n^{(2)} \\ (W_kx_n)^*=k_n^*= k_n^{(1)} - ik_n^{(2)} \\ e^{i(m-n)\theta}=\cos((m-n)\theta) + i \sin((m-n)\theta) \\
$$
继续可得：
$$
g(x_m,x_n,m-n)=Re[(W_qx_m)(W_kx_n)^{*}e^{i(m-n)\theta}] \\ = Re[(q_m^{(1)} + iq_m^{(2)})(k_n^{(1)} - ik_n^{(2)})(\cos((m-n)\theta) + i \sin((m-n)\theta))] \\ = Re[((q_m^{(1)}k_n^{(1)} + q_m^{(2)}k_n^{(2)}) + i(q_m^{(2)}k_n^{(1)} - q_m^{(1)}k_n^{(2)}))(\cos((m-n)\theta) + i \sin((m-n)\theta))] \\ = (q_m^{(1)}k_n^{(1)} + q_m^{(2)}k_n^{(2)})\cos((m-n)\theta) - (q_m^{(2)}k_n^{(1)} - q_m^{(1)}k_n^{(2)})\sin((m-n)\theta) \\
$$
接下来我们就要证明函数 $g$ 的计算公式是成立的。

首先回顾一下 attention 操作， 位置 m 的 query 和位置 n 的 key 会做一个内积操作：
$$
f_q(x_m,m)=[q_m^{(1)} \cos (m\theta) - q_m^{(2)} \sin (m\theta), q_m^{(2)}\cos (m\theta) + q_m^{(1)}\sin (m\theta)] \\ f_k(x_n,n) =[k_n^{(1)} \cos (n\theta) - k_n^{(2)} \sin (n\theta), k_n^{(2)}\cos (n\theta) + k_n^{(1)}\sin (n\theta)] \\ <f_q(x_m,m),f_k(x_n,n)> = \\ (q_m^{(1)} \cos (m\theta) - q_m^{(2)} \sin (m\theta))(k_n^{(1)} \cos (n\theta) - k_n^{(2)} \sin (n\theta)) \\+ (q_m^{(2)}\cos (m\theta) + q_m^{(1)}\sin (m\theta))(k_n^{(2)}\cos (n\theta) + k_n^{(1)}\sin (n\theta))\\ =q_m^{(1)} \cos (m\theta) k_n^{(1)} \cos (n\theta) - q_m^{(1)} \cos (m\theta)k_n^{(2)} \sin (n\theta)\\ - q_m^{(2)} \sin (m\theta)k_n^{(1)} \cos (n\theta) + q_m^{(2)} \sin (m\theta)k_n^{(2)} \sin (n\theta) \\ + q_m^{(2)}\cos (m\theta)k_n^{(2)}\cos (n\theta) + q_m^{(2)}\cos (m\theta)k_n^{(1)}\sin (n\theta) \\ + q_m^{(1)}\sin (m\theta)k_n^{(2)}\cos (n\theta) + q_m^{(1)}\sin (m\theta)k_n^{(1)}\sin (n\theta) \\
$$
接着继续之前先复习一下三角函数的一些性质`[3]`：
$$
\sin(a+b) = \sin a \cos b + \cos a \sin b \\ \sin(a-b) = \sin a \cos b - \cos a \sin b \\ \cos(a+b) = \cos a \cos b - \sin a \sin b \\ \cos(a-b) = \cos a \cos b + \sin a \sin b \\
$$
好了回到上面那堆式子，我们整理一下：
$$
<f_q(x_m,m),f_k(x_n,n)> = \\ q_m^{(1)}k_n^{(1)}(\cos(m\theta)\cos(n\theta) + \sin(m\theta)\sin(n\theta) ) \\ + q_m^{(1)}k_n^{(2)}(-\cos(m\theta)\sin(n\theta) + \sin(m\theta)\cos(n\theta) ) \\ + q_m^{(2)}k_n^{(1)}(-\sin(m\theta)\cos(n\theta) + \cos(m\theta)\sin(n\theta) ) \\ + q_m^{(2)}k_n^{(2)}(\sin(m\theta)\sin(n\theta) + \cos(m\theta)\cos(n\theta) ) \\ = q_m^{(1)}k_n^{(1)}\cos((m-n)\theta) \\ + q_m^{(1)}k_n^{(2)}\sin((m-n)\theta) \\ - q_m^{(2)}k_n^{(1)}\sin((m-n)\theta) \\ + q_m^{(2)}k_n^{(2)}\cos((m-n)\theta) \\ = (q_m^{(1)}k_n^{(1)} + q_m^{(2)}k_n^{(2)})\cos((m-n)\theta) + (q_m^{(1)}k_n^{(2)}- q_m^{(2)}k_n^{(1)})\sin((m-n)\theta) \\ = (q_m^{(1)}k_n^{(1)} + q_m^{(2)}k_n^{(2)})\cos((m-n)\theta) - (q_m^{(2)}k_n^{(1)} - q_m^{(1)}k_n^{(2)})\sin((m-n)\theta) \\ =g(x_m,x_n,m-n) \\
$$
这就证明上述关系是成立的，位置 m 的 query 和位置 n 的 key 的内积就是函数 $g$。

把上面的式子用矩阵向量乘的形式来表达就是：
$$
<f_q(x_m,m),f_k(x_n,n)> \\ =\begin{pmatrix} \begin{pmatrix} \cos (m\theta) & -\sin (m\theta) \\ \sin (m\theta) & \cos (m\theta) \end{pmatrix} \begin{pmatrix} q_m^{(1)} \\ q_m^{(2)} \end{pmatrix} \end{pmatrix}^T \begin{pmatrix} \begin{pmatrix} \cos (n\theta) & -\sin (n\theta) \\ \sin (n\theta) & \cos (n\theta) \end{pmatrix} \begin{pmatrix} k_n^{(1)} \\ k_n^{(2)} \end{pmatrix} \end{pmatrix} \\ = \begin{pmatrix} q_m^{(1)} & q_m^{(2)} \\ \end{pmatrix} \begin{pmatrix} \cos (m\theta) & \sin (m\theta) \\ -\sin (m\theta) & \cos (m\theta) \end{pmatrix} \begin{pmatrix} \cos (n\theta) & -\sin (n\theta) \\ \sin (n\theta) & \cos (n\theta) \end{pmatrix} \begin{pmatrix} k_n^{(1)} \\ k_n^{(2)} \end{pmatrix} \\ = \begin{pmatrix} q_m^{(1)} & q_m^{(2)} \\ \end{pmatrix} \begin{pmatrix} \cos(m\theta)\cos(n\theta) + \sin(m\theta)\sin(n\theta) & -\cos(m\theta)\sin(n\theta) + \sin(m\theta)\cos(n\theta) \\ -\sin(m\theta)\cos(n\theta) + \cos(m\theta)\sin(n\theta) & \sin(m\theta)\sin(n\theta) + \cos(m\theta)\cos(n\theta) \end{pmatrix} \begin{pmatrix} k_n^{(1)} \\ k_n^{(2)} \end{pmatrix} \\ =\begin{pmatrix} q_m^{(1)} & q_m^{(2)} \\ \end{pmatrix} \begin{pmatrix} \cos((m-n)\theta) & -\sin((m-n)\theta) \\ \sin((m-n)\theta) & \cos((m-n)\theta) \end{pmatrix} \begin{pmatrix} k_n^{(1)} \\ k_n^{(2)} \end{pmatrix} \\
$$
然后上面的讲解是假定的词嵌入维度是2维向量，而对于`d >= 2` 的通用情况，则是将词嵌入向量元素按照两两一组分组，每组应用同样的旋转操作且每组的旋转角度计算方式如下：
$$
\theta_j=10000^{-2(j-1)/d}, j \in [1,2,...,d/2] \\
$$
所以简单来说 RoPE 的 self-attention 操作的流程是，对于 token 序列中的每个词嵌入向量，首先计算其对应的 query 和 key 向量，然后对每个 token 位置都计算对应的旋转位置编码，接着对每个 token 位置的 query 和 key 向量的元素按照 两两一组 应用旋转变换，最后再计算 query 和 key 之间的内积得到 self-attention 的计算结果。

## RoPE 的性质

### 远程衰减

可以看到, RoPE形式上和Sinusoidal位置编码有点相似, 只不过Sinusoidal位置编码是加性的, 而RoPE可以视为乘性的。在 $\theta_{i}$ 的选择上, 我们同样沿用了Sinusoidal位置编码的方案, 即 $\theta_{i}=10000^{-2 i / d}$, 它可以带来一定的远程衰减性。

具体证明如下：将 $q, k$ 两两分组后, 它们加上RoPE后的内积可以用复数乘法表示为

$$
\left(\boldsymbol{\mathcal { R }}_{m} \boldsymbol{q}\right)^{\top}\left(\boldsymbol{\mathcal { R }}_{n} \boldsymbol{k}\right)=\operatorname{Re}\left[\sum_{i=0}^{d / 2-1} \boldsymbol{q}_{[2 i: 2 i+1]} \boldsymbol{k}_{[2 i: 2 i+1]}^{*} e^{\mathrm{i}(m-n) \theta_{i}}\right]
$$

记 $h_{i}=\boldsymbol{q}_{[2 i: 2 i+1]} \boldsymbol{k}_{[2 i: 2 i+1]}^{*}, S_{j}=\sum_{i=0}^{j-1} e^{\mathrm{i}(m-n) \theta_{i}}$, 并约定 $h_{d / 2}=0, S_{0}=0$, 那么由Abel变换（分部求和法）可以得到:

$$
\sum_{i=0}^{d / 2-1} \boldsymbol{q}_{[2 i: 2 i+1]} \boldsymbol{k}_{[2 i: 2 i+1]}^{*} e^{\mathrm{i}(m-n) \theta_{i}}=\sum_{i=0}^{d / 2-1} h_{i}\left(S_{i+1}-S_{i}\right)=-\sum_{i=0}^{d / 2-1} S_{i+1}\left(h_{i+1}-h_{i}\right)
$$

所以

$$
\begin{aligned}
\left|\sum_{i=0}^{d / 2-1} \boldsymbol{q}_{[2 i: 2 i+1]} \boldsymbol{k}_{[2 i: 2 i+1]}^{*} e^{\mathrm{i}(m-n) \theta_{i}}\right| & =\left|\sum_{i=0}^{d / 2-1} S_{i+1}\left(h_{i+1}-h_{i}\right)\right| \\
& \leq \sum_{i=0}^{d / 2-1}\left|S_{i+1}\right|\left|h_{i+1}-h_{i}\right| \\
& \leq\left(\max _{i}\left|h_{i+1}-h_{i}\right|\right) \sum_{i=0}^{d / 2-1}\left|S_{i+1}\right|
\end{aligned}
$$

因此我们可以考察 $\frac{1}{d / 2} \sum_{i=1}^{d / 2}\left|S_{i}\right|$ 随着相对距离的变化情况来作为衰减性的体现, 我们可以可以看到随着相对距离的变大, 内积结果有衰减趋势的出现。因此, 选择 $\theta_{i}=10000^{-2 i / d}$, 确实能带来一定的远程衰减性。

### 线性场景

最后, 我们指出, RoPE是目前唯一一种可以用于线性Attention的相对位置编码。这是因为其他的相对位置编码, 都是直接基于Attention矩阵进行操作的, 但是线性Attention并没有事先算出Attention矩阵, 因此也就不存在操作Attention矩阵的做法, 所以其他的方案无法应用到线性Attention中。而对于RoPE来说, 它是用绝对位置编码的方式来实现相对位置编码, 不需要操作Attention矩阵, 因此有了应用到线性Attention的可能性。

线性Attention的常见形式是:

$$
\operatorname{Attention}(\boldsymbol{Q}, \boldsymbol{K}, \boldsymbol{V})_{i}=\frac{\sum_{j=1}^{n} \operatorname{sim}\left(\boldsymbol{q}_{i}, \boldsymbol{k}_{j}\right) \boldsymbol{v}_{j}}{\sum_{j=1}^{n} \operatorname{sim}\left(\boldsymbol{q}_{i}, \boldsymbol{k}_{j}\right)}=\frac{\sum_{j=1}^{n} \phi\left(\boldsymbol{q}_{i}\right)^{\top} \varphi\left(\boldsymbol{k}_{j}\right) \boldsymbol{v}_{j}}{\sum_{j=1}^{n} \phi\left(\boldsymbol{q}_{i}\right)^{\top} \varphi\left(\boldsymbol{k}_{j}\right)}
$$

其中 $\phi, \varphi$ 是值域非负的激活函数。可以看到, 线性Attention也是基于内积的, 所以很自然的想法是可以将RoPE插入到内积中:

$$
\frac{\sum_{j=1}^{n}\left[\boldsymbol{\mathcal { R }}_{i} \phi\left(\boldsymbol{q}_{i}\right)\right]^{\top}\left[\boldsymbol{\mathcal { R }}_{j} \varphi\left(\boldsymbol{k}_{j}\right)\right] \boldsymbol{v}_{j}}{\sum_{j=1}^{n}\left[\boldsymbol{\mathcal { R }}_{i} \phi\left(\boldsymbol{q}_{i}\right)\right]^{\top}\left[\boldsymbol{\mathcal { R }}_{j} \varphi\left(\boldsymbol{k}_{j}\right)\right]}
$$

但这样存在的问题是, 内积 $\left[\boldsymbol{R}_{i} \phi\left(\boldsymbol{q}_{i}\right)\right]^{\top}\left[\boldsymbol{R}_{j} \varphi\left(\boldsymbol{k}_{j}\right)\right]$ 可能为负数, 因此它不再是常规的概率注意力, 而且分母有为 $\mathrm{o}$ 的风险,可能会带来优化上的不稳定。考虑到 $\boldsymbol{R}_{i}, \boldsymbol{R}_{j}$ 都是正交矩阵, 它不改变向量的模长, 因此我们可以抛弃常规的概率归一化要求, 使用如下运算作为一种新的线性Attention:

$$
\frac{\sum_{j=1}^{n}\left[\boldsymbol{\mathcal { R }}_{i} \phi\left(\boldsymbol{q}_{i}\right)\right]^{\top}\left[\boldsymbol{\mathcal { R }}_{j} \varphi\left(\boldsymbol{k}_{j}\right)\right] \boldsymbol{v}_{j}}{\sum_{j=1}^{n} \phi\left(\boldsymbol{q}_{i}\right)^{\top} \varphi\left(\boldsymbol{k}_{j}\right)}
$$

也就是说，RoPE只插入分子中，而分母则不改变，这样的注意力不再是基于概率的（注意力矩阵不再满足非负归一性）,但它某种意义上来说也是一个归一化方案，而且也没有证据表明非概率式的注意力就不好（比如Nyströmformer也算是没有严格依据概率分布的方式构建注意力）, 所以我们将它作为候选方案之一进行实验, 而我们初步的实验结果显示这样的线性Attention也是有效的。

### RoPE的长度扩展

在LLM的应用中，有一个非常重要的参数，叫做LLM支持的上下文长度(max context length)。更长的上下文长度允许我们进行更多轮次的对话，允许我们对更长的本文进行总结分析，也允许我们生成更长的文章。但是在训练LLM的时候，我们的训练语料大部分是不够长的，许多LLM训练时候设计的最大文本长度都是只有2k，也就是最长2048个token。那么，能否在训练的时候使用较短的文本，而在推理的时候扩展到长文本上呢？

是有可能的，我们可以对RoPE进行长度扩展。下面我们介绍3种扩展方案。

- 第一种是直接外推

直接外推其实就是继续沿用现有的位置编码公式，不做任何修改。在扩展长度不太长的时候，例如由2k扩展到2.5k时，这种方法可能对性能的影响并不大。因为旋转位置编码只和相对位置m-n的大小有关，一般具有远程衰减性，即相对距离越大的两个token，其相关性一般越弱。

因此如果我们的模型已经从训练数据那里学习到了token之间的相关性相对于相对距离在0-2k的一个合适的衰减规律的时候，可以设想把这个规律应用到0-2.5k也是没有太大的问题的。但是如果我们要扩展到更长的长度，例如从2k扩展到32k，这种直接外推的方案通常会严重地影响性能。因为我们学习到的衰减规律有可能在5k的那里就完全衰减截断基本降为0了，这样我们就无法捕捉相对距离长于5k的两个token之间的相互作用，外推就会导致性能下降。

总结一下，直接外推对衰减规律在长距离情况下的使用容易出现问题，导致性能下降。为了减少长度外推对性能的影响，我们可以让训练好的模型在更长的上下文上做少许步骤的微调。

- 第二种是线性内插

线性内插需要改变位置编码公式，等效于将位置序号等比例缩小。

编码公式变化如 ，当从2k扩展到32k，等效于需要将位置序号变成原来的1/16.线性内插没有改变模型学习到的衰减规律的应用范围，不考虑微调的话，其效果一般好于直接外推方案。但是，扩展倍数非常大的时候，例如从2k扩展到32k，其性能也会明显的受到影响。因为在这种情况下，衰减规律在短距离情况下的使用会受到较严重的影响，本来距离为1的两个token，长度扩展后相当于变成了距离为1/16，衰减规律在短距离时可能具有非常大的变化率，因此对相关性的评估可能会极端地偏离合理值。

应用线性内插时，在长文本上做少许步骤的微调也能够明显地改善性能。

- 第三种是NTK扩展方式

这种方式综合了外推和内插的优点，做长度扩展后即使不微调也能够保持较好的性能。

前面的分析我们知道直接外推对衰减规律在长距离情况下的使用容易出问题，在短距离情况下的使用不受影响。而线性内插对衰减规律在短距离情况下的使用容易出现问题，在长距离的情况下影响较小。我们能否将它们综合起来，在短距离情况下具有外推特性(与扩展前基本一致)，在长距离情况下具有内插特性(缩放到扩展前的范围)，从而使得长距离情况下和短距离情况下衰减规律的使用都不太受到影响呢。

我们观察RoPE位置编码第行的元素计算公式 ，可以发现越大，三角函数对应的角频率系数越小，或者说越低频，对应的三角函数变化越慢。容易得到如下直观结论：短距离之间的差异(例如1和5的差异)，主要体现在高频分量(i比较小)上，长距离之间的差异(例如5000和10000的差异)，主要体现在低频分量(i比较大)上。

为了在短距离情况下具有外推特性，而在长距离情况下具有内插特性，我们可以设计一个和有关的位置序号缩放因子，使得在最高频()时取值为1(与扩展前基本一致)，而在最低频时()恰好为缩放倍数的倒数(缩放到扩展前的范围)。一种有效的选择方案是的指数函数，其效果相当于对中的做一个缩放。NTK扩展方式的要点是高频外推，低频内插，实现方法是直接对底数base进行缩放，类似进制编码转换。采用NTK扩展到长文本，即使不做微调，性能会只会略有下降。

## 代码实现

旋转位置嵌入的简单实现将使用前面所示的块对角矩阵形式。在实践中，以这种方式实现旋转位置嵌入的效率非常低，而且更优化的形式很容易获得。RoPE 的原始实现可在[roformer](https://github.com/ZhuiyiTechnology/roformer)和[bert4keras中找到](https://github.com/bojone/bert4keras)中找到。

此外，我们还在[x-transformers](https://github.com/lucidrains/x-transformers)、[GPT-Neo](https://github.com/EleutherAI/gpt-neo)、[GPT-NeoX](https://github.com/EleutherAI/gpt-neox)和[Mesh Transformer JAX](https://github.com/kingoflolz/mesh-transformer-jax)中实现了旋转位置嵌入。以下是从这些代码库中提取的 PyTorch 实现。

```python
import torch

class Rotary(torch.nn.Module):
    def __init__(self, dim, base=10000):
        super().__init__()
        inv_freq = 1.0 / (base ** (torch.arange(0, dim, 2).float() / dim))
        self.register_buffer("inv_freq", inv_freq)
        self.seq_len_cached = None
        self.cos_cached = None
        self.sin_cached = None

    def forward(self, x, seq_dim=1):
        seq_len = x.shape[seq_dim]
        if seq_len != self.seq_len_cached:
            self.seq_len_cached = seq_len
            t = torch.arange(x.shape[seq_dim], device=x.device).type_as(self.inv_freq)
            freqs = torch.einsum("i,j->ij", t, self.inv_freq)
            emb = torch.cat((freqs, freqs), dim=-1).to(x.device)
            self.cos_cached = emb.cos()[:, None, None, :]
            self.sin_cached = emb.sin()[:, None, None, :]
        return self.cos_cached, self.sin_cached


# rotary pos emb helpers:

def rotate_half(x):
    x1, x2 = x[..., : x.shape[-1] // 2], x[..., x.shape[-1] // 2 :]
    return torch.cat(
        (-x2, x1), dim=x1.ndim - 1
    )  # dim=-1 triggers a bug in torch < 1.8.0


@torch.jit.script
def apply_rotary_pos_emb(q, k, cos, sin):
    return (q * cos) + (rotate_half(q) * sin), (k * cos) + (rotate_half(k) * sin)
```



## 总结

从理论上来看，RoPE与Sinusoidal位置编码有些相通之处，但RoPE不依赖于泰勒展开，更具严谨性与可解释性；从预训练模型RoFormer的结果来看，RoPE具有良好的外推性，应用到Transformer中体现出较好的处理长文本的能力。此外，RoPE还是目前唯一一种可用于线性Attention的相对位置编码。

## Reference

[1] [https://arxiv.org/pdf/2104.09864.pdf](https://arxiv.org/pdf/2104.09864.pdf)

[2] [https://en.wikipedia.org/wiki/Euler's_formula](https://en.wikipedia.org/wiki/Euler's_formula)

[3] [https://en.wikipedia.org/wiki/List_of_trigonometric_identities](https://en.wikipedia.org/wiki/List_of_trigonometric_identities)

[4] [https://github.com/facebookresearch/llama/tree/main](https://github.com/facebookresearch/llama/tree/main)

[5] [https://zh.wikipedia.org/wiki/旋转矩阵](https://zh.wikipedia.org/wiki/%E6%97%8B%E8%BD%AC%E7%9F%A9%E9%98%B5)

[6] Jianlin Su. 让研究人员绞尽脑汁的 Transformer 位置编码. https://kexue.fm/archives/8130, 2021. [Online; accessed 18-April-2021].

[7] Jianlin Su. Transformer 升级之路：2、博采众长的旋转式位置编码. https://kexue.fm/archives/8265, 2021. [Online; accessed 18-April-2021].

[8] Jianlin Su, Yu Lu, Shengfeng Pan, Bo Wen, and Yunfeng Liu. RoFormer: Enhanced Transformer with Rotary Position Embedding. *arXiv preprint [arXiv:2104.09864](https://arxiv.org/abs/2104.09864)*, 2021.
