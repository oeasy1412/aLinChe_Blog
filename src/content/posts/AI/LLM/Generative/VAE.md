---
title: VAE
published: 2026-07-01
description: VAE
tags: [Generative, Diffusion, LLM]
category: LLM
draft: false
---

# 1. 深入浅出生成算法数学基础：从凸函数到 KL 散度

## 1. 一切的起点：凸函数 (Convex Function)
**直观理解：**
在数学中，凸函数最直观的几何特征就是“**弦在弧上**”。如果你在函数图像上任取两点连成一条线段（弦），这条线段上的每一个点都会高于（或等于）对应位置的函数曲线（弧）。

**数学公式：**
对于定义域内的任意两点 $x_1$ 和 $x_2$，以及任意比例参数 $\lambda \in [0, 1]$，(下凸)凸函数 $f(x)$ 满足：
$$ f(\lambda x_1 + (1-\lambda)x_2) \le \lambda f(x_1) + (1-\lambda)f(x_2) $$

## 2. Jensen 不等式
**数学公式：**
对于任意一个**凸函数** $f(x)$，假设有一组参数 $x_1, x_2, \dots, x_M$，以及对应的权重 $\lambda_1, \lambda_2, \dots, \lambda_M$。（这些权重满足 $\lambda_i \ge 0$ 且 $\sum_{i=1}^M \lambda_i = 1$），那么必然满足以下不等式：

$$ f\left(\sum_{i=1}^M \lambda_i x_i\right) \le \sum_{i=1}^M \lambda_i f(x_i) $$

**直观理解：**
- **等式左边 $f\left(\sum \lambda_i x_i\right)$：【先加权，后映射】**
    我们先把所有的输入参数 $x_i$ 按照权重 $\lambda_i$ 混合起来（求加权平均值/重心），得到一个新的点，然后再把这个混合后的点代入函数 $f$ 中求值。
    - *👉 几何意义：先在 X 轴上找到这群点的“重心”，然后看这个重心对应的**“函数曲线上的点”**。*
- **等式右边 $\sum \lambda_i f(x_i)$：【先映射，后加权】**
    我们先分别算出每个输入参数对应的函数值 $f(x_1), f(x_2)...$，然后再对这些**函数值（即 Y 轴上的高度）**按照同样的权重求加权平均。
    - *👉 几何意义：找到这群点在函数曲线上的位置，然后直接在空间中求这些点的“重心”。这个重心必然悬空落在连接这些点的**“多边形弦（割线面）上”**。*

将权重 $\lambda_i$ 视为 **概率 $P(x_i)$**，数学期望 $\mathbb{E}[X] = \sum P(x_i) x_i$，而函数值 $f(x)$ 按概率加权求和 $\sum P(x_i) f(x_i)$ 即为 $\mathbb{E}[f(X)]$。

此时，对于随机变量 $X$，有：
$$ f(\mathbb{E}[X]) \le \mathbb{E}[f(X)] $$
*(口诀：期望的函数 小于等于 函数的期望)*

### Jensen 不等式 在**生成模型**中的意义
在 AI 领域，我们经常需要最大化数据的对数似然 $\log P(X)$。但很多时候 $P(X)$ 的直接计算极其困难（包含复杂的积分）。

> $$ \log P(X) = \log \left( \int P(X|Z) P(Z) dZ \right) $$
> $$ \nabla_\theta \log P(X) = \frac{1}{\int P(X|Z)P(Z) dZ} \cdot \nabla_\theta \left( \int P(X|Z) P(Z) dZ \right) $$
> 分母上的 $\int P(X|Z)P(Z) dZ$ 是一个极其高维的积分。假如隐变量 $Z$ 是 256 维的向量，你要对 256 维空间做连续积分，这会导致**维度灾难**，就算是超级计算机算到宇宙毁灭也算不出来精确值（x
> 
> 而且它也**无法使用蒙特卡洛采样(Monte Carlo)**：“积分算不出，那我随机采样几个 $Z$ 近似代替不就行了吗？”的思路不可取，因为在茫茫的高维 $Z$ 空间里，随便盲抽一个 $Z$（比如瞎猜一组猫的特征），它能解码出现实图片 $X$ 的 **似然概率** $P(X|Z)$ 几乎为 0。（我连十连出金都没见过几个.jpg）

利用对于对数函数的 Jensen 不等式，我们可以把 $\log$ 移到期望的外面，从而将复杂的极大似然估计转化为优化一个**下界 (Lower Bound)**。这就是 VAE（变分自编码器）中著名的 **`ELBO`(Evidence Lower Bound，证据下界)** 的由来（后文会细讲）。

---

## 3. 度量分布的尺子：KL 散度 (KL Divergence)
在生成模型中，我们往往希望我们用神经网络生成的概率分布 $Q(x)$ 能够尽可能逼近真实的数据分布 $P(x)$。如何衡量这两个分布的差异？这就需要用到 **KL 散度**（相对熵）。

**数学公式：**
连续分布下，分布 $P$ 和分布 $Q$ 的 KL散度定义为：
$$ D_{KL}(P || Q) = \int P(x) \log \frac{P(x)}{Q(x)} dx $$
或者用概率论中期望的写法（离散与连续通用）：
$$ D_{KL}(P || Q) = \mathbb{E}_{x \sim P} \left[ \log \frac{P(x)}{Q(x)} \right] $$

**核心性质：**
1. **非对称性**：$D_{KL}(P || Q) \neq D_{KL}(Q || P)$，所以它不是严格意义上的“距离”。
2. **非负性**：$D_{KL}(P || Q) \ge 0$，当且仅当两个分布完全相同时取等号。

### 核心推导：用 Jensen 不等式证明 KL散度 $\ge 0$
我们来证明 $D_{KL}(P || Q) \ge 0$：
$$ D_{KL}(P || Q) = \mathbb{E}_{x \sim P} \left[ \log \frac{P(x)}{Q(x)} \right] $$
利用对数的性质 $\log(a/b) = -\log(b/a)$，将其翻转，并将负号留在期望内部：
$$ = \mathbb{E}_{x \sim P} \left[ -\log \frac{Q(x)}{P(x)} \right] $$
因为 $- \log$ 是凸函数，根据 **Jensen 不等式**，$\mathbb{E}[-\log(X)] \ge \log(\mathbb{E}[X])$。
$$ \mathbb{E}_{x \sim P} \left[ -\log \left( \frac{Q(x)}{P(x)} \right) \right] \ge -\log \left( \mathbb{E}_{x \sim P} \left[ \frac{Q(x)}{P(x)} \right] \right) $$
展开期望公式：
$$ = - \log \left( \int P(x) \frac{Q(x)}{P(x)} dx \right) = - \log \left( \int Q(x) dx \right) = - \log(1) = 0 $$
**结论：** $D_{KL}(P || Q) \ge 0$。证毕！

---

# 2. VAE 变分自编码器 的核心思路
**代表技术**：VAE (2014)

> 论文：**Auto-Encoding Variational Bayes** · Kingma, Welling · arXiv：[1312.6114](https://arxiv.org/abs/1312.6114)

## 1. 为什么需要 VAE？普通的 AE 哪里不够好？
要理解 VAE，首先要回顾它的前身——**自编码器(Auto-Encoder, 简称 AE)**。
普通的 AE 包含一个**编码器(Encoder)** 和一个**解码器(Decoder)**：
- **编码器Encoder**：将高维的输入图像 $X$ 压缩成低维的隐向量 $Z$（比如提取出一张猫图的特征）。
- **解码器Decoder**：将隐向量 $Z$ 还原成图像 $X'$。

**AE 的致命缺陷：无法用于生成**
在 AE 中，每张图像都被编码成了隐空间 (Latent Space) 中的一个**确定的点**。这就导致隐空间是**稀疏且不连续的**。如果你在隐空间中随机插值取一个没有被训练数据覆盖到的“空白点”扔给解码器，它很可能会输出一团毫无意义的噪点。
> 💡 换句话说，AE 只适合做**数据压缩**，不能做**数据生成**。

## 2. VAE 的核心思路：从“确定的点”到“概率分布”
为了让模型具备“生成”能力，VAE 提出了一个极其优美的改进思路：**既然确定的点容易造成断层，那我们就把输入映射为一个区域（即概率分布）！**

在 VAE 中：
1. **编码器** 不再输出一个具体的隐向量$Z$，而是输出一个**概率分布的参数**。通常我们假设这个分布是高斯分布，所以编码器会输出 均值 $\mu$ 和方差 $\sigma^2$。
2. **采样(Sampling)**：在这个均值和方差确定的高斯分布 $\mathcal{N}(\mu, \sigma^2)$ 中，**随机采样**出一个向量 $Z$ 作为隐向量。
3. **解码器** 将采样得到的 隐向量 $Z$ 还原成 图像 $X'$。

### 💡 为什么 VAE 这样做有效？
由于加入了随机采样，即使是同一张图片，每次编码后参与解码的 $Z$ 都会有微小的扰动。这逼迫解码器必须学会：**在 $\mu$ 附近的这一片连续区域内，不管怎么采样，都能解码出清晰的图像**。这就使得隐空间变得相对**连续且平滑**了，不同图像的区域与区域之间也会发生重叠融合 (**Interpolation**)，赋予了模型真正的生成能力。

## 3. VAE 数学上的落地：损失函数 与 KL 散度
直观上理解了 VAE 的架构，接下来我们面临的问题是：**如何用数学语言定义它，并训练这个网络？** 

我们先明确一下网络中各个部件对应的数学符号：
- **$X$ (观测数据)**：真实的数据，比如输入给网络的真实图片。
- **$Z$ (隐变量 / Latent Variable)**：图像被压缩降维后，在隐空间中的特征表示。
- **$P$ (解码侧)**：代表真实或期望的生成分布。
    - **$P(Z)$**：隐变量的**先验分布(Prior)**。即在没看到图片之前，我们期望隐变量长什么样（通常人为设为**多元标准正态分布** $\mathcal{N}(0, I)$，其中 $I$ 是单位矩阵）。
    - **$P(X)$**：真实图片的数据分布（我们的**最终目标**就是最大化这个分布的对数似然 $\log P(X)$）。
    - **$P_\theta(X|Z)$**：**解码器(Decoder)**。即给定一个特征 $Z$，把它还原成真实图片 $X$ 的概率。其中 $\theta$ 是解码器**Decoder的可训练参数**。
- **$Q$ (编码侧)**：因为真实的**后验分布(Posterior)** $P(Z|X)$ 算不出来，我们引入一个变分分布 $Q_\phi$ 来近似它。
    - **$Q_\phi(Z|X)$**：**编码器(Encoder)**。即输入一张图片 $X$，神经网络输出的对应隐变量 $Z$ 的概率分布（通常输出均值 $\mu_\phi$ 和方差 $\sigma^2_\phi$ ），其中 $\phi$ 是编码器**Encoder的可训练参数**。

![VAE](images/VAE.png)

### 💡 为什么不去计算真实的后验分布 $P(Z|X)$ ？
在理想状态下，当我们输入一张图片 $X$，我们最想知道的是它最完美、最真实的隐变量分布，即**真实的后验分布 $P(Z|X)$**。

然而，这是一个数学上的“死胡同”，人类和计算机都难以精确求出它。

根据**贝叶斯定理**： $ P(Z|X) = \frac{P(X|Z) \cdot P(Z)}{P(X)} $
1. **分子 $P(X|Z)$（似然）**：这是解码器，用神经网络拟合，能算！
2. **分子 $P(Z)$（先验）**：人为规定的多元标准正态分布，能算！
3. **分母 $P(X)$（万恶之源！）**： 
    根据全概率公式，$P(X) = \int P(X|Z)P(Z) dZ$。这意味着要在极高维的连续特征空间里做积分，计算量非常大（**维度灾难**，这一点在前面 Jensen 不等式部分已经证明过了）。

既然直接求 $\log P(X)$ 走不通，用我们引入的编码器 $Q_\phi(Z|X)$，有：
$$ - \log P(X) = - \log \left( \int Q_\phi(Z|X) \frac{P_\theta(X|Z)P(Z)}{Q_\phi(Z|X)} dZ \right) = - \log \left( \mathbb{E}_{Z \sim Q_\phi(Z|X)} \left[ \frac{P_\theta(X|Z)P(Z)}{Q_\phi(Z|X)} \right] \right) $$

根据 **Jensen 不等式**，有：
$$ - \log P(X) \le - \mathbb{E}_{Z \sim Q_\phi(Z|X)} \left[ \log \left( \frac{P_\theta(X|Z)P(Z)}{Q_\phi(Z|X)} \right) \right] = \mathbb{E}_{Z \sim Q_\phi(Z|X)} \left[ \log \frac{Q_\phi(Z|X)}{P(Z)} - \log P_\theta(X|Z) \right] $$
$$ = \mathbb{E}_{Z \sim Q_\phi(Z|X)} \left[ \log \frac{Q_\phi(Z|X)}{P(Z)} \right] - \mathbb{E}_{Z \sim Q_\phi(Z|X)} \left[ \log P_\theta(X|Z) \right] = - \text{ELBO} $$

当期望 $\mathbb{E}$ 被提到最外面后，由莱布尼茨积分法则，求导算子可以直接穿透期望，即：$\nabla \mathbb{E}[...] = \mathbb{E}[\nabla ...]$。

这就将一个无法计算的极大似然估计问题，巧妙转化为了**优化一个可微分的下界(ELBO)**。

### 最终落地：VAE 的损失函数
VAE 的目标是**最大化生成数据的对数似然** $\log P(X)$。经过变分推断（引入隐变量 $Z$ ），我们得到 VAE 的核心损失函数（其实也就是最大化 ELBO 的相反数）：

$$ \mathcal{L}_{VAE} = - \mathbb{E}_{Z \sim Q_\phi(Z|X)} [\log P_\theta(X|Z)] + D_{KL}(Q_\phi(Z|X) \parallel P(Z)) $$
$$ \mathcal{L}_{VAE} = \mathcal{L}_{rec} + \mathcal{L}_{KL} $$

这个由两部分组成的损失函数，恰好对应了 VAE 的**两个设计目标**：
### (1) 重构损失 (`Reconstruction Loss`) : $- \mathbb{E} [\log P_\theta(X|Z)]$
这部分等价于输入输出的均方误差(MSE)或交叉熵，它的目的是让解码器 $P_\theta(X|Z)$ 能够**尽可能完美地还原输入图像**。
### (2) 相似度/正则化 损失 (`KL Divergence Loss`) : $D_{KL}(Q_\phi(Z|X) \parallel P(Z))$
如果只有重构损失，神经网络会“偷懒”：它会把方差 $\sigma^2$ 学成无限趋近于 0，这样采样就又退化成了确定的点，VAE 就又变回了普通的 AE。
为了防止这一点，我们强制要求编码器输出的分布 $Q_\phi(Z|X)$ 必须接近一个先验分布 $P(Z)$（通常设为**多元标准正态分布 $\mathcal{N}(0, I)$**），通过 KL 散度把各个图像的特征分布向先验分布 $P(Z)$ 拉近，使得隐空间的数据点紧凑地聚集成一个实心球体，消除簇与簇之间的断层空隙。

> P.S. 由数学计算"易得"：$ \mathcal{L}_{VAE} = \mathcal{L}_{rec} + \mathcal{L}_{KL} = \text{MSE} - \frac{1}{2} \sum_{i=1}^d (1 + \log(\sigma_i^2) - \mu_i^2 - \sigma_i^2) $
> ```python
> recons_loss = F.mse_loss(recon_x, x, reduction='sum')
> kl_loss = -0.5 * torch.sum(1 + log_var - mu.pow(2) - log_var.exp())
> ```
> [pytorch/examples/vae 示例代码](https://github.com/pytorch/examples/blob/main/vae/main.py#L80)

## 4. 还没完！解决工程问题：**重参数化** (`Reparameterization`)
到这里，VAE 的理论看似完美，但在用 PyTorch 实际写代码时会遇到一个致命的工程问题：**“采样”这个动作是随机的，不可导！**

神经网络依赖反向传播(Backpropagation)更新梯度，但梯度无法穿过一个包含随机性的“采样节点”。

为了解决这个问题，VAE 提出了 **重参数化技巧**：
我们不直接从 $\mathcal{N}(\mu, \sigma^2)$ 中采样 $Z$，而是：
1. 先从多元标准正态分布 $\mathcal{N}(0, I)$ 中采样一个纯噪声 $\epsilon$。
2. 然后通过公式进行线性变换：$ Z = \mu + \sigma \odot \epsilon $

在这个公式里，$\epsilon$ 是不需要求导的常数噪声，而 $\mu$ 和 $\sigma$ 是网络层输出的确定的参数。这样一来，随机性被成功地剥离到了一个独立的旁支上，梯度就可以顺畅地沿着 $\mu$ 和 $\sigma$ 反向传播回编码器了！

## 5. VAE 小结一下
VAE 的核心目的是**学习数据的隐变量分布，并构建一个连续、结构化且致密的潜在空间(`Latent Space`)**。

相比于传统的 AE，VAE 最大的魅力在于：
1. **强大的生成能力**：训练完成后，我们甚至可以直接丢弃编码器，只需在多元标准正态分布 $\mathcal{N}(0,I)$ 中随机抓取一个向量扔给解码器，就能生成一张**现实中不存在却栩栩如生**的全新图片。
2. **平滑的特征过渡**：由于隐空间的连续性，我们可以提取两个不同图像的隐向量，在它们之间进行线性插值（比如从戴眼镜的男人平滑过渡到不戴眼镜的女人，bushi），解码器可以输出相对丝滑的中间渐变过程。

可以说，从把 $\log P(x)$ 拆解为 ELBO，到利用 KL散度约束分布，再到重参数化技巧打通反向传播，VAE 将严谨的数学理论与精妙的工程技巧结合到了极致。

---

# 3. 流匹配 (`Flow Matching`)
## 1. 核心直觉：从“跳跃的采样”到“流动的向量场”
- 在 VAE 中，我们借助重参数化技巧，从先验分布中“抓取”一个随机点，通过解码器**一步跨越**隐空间与像素空间的鸿沟，直接生成图像。
- 与之不同的是，扩散模型(Diffusion)引入了随机微分方程（`SDE`）来刻画含噪演化，而流匹配(Flow Matching)则进一步将其简化为确定性常微分方程（`ODE`）。二者的核心共性在于：摒弃了“点映射”的思维，转而通过构建**连续的向量场**，完成从简单噪声分布到复杂数据分布的**概率流变换**。
- **流匹配的核心思想**：通过流(Flow)的思想，把已知分布一步步流动转换为真实的目标分布。可以看作概率密度在流动。

## 2. 数学基石：常微分方程 (ODE) 与 目标匹配
既然是描述随时间变化的“流动”，最合适的数学工具就是**常微分方程 (`ODE`, Ordinary Differential Equation)**：
$$ \frac{d x_t}{d t} = v_t(x_t) $$
- $x_t$ 是 $t$ 时刻 点 $x$ 在多维空间的位置，其中 $t \in [0,1]$，0 时刻为初始位置，1 时刻是目标位置，位置连起来即为**轨迹(Trajectory)**。
- $v_t(x_t)$ 是 $t$ 时刻在 $x_t$ 位置 的**向量场/速度场(Vector Field)**，即流动规则。
  - 在一个多维空间，如果给定了初始位置 $x_0$ 和向量场 $v_t(x)$，就可以通过 ODE 解出任意时刻的 $x_t$ 并确定唯一的轨迹 $X$。
- **流(Flow)**：$ \phi_t(x_0) = x_t $，有：$ \frac{d \phi_t(x_0)}{dt}  = v_t(\phi_t(x_0)) $，其核心为 $P_0(x_0) \to P_t(x_t)$


如果我们拥有一个真实的、完美引导噪声走向数据的“理想向量场” $u_t^{marginal}(x)$，那么我们只需要训练一个神经网络 $v_\theta(x, t)$ 去逼近它，就解决问题了。

这就是**流匹配目标函数 (Flow Matching Objective)**：$ \mathcal{L}_{FM}(\theta) = \mathbb{E}_{t \sim U[0,1], x \sim p_t(x)} \left[ ||v_\theta(x_t, t) - u_t^{marginal}(x_t)||^2 \right] $

## 3. 条件流匹配
但是事实是我们根本不知道那个全局的、理想的向量场 $u_t^{marginal}(x)$ 究竟长什么样，

---

## TODO：DDPM，EDM，条件流匹配CFM，直流最优传输，边缘化定理，Reflow + Distill

---

# 参考文献：
- 讲解：https://www.bilibili.com/video/BV1xFxMz1EMS
- VAE：https://arxiv.org/abs/1312.6114
- DDPM：https://arxiv.org/abs/2006.11239
- EDM：https://arxiv.org/abs/2206.00364
- CFM：https://arxiv.org/abs/2210.02747
