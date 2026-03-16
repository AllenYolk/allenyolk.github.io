---
title: "PyTorch: Tensor Version and Data"
lang: zh-cn
tags:
  - "#pytorch"
draft: "false"
---

本笔记将结合代码解析 PyTorch 中 `torch.Tensor.data` 以及 `torch.Tensor._version` 的底层机制。

## `torch.Tensor._version` 机制

`_version` 是 PyTorch tensor 的一个属性，用于追踪原地 (in-place) 修改次数。每当 tensor 发生原地操作 (如 `add_`、`copy_` 等) 时，`_version` 都会自增。如此一来，autograd 可以检测到数据是否被意外修改，防止梯度计算出错。

以下面的代码为例：

```python
import torch

x = torch.tensor(2.0)
w = torch.tensor(0.1, requires_grad=True)
y = w * x
print("version after forward:", x._version)
# version after forward: 0

x.add_(1)
print("version after inplace:", x._version)
# version after inplace: 1

y.backward()
# RuntimeError: one of the variables needed for gradient computation has been modified by an inplace operation: [torch.FloatTensor []] is at version 1; expected version 0 instead. ...
```

1. 执行乘法 `y = w * x` 时，由于 `w` 需要求梯度且 $\frac{\partial y}{\partial w}=x$，故 `x` 将被保存起来，以供反向传播使用 (即 `ctx.save_for_backward(x)`)。此时的 `x` 并未经过任何原地修改，因此 `x._version` 是 `0`。`ctx.save_for_backward(x)` 除了保存 `x` 外，还记录了此刻 (forward 时) 的 `x._version`。
2. 执行 `x.add_(1)` 后，`x` 被原地修改，版本号自增。因此，此时 `x._version` 变成了 `1`。
3. 执行 `y.backward()` 时，自动微分引擎开始执行乘法的 `backward()`。此时，需要取出被保存起来的 `x`。此刻的 `x._version` 是 `1`，但前向传播期间记录下来的、当时的 `x._version` 是 `0`。版本号不匹配，报运行时错误。

**总结**：PyTorch 在 forward 阶段记录所保存张量的 `_version`，在 backward 时检查张量版本一致性。若发生了原地修改导致张量 `_version` 改变，则立即抛出错误以避免梯度计算错误。

## `torch.Tensor.data` 机制

`torch.Tensor.data` 用于直接访问张量的底层数据。它返回一个与原张量**共享底层存储** 的**新 `torch.Tensor` 对象**，且该张量**被从计算图中分离**。

一个最直接的示例：

```python
x = torch.tensor(2.0, requires_grad=True)  
y = x.data

print(x.untyped_storage().data_ptr() == y.untyped_storage().data_ptr())
# True
print(id(x) == id(y))
# False
```

可见，`x.data` 和 `x` 虽然共享一片存储，但却是两个不同的对象。

> [!note]
> `torch.Tensor` 对象其实是一组元数据 (metadata，包括 shape，stride，version 等信息)。它指向了一段存有张量数据的存储空间。

另一个较复杂的示例：

```python
import torch

x = torch.tensor(2.0, requires_grad=True)
print(x)
# tensor(2., requires_grad=True)

z = x.data
print(z)
# tensor(2.)  

z.add_(1)
print(z)
# tensor(3.)  

print(x)
# tensor(3., requires_grad=True)

x.add_(1)
# RuntimeError: a leaf Variable that requires grad is being used in an in-place operation.
```

可以看到：

* `z = x.data` 所得的 `z` 并不需要求梯度，说明 `.data` 返回的张量被从计算图中分离。
* `z.add_(1)` 成功地使 `x` 的值发生变化，说明 `.data` 返回的张量和原张量共享底层存储。
* `x.add_(1)` 则会导致运行时错误，因为需要求梯度的叶子节点不允许被原地修改。

利用 `.data`，可以绕开 PyTorch 的某些安全保护机制，强行修改张量值。若能在保证后果已知且安全的前提下妥善使用，可以轻松完成一些有难度的操作。

## 用 `.data` 绕过张量版本检查

由于 `.data` 创建了新的张量对象，故对 `x.data` 的原地修改只会使 `x.data._version` 自增，而不影响 `x._version`。例如：

```python
import torch

B, N = 3, 4
x = torch.zeros(B, N)
print(f"x._version={x._version}, x.data._version={x.data._version}")
# x._version=0, x.data._version=0

x.add_(1)
print(f"x._version={x._version}, x.data._version={x.data._version}")
# x._version=1, x.data._version=0  

y = x.data
print(f"x._version={x._version}, y._version={y._version}")
# x._version=1, y._version=0  

y.add_(1)
print(f"x._version={x._version}, y._version={y._version}")
# x._version=1, y._version=1

print(torch.allclose(x, y))
# True
```

可以看出：

* `.data` 所创建的新张量 `_version` 为 `0`。
* 对 `x.data` 的原地修改不会影响 `x` 的 `_version`。
* 对 `x.data` 的原地修改影响了 `x` 的值。

因此，`.data` 提供了一种 **绕过 autograd 张量版本检查** 的方式。例如：

```python
import torch
import torch.nn as nn
from torch import autograd
import torch.nn.functional as F
import einops


class ETLinear(autograd.Function):
    @staticmethod
    def forward(ctx, x, et, weight, bias):
        if any(ctx.needs_input_grad):
            ctx.save_for_backward(et, weight)
        return F.linear(x, weight, bias)

    @staticmethod
    def backward(ctx, grad_output):
        grad_x, grad_weight, grad_bias = None, None, None

        if any(ctx.needs_input_grad):
            et, weight = ctx.saved_tensors
            print("ET version in backward:", et._version)
            print("ET: ", et)
            if ctx.needs_input_grad[0]:
                grad_x = einops.einsum(
                 grad_output, weight, "... o, o i -> ... i"
             )
            if ctx.needs_input_grad[2]:
                grad_weight = einops.einsum(
                 grad_output, et, "... o, ... i -> o i"
             )
            if ctx.needs_input_grad[3]:
                grad_bias = einops.reduce(
                 grad_output, "... o -> o", "sum"
             )

        return grad_x, None, grad_weight, grad_bias
```

这里的 `ETLinear` 前向传播与 `nn.Linear` 类似，但反向传播时对权重梯度 `grad_weight` 的计算则基于资格迹 (eligibility trace) 变量 `et` 进行。这一算子可广泛应用于 SNN 在线学习。考虑一个 `ETLinear-Neuron` 块，其需按序执行如下计算步骤：

1. `y = ETLinear(x)`
2. `s = Neuron(y)`
3. 基于 `x`，`y` 以及 `Neuron` 的目前状态，更新资格迹变量 `et`
4. 计算 `Neuron` 的反向传播
5. 使用步骤 3 中更新的 `et` 来计算 `ETLinear` 的反向传播

显然，在步骤一前向传播的时候，需要把未更新的 `et` 输入到 `ETLinear` 中并暂存。步骤 3 需要更新 `et` 的值，容易想到使用原地修改来实现。然而，一旦原地修改了 `et`，便会导致 `et._version` 自增，在 `ETLinear` 的反向传播开始时无法通过 `et` 的版本检查。

为了更新 `et`，可以对 `et.data` 做原地修改，如下所示：

```python
B, N, M = 2, 3, 3
x, et = torch.rand(B, N), torch.zeros(B, N)
f = nn.Linear(N, M)
y = ETLinear.apply(x, et, f.weight, f.bias)
et.data.add_(torch.rand(B, N))
y.sum().backward()
```

这段代码可以成功通过版本检查，实现基于更新后的资格迹的反向传播。

## 小结

`.data` 的行为可以概括为：

1. 返回一个新的张量对象
2. 新张量和原张量**共享底层存储**
3. 新张量的**元数据完全独立**于原张量
4. 新张量被从计算图中**分离**

这些机制使 `.data` 成为一种**直接访问底层张量数据的接口**，可以绕过 PyTorch 的保护机制 (包括 `_version` 版本检查)。然而，这也意味着：如果使用不当，`.data` 可能导致意料之外的错误。使用 `.data` 时请务必小心！
