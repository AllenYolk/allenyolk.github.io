---
title: Normalization-free Networks
lang: zh-cn
tags:
  - machine-learning
  - normalization
draft: "false"
---

在深度网络中，batch normalization (BN)和layer normalization (LN) 等归一化层的主要作用是：

- 减少内部协变量偏移（internal covariate shift），保持激活分布稳定
- 使loss landscape光滑，令训练更稳定
- 提供正则化效果，提高泛化能力

在传统认识中，归一化对于深层网络稳定训练是必要的。但近年来研究显示，这些功能可以通过其他方式达到。

## Normlizer-free ResNets[^1][^2]

核心思想：

- 精确控制信号（activations, gradients）的方差传播，而不依赖归一化层
- 使用 **Scaled Weight Standardization（缩放权重标准化）**
- 采用 **Adaptive Gradient Clipping（AGC，自适应梯度裁剪）** 解决训练不稳问题
- 增强正则化（Dropout / Stochastic Depth）替代 Norm 带来的正则化效果

结果：

- 在大规模分类任务上，NFNets 可 **达到或超越带 BN 的 baseline 性能**
- 支持大 batch、强增强、迁移学习场景
- 提高训练效率（NFNet-F1 训练速度比 EfficientNet-B7 快约 8.7×）

*Still Under Construction...*

[^1]: Brock, A., De, S., & Smith, S.L. (2021). Characterizing signal propagation to close the performance gap in unnormalized ResNets. ICLR 2021.
[^2]: Brock, A., De, S., Smith, S. L., & Simonyan, K. (2021). High-Performance Large-Scale Image Recognition Without Normalization. ICML 2021.
