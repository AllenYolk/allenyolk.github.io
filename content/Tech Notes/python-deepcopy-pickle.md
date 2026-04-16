---
title: "Python: Deep Copy and Pickle"
lang: zh-CN
tags:
  - python
  - pytorch
  - triton
draft: "false"
---
在 Python 开发时，常会遇到 `TypeError: cannot pickle '…' object`。这通常是因为我们试图 deep copy 或者序列化一个对象，但该对象的某个属性却是不可序列化的。

## Pickle 详解

### 基本概念

`pickle` 是 Python 的**序列化**协议，可将 Python 对象转换成字节流。`pickle` 包提供的接口可以对象序列化到内存或者硬盘上，也可以将内存或硬盘上的字节流转换成对象。

```python
import pickle


class A:
    def __init__(self, x):
        self.x = x
        self.y = [x, x, x]


a = A(x=10)

# To and from memory
buffer = pickle.dumps(a)
print(type(buffer)) # <class 'bytes'>
restored = pickle.loads(buffer)
print(restored.x == a.x) # True
print(restored.y == a.y) # True

# To and from disk
with open("a.pkl", "wb") as f:
    pickle.dump(a, f)
with open("a.pkl", "rb") as f:
    restored = pickle.load(f)
print(restored.x == a.x) # True
print(restored.y == a.y) # True
```

> [!note]
> `pickle.dumps` 和 `pickle.loads` 中的 `s` 代表 `str` 。在 Python 2 中，字节流是 `str` 类型；到了 Python 3，才有了单独的 `bytes` 类型。`pickle` 接口采用了老的命名风格。

容易发现，`pickle.loads` 和 `pickle.load` 不仅可以加载数据，还能够自动还原数据类别。这是因为它执行了下述步骤：

1. **识别模块和类名**：读取字节流中的元数据，发现该对象属于 `A` 这个类
2. **寻找类定义**：在==当前的 Python 运行环境==中寻找这个类。如果找到了，它会创建一个该类的新实例
3. **恢复属性和方法**：将保存好的属性值填回对象字典 `__dict__`

### 自定义序列化

可以用以下方式自定义对象序列化过程。多组协议同时被定义时，优先使用靠前的协议。

> [!note]
> 为了便于理解，这里只展示最常用的两类序列化自定义方法。欲知更多的自定义方法和完整的优先级链条，可参考 [Python Pickle文档](https://docs.python.org/3/library/pickle.html)。

#### 1. `__reduce__`

底层的控制手段。`__reduce__` 需要返回一个元组，通常包含：

1. 一个**可调用对象**，用于构造或重建对象
2. 一个**参数元组**，传递给该可调用对象

于是，为了序列化该对象，只需要序列化 `__reduce__` 返回的这个元组。

```python
class MyClass:
    def __init__(self, name):
        self.name = name

    def __reduce__(self):
        return (self.__class__, ("Fixed Name",))
```

简单来说，`__reduce__` 的含义为：*如果你不知道怎么序列化我，就请把我视作“一个构造函数 + 一组参数”*。需注意的是，如果 `__reduce__` 返回的参数元组中仍包含不可序列化对象，则序列化仍会失效！

#### 2. `__getstate__` & `__setstate__`

为了序列化该对象，只需序列化其状态字典。`__getstate__` 和 `__setstate__` 允许用户精细控制**哪些属性应该被序列化**以及**从字节流恢复时该如何初始化**。

```python
class MyModel:
	def __init__(self, x):
		self.x = x
		self.triton_kernel = get_triton_kernel() # not serializable

	def __getstate__(self):  
		state = self.__dict__.copy()  
		if 'triton_kernel' in state: 
			del state['triton_kernel']
		return state

	def __setstate__(self, state):
	    self.__dict__.update(state)
	    self.triton_kernel = get_triton_kernel() 
```

这个示例中，`triton_kernel` 不可被序列化。因此，在 `__getstate__` 中将 `triton_kernel` 从状态字典中删去，并在 `__setstate__` 中将 `triton_kernel` 添加回来即可。

#### 3. 默认序列化

直接序列化对象的 `__dict__` 字典。若字典中存在不可序列化的部分，则报错。

## Deep Copy 详解

### 基本概念

`copy.deepcopy` 不同于赋值或浅拷贝：它会**递归复制**对象的所有层级，从而获得一个和原对象完全断绝关系的新对象。

以嵌套列表为例：

```python
import copy

original = [["Alice", 95], ["Bob", 80]]
shallow_copied = copy.copy(original)
deep_copied = copy.deepcopy(original)

original[0][1] = 100 
print(f"原始数据: {original}")        # [['Alice', 100], ['Bob', 80]]
print(f"浅拷贝结果: {shallow_copied}")  # [['Alice', 100], ['Bob', 80]]
print(f"深拷贝结果: {deep_copied}")     # [['Alice', 95], ['Bob', 80]]
```

浅拷贝后，`shallow_copied[0]` 仍然指向 `original[0]`；而深拷贝后，`deep_copied[0]` 指向了一个完全独立的新子列表。

### 自定义 Deep Copy

`copy.deepcopy` 处理复杂对象时，Python 按以下优先级链条寻找具体的拷贝方法：

#### 1. `__deepcopy__`

完全绕过序列化，直接完成拷贝。该方法除了接收 `self` 以外，还接收一个 `memo` 字典，用于追踪已经拷贝的对象，防止无限递归。该方法返回的对象将作为 `copy.deepcopy(obj)` 的返回值。

```python
class FileLogger:
    def __init__(self, filename):
        self.filename = filename
        self.file = open(filename, 'a')

    def __deepcopy__(self, memo): # copy.deepcopy(obj) <- obj.__deepcopy__(memo)
        new_obj = FileLogger(self.filename)
        memo[id(self)] = new_obj
        return new_obj
```

关于 `memo` 的作用，见下面的例子：

```python
import copy

class Person:
	def __init__(self, name):
		self.name = name
		self.friend = None

	def __deepcopy__(self, memo):
		if id(self) in memo:
			return memo[id(self)]
		new_obj = BadPerson(self.name)
		new_obj.friend = copy.deepcopy(self.friend)
		return new_obj

allen = Person("Allen")
billy = Person("Billy")

allen.friend = billy
billy.friend = allen
new_allen = copy.deepcopy(allen)
```

倘若不对 `id(self) in memo` 的情况进行特判，则会导致无限递归深拷贝。

#### 2. Pickle 序列化协议

通过**序列化 + 立即反序列化**来模拟深拷贝。

按照 [[#自定义序列化]] 一节中描述的顺序来寻找序列化和反序列化方式。

1. `__reduce__`
2. `__getstate__` & `__setstate__`

注意，若上述方法没有被定义，深拷贝并不会考虑使用默认序列化方式（序列化 `__dict__`），而是降级到默认深拷贝方式。

#### 3. 默认深拷贝

递归地遍历并拷贝 `__dict__` 中的所有属性。若 `__dict__` 中存在不可深拷贝的部分，则报错。

## 案例分析

Triton 内核 `triton.JITFunction` 是一个非常贵的复杂对象，不可序列化。若 `torch.nn.Module` 的某个属性是一个 Triton 内核，则该模块无法序列化，无法被深拷贝。此时，建议采取以下对策：

1. **状态剔除**：实现 `__getstate__`，将 Triton 内核从 state 中剔除；再自定义 `__setstate__`，在反序列化时将 Triton 内核添加回来。这能同时解决深拷贝和潜在的 `torch.save` 的问题。

```python
class TritonLIF(nn.Module):
    def __init__(self, beta=0.5, detach_reset=True, *args, **kwargs):
        super().__init__()
		...
        self.sg_fn = surrogate_kernels.atan_surrogate_backward
        self.kernel = lif_ops.MultistepLIFFunction

    def __getstate__(self):
        state = self.__dict__.copy()
        del state['sg_fn']
        del state['kernel']
        return state

    def __setstate__(self, state):
        self.__dict__.update(state)
        self.sg_fn = surrogate_kernels.atan_surrogate_backward
        self.kernel = lif_ops.MultistepLIFFunction
```

2. **自定义深拷贝**：如果只需解决深拷贝问题，实现 `__deepcopy__` 并对故障属性进行“引用赋值”是最快捷的手段。