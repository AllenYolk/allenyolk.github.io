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
\end{cases}\ ，
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

**Fixup Initialization** [^fixup]：

* 通过修改残差网络的初始化权重，使其在训练初期信号衰减/爆炸得到控制
* 可在 ResNet 上训练深层网络而不需要 BN
* 训练稳定性和 SOTA 性能通常稍逊于 NFNets，但方法更简单

## 噪声注入

**NoMorelization**[^nomore]：用 **两标量 + 噪声注入** 模拟归一化作用，计算成本极低，速度快，并适用于卷积和 Transformer 等架构。

…

[^nf1]: Brock, A., De, S., & Smith, S.L. (2021). Characterizing signal propagation to close the performance gap in unnormalized ResNets. ICLR 2021.
[^nf2]: Brock, A., De, S., Smith, S. L., & Simonyan, K. (2021). High-Performance Large-Scale Image Recognition Without Normalization. ICML 2021.
[^ottt]: Xiao, M., Meng, Q., Zhang, Z., He, D., & Lin, Z. (2022). Online Training Through Time for Spiking Neural Networks. NeurIPS 2022.
[^tnnls]: Civitelli, E., Sortino, A., Lapucci, M., Bagattini, F., & Galvan, G. (2025). A Robust Initialization of Residual Blocks for Effective ResNet Training Without Batch Normalization. IEEE Transactions on Neural Networks and Learning Systems, 36(1), 1947–1952.
[^fixup]: Zhang, H., Dauphin, Y., & Ma, T. (2019). Fixup Initialization: Residual Learning Without Normalization. ICLR 2019.
[^nomore]: Liu, C., Yang, Y., Ding, Y., & Lu, H. (2022). NoMorelization: Building Normalizer-Free Models from a Sample's Perspective. arXiv preprint arXiv:2210.06932.
