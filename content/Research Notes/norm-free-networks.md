---
title: Normalization-free Networks
lang: zh-cn
tags:
  - machine-learning
  - normalization
  - snn
draft: "false"
---

在深度网络中，batch normalization (BN) 和 layer normalization (LN) 等归一化层的主要作用有：

* 减少内部协变量偏移（internal covariate shift），保持激活分布稳定
* 使 loss landscape 光滑，令大学习率、大 batch size 下的训练更稳定
* 提供了正则化效果，提升模型泛化能力：例如，BN 训练时统计量基于当前 mini batch 计算；这些统计量随着训练 step 不断变化，与整个数据集上的真实统计量不相等，相当于引入了微小噪声。

但归一化层存在缺点。以 BN 为例：

* 时空开销大：计算过程较为复杂，且需要保存较多中间结果和统计量
* BN 打破 batch 中样本之间的独立性：例如，在做时间序列预测任务时，经常会从长序列中截取出若干有重叠的片段；在这样的设定下，使用 BN 会导致训练时信息泄露。

> [!note]
> 在 SNN 领域中，BN 的使用通常被认为是合理的：完成训练后，BN 可以与线性层融合，从而在推理阶段不引入额外浮点操作。而 LN 等模块的使用则是不合理的：LN 无法和线性层融合，无法部署到神经形态器件上。

在传统认识中，归一化对于深层网络稳定训练是必要的。但有一系列研究显示，归一化层的功能可以通过其他方式达到。基于这些方式搭建的无归一化网络 (normalization-free networks) 可以应用于无法使用归一化的特殊场景，且性能也较为理想。

## Normlizer-free ResNets

**Normalizer-free ResNets (NF-ResNets)** 是最被认可、性能最好的一类 Normalization-free 网络。这类模型在 ICLR 2021[^nf1] 中被提出，并在 ICML 2021[^nf2] 中得到改进。

* 在大规模分类任务上，NF-ResNets 可达到或超越带 BN 的 baseline 性能
* 提高训练效率：相比同样正确率的带 BN 的网络，训练速度提升 $8.7\times$

### 控制激活值方差

### Scaled Weight Standardization (sWS)

为避免激活值统计量偏移，对权重施加标准化：

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

### Adaptive Gradient Clipping

…

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
[^fixup]: Zhang, H., Dauphin, Y., & Ma, T. (2019). Fixup Initialization: Residual Learning Without Normalization. ICLR 2019.
[^nomore]: Liu, C., Yang, Y., Ding, Y., & Lu, H. (2022). NoMorelization: Building Normalizer-Free Models from a Sample's Perspective. arXiv preprint arXiv:2210.06932.
