---
title: Normalization-free Networks
lang: zh-cn
tags:
  - machine-learning
  - normalization
draft: "false"
---

在深度网络中，batch normalization (BN) 和 layer normalization (LN) 等归一化层的主要作用有：

* 减少内部协变量偏移（internal covariate shift），保持激活分布稳定
* 使 loss landscape 光滑，令大学习率、大 batch size 下的训练更稳定
* 提供了正则化效果，提升模型泛化能力：例如，BN 训练时统计量基于当前 mini batch 计算；这些统计量随着训练 step 不断变化，与整个数据集上的真实统计量不相等，相当于引入了微小噪声。

但归一化层存在缺点。以 BN 为例：

* 时空开销大：计算过程较为复杂，且需要保存较多中间结果和统计量
* BN 打破 batch 中样本之间的独立性：例如，在做时间序列预测任务时，经常会从长序列中截取出若干有重叠的片段；在这样的设定下，使用 BN 会导致训练时信息泄露。
* 训练效果受 batch size 影响大：batch size 极小时，BN 统计量偏移严重，导致模型性能极差

> [!note]
> 在 SNN 领域中，BN 的使用通常被认为是合理的：完成训练后，BN 可以与线性层融合，从而在推理阶段不引入额外浮点操作。而 LN 等模块的使用则是不合理的：LN 无法和线性层融合，无法部署到神经形态器件上。

在传统认识中，归一化对于深层网络稳定训练是必要的。但有一系列研究显示，归一化层的功能可以通过其他方式达到。基于这些方式搭建的无归一化网络 (normalization-free networks) 可以应用于无法使用归一化的特殊场景，且性能也较为理想。

## Normlizer-free ResNets

**Normalizer-free ResNets (NF-ResNets)** 是最被认可、性能最好的一类 Normalization-free 网络。这类模型在 ICLR 2021[^nf1] 中被提出，并在 ICML 2021[^nf2] 中得到改进。

* 在大规模分类任务上，NF-ResNets 可达到或超越带 BN 的 baseline 性能
* 提高训练效率：相比同样正确率的带 BN 的网络，训练速度提升 $8.7\times$

### 控制激活值方差

为了观察前向传播阶段信号统计量的变化趋势，NF-ResNet 的原论文提出了**信号传播图**[^nf1]。

给定一个初始化后的深度神经网络，将含有 $N$ 个样本的一个 mini batch 输入其中；这些样本可以来自数据集，也可以从标准正态分布中直接采样获得（对最终观测到的信号传播趋势没有太大影响）。选定网络 $L$ 个关键位置，将这些位置处的激活值 $\{\mathbf{x}^{(l)}\}_{l=1}^L$ 作为考察对象。假设 $\mathbf{x}^{(l)}\in\mathbb{R}^{N\times C_l \times \dots}$，其中省略号代表空间维度。信号传播图至少含两个子图：

* **通道均方的平均值 (average channel squared mean)**：$\operatorname{Avg}_c\left[\operatorname{Avg}(\mathbf{x}^{(l)}_{.,c,\dots})^2\right]$，其中 $\mathbf{x}^{(l)}_{.,c,\dots}$ 代表 $\mathbf{x}^{(l)}$ 的第 $c$ 个通道。换言之，先求逐通道的平均，平方过后，再对所有通道求平均。
* **通道方差的平均值 (average channel variance)**：$\operatorname{Avg}_c\left[\operatorname{Var}(\mathbf{x}^{(l)}_{.,c,\dots})\right]$

对于 ResNet 而言，通常选择每个残差块的输出（$\mathbf{x}^{(l)}+f(\mathbf{x}^{(l)})$，加和后）作为考察对象，且另外考虑残差分支输出张量（$f(\mathbf{x}^{(l)})$，加和前）的通道方差平均值 (average channel variance on the end of residual branch)。下图展示了一个含 4 个 stage、200 个残差块（600 层）的、经过 Kaiming 初始化的 pre-act ResNet 中这些统计量关于深度 $l$ 的变化趋势 [^nf1]。

![[norm-free-networks.png]]

对于每个残差块 $\mathbf{x}^{(l+1)}=\mathbf{x}^{(l)}+f(\mathbf{x}^{(l)})$ 而言，有

$$
\operatorname{Var}[\mathbf{x}^{(l+1)}]=\operatorname{Var}[\mathbf{x}^{(l)}]+\operatorname{Var}[f(\mathbf{x}^{(l)})].
$$

残差分枝 $f$ 中蕴含 BN，且 Kaiming 初始化后的卷积层能保持信号方差。故在初始化结束时，$\operatorname{Var}[f(\mathbf{x}^{(l)})]$ 值恒定，不受 $l$ 的影响，如上图 (c)；$\operatorname{Var}[\mathbf{x}^{(l)}]$ 随着 stage 内 $l$ 的增加而线性增长，如上图 (b)。另外，在 stage transition 时，主干网络上的 BN（见下图）将信号方差重置到较低水平，如上图 (b) 所示。

![[norm-free-networks-1.png]]

为了让 NF-ResNet 的方差变化模式和有 BN 的 ResNet 相似，Brock 等人修改了残差连接的计算方式：

$$
\mathbf{x}^{(l+1)}=\mathbf{x}^{(l)}+\alpha f(\mathbf{x}^{(l)}/\beta^{(l)})
$$

其中：

* $f$ 是不含 BN 的残差分枝，且满足 $\operatorname{Var}[f(\mathbf{x})]=\operatorname{Var}[\mathbf{x}]$，即保持信号方差（后续章节将介绍如何在不使用 BN 的前提下保持信号方差）
* $\beta^{(l)}=\sqrt{\operatorname{Var}[\mathbf{x}^{(l)}]}$，用于确保 $f$ 的输入方差为 1；
* $\alpha$ 是人为设定的标量因子，用于模拟同一个 stage 内的方差增长

于是，有

$$
\operatorname{Var}[\mathbf{x}^{(l+1)}]=\operatorname{Var}[\mathbf{x}^{(l)}]+\alpha^2
$$

又由于网络输入经过归一化增强后保证 $\mathbf{x}^{(0)}$，故可以提前设置 $\beta^{(l)}$ 的取值

$$
\beta^{(l)} = \sqrt{1+l\cdot \alpha^2}
$$

最后，为了模拟 transition 块对方差的重制效果，NF-ResNet 将 $\mathbf{x}^{(l)}/\beta^{(l)}$（而非 $\mathbf{x}^{(l)}$）输入到 transition 块跳跃链接的卷积层中。由于初始化后该卷积层能保持方差，故 transition 块输出的方差为 $1+\alpha^2$，达到了重置方差的目的。

### Scaled Weight Standardization (sWS)

为了实现上一节所提到的“保持信号方差的无 BN 残差分枝”，对权重施加标准化：

$$
\hat{W}_{ij} = \gamma\frac{W_{ij}-\mu_i}{\sqrt{N}\sigma_i},
$$

其中，$i$ 是输出通道的下标，$N$ 是扇入，$\mu_i$ 和 $\sigma_i$ 分别指代某个输出通道中权重的均值和标准差，而 $\gamma$ 是一个恒定标量。$\gamma$ 的作用是，使得“激活函数 - 线性层”模块能保持信号方差，即：

$$
\operatorname{Var}[\mathbf{\hat{W}}g(\mathbf{x})]=\operatorname{Var}[\mathbf{x}],
$$

其中 $g$ 代表激活函数。对于 ReLU 而言，$\gamma=\sqrt{\frac{2}{1-1/ \pi}}$；而 SNN 中常取 $\gamma=2.74$ [^ottt]。类似于 BN，可以为 sWS 再添加一个可学习的、逐输出通道的放缩因子 $\xi$，从而有

$$
\hat{W}_{ij} = \xi\cdot\gamma\frac{W_{ij}-\mu_i}{\sqrt{N}\sigma_i}.
$$

sWS 一般施加于网络骨干中的所有线性层权重。偏置和其他仿射变换参数通常不做 sWS 处理。另外，sWS 通常不施加在线性分类头上。

### Adaptive Gradient Clipping (AGC)

施加上述两点改动后，前向传播信号的方差得以很好地维持。但有分析表明，NF-ResNet 浅层权重的梯度方差是深层权重梯度方差的指数倍，从而导致梯度爆炸 [^tnnls]。AGC 通过适应性地约束梯度范数来缓解这一现象。

常规的梯度裁剪 (GC) 直接对梯度的范数或各元素绝对值施加限制。以基于 L2 范数的梯度裁剪为例：

$$
\mathbf{\hat{G}} =
\begin{cases}
\frac{\lambda}{\|\mathbf{G}\|_2} \mathbf{G} \quad &\text{if }\|\mathbf{G}\|_2>\lambda \\
\mathbf{G} &\text{else}
\end{cases}\ ,
$$

此处的裁剪阈值 $\lambda \in \mathbb{R}^+$ 至关重要，需要花大功夫调优。AGC 则让梯度裁剪阈值取决于权重范数大小：

$$
\mathbf{\hat{G}}_i =
\begin{cases}
\frac{\lambda \|\mathbf{W}_i\|^*_2}{\|\mathbf{G}_i\|_2}\mathbf{G}_i \quad &\text{if } \frac{\|\mathbf{G}_i\|_2}{\|\mathbf{W}_i\|^*_2} > \lambda \\
\mathbf{G}_i &\text{else}
\end{cases}\ ,\
\text{where } \|\mathbf{W}_i\|_2^*=\max\left(\|\mathbf{W}_i\|_2, \epsilon\right)
$$

此处的 $i$ 是输出通道的下标，即对每个输出通道单独做梯度裁剪；$\epsilon=10^{-3}$ 用于防止初始化为 0 的参数永远不更新。裁剪阈值 $\lambda$ 通常被设置为 0.01，0.1 或 1，取决于具体的任务和网络类型。相比常规 GC，AGC 对超参数 $\lambda$ 更不敏感，且能适应更多种权重初始化策略。在 AGC 的帮助下，NF-ResNet 达到了当时的 ImageNet 分类 SOTA[^nf2]。

## 初始化

**Fixup Initialization** [^fixup] 调整传统初始化策略并添加少量可学习标量，用以缓解 Normalization-free ResNet 训练时的梯度爆炸问题 [^tnnls]。

假设网络中共含有 $L$ 个残差块，其中第 $l$ 个残差块含 $m^{(l)}$ 个权重层。Fixup 初始化规则为：

1. 将每个残差块的最后一个权重层初始化为全 0，从而使残差分枝的输出为 $f^{(l)}(\mathbf{x}^{(l)})=0$。即：除了 transition 块以外的其他残差块都被初始化成恒等映射。
2. 对于残差分枝中的其他权重，先做标准的 Kaiming 初始化，再放缩 $L^{-\frac{1}{2m^{(l)}-2}}$ 倍。这一放缩避免了梯度爆炸。
3. 在每个残差分枝加和前，施加一个可学习且初始化为 $1$ 的标量放缩：$\mathbf{x}^{(l+1)}=\alpha^{(l)}f^{(l)}(\mathbf{x}^{(l)})+\mathbf{x}^{(l)}$。在每个线性变换和激活函数前，施加一个可学习且初始化为 $0$ 的标量偏置。这些标量模拟了 BN 层的可学习仿射变换，但更省参数。

Fixup 初始化的优势为：只涉及初始化方法的改动和少量额外标量，不涉及训练期间的权重标准化和梯度裁剪。工程实现方便，训练速度更快。但其缺点在于：缺少对训练过程的约束，信号可能随着训练的进行而逐渐爆炸，故性能不如带 BN 的 ResNet 以及 NF-ResNet。

## 噪声注入

**NoMorelization**[^nomore] 一文从单个样本的角度审视 BN，发现 BN 实际上对样本做了 L2 正则化和噪声注入两个步骤。不妨假设 batch 中含有 $N$ 个样本，且每个样本 $x_i$ 都是标量。若忽略仿射变换以及分母上的标准差，则 BN 可表示为

$$
\begin{aligned}
\hat{x}_i &= x_i - \frac{1}{N}\sum_{n=1}^N x_n \\
&= \frac{N-1}{N}x_i + \frac{1}{N}\sum_{n=1,\ n\ne i}^N x_n \ ,
\end{aligned}
$$

第一项是对样本 $x_i$ 的固定比例衰减，类似于 L2 正则化对权重的衰减效应；第二项与当前样本无关，故可解释为噪声。实际情况下，batch 中信号样本并不遵从独立同分布的正态分布，难以对这里的噪声进行直接建模。对于这个噪声项，NoMorelization 的作者持有以下观点 [^nomore]：

* 这个噪声起到了正则化的作用。故删去 BN 后，网络性能变差。
* 然而，这个噪声的形式过于复杂，增加了训练难度。若在保留噪声的前提下，简化噪声形式，则可能提升模型性能。

这一观点可以自然拓展到其他类型的 normalization 上。于是，NoMorelization 直接将噪声项简化为高斯噪声，并得到如下的模块公式：

$$
\mathbf{\hat{x}} = \alpha\mathbf{x}+\beta+\gamma\cdot\boldsymbol{\delta},
$$

其中 $\boldsymbol{\delta}$ 是均值为 0 的高斯噪声向量，$\alpha$ 和 $\beta$ 是可学习的标量，$\gamma$ 是不可学习的、用于控制噪声强度的超参数。实际应用中，NoMorelization 模块并非直接替换原有的 Normalization 层，而是添加在残差分枝结尾处（加和之前），即

$$
\mathbf{x}^{(l+1)} = \left[\alpha^{(l)} f^{(l)}(\mathbf{x}^{(l)})+\beta^{(l)}+\gamma\cdot\boldsymbol{\delta}\right]+\mathbf{x}^{(l)},
$$

这种情况下，$\alpha$ 和 $\beta$ 都初始化为 0，从而使残差块被初始化为恒等映射。替代 BN 时，通常 $\gamma=0.1$；替代 LN 时，通常 $\gamma=10^{-4}$。噪声仅在训练时施加；推理时，噪声项被移除。

> [!note]
> 这里只是阐述原文观点和方法。请客观理性地看待！

NoMorelization 形式简洁，易于实现，训练开销低，且可替换 BN、LN 等多种 normalization 层并应用于 CNN、Transformer 等多种架构中。原文实验表明，NoMorelization 可以达到和传统归一化方式相似（甚至略高）的性能；若结合 sWS 和 AGC 等正则化方法，则可达到显著更高的性能。

[^nf1]: Brock, A., De, S., & Smith, S.L. (2021). Characterizing signal propagation to close the performance gap in unnormalized ResNets. ICLR 2021.
[^nf2]: Brock, A., De, S., Smith, S. L., & Simonyan, K. (2021). High-Performance Large-Scale Image Recognition Without Normalization. ICML 2021.
[^ottt]: Xiao, M., Meng, Q., Zhang, Z., He, D., & Lin, Z. (2022). Online Training Through Time for Spiking Neural Networks. NeurIPS 2022.
[^tnnls]: Civitelli, E., Sortino, A., Lapucci, M., Bagattini, F., & Galvan, G. (2025). A Robust Initialization of Residual Blocks for Effective ResNet Training Without Batch Normalization. IEEE Transactions on Neural Networks and Learning Systems, 36(1), 1947–1952.
[^fixup]: Zhang, H., Dauphin, Y., & Ma, T. (2019). Fixup Initialization: Residual Learning Without Normalization. ICLR 2019.
[^nomore]: Liu, C., Yang, Y., Ding, Y., & Lu, H. (2022). NoMorelization: Building Normalizer-Free Models from a Sample's Perspective. arXiv preprint arXiv:2210.06932.
