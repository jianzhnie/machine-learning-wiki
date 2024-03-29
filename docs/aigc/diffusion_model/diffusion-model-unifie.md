# 理解扩散模型：一个统一的视角

扩散模型作为生成模型显示出令人难以置信的能力[1, 2, 3]; 事实上，它们是为目前最先进的文本条件图像生成模型，例如 Imagen[4]和 DALL-E 2[5]. 在本教程中，我们从 variational 和 score-based 的角度回顾、揭开和统一对扩散模型的理解。

本教程背后的最初灵感来自于我在暑期实习期间与 Google Research 的大佬们进行的多次讨论。Josh Dillon 关于扩散模型的 ELBO 推导的精彩注释帮助塑造了我对扩散模型的变分视角的理解；经过调整，它们形成了下面列出的许多证明。我还要感谢 Ben Poole 围绕 Tweedie 公式进行的许多富有启发性的讨论，以及 Durk Kingma 对 VDM、VAE 和 HVAE 及其联系的有益见解。此外，我从 Yang Song 的许多关于 score-based diffusion 的工作和他的[博客](https://yang-song.github.io/blog/)中学到了很多东西，其中精简的部分包括 score-based generative models 部分。Sander Dieleman 的深度[博文](https://benanne.github.io/2022/05/26/guidance.html)指南上写得很清楚；指导部分提供了对其的删节。Lilian Weng 的[博客](https://lilianweng.github.io/posts/2021-07-11-diffusion-models/)也专业地展示了这些材料，并对她的逐步推导进行了修改，直接激发并形成了以下证明。我强烈鼓励人们阅读他们的工作；最后提供了其他有用的相关资源的非详尽列表。我还要感谢许多帮助审查、编辑和评论本教程草稿的人 - 非常感谢 Yang Song、Durk Kingma、Ben Poole、Jonathan Ho、Yiding Jiang、Ting Chen、Jeremy Cohen, and Chen Sun!

## 介绍

给定来自兴趣分布观察样本 $\boldsymbol{x}$ ，生成模型的目标是学习对其真实数据的分布 $p(\boldsymbol{x})$ 进行建模.   一旦完成学习，我们就可以从我们的近似模型中随意生成新的样本。此外，在某些公式下，我们还能够使用学习模型来评估观察或采样数据的可能性。

当前文献中有几个众所周知的方向，我们将仅在较高层次上进行简要介绍。生成对抗网络 (GAN) 模拟以对抗方式学习的复杂分布的采样过程。另一类称为“基于似然”的生成模型试图学习一种模型，该模型将高可能性分配给观察到的数据样本。包括自回归模型、归一化流和变分自编码器 (VAE)。另一种类似的方法是基于能量的建模，其中将分布学习为任意灵活的能量函数，然后对其进行归一化。基于分数的生成模型高度相关；他们不是学习对能量函数本身建模，而是学习基于能量的模型分数作为神经网络。在这项工作中，我们探索和审查扩散模型，正如我们将展示的那样，这些模型具有基于可能性和基于分数的解释。我们以极其详细的方式展示了这些模型背后的数学原理，目的是让任何人都可以跟进并理解扩散模型是什么以及它们是如何工作的。

## 背景

对于许多模态，我们可以将观察到的数据视为由关联的看不见的隐变量表示或生成，我们可以用随机变量$z$表示. 直觉上，潜变量背后的思想可以通过柏拉图的[洞穴寓言](https://en.wikipedia.org/wiki/Allegory_of_the_cave)来表达。寓言中，一群人一生都被锁在一个山洞里，只能看到投射在他们面前墙壁上的二维影子，这些影子是由火堆前经过的看不见的三维物体产生的。对这样的人来说，他们所观察到的一切，其实都是由他们永远看不到的高维抽象概念决定的。

类似地，我们在现实世界中遇到的对象也可能是一些更高层次表示的函数生成的；例如，此类表示可以封装抽象属性，例如颜色、大小、形状等。那么，我们所观察到的就可以解释为这种抽象概念的三维投影或实例化，就像洞穴人所观察到的实际上是三维物体的二维投影一样。尽管洞穴人永远看不到（甚至完全理解）隐藏的物体，但他们仍然可以对它们进行推理和推论；以类似的方式，我们可以近似描述我们观察到的数据的潜在表示。

尽管柏拉图的寓言将隐变量背后的想法说明为决定观察结果的潜在不可观察的表示，但这种类比的一个警告是，在生成建模中，我们通常寻求学习低维潜在表示而不是高维潜在表示。这是因为如果没有强大的先验知识，试图学习比观察更高维的表示是徒劳的。另一方面，学习低维潜能也可以看作是一种压缩形式，并且可以潜在地揭示描述观察结果的具有语义意义的结构。

### 证据下限

在数学上，我们可以想象隐变量和我们观察到的数据是通过联合分布 $p(\boldsymbol{x} , \boldsymbol{z})$ 建模的. 一种称为“基于似然”的生成建模方法学习模型以最大化似然 $p(\boldsymbol{x})$， 对于所有观察到的$x$. 我们可以通过两种方式来操纵这个联合分布来恢复我们观察到的数据的可能性 $p(\boldsymbol{x})$;  我们可以明确地[边缘化](https://en.wikipedia.org/wiki/Marginal_likelihood)隐变量$z$，这样可以积掉 Z:
$$
\begin{equation}
\label{eq:1}
p(\boldsymbol{x}) = \int p(\boldsymbol{x}, \boldsymbol{z})d\boldsymbol{z}
\end{equation}
$$
或者，我们也可以求助于[概率链式法则](https://en.wikipedia.org/wiki/Chain_rule_(probability))：
$$
\begin{equation}
\label{eq:2}
p(\boldsymbol{x}) = \frac{p(\boldsymbol{x}, \boldsymbol{z})}{p(\boldsymbol{z}\mid\boldsymbol{x})}
\end{equation}
$$
直接计算并最大化似然$p(\boldsymbol{x})$很难，因为它要么涉及整合所有在等式1中隐变量$\boldsymbol{z}$，这对于复杂模型来说是棘手的，或者它涉及访问在等式2中真实潜在编码器$p(\boldsymbol{z}\mid\boldsymbol{x})$. 然而，使用这两个方程，我们可以推导出证据下限( ELBO )，顾名思义，它是证据的[下限](https://en.wikipedia.org/wiki/Upper_and_lower_bounds)。在这种情况下，证据被量化为观察到的数据的对数似然。然后，最大化 ELBO 成为优化隐变量模型的代理目标；在最好的情况下，它变得完全等同于证据。形式上，ELBO 的方程是：
$$
\begin{equation}
\mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{p(\boldsymbol{x}, \boldsymbol{z})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right]
\end{equation}
$$
为了明确与证据的关系，我们可以在数学上写成：
$$
\begin{equation}
\log p(\boldsymbol{x}) \geq \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{p(\boldsymbol{x}, \boldsymbol{z})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right]
\end{equation}
$$
这里，$q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})$ 是我们寻求优化的具有参数$\boldsymbol{\phi}$的灵活近似变分分布。直观上，它可以被认为是一个可参数化的模型，通过学习来估计给定观察的隐变量的真实分布$x$; 换句话说，它试图近似真实的后验 $p(\boldsymbol{z}\mid\boldsymbol{x})$. 正如我们将在探索变分自动编码器时看到的那样，当我们通过调整参数$\boldsymbol{\phi}$来最大化 ELBO，我们可以访问可用于对真实数据分布进行建模并从中采样的组件，从而学习生成模型。现在，让我们尝试更深入地探讨为什么 ELBO 是我们想要最大化的目标。

让我们从推导 ELBO 开始，使用等式1:
$$
\begin{align}
\log p(\boldsymbol{x})
&= \log \int p(\boldsymbol{x}, \boldsymbol{z})d\boldsymbol{z}\\
&= \log \int \frac{p(\boldsymbol{x}, \boldsymbol{z})q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}d\boldsymbol{z}\\
&= \log \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\frac{p(\boldsymbol{x}, \boldsymbol{z})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right]\\
&\geq \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log \frac{p(\boldsymbol{x}, \boldsymbol{z})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right] \label{eq:8}
\end{align}
$$
在这个推导中，我们通过应用 Jensen 不等式直接得出我们的下界。然而，这并没有为我们提供关于引擎盖下实际发生的事情的有用信息。至关重要的是，这个证明并没有给出关于为什么 ELBO 实际上是证据的下限的直觉，因为Jensen 不等式用手把它推开了。此外，仅知道 ELBO 确实是数据的下限并不能真正告诉我们为什么要将其作为目标最大化。为了更好地理解证据与 ELBO 之间的关系，让我们进行另一个推导，这次使用方程式2:
$$
\begin{align}
\log p(\boldsymbol{x}) & = \log p(\boldsymbol{x}) \int q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})dz\\
          & = \int q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})\log p(\boldsymbol{x})dz\\
          & = \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log p(\boldsymbol{x})\right]\\
          & = \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{p(\boldsymbol{x}, \boldsymbol{z})}{p(\boldsymbol{z}\mid\boldsymbol{x})}\right]\\
          & = \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{p(\boldsymbol{x}, \boldsymbol{z})q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}{p(\boldsymbol{z}\mid\boldsymbol{x})q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right]\\
          & = \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{p(\boldsymbol{x}, \boldsymbol{z})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right] + \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}{p(\boldsymbol{z}\mid\boldsymbol{x})}\right]\\
          & = \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{p(\boldsymbol{x}, \boldsymbol{z})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right] + \mathcal{D}_{\text{KL}}(q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x}) \mid\mid p(\boldsymbol{z}\mid\boldsymbol{x}))\label{eq:15}\\
          & \geq \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{p(\boldsymbol{x}, \boldsymbol{z})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right]
\end{align}
$$
从这个推导中，我们清楚地观察到方程式15证据等于 ELBO 加上近似后验 $q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})$和真正的后验$p(\boldsymbol{z}\mid\boldsymbol{x})$之间的 KL 散度, 事实上，正是这个 KL 散度项被 Jensen 的不等式方程式神奇地移除了,  在8个的一个推导中。理解这个项不仅是理解 ELBO 与证据之间关系的关键，也是理解为什么优化 ELBO 是一个适当目标的原因。

首先，我们现在知道为什么 ELBO 确实是一个下界：证据与 ELBO 之间的差异是严格非负的 KL 项，因此 ELBO 的值永远不会超过证据。

其次，我们探讨了为什么我们寻求最大化 ELBO。引入了隐变量 $z$ 我们想要建模，我们的目标是学习描述我们观察到的数据的潜在结构。换句话说，我们想要优化变分后验$q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})$的参数, 以精确匹配真实的后验分布$p(\boldsymbol{z}\mid\boldsymbol{x})$，这是通过最小化他们的 KL 散度（理想情况下为零）来实现的。不幸的是，直接最小化这个 KL 散度项是很棘手的，因为我们无法获得真实分布 $p(\boldsymbol{z}\mid\boldsymbol{x})$。但是，请注意，在等式15的左侧，我们数据的可能性（因此我们的证据项日志⁡$\log p(\boldsymbol{x})$ 始终是一个关于$\boldsymbol{\phi}$ 的常数，因为它是通过从联合分布$p(\boldsymbol{x}, \boldsymbol{z})$边缘化所有潜在的$\boldsymbol{z}$来计算的, 并且不依赖于任何$\boldsymbol{\phi}$ 。由于 ELBO 和 KL Divergence 项总和为常数，因此 ELBO 项的任何最大化关于$\boldsymbol{\phi}$ 必然调用等同的 KL 散度项的最小化。因此，ELBO 可以最大化作为学习如何完美地模拟真实潜在后验分布的代理；我们越优化 ELBO，我们的近似后验就越接近真实的后验。此外，一旦经过训练，ELBO 也可用于估计观察到的或生成的数据的可能性，因为它学会了近似模型证据 $\log p(\boldsymbol{x})$.

### 变分自动编码器

在变分自动编码器 (VAE) 的默认公式中[6, 7]，我们直接最大化 ELBO。这种方法是可变的，因为我们优化最佳$q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})$在一系列参数化的潜在后验分布中. 之所以称为自动编码器，是因为它让人联想到传统的自动编码器模型，其中输入数据经过训练以在经历中间瓶颈表示步骤后进行自我预测。为了明确这种联系，让我们进一步剖析 ELBO ：
$$
\begin{align}
\mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{p(\boldsymbol{x}, \boldsymbol{z})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right]
&= \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{p_{\boldsymbol{\theta}}(\boldsymbol{x}\mid\boldsymbol{z})p(\boldsymbol{z})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right]\\
&= \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log p_{\boldsymbol{\theta}}(\boldsymbol{x}\mid\boldsymbol{z})\right] + \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log\frac{p(\boldsymbol{z})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\right]\\
&= \underbrace{\mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log p_{\boldsymbol{\theta}}(\boldsymbol{x}\mid\boldsymbol{z})\right]}_\text{reconstruction term} - \underbrace{\mathcal{D}_{\text{KL}}(q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x}) \mid\mid p(\boldsymbol{z}))}_\text{prior matching term} \label{eq:19}
\end{align}
$$
在这种情况下，我们学习了一个中间瓶颈分布$q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})$，这个过程可以被视为编码器；它将输入转换为可能潜在的分布。同时，我们学习了一个确定性函数 $p_{\boldsymbol{\theta}}(\boldsymbol{x}\mid\boldsymbol{z})$, 转换给定的潜在向量$z$ 到观察$x$，这个过程可以理解为解码器。

<div align=center>
<img width="300" src="https://calvinyluo.com/assets/images/diffusion/vae.webp"/>
</div>

<div align=center>图 1. 可视化变分自动编码器。通过重新参数化技巧共同学习潜在的编码器和解码器[6, 7].</div

等式19中的两项每个都有直观的描述：第一项衡量了解码器从我们的变分分布中重建的似然；这确保了学习到的分布正在建模有效的潜在数据，原始数据可以从中重新生成。第二项衡量了学习到的变分分布与对隐变量持有的先验信念的相似程度。最小化这一项会鼓励编码器实际学习分布，而不是坍缩成 Dirac delta 函数。因此，最大化 ELBO 等同于最大化其第一项并最小化其第二项。

VAE 的一个决定性特征是 ELBO 如何在参数上联合优化$\boldsymbol{\phi}$和$\boldsymbol{\theta}$. VAE 的编码器通常选择为具有对角协方差的多元高斯模型，而先验通常被选择为标准的多元高斯：
$$
\begin{align}
    q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x}) &= \mathcal{N}(\boldsymbol{z}; \boldsymbol{\mu}_{\boldsymbol{\phi}}(\boldsymbol{x}), \boldsymbol{\sigma}_{\boldsymbol{\phi}}^2(\boldsymbol{x})\textbf{I})\\
    p(\boldsymbol{z}) &= \mathcal{N}(\boldsymbol{z}; \boldsymbol{0}, \textbf{I})
\end{align}
$$
然后，可以分析计算 ELBO 的 KL 散度项，并且可以使用蒙特卡罗估计来近似重建项。我们的目标可以改写为：
$$
\begin{align}
& \quad \,\underset{\boldsymbol{\phi}, \boldsymbol{\theta}}{\arg\max}\, \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})}\left[\log p_{\boldsymbol{\theta}}(\boldsymbol{x}\mid\boldsymbol{z})\right] - \mathcal{D}_{\text{KL}}(q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x}) \mid\mid p(\boldsymbol{z})) \nonumber \\
& \approx \underset{\boldsymbol{\phi}, \boldsymbol{\theta}}{\arg\max}\, \sum_{l=1}^{L}\log p_{\boldsymbol{\theta}}(\boldsymbol{x}\mid\boldsymbol{z}^{(l)}) - \mathcal{D}_{\text{KL}}(q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x}) \mid\mid p(\boldsymbol{z}))
\end{align}
$$
对于数据集中的每个观测$x$， 隐变量 $\{\boldsymbol{z}^{(l)}\}_{l=1}^L $ 是从中$q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})$ 采样得到。然而，这个默认设置出现了一个问题：每个$\boldsymbol{z}^{(l)}$我们的损失计算是通过随机抽样程序生成的，该程序通常是不可微分的。幸运的是，这可以通过重新参数化技巧来解决 [6, 7] 当$q_{\boldsymbol{\phi}}(\boldsymbol{z}\mid\boldsymbol{x})$旨在对某些分布进行建模，包括多元高斯分布。

重新参数化技巧将随机变量重写为噪声变量的确定性函数；这允许通过梯度下降优化非随机项。例如，来自正态分布的样本$x \sim \mathcal{N}(x;\mu, \sigma^2)$ 任意平均值$\mu$和方差$\sigma^2$可以重写为：

$$\begin{align}
    x &= \mu + \sigma\epsilon \quad \text{with } \epsilon \sim \mathcal{N}(\epsilon; 0, \text{I})
\end{align}$$

换句话说，任意高斯分布可以解释为标准高斯分布（其中$\epsilon$是一个样本），它们的均值通过加法从零移动到目标均值$\mu$，它们的方差被目标方差拉伸 $\sigma^2$. 因此，通过重新参数化技巧，可以通过从标准高斯采样，按目标标准差缩放结果，然后将其移动目标均值来执行任意高斯分布的采样。

在 VAE 中，每个$\sigma^2$因此被计算为输入的确定性函数$x$和辅助噪声变量$\boldsymbol{\epsilon}$:

$$\begin{align}
    \boldsymbol{z} &= \boldsymbol{\mu}_{\boldsymbol{\phi}}(\boldsymbol{x}) + \boldsymbol{\sigma}_{\boldsymbol{\phi}}(\boldsymbol{x})\odot\boldsymbol{\epsilon} \quad \text{with } \boldsymbol{\epsilon} \sim \mathcal{N}(\boldsymbol{\epsilon};\boldsymbol{0}, \textbf{I})
\end{align}$$

其中⊙代表元素乘积。在这个重新参数化的版本下，然后可以根据$\boldsymbol{\phi}$ 计算梯度，优化$\boldsymbol{\mu}_{\boldsymbol{\phi}}$和$\boldsymbol{\sigma}_{\boldsymbol{\phi}}$. 因此，VAE 利用重新参数化技巧和蒙特卡罗估计来联合优化 ELBO $\boldsymbol{\phi}$ 和$\boldsymbol{\theta}$.

训练 VAE 后，可以通过直接从潜在空间$p(\boldsymbol{z})$采样来生成新数据, 然后通过解码器运行它。变分自动编码器的维数$z$ 小于输入$x$，因为我们可能会学习紧凑、有用的表示。此外，当学习到具有语义意义的潜在空间时，可以在将潜在向量传递给解码器之前对其进行编辑，以更精确地控制生成的数据。

### Hierarchical VAE

分层变分自动编码器 (HVAE)[8, 9]是 VAE 的推广，它扩展到隐变量的多个层次结构。在这种表述下，隐变量本身被解释为从其他更高层次、更抽象的隐变量中生成。直觉上，正如我们将三维观察对象视为从更高层次的抽象潜能中生成一样，柏拉图洞穴中的人们将三维对象视为生成二维观察的潜能。因此，从柏拉图的洞穴居民的角度来看，他们的观察可以被视为由深度二（或更多）的潜在层次建模。

<div align=center>
<img width="500" src="https://calvinyluo.com/assets/images/diffusion/hvae.webp"/>
</div>

<div align=center>图 2. 马尔可夫分层变分自动编码器。生成过程被建模为马尔可夫链，其中每个隐变量仅从前一个隐变量生成.</div

而在一般的 HVAE 中有$T$层次结构级别，每个隐变量都允许以所有先前的隐变量为条件，在这项工作中，我们关注一种特殊情况，我们称之为马尔可夫 HVAE (MHVAE)。在 MHVAE 中，生成过程是马尔可夫链；也就是说，层次结构中的每个转换都是马尔可夫的，其中解码每个潜在的$\boldsymbol{z}_t$仅以先前的潜在条件为条件$\boldsymbol{z}_{t+1}$. 从直观和视觉上看，这可以看作是将 VAE 简单地堆叠在一起，如上图所示；描述这个模型的另一个合适的项是递归 VAE。在数学上，我们将马尔可夫 HVAE 的联合分布表示为：
$$
\begin{align}
    p(\boldsymbol{x}, \boldsymbol{z}_{1:T}) &= p(\boldsymbol{z}_T)p_{\boldsymbol{\theta}}(\boldsymbol{x}\mid\boldsymbol{z}_1)\prod_{t=2}^{T}p_{\boldsymbol{\theta}}(\boldsymbol{z}_{t-1}\mid\boldsymbol{z}_{t}) \label{eq:20}
\end{align}
$$
及其后验为：
$$
\begin{align}
    q_{\boldsymbol{\phi}}(\boldsymbol{z}_{1:T}\mid\boldsymbol{x}) &= q_{\boldsymbol{\phi}}(\boldsymbol{z}_1\mid\boldsymbol{x})\prod_{t=2}^{T}q_{\boldsymbol{\phi}}(\boldsymbol{z}_{t}\mid\boldsymbol{z}_{t-1}) \label{eq:21}
\end{align}
$$
然后，我们可以轻松地将 ELBO 扩展为：
$$
\begin{align}
\log p(\boldsymbol{x}) &= \log \int p(\boldsymbol{x}, \boldsymbol{z}_{1:T}) d\boldsymbol{z}_{1:T}\\
&= \log \int \frac{p(\boldsymbol{x}, \boldsymbol{z}_{1:T})q_{\boldsymbol{\phi}}(\boldsymbol{z}_{1:T}\mid\boldsymbol{x})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}_{1:T}\mid\boldsymbol{x})} d\boldsymbol{z}_{1:T}\\
&= \log \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}_{1:T}\mid\boldsymbol{x})}\left[\frac{p(\boldsymbol{x}, \boldsymbol{z}_{1:T})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}_{1:T}\mid\boldsymbol{x})}\right]\\
&\geq \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}_{1:T}\mid\boldsymbol{x})}\left[\log \frac{p(\boldsymbol{x}, \boldsymbol{z}_{1:T})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}_{1:T}\mid\boldsymbol{x})}\right] \label{eq:25}
\end{align}
$$
然后我们可以插入我们的联合分布（等式23) 和后验 (方程24) 转化为方程28产生另一种形式：
$$
\begin{align}
\mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}_{1:T}\mid\boldsymbol{x})}\left[\log \frac{p(\boldsymbol{x}, \boldsymbol{z}_{1:T})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}_{1:T}\mid\boldsymbol{x})}\right]
&= \mathbb{E}_{q_{\boldsymbol{\phi}}(\boldsymbol{z}_{1:T}\mid\boldsymbol{x})}\left[\log \frac{p(\boldsymbol{z}_T)p_{\boldsymbol{\theta}}(\boldsymbol{x}\mid\boldsymbol{z}_1)\prod_{t=2}^{T}p_{\boldsymbol{\theta}}(\boldsymbol{z}_{t-1}\mid\boldsymbol{z}_{t})}{q_{\boldsymbol{\phi}}(\boldsymbol{z}_1\mid\boldsymbol{x})\prod_{t=2}^{T}q_{\boldsymbol{\phi}}(\boldsymbol{z}_{t}\mid\boldsymbol{z}_{t-1})}\right]
\end{align}
$$
正如我们将在下面展示的，当我们研究变分扩散模型时，这个目标可以进一步分解为可解释的组件。

## 变分扩散模型

考虑变分扩散模型 (VDM) 的最简单方法[1, 2, 10], 简单来说就是一个具有三个关键限制的马尔可夫分层变分自动编码器：

- 隐变量维度正好等于数据维度
- 每个时间步的潜在编码器结构是未被学习的；它被预定义为线性高斯模型。换句话说，它是以前一个时间步的输出为中心的高斯分布
- 潜在编码器的高斯参数随时间变化，最终时间步的潜在分布是一个标准的高斯分布

此外，我们由标准马尔可夫VAE转换到分层变分自动编码器的过程中保持马尔可夫属性。

让我们扩展这些假设的含义。从第一个限制开始，由于一些符号的滥用，我们现在可以将真实数据样本和隐变量表示为$\boldsymbol{x}_t$， 其中$t=0$代表真实的数据样本， $t \in \left[1, T\right]$ 表示对应的潜在层次结构，索引为$t$. VDM 后验与 MHVAE 后验相同（方程24), 但现在可以改写为：
$$
\begin{align}
    q(\boldsymbol{x}_{1:T}\mid\boldsymbol{x}_0) = \prod_{t = 1}^{T}q(\boldsymbol{x}_{t}\mid\boldsymbol{x}_{t-1})
\end{align}
$$
从第二个假设，我们知道编码器中每个隐变量的分布都是以其先前的分层隐变量为中心的高斯分布。与马尔可夫 HVAE 不同，编码器在每个时间步的结构 $t$ 未被学习过；它固定为线性高斯模型，其中均值和标准差可以预先设置为超参数[2]，或者作为参数学习[10]. 我们用均值 $\boldsymbol{\mu}_t(\boldsymbol{x}_t) = \sqrt{\alpha_t} \boldsymbol{x}_{t-1}$ 和方差 $\boldsymbol{\Sigma}_t(\boldsymbol{x}_t) = (1 - \alpha_t) \textbf{I}$ 参数化高斯编码器，其中选择系数的形式使得隐变量的方差保持在相似的范围内；换句话说，编码过程是保持方差的。请注意，允许替代高斯参数化，并导致类似的推导。主要的收获是$\alpha_t$是一个（潜在可学习的）系数，可以随层次深度$t$变化，为了灵活性。在数学上，编码器转换表示为：
$$
\begin{align}
    q(\boldsymbol{x}_{t}\mid\boldsymbol{x}_{t-1}) = \mathcal{N}(\boldsymbol{x}_{t} ; \sqrt{\alpha_t} \boldsymbol{x}_{t-1}, (1 - \alpha_t) \textbf{I}) \label{eq:27}
\end{align}
$$
由第三个假设，我们知道$\alpha_t$根据固定的或可学习的时间表随着时间的推移而演变，这样最终潜在的分布$p(\boldsymbol{x}_T)$是标准高斯分布。然后我们可以更新马尔可夫 HVAE 的联合分布（方程23) 将 VDM 的联合分布写为：

$$
\begin{align}
p(\boldsymbol{x}_{0:T}) &= p(\boldsymbol{x}_T)\prod_{t=1}^{T}p_{\boldsymbol{\theta}}(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_t) \label{eq:36} \\
\text{where,}&\nonumber\\
p(\boldsymbol{x}_T) &= \mathcal{N}(\boldsymbol{x}_T; \boldsymbol{0}, \textbf{I})
\end{align}
$$
总的来说，这组假设描述的是图像输入随时间的稳定噪声化；我们通过添加高斯噪声来逐步破坏图像，直到最终它变得与纯高斯噪声完全相同。从视觉上看，这个过程如下图所示。

<div align=center>
<img width="500" src="https://calvinyluo.com/assets/images/diffusion/vdm_base.webp"/>
</div>
<div align=center> 变分扩散模型的视觉表示。随着时间的推移，一个输入被稳定地噪声化，直到它变得与高斯噪声相同；扩散模型学习逆转这个过程.</div

请注意，我们的编码器分布$q(\boldsymbol{x}_t\mid\boldsymbol{x}_{t-1})$不再被$\boldsymbol{\phi}$参数化，因为它们在每个时间步都有定义好化的均值和方差参数化，可以完全建模为高斯模型. 因此，在 VDM 中，我们只对学习条件 $p_{\boldsymbol{\theta}}(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_{t})$ 感兴趣, 以便我们可以模拟新数据。优化 VDM 后，采样过程就像从中$p(\boldsymbol{x}_T)$采样高斯噪声一样简单并迭代运行$p_{\boldsymbol{\theta}}(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_{t})$ 去噪转换$t$ 步骤生成$\boldsymbol{x}_0$.

像任何分层 VAE ，可以通过最大化 ELBO 来优化 VDM[1]. ELBO 的一种推导结果为以下形式：
$$
\begin{align}
\log p(\boldsymbol{x})
&\geq \mathbb{E}_{q(\boldsymbol{x}_{1:T}\mid\boldsymbol{x}_0)}\left[\log \frac{p(\boldsymbol{x}_{0:T})}{q(\boldsymbol{x}_{1:T}\mid\boldsymbol{x}_0)}\right] \label{eq:34}\\
&=  \begin{aligned}[t]
      \underbrace{\mathbb{E}_{q(\boldsymbol{x}_{1}\mid\boldsymbol{x}_0)}\left[\log p_{\theta}(\boldsymbol{x}_0\mid\boldsymbol{x}_1)\right]}_\text{reconstruction term} &- \underbrace{\mathbb{E}_{q(\boldsymbol{x}_{T-1}\mid\boldsymbol{x}_0)}\left[\mathcal{D}_{\text{KL}}(q(\boldsymbol{x}_T\mid\boldsymbol{x}_{T-1}) \mid\mid p(\boldsymbol{x}_T))\right]}_\text{prior matching term} \\
      &- \sum_{t=1}^{T-1}\underbrace{\mathbb{E}_{q(\boldsymbol{x}_{t-1}, \boldsymbol{x}_{t+1}\mid\boldsymbol{x}_0)}\left[\mathcal{D}_{\text{KL}}(q(\boldsymbol{x}_t\mid\boldsymbol{x}_{t-1}) \mid\mid p_{\theta}(\boldsymbol{x}_{t}\mid\boldsymbol{x}_{t+1}))\right]}_\text{consistency term}
    \end{aligned} \label{eq:45}
\end{align}
$$
证明给我看！

ELBO 的派生形式可以根据其各个组成部分来解释：

1. $\mathbb{E}_{q(\boldsymbol{x}_{1}\mid\boldsymbol{x}_0)}\left[\log p_{\boldsymbol{\theta}}(\boldsymbol{x}_0\mid\boldsymbol{x}_1)\right]$ 可以解释为重建项，预测给定第一步潜在的原始数据样本的对数概率。该项也出现在普通 VAE 中，并且可以进行类似的训练。
2. $\mathbb{E}_{q(\boldsymbol{x}_{T-1}\mid\boldsymbol{x}_0)}\left[\mathcal{D}_{\text{KL}}(q(\boldsymbol{x}_T\mid\boldsymbol{x}_{T-1})) \mid\mid p(\boldsymbol{x}_T))\right]$是先验匹配项；当最终的潜在分布与高斯先验匹配时，它被最小化。该项不需要优化，因为它没有可训练的参数；此外，由于我们假设足够大$t$， 最终的分布是高斯分布，这一项实际上变为零。
3. $\mathbb{E}_{q(\boldsymbol{x}_{t-1}, \boldsymbol{x}_{t+1}\mid\boldsymbol{x}_0)}\left[\mathcal{D}_{\text{KL}}(q(\boldsymbol{x}_{t}\mid\boldsymbol{x}_{t-1}) \mid\mid p_{\boldsymbol{\theta}}(\boldsymbol{x}_{t}\mid\boldsymbol{x}_{t+1}))\right]$是一致性项；它努力使分布在 $\alpha_t$ 前后一致的过程。也就是说，对于每个中间时间步，噪声较大图像的去噪步骤应与干净图像的相应噪声步骤相匹配；KL 散度在数学上反映了这一点。当我们训练$p_{\theta}(\boldsymbol{x}_t\mid\boldsymbol{x}_{t+1})$时，这个项被最小化来匹配高斯分布$q(\boldsymbol{x}_t\mid\boldsymbol{x}_{t-1})$，这是在等式31定义的.

从视觉上看，ELBO 的这种解释如下图所示。优化 VDM 的成本主要由第三项决定，因为我们必须优化所有时间步长$t$.

<div align=center>
<img width="500" src="https://calvinyluo.com/assets/images/diffusion/first_derivation.webp"/>
</div>

可以通过确保对于每个中间隐变量，其上方隐变量的后验匹配其之前隐变量的高斯破坏来优化 VDM。在这个图中，对于每个中间隐变量，我们最小化由粉色和绿色箭头表示的分布之间的差异。

在这种推导下，ELBO 的所有项都按预期计算，因此可以使用蒙特卡洛估计值进行近似。然而，实际上使用我们刚刚推导出的项来优化 ELBO 可能不是最优的；因为一致性项被计算为对两个随机变量$\left\{\boldsymbol{x}_{t-1}, \boldsymbol{x}_{t+1}\right\}$的期望对于每个时间步长，其蒙特卡洛估计的方差可能高于每个时间步长仅使用一个随机变量估计的项。因为它是通过总结计算的$t$−1个一致性条款，ELBO 的最终估计值可能会有很大的方差对于较大的$T$值。

让我们尝试为我们的 ELBO 推导一种形式，其中每一项都被计算为一次仅对一个随机变量的期望。关键的见解是我们可以将编码器转换重写为$q(\boldsymbol{x}_t\mid\boldsymbol{x}_{t-1}) = q(\boldsymbol{x}_t\mid\boldsymbol{x}_{t-1}, \boldsymbol{x}_0)$，其中由于马尔可夫属性，额外的条件项是多余的。然后，根据贝叶斯规则，我们可以将每个转换重写为：

$$
\begin{align}
q(\boldsymbol{x}_t \mid \boldsymbol{x}_{t-1}, \boldsymbol{x}_0) = \frac{q(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_t, \boldsymbol{x}_0)q(\boldsymbol{x}_t\mid\boldsymbol{x}_0)}{q(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_0)}
\end{align}
$$
有了这个新方程，ELBO 可以导出为以下形式[1, 2]:

$$
\begin{align}
\log p(\boldsymbol{x})
&\geq \mathbb{E}_{q(\boldsymbol{x}_{1:T}\mid\boldsymbol{x}_0)}\left[\log \frac{p(\boldsymbol{x}_{0:T})}{q(\boldsymbol{x}_{1:T}\mid\boldsymbol{x}_0)}\right]\\
&=  \begin{aligned}[t]
      \underbrace{\mathbb{E}_{q(\boldsymbol{x}_{1}\mid\boldsymbol{x}_0)}\left[\log p_{\boldsymbol{\theta}}(\boldsymbol{x}_0\mid\boldsymbol{x}_1)\right]}_\text{reconstruction term} &- \underbrace{\mathcal{D}_{\text{KL}}(q(\boldsymbol{x}_T\mid\boldsymbol{x}_0) \mid\mid p(\boldsymbol{x}_T))}_\text{prior matching term} \\
      &- \sum_{t=2}^{T} \underbrace{\mathbb{E}_{q(\boldsymbol{x}_{t}\mid\boldsymbol{x}_0)}\left[\mathcal{D}_{\text{KL}}(q(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_t, \boldsymbol{x}_0) \mid\mid p_{\boldsymbol{\theta}}(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_t))\right]}_\text{denoising matching term}
    \end{aligned} \label{eq:51}
\end{align}
$$
证明给我看！

因此，ELBO 可以分解为单个项的总和，这些项一次最多是一个随机变量的期望值。这个公式也有一个优雅的解释，当单独检查每个项时就会发现：

1. $\mathbb{E}_{q(\boldsymbol{x}_{1}\mid\boldsymbol{x}_0)}\left[\log p_{\boldsymbol{\theta}}(\boldsymbol{x}_0\mid\boldsymbol{x}_1)\right]$可以解释为重构项；就像它在普通 VAE 的 ELBO 中的类似物一样，可以使用蒙特卡罗估计来近似和优化该项。
2. $\mathcal{D}_{\text{KL}}(q(\boldsymbol{x}_T\mid\boldsymbol{x}_0) \mid\mid p(\boldsymbol{x}_T))$是先验匹配项；它表示最终噪声化输入的分布与标准高斯先验的接近程度。它没有可训练的参数，并且在我们的假设下也等于零。
3. $\mathbb{E}_{q(\boldsymbol{x}_{t}\mid\boldsymbol{x}_0)}\left[\mathcal{D}_{\text{KL}}(q(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_t, \boldsymbol{x}_0) \mid\mid p_{\boldsymbol{\theta}}(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_t))\right]$是去噪匹配项。我们学习所需的去噪过渡步骤 $p_{\boldsymbol{\theta}}(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_t)$ 作为易于处理的真实去噪过渡步骤的近似值$q(\boldsymbol{x}_{t-1}\mid\boldsymbol{x}_{t}, \boldsymbol{x}_0)$过渡步骤可以充当地面实况信号，因为它定义了如何对嘈杂的图像进行去噪$\boldsymbol{x}_t$可以访问最终的、完全去噪的图像$\boldsymbol{x}_0$。因此，当两个去噪步骤尽可能接近地匹配时（通过它们的 KL 散度来衡量），该项将被最小化。

下图描述了这种 ELBO 分解的直观解释：

<div align=center>
<img width="500" src="https://calvinyluo.com/assets/images/diffusion/second_derivation.webp"/>
</div>

VDM 也可以通过学习每个潜在的去噪步骤来优化，方法是将它与一个易于计算的地面实况去噪步骤相匹配。这再次通过将绿色箭头表示的分布与粉红色箭头表示的分布相匹配来直观地表示。艺术自由在这里发挥作用；在全图中，每个粉红色箭头也必须来自真实图像，因为它也是一个条件项。

作为旁注，人们观察到在两个 ELBO 推导过程中（方程式35和方程式38), 仅使用马尔可夫假设；因此，这些公式适用于任何任意马尔可夫 HVAE。此外，当我们设置$t$=1个，VDM 的两种 ELBO 解释都精确地重新创建了普通 VAE 的 ELBO 方程，如方程式中所写19.

在 ELBO 的推导中，大部分优化成本再次位于求和项中，它主导了重建项。而每个 KL 散度项丁吉隆坡(�($\alpha_t$−1个∀$\alpha_t$,�0)dddd��($\alpha_t$−1个∀$\alpha_t$))由于同时学习编码器增加了复杂性，因此很难在任意复杂的马尔可夫 HVAE 中最小化任意后验，在 VDM 中，我们可以利用高斯过渡假设来使优化变得易于处理。根据贝叶斯法则，我们有：

�($\alpha_t$−1个∀$\alpha_t$,�0)=�($\alpha_t$∀$\alpha_t$−1个,�0)�($\alpha_t$−1个∀�0)�($\alpha_t$∀�0)

我们已经知道�($\alpha_t$∀$\alpha_t$−1个,�0)=�($\alpha_t$∀$\alpha_t$−1个)=否($\alpha_t$;$\alpha_t$$\alpha_t$−1个,(1个−$\alpha_t$)我)根据我们关于编码器转换的假设（等式31), 剩下的就是对形式的推导�($\alpha_t$∀�0)和�($\alpha_t$−1个∀�0). 幸运的是，通过利用 VDM 的编码器转换是线性高斯模型这一事实，这些也变得易于处理。回想一下，在重新参数化技巧下，样本$\alpha_t$～�($\alpha_t$∀$\alpha_t$−1个)可以重写为：

(39)$\alpha_t$=$\alpha_t$$\alpha_t$−1个+1个−$\alpha_t$�和 �～否(�;0,我)

同样，样本$\alpha_t$−1个～�($\alpha_t$−1个∀$\alpha_t$−2个)可以重写为：

(40)$\alpha_t$−1个=$\alpha_t$−1个$\alpha_t$−2个+1个−$\alpha_t$−1个�和 �～否(�;0,我)

然后，形式�($\alpha_t$∀�0)可以通过重复应用重新参数化技巧递归导出。假设我们可以访问 2$t$随机噪声变量{$\alpha_t$＊,$\alpha_t$}$t$=0$t$～iid否(�;0,我). 然后，对于任意样本$\alpha_t$～�($\alpha_t$∀�0)，我们可以将其重写为：

(41)$\alpha_t$=$\alpha_t$$\alpha_t$−1个+1个−$\alpha_t$$\alpha_t$−1个＊(42)=$\alpha_t$($\alpha_t$−1个$\alpha_t$−2个+1个−$\alpha_t$−1个$\alpha_t$−2个＊)+1个−$\alpha_t$$\alpha_t$−1个＊(43)=$\alpha_t$$\alpha_t$−1个$\alpha_t$−2个+$\alpha_t$−$\alpha_t$$\alpha_t$−1个$\alpha_t$−2个＊+1个−$\alpha_t$$\alpha_t$−1个＊(44)=$\alpha_t$$\alpha_t$−1个$\alpha_t$−2个+$\alpha_t$−$\alpha_t$$\alpha_t$−1个2个+1个−$\alpha_t$2个$\alpha_t$−2个(45)=$\alpha_t$$\alpha_t$−1个$\alpha_t$−2个+$\alpha_t$−$\alpha_t$$\alpha_t$−1个+1个−$\alpha_t$$\alpha_t$−2个(46)=$\alpha_t$$\alpha_t$−1个$\alpha_t$−2个+1个−$\alpha_t$$\alpha_t$−1个$\alpha_t$−2个(47)=……(48)=‖一世=1个$t$�一世�0+1个−‖一世=1个$t$�一世�0(49)=�¯$t$�0+1个−�¯$t$�0(50)～否($\alpha_t$;�¯$t$�0,(1个−�¯$t$)我)

在等式中44我们利用了这样一个事实，即[两个独立的高斯随机变量的总和](https://en.wikipedia.org/wiki/Sum_of_normally_distributed_random_variables)仍然是高斯分布，均值是两个均值的总和，方差是两个方差的总和。口译1个−$\alpha_t$$\alpha_t$−1个＊作为高斯样本否(0,(1个−$\alpha_t$)我)， 和$\alpha_t$−$\alpha_t$$\alpha_t$−1个$\alpha_t$−2个＊作为高斯样本否(0,($\alpha_t$−$\alpha_t$$\alpha_t$−1个)我)，然后我们可以将它们的总和视为从高斯采样的随机变量否(0,(1个−$\alpha_t$+$\alpha_t$−$\alpha_t$$\alpha_t$−1个)我)=否(0,(1个−$\alpha_t$$\alpha_t$−1个)我). 然后可以使用重新参数化技巧将来自该分布的样本表示为1个−$\alpha_t$$\alpha_t$−1个$\alpha_t$−2个, 如等式46.

因此，我们推导出高斯形式的�($\alpha_t$∀�0). 可以修改此推导以还产生高斯参数化描述�($\alpha_t$−1个∀�0). 然后，我们可以将这些高斯分布代入贝叶斯规则展开�($\alpha_t$−1个∀$\alpha_t$,�0)，并证明它简化为以下内容：

(51)�($\alpha_t$−1个∀$\alpha_t$,�0)=�($\alpha_t$∀$\alpha_t$−1个,�0)�($\alpha_t$−1个∀�0)�($\alpha_t$∀�0)(52)=否($\alpha_t$;$\alpha_t$$\alpha_t$−1个,(1个−$\alpha_t$)我)否($\alpha_t$−1个;�¯$t$−1个�0,(1个−�¯$t$−1个)我)否($\alpha_t$;�¯$t$�0,(1个−�¯$t$)我)(53)∝否($\alpha_t$−1个;$\alpha_t$(1个−�¯$t$−1个)$\alpha_t$+�¯$t$−1个(1个−$\alpha_t$)�01个−�¯$t$⏟��($\alpha_t$,�0),(1个−$\alpha_t$)(1个−�¯$t$−1个)1个−�¯$t$我⏟��($t$))

证明给我看！

因此，我们已经表明，在每一步，$\alpha_t$−1个～�($\alpha_t$−1个∀$\alpha_t$,�0)服从正态分布，均值��($\alpha_t$,�0)这是一个函数$\alpha_t$和�0和方差��($t$)作为函数�系数。这些�系数在每个时间步都是已知和固定的；它们要么在建模为超参数时永久设置

[2]

，或被视为寻求对它们建模的网络的当前推理输出

[10]

. 以下等式

53

，我们可以将方差方程重写为

��($t$)=��2个($t$)我

， 其中：



(54)��2个($t$)=(1个−$\alpha_t$)(1个−�¯$t$−1个)1个−�¯$t$

为了匹配近似去噪过渡步骤��($\alpha_t$−1个∀$\alpha_t$)到地面实况去噪过渡步骤�($\alpha_t$−1个∀$\alpha_t$,�0)尽可能接近，我们也可以将其建模为高斯分布。此外，正如所有�已知项在每个时间步被冻结，我们可以立即构造近似去噪转换步骤的方差也为��($t$)=��2个($t$)我. 我们必须参数化它的均值��($\alpha_t$,$t$)作为函数$\alpha_t$, 然而, 因为��($\alpha_t$−1个∀$\alpha_t$)不条件�0.

我们利用[两个高斯分布之间的 KL 散度](https://en.wikipedia.org/wiki/Kullback/%E2/%80/%93Leibler_divergence#Multivariate_normal_distributions)为：

(55)丁吉隆坡(否(�;��,��)dddd否(是;�是,�是))=1个2个[日志⁡∀�是∀∀��∀−�+特(�是−1个��)+(�是−��)$t$�是−1个(�是−��)]

在我们的例子中，我们可以将两个高斯分布的方差设置为完全匹配，优化 KL 散度项可以减少两个分布的均值之间的差异：

参数⁡分钟�丁吉隆坡(�($\alpha_t$−1个∀$\alpha_t$,�0)dddd��($\alpha_t$−1个∀$\alpha_t$))(56)=参数⁡分钟�丁吉隆坡(否($\alpha_t$−1个;��,��($t$))dddd否($\alpha_t$−1个;��,��($t$)))(57)=参数⁡分钟�1个2个��2个($t$)[‖��−��‖2个2个]

证明给我看！

我们写的地方��作为简写��($\alpha_t$,�0)， 和��作为简写��($\alpha_t$,$t$)为简洁起见。换句话说，我们要优化一个��($\alpha_t$,$t$)那匹配��($\alpha_t$,�0), 从我们推导出的方程53，采用以下形式：

(58)��($\alpha_t$,�0)=$\alpha_t$(1个−�¯$t$−1个)$\alpha_t$+�¯$t$−1个(1个−$\alpha_t$)�01个−�¯$t$

作为��($\alpha_t$,$t$)也有条件$\alpha_t$, 我们可以匹配��($\alpha_t$,�0)通过将其设置为以下形式来关闭：

(59)��($\alpha_t$,$t$)=$\alpha_t$(1个−�¯$t$−1个)$\alpha_t$+�¯$t$−1个(1个−$\alpha_t$)�^�($\alpha_t$,$t$)1个−�¯$t$

其中�^�($\alpha_t$,$t$)由寻求预测的神经网络参数化�0来自嘈杂的图像$\alpha_t$和时间索引$t$. 然后，优化问题简化为：

参数⁡分钟�丁吉隆坡(�($\alpha_t$−1个∀$\alpha_t$,�0)dddd��($\alpha_t$−1个∀$\alpha_t$))(60)=参数⁡分钟�丁吉隆坡(否($\alpha_t$−1个;��,��($t$))dddd否($\alpha_t$−1个;��,��($t$)))(61)=参数⁡分钟�1个2个��2个($t$)�¯$t$−1个(1个−$\alpha_t$)2个(1个−�¯$t$)2个[‖�^�($\alpha_t$,$t$)−�0‖2个2个]

证明给我看！

因此，优化 VDM 归结为学习一个神经网络，以根据它的任意噪声版本预测原始地面实况图像

[2]

. 此外，最小化我们导出的 ELBO 目标的总和项（等式

38

) 可以通过最小化所有时间步长的期望来近似所有噪声水平：



参数⁡分钟�∑$t$=2个$t$乙�($\alpha_t$∀�0)[丁吉隆坡(�($\alpha_t$−1个∀$\alpha_t$,�0)dddd��($\alpha_t$−1个∀$\alpha_t$))](62)=参数⁡分钟�乙$t$～ü{2个,$t$}[乙�($\alpha_t$∀�0)[1个2个��2个($t$)�¯$t$−1个(1个−$\alpha_t$)2个(1个−�¯$t$)2个[‖�^�($\alpha_t$,$t$)−�0‖2个2个]]]

然后可以在时间步长上使用随机样本对其进行优化。

------

### 学习噪声参数

让我们研究如何联合学习 VDM 的噪声参数

[10]

. 一种可能的方法是建模

$\alpha_t$

使用神经网络

�^�($t$)

带参数

�

. 然而，这是低效的，因为推理必须在每个时间步执行多次

$t$

计算

�¯$t$

. 虽然缓存可以减轻这种计算成本，但我们还可以推导出另一种学习扩散噪声参数的方法。通过从方程式中替换我们的方差方程式

54

进入我们在方程式中导出的每时间步长目标

61

，我们可以减少：



(63)1个2个��2个($t$)�¯$t$−1个(1个−$\alpha_t$)2个(1个−�¯$t$)2个[‖�^�($\alpha_t$,$t$)−�0‖2个2个]=1个2个(�¯$t$−1个1个−�¯$t$−1个−�¯$t$1个−�¯$t$)[‖�^�($\alpha_t$,$t$)−�0‖2个2个]

证明给我看！

从方程式召回50那�($\alpha_t$∀�0)是形式的高斯否($\alpha_t$;�¯$t$�0,(1个−�¯$t$)我). 然后，按照[信噪比 (SNR)](https://en.wikipedia.org/wiki/Signal-to-noise_ratio#Alternate_definition)的定义为�2个�2个，我们可以在每个时间步写下 SNR$t$作为：

(64)信噪比($t$)=�¯$t$1个−�¯$t$

那么，我们推导出的方程63（和方程式61) 可以简化为：

(65)1个2个��2个($t$)�¯$t$−1个(1个−$\alpha_t$)2个(1个−�¯$t$)2个[‖�^�($\alpha_t$,$t$)−�0‖2个2个]=1个2个(信噪比($t$−1个)−信噪比($t$))[‖�^�($\alpha_t$,$t$)−�0‖2个2个]

顾名思义，SNR 表示原始信号与存在的噪声量之间的比率；较高的 SNR 表示较多的信号，较低的 SNR 表示较多的噪声。在扩散模型中，我们要求 SNR 随着时间步长单调递减$t$增加；这形式化了扰动输入的概念$\alpha_t$随着时间的推移变得越来越嘈杂，直到它变得与标准高斯相同$t$=$t$.

简化方程式中的目标之后65，我们可以使用神经网络在每个时间步直接参数化 SNR，并与扩散模型一起学习

[10]

. 由于 SNR 必须随时间单调下降，我们可以将其表示为：



(66)信噪比($t$)=exp(−��($t$))

其中��($t$)被建模为具有参数的单调递增神经网络�. 否定��($t$)导致单调递减函数，而指数迫使结果项为正。请注意，方程式中的目标62现在必须优化�还有：

参数⁡分钟�,�∑$t$=2个$t$乙�($\alpha_t$∀�0)[丁吉隆坡(�($\alpha_t$−1个∀$\alpha_t$,�0)dddd��($\alpha_t$−1个∀$\alpha_t$))](67)=参数⁡分钟�,�乙$t$～ü{2个,$t$}[乙�($\alpha_t$∀�0)[1个2个(信噪比($t$−1个)−信噪比($t$))[‖�^�($\alpha_t$,$t$)−�0‖2个2个]]]

通过在等式中结合我们的 SNR 参数化66根据我们在等式中对 SNR 的定义64，我们还可以明确地推导出优雅的形式�¯$t$以及价值1个−�¯$t$:

(68)�¯$t$1个−�¯$t$=exp(−��($t$))(69)∴�¯$t$=乙状结肠(−��($t$))(70)∴1个−�¯$t$=乙状结肠(��($t$))

这些项对于各种计算都是必需的；例如，在优化过程中，它们被用来创建任意噪声$\alpha_t$从输入�0使用重新参数化技巧，如等式所示49.

------

### 三种等价解释

正如我们之前证明的那样，可以通过简单地学习神经网络来预测原始自然图像来训练变分扩散模型�0来自任意噪声版本$\alpha_t$及其时间索引$t$. 然而，�0还有两个等价的参数化，这导致了对 VDM 的两个进一步解释。

首先，我们可以利用重新参数化技巧。在我们推导的形式中�($\alpha_t$∀�0)，我们可以重新排列方程49表明：

(71)�0=$\alpha_t$−1个−�¯$t$�0�¯$t$

将其插入我们之前得出的真实去噪转换均值��($\alpha_t$,�0)，我们可以推导出以下替代参数化

[2]

:



(72)��($\alpha_t$,�0)=$\alpha_t$(1个−�¯$t$−1个)$\alpha_t$+�¯$t$−1个(1个−$\alpha_t$)�01个−�¯$t$(73)=1个$\alpha_t$$\alpha_t$−1个−$\alpha_t$1个−�¯$t$$\alpha_t$�0

证明给我看！

因此，我们可以设置我们近似的去噪转换均值��($\alpha_t$,$t$)作为：

(74)��($\alpha_t$,$t$)=1个$\alpha_t$$\alpha_t$−1个−$\alpha_t$1个−�¯$t$$\alpha_t$�^�($\alpha_t$,$t$)

相应的优化问题变为：

参数⁡分钟�丁吉隆坡(�($\alpha_t$−1个∀$\alpha_t$,�0)dddd��($\alpha_t$−1个∀$\alpha_t$))(75)=参数⁡分钟�丁吉隆坡(否($\alpha_t$−1个;��,��($t$))dddd否($\alpha_t$−1个;��,��($t$)))(76)=参数⁡分钟�1个2个��2个($t$)(1个−$\alpha_t$)2个(1个−�¯$t$)$\alpha_t$[‖�0−�^�($\alpha_t$,$t$)‖2个2个]

证明给我看！

这里，�^�($\alpha_t$,$t$)是一个学习预测源噪声的神经网络�0～否(�;0,我)这决定了$\alpha_t$从�0. 因此，我们表明通过预测原始图像来学习 VDM�0相当于学习预测噪声；然而，根据经验，一些工作发现预测噪声会带来更好的性能

[2, 4]

.



为了推导出变分扩散模型的第三种常见解释，我们求助于 Tweedie 公式

[11]

. 在英语中，Tweedie 公式指出，给定从中抽取的样本，指数族分布的真实均值可以通过样本的最大似然估计（也称为经验均值）加上一些涉及估计分数的校正项来估计。在只有一个观察样本的情况下，经验平均值就是样本本身。它通常用于减轻样本偏差；如果观察到的样本都位于基础分布的一端，那么负分数会变大，并将样本的朴素最大似然估计修正为真实均值。



在数学上，对于高斯变量�～否(�;��,��), Tweedie 的公式指出：

乙[��∀�]=�+��∇�日志⁡�(�)

在这种情况下，我们将其应用于预测的真实后验均值$\alpha_t$给出了它的样本。从等式50， 我们知道：

�($\alpha_t$∀�0)=否($\alpha_t$;�¯$t$�0,(1个−�¯$t$)我)

然后，根据 Tweedie 公式，我们有：

(77)乙[�$\alpha_t$∀$\alpha_t$]=$\alpha_t$+(1个−�¯$t$)∇$\alpha_t$日志⁡�($\alpha_t$)

我们写的地方∇$\alpha_t$日志⁡�($\alpha_t$)作为∇日志⁡�($\alpha_t$)为了符号简单。根据 Tweedie 公式，真实值的最佳估计值是$\alpha_t$产生自，�$\alpha_t$=�¯$t$�0, 定义为：

(78)�¯$t$�0=$\alpha_t$+(1个−�¯$t$)∇日志⁡�($\alpha_t$)(79)∴�0=$\alpha_t$+(1个−�¯$t$)∇日志⁡�($\alpha_t$)�¯$t$

然后，我们可以代入方程79进入我们之前推导的地面实况去噪过渡均值��($\alpha_t$,�0)再次推导出一个新的参数化：

(80)��($\alpha_t$,�0)=$\alpha_t$(1个−�¯$t$−1个)$\alpha_t$+�¯$t$−1个(1个−$\alpha_t$)�01个−�¯$t$(81)=1个$\alpha_t$$\alpha_t$+1个−$\alpha_t$$\alpha_t$∇日志⁡�($\alpha_t$)

证明给我看！

因此，我们也可以设置我们近似的去噪过渡均值��($\alpha_t$,$t$)作为：

(82)��($\alpha_t$,$t$)=1个$\alpha_t$$\alpha_t$+1个−$\alpha_t$$\alpha_t$秒�($\alpha_t$,$t$)

相应的优化问题变为：

参数⁡分钟�丁吉隆坡(�($\alpha_t$−1个∀$\alpha_t$,�0)dddd��($\alpha_t$−1个∀$\alpha_t$))(83)=参数⁡分钟�丁吉隆坡(否($\alpha_t$−1个;��,��($t$))dddd否($\alpha_t$−1个;��,��($t$)))(84)=参数⁡分钟�1个2个��2个($t$)(1个−$\alpha_t$)2个$\alpha_t$[‖秒�($\alpha_t$,$t$)−∇日志⁡�($\alpha_t$)‖2个2个]

证明给我看！

这里，秒�($\alpha_t$,$t$)是学习预测得分函数的神经网络∇$\alpha_t$日志⁡�($\alpha_t$), 这是梯度$\alpha_t$在数据空间中，对于任意噪声水平$t$.

精明的读者会注意到分数函数∇日志⁡�($\alpha_t$)看起来与源噪声的形式非常相似�0. 这可以通过组合 Tweedie 公式（方程式79）与重新参数化技巧（等式71):

(85)�0=$\alpha_t$+(1个−�¯$t$)∇日志⁡�($\alpha_t$)�¯$t$=$\alpha_t$−1个−�¯$t$�0�¯$t$(86)∴(1个−�¯$t$)∇日志⁡�($\alpha_t$)=−1个−�¯$t$�0(87)∇日志⁡�($\alpha_t$)=−1个1个−�¯$t$�0

事实证明，这两项相差一个随时间变化的常数因子！得分函数衡量如何在数据空间中移动以最大化对数概率；直觉上，由于源噪声被添加到自然图像中以破坏它，因此朝相反方向移动会使图像“去噪”，这将是增加后续对数概率的最佳更新。我们的数学证明证明了这种直觉；我们已经明确表明，学习对评分函数建模等同于对源噪声的负建模（达到比例因子）。

因此，我们推导出三个等效目标来优化 VDM：学习神经网络来预测原始图像�0, 源噪声�0，或图像在任意噪声水平下的得分∇日志⁡�($\alpha_t$)

[2, 10]

. VDM 可以通过随机采样时间步长进行可扩展训练

$t$

并最小化基于真实目标的预测范数。



------

## 基于分数的生成模型

我们已经表明，可以通过优化神经网络简单地学习变分扩散模型秒�($\alpha_t$,$t$)预测得分函数∇日志⁡�($\alpha_t$). 然而，在我们的推导中，分数项来自特威迪公式的应用；这并不一定能为我们提供很好的直觉或洞察分数函数到底是什么或者为什么值得建模。幸运的是，我们可以看看另一类生成模型，基于分数的生成模型

[3, 12, 13]

，正是出于这种直觉。[Yang Song 的优秀博客文章](https://yang-song.github.io/blog/2021/score/)对此主题进行了适当、彻底的处理，本节中大量引用了该文章。事实证明，我们可以证明我们之前导出的 VDM 公式具有等效的基于分数的生成建模公式，使我们能够随意在这两种解释之间灵活切换。



为了开始理解为什么优化得分函数有意义，我们绕道而行，重新审视基于能量的模型

[14, 15]

. 任意灵活的概率分布可以写成以下形式：



(88)��(�)=1个��电子−��(�)

其中��(�)是一个任意灵活的、可参数化的函数，称为能量函数，通常由神经网络建模，并且��是一个归一化常数，以确保∫��(�)��=1个. 学习这种分布的一种方法是最大似然法；然而，这需要简单地计算归一化常数��=∫电子−��(�)��，这对于复杂的情况可能是不可能的��(�)职能。

避免计算或建模归一化常数的一种方法是使用神经网络秒�(�)学习得分函数∇日志⁡�(�) 分布的 �(�)反而。这是由于观察到对方程两边的对数求导88产量：

(89)∇�日志⁡��(�)=∇�日志⁡(1个��电子−��(�))(90)=∇�日志⁡1个��+∇�日志⁡电子−��(�)(91)=−∇���(�)(92)≈秒�(�)

它可以自由地表示为神经网络，而不涉及任何归一化常数。请注意，我们可以使用神经网络对能量函数进行建模��(�)，从中我们可以通过随意计算其负梯度来查询分数；或者，更常见的是，我们可以直接使用神经网络对得分函数进行建模秒�(�). 然后可以通过最小化估计分数和地面实况分数函数之间的 Fisher Divergence 来优化神经网络：

(93)乙�(�)[‖秒�(�)−∇日志⁡�(�)‖2个2个]

分数函数代表什么？对于每一个�，取其对数似然的梯度�本质上描述了数据空间中的哪个方向移动以进一步增加其可能性。直觉上，分数函数在整个空间上定义了一个向量场，该向量场包含数据�居住，指向模式。从视觉上看，这由下图右侧图中的黑色箭头表示。

![Langevin Dynamics 沿玩具轮廓可视化采样路径](https://calvinyluo.com/assets/images/diffusion/score_3d.jpeg)

![根据地面实况分数可视化 Langevin Dynamics 的采样路径](https://calvinyluo.com/assets/images/diffusion/score_sample.webp)

使用 Langevin 动力学生成的三个随机采样轨迹的可视化，所有这些都从相同的初始化点开始，用于高斯混合。左图在三维轮廓上绘制了这些采样轨迹，而右图绘制了采样轨迹与地面实况得分函数的对比。由于 Langevin 动力学采样过程中的随机噪声项，我们能够从相同的初始化点生成不同模式的样本；没有它，从固定点采样将始终确定性地遵循每次试验的相同模式的分数。

然后，通过学习真实数据分布的得分函数，我们可以通过从同一空间中的任意点开始并迭代地跟随得分直到达到模式来生成样本。这种采样过程被称为 Langevin 动力学，在数学上被描述为：

(94)�一世+1个←�一世+�∇日志⁡�(�一世)+2个��,一世=0,1个,...,钾

其中�0从先验分布（例如均匀分布）中随机抽样，并且�～否(�;0,我)是一个额外的噪声项，以确保生成的样本不会总是折叠到一个模式，而是围绕它徘徊以实现多样性。此外，由于学习到的分数函数是确定性的，因此使用涉及的噪声项进行采样会增加生成过程的随机性，从而使我们能够避免确定性轨迹。当从位于多个模式之间的位置初始化采样时，这特别有用。上图显示了 Langevin 动态采样和噪声项的好处的直观描述。

请注意，方程式中的目标93依赖于访问真实评分函数，对于复杂的分布，例如自然图像建模，我们无法使用该函数。幸运的是，称为分数匹配的替代技术

[16, 17, 18, 19]

已经推导出来最小化这种 Fisher 散度，而无需知道地面真值分数，并且可以通过随机梯度下降进行优化。



总的来说，学习将分布表示为分数函数并使用它通过马尔可夫链蒙特卡罗技术（例如 Langevin 动力学）生成样本，被称为基于分数的生成建模

[3, 12, 13]

.



正如 Song 和 Ermon 所详述的那样，普通分数匹配存在三个主要问题

[3]

. 首先，分数函数在以下情况下定义不明确

�

位于高维空间中的低维流形上。这可以从数学上看出；不在低维流形上的所有点的概率为零，其对数未定义。这在尝试学习基于自然图像的生成模型时尤其不方便，因为已知自然图像位于整个环境空间的低维流形上。



其次，通过普通分数匹配训练的估计分数函数在低密度区域将不准确。从我们在等式中最小化的目标可以明显看出这一点93. 因为这是一个期待�(�)，并明确地对其中的样本进行训练，对于很少见或未见的例子，该模型将不会收到准确的学习信号。这是有问题的，因为我们的采样策略涉及从高维空间中的随机位置开始，这很可能是随机噪声，并根据学习的得分函数移动。由于我们遵循的是嘈杂或不准确的分数估计，因此最终生成的样本也可能不是最优的，或者需要更多的迭代才能收敛到准确的输出。

最后，Langevin 动力学采样可能不会混合，即使它是使用地面实况分数执行的。假设真实数据分布是两个不相交分布的混合：

(95)�(�)=�1个�1个(�)+�2个�2个(�)

然后，当计算分数时，这些混合系数会丢失，因为对数操作将系数从分布中分离出来，而梯度操作会将其归零。为了形象化这一点，请注意上面右图中显示的地面真值得分函数与三个分布之间的不同权重无关；尽管右下角的模式在实际的高斯混合中具有更高的权重，但从描绘的初始化点进行的 Langevin 动力学采样具有大致相等的机会到达每个模式。

事实证明，这三个缺点可以通过向数据中添加多级高斯噪声来同时解决。首先，由于高斯噪声分布的支持是整个空间，扰动数据样本将不再局限于低维流形。其次，添加大的高斯噪声会增加每个模式在数据分布中覆盖的区域，在低密度区域添加更多的训练信号。最后，添加方差增加的多级高斯噪声将导致符合地面实况混合系数的中间分布。

形式上，我们可以选择噪声级的正序列{$\alpha_t$}$t$=1个$t$并定义一系列逐渐扰动的数据分布：

(96)�$\alpha_t$($\alpha_t$)=∫�(�)否($\alpha_t$;�,$\alpha_t$2个我)��

然后，一个神经网络秒�(�,$t$)使用得分匹配来学习，以同时学习所有噪声水平的得分函数：

(97)参数⁡分钟�∑$t$=1个$t$�($t$)乙�$\alpha_t$($\alpha_t$)[‖秒�(�,$t$)−∇日志⁡�$\alpha_t$($\alpha_t$)‖2个2个]

其中�($t$)是一个以噪声水平为条件的正加权函数$t$. 请注意，此目标几乎与公式中导出的目标完全匹配84训练变分扩散模型。此外，作者提出将退火 Langevin 动力学采样作为生成过程，其中通过为每个样本运行 Langevin 动力学来生成样本$t$=$t$,$t$−1个,...,2个,1个按顺序。初始化是从一些固定的先验（比如uniform）中选择的，之后的每个采样步骤都是从之前模拟的最终样本开始的。因为噪音水平随着时间步长稳定下降$t$，并且我们随着时间的推移减小步长，样本最终会收敛到真实模式。这直接类似于在变分扩散模型的马尔可夫 HVAE 解释中执行的采样过程，其中随机初始化的数据向量在降低的噪声水平上迭代细化。

因此，我们在变分扩散模型和基于分数的生成模型之间建立了明确的联系，无论是在训练目标还是采样过程中。

一个问题是如何自然地将扩散模型推广到无限数量的时间步长。在马尔可夫 HVAE 观点下，这可以解释为将层次结构的数量扩展到无穷大$t$→∞. 从等效的基于分数的生成模型的角度来表示这一点更清楚；在无限数量的噪声尺度下，图像在连续时间上的扰动可以表示为随机过程，因此可以用随机微分方程 (SDE) 来描述。然后通过反转 SDE 进行采样，这自然需要估计每个连续值噪声水平的得分函数

[12]

. SDE 的不同参数化本质上描述了随时间变化的不同扰动方案，从而实现噪声过程的灵活建模

[10]

. [邀请感兴趣的读者在杨松的博客上](https://yang-song.github.io/blog/2021/score/)进一步探讨该主题。



------

## 指导

到目前为止，我们只专注于对数据分布进行建模�(�). 然而，我们通常也对学习条件分布感兴趣�(�∀是)，这将使我们能够明确地控制我们通过调节信息生成的数据是. 这构成了级联扩散模型等图像超分辨率模型的支柱

[20]

，以及最先进的图像文本模型，例如 DALL-E 2

[5]

和图像

[4]

.



添加条件信息的一种自然方法是在每次迭代时简单地与时间步长信息一起添加。回忆一下方程式中的联合分布32:

�(�0:$t$)=�($\alpha_t$)‖$t$=1个$t$��($\alpha_t$−1个∀$\alpha_t$)

然后，为了将其转化为条件扩散模型，我们可以简单地添加任意条件信息是在每个过渡步骤为：

(98)�(�0:$t$∀是)=�($\alpha_t$)‖$t$=1个$t$��($\alpha_t$−1个∀$\alpha_t$,是)

例如，是可以是图像文本生成中的文本编码，也可以是用于执行超分辨率的低分辨率图像。因此，我们能够像以前一样通过预测来学习 VDM 的核心神经网络�^�($\alpha_t$,$t$,是)≈�0,�^�($\alpha_t$,$t$,是)≈�0， 或者秒�($\alpha_t$,$t$,是)≈∇日志⁡�($\alpha_t$∀是)对于每个所需的解释和实现。

这种普通公式的一个警告是，以这种方式训练的条件扩散模型可能会学会忽略或淡化任何给定的条件信息。因此，建议将指导作为一种更明确地控制模型赋予条件信息的权重的方法，但代价是样本多样性。两种最流行的指导形式被称为分类器指导

[12, 21]

和无分类器指导

[22]

. 可以在 Sander Dieleman 的深入[博客文章](https://benanne.github.io/2022/05/26/guidance.html)中找到对指导主题的出色处理，以及它如何扩展到自回归模型；在这里，我们只关注扩散模型的指导。



### 分类指导

让我们从扩散模型的基于分数的公式开始，我们的目标是学习∇日志⁡�($\alpha_t$∀是)，条件模型的分数，在任意噪声水平$t$. 回顾∇是简写∇$\alpha_t$为了简洁起见。根据贝叶斯法则，我们可以推导出如下等价形式：

(99)∇日志⁡�($\alpha_t$∀是)=∇日志⁡(�($\alpha_t$)�(是∀$\alpha_t$)�(是))(100)=∇日志⁡�($\alpha_t$)+∇日志⁡�(是∀$\alpha_t$)−∇日志⁡�(是)(101)=∇日志⁡�($\alpha_t$)⏟无条件得分+∇日志⁡�(是∀$\alpha_t$)⏟对抗梯度

我们利用了梯度的事实日志⁡�(是)关于$\alpha_t$为零。

我们最终的推导结果可以解释为学习一个无条件的得分函数与分类器的对抗梯度相结合�(是∀$\alpha_t$). 因此，在分类器指导中

[12, 21]

，无条件扩散模型的分数是按照先前推导的方式学习的，以及一个接受任意噪声的分类器

$\alpha_t$

并尝试预测条件信息

是

. 然后，在采样过程中，用于退火 Langevin 动力学的整体条件得分函数被计算为无条件得分函数和噪声分类器的对抗梯度之和。



为了引入细粒度控制来鼓励或阻止模型考虑条件信息，分类器指导通过�超参数项。在分类器指导下学习的得分函数可以总结为：

(102)∇日志⁡�($\alpha_t$∀是)=∇日志⁡�($\alpha_t$)+�∇日志⁡�(是∀$\alpha_t$)

直觉上，当�=0条件扩散模型学会完全忽略条件信息，并且何时�很大，条件扩散模型学会生成严重遵守条件信息的样本。这将以样本多样性为代价，因为它只会产生易于从中重新生成所提供的调节信息的数据，即使在嘈杂的水平下也是如此。

分类器指导的一个明显缺点是它依赖于单独学习的分类器。由于分类器必须处理任意噪声输入，而大多数现有的预训练分类模型并未针对此进行优化，因此必须与扩散模型一起进行临时学习。

### 无分类器指导

在无分类器指导中

[22]

，作者放弃了单独分类器模型的训练，转而使用无条件扩散模型和条件扩散模型。为了推导无分类器指导下的得分函数，我们可以首先重新排列方程式

101

表明：



(103)∇日志⁡�(是∀$\alpha_t$)=∇日志⁡�($\alpha_t$∀是)−∇日志⁡�($\alpha_t$)

然后，将其代入等式102，我们得到：

(104)∇日志⁡�($\alpha_t$∀是)=∇日志⁡�($\alpha_t$)+�(∇日志⁡�($\alpha_t$∀是)−∇日志⁡�($\alpha_t$))(105)=∇日志⁡�($\alpha_t$)+�∇日志⁡�($\alpha_t$∀是)−�∇日志⁡�($\alpha_t$)(106)=�∇日志⁡�($\alpha_t$∀是)⏟条件分数+(1个−�)∇日志⁡�($\alpha_t$)⏟无条件得分

再次，�是一个控制我们学到的条件模型对条件信息的关心程度的项。当�=0，学习到的条件模型完全忽略了调节器，学习了一个无条件扩散模型。当�=1个，该模型在没有指导的情况下显式学习普通条件分布。当�>1个，扩散模型不仅优先考虑条件得分函数，而且向远离无条件得分函数的方向移动。换句话说，它降低了生成不使用条件信息的样本的概率，有利于明确使用条件信息的样本。这也具有以生成与条件信息准确匹配的样本为代价降低样本多样性的效果。

因为学习两个单独的扩散模型是昂贵的，我们可以将条件和非条件扩散模型一起学习作为一个单一的条件模型；可以通过用固定常数值（例如零）替换条件信息来查询无条件扩散模型

[22]

. 这本质上是对条件信息进行随机丢弃。Classifier-Free Guidance 是优雅的，因为它使我们能够更好地控制我们的条件生成过程，同时只需要训练一个单一的扩散模型。



------

## 关闭

请允许我们在这篇博文中概括一下我们的发现。首先，我们推导出变分扩散模型作为马尔可夫分层变分自动编码器的一个特例，其中三个关键假设支持 ELBO 的易处理计算和可扩展优化。然后，我们证明优化 VDM 归结为学习神经网络以预测三个潜在目标之一：来自任意噪声化的原始源图像，来自任意噪声化图像的原始源噪声，或噪声化的得分函数任意噪声水平的图像。然后，我们更深入地研究学习得分函数的意义，并通过 Tweedie 公式将扩散模型的变分视角与基于得分的生成建模视角明确联系起来。最后，

尽管扩散模型在生成建模方面取得了令人难以置信的成功，但仍然存在一些需要考虑的缺点，这些缺点是未来工作中令人兴奋的方向：

- 作为人类，这不太可能是我们自然建模和生成数据的方式；我们似乎并没有生成新的样本作为我们迭代去噪的随机噪声。
- VDM 不产生可解释的隐变量。VAE 希望通过编码器的优化来学习结构化潜在空间，而在 VDM 中，每个时间步长的编码器已经作为线性高斯模型给出，无法灵活优化。因此，中间隐变量被限制为原始输入的噪声版本。
- 潜在被限制在与原始输入相同的维度，进一步挫败了学习有意义的、压缩的潜在结构的努力。
- 采样是一个昂贵的过程，因为必须在两种公式下运行多个去噪步骤。时间步数$t$通常非常大以确保最终的潜在是完全高斯噪声；在采样期间，我们必须遍历所有这些时间步长以生成样本。

最后一点，扩散模型的成功凸显了分层 VAE 作为生成模型的力量。我们已经表明，当我们推广到无限的潜在层次结构时，即使编码器是微不足道的并且潜在维度是固定的并且假设马尔可夫转换，我们仍然能够学习强大的数据模型。这表明在一般的深度 HVAE 的情况下可以实现进一步的性能提升，其中可以潜在地学习复杂的编码器和具有语义意义的潜在空间。

为了获得更多资源和更深入的见解，我鼓励好奇的读者查看以下（非详尽的）论文和博客列表：

For further resources and deeper insights, I impel the inquisitive reader to check out the following (non-exhaustive) list of papers and blogs:

- Blogs:
  - [What are Diffusion Models?](https://lilianweng.github.io/posts/2021-07-11-diffusion-models/)
  - [Generative Modeling by Estimating Gradients of the Data Distribution (blog)](https://yang-song.github.io/blog/2021/score/)
  - [Guidance: A Cheat Code for Diffusion Models](https://benanne.github.io/2022/05/26/guidance.html)
  - [Score based diffusions explained in just one paragraph](https://kidger.site/thoughts/score-based-diffusions-explained-in-just-one-paragraph/)
  - [A Path to the VDM Diffusion Loss](https://blog.alexalemi.com/diffusion.html)
- Papers:
  - [Deep Unsupervised Learning using Nonequilibrium Thermodynamics](https://arxiv.org/abs/1503.03585)
  - [Denoising Diffusion Probabilistic Models](https://arxiv.org/abs/2006.11239)
  - [Generative Modeling by Estimating Gradients of the Data Distribution (paper)](https://arxiv.org/abs/2006.09011)
  - [Score-Based Generative Modeling through Stochastic Differential Equations](https://arxiv.org/abs/2011.13456)
  - [Diffusion Models Beat GANs on Image Synthesis](https://arxiv.org/abs/2105.05233)
  - [Classifier-Free Diffusion Guidance](https://arxiv.org/abs/2207.12598)
