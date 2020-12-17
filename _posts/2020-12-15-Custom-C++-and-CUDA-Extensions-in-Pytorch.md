---
layout: post
title: "CUSTOM C++ AND CUDA EXTENSIONS in Pytorch"
tag: pytorch
---

学习一下Pytorch官方文档，记录一些重点。

<!--more-->

#### Building with `setuptools`

`setuptools`是用一个用于python库的快速发布、编译的工具。

利用`setuptools`之前，先写一个`setup.py`用于编译C++ Code。

```python
from setuptools import setup, Extension
from torch.utils import cpp_extension

setup(name='lltm_cpp',
      ext_modules=[cpp_extension.CppExtension('lltm_cpp', ['lltm.cpp'])],
      cmdclass={'build_ext': cpp_extension.BuildExtension})
```

**CppExtension**是一个pytorch实现的wrapper，将pytorch的各种路径传递给C++。



```python
Extension(
   name='lltm_cpp',
   sources=['lltm.cpp'],
   include_dirs=cpp_extension.include_paths(),
   language='c++')
# 与上面等价，但是没怎么看懂，但不影响下面具体的C++ Kernel实现。
```



#### Writing the C++ Op

**Small case for easy understanding:**

```c++
#include <torch/extension.h> 
// <torch/extension.h> is the one-stop header to include all the necessary PyTorch bits to write C++ extensions
#include <iostream>

torch::Tensor d_sigmoid(torch::Tensor z){
    auto s = torch::sigmoid(z);
    return (1 - s) * s;
}
```

`<torch/extension.h>` 包括：

- ATen library, 是pytorch中已经实现的各种Tensor操作。
- [<font color=#ed5a65>pybind11</font>](https://github.com/pybind/pybind11)用于对C++11和Python进行无缝的胶合， create Python bindings for our C++ code。
- 管理ATen和pybind11之间交互的Headers。

这里，就利用ATen已有的sigmoid操作实现了d_sigmoid。其中用于操作的数据类型都是`torch::Tensor`。



**Forward Pass of LLTM**

```c++
#include <vector>
std::vector<at::Tensor> lltm_forward(torch::Tensor input, torch::Tensor weights,
                                     torch::Tensor bias, torch::Tensor old_h,
                                     torch::Tensor old_cell) 
{
    auto X = torch::cat({old_h, input}, /*dim*/1);
    auto gate_weights = torch::addmm(bias, X, weights.transpose(0, 1));
  	auto gates = gate_weights.chunk(3, /*dim=*/1);

  	auto input_gate = torch::sigmoid(gates[0]);
  	auto output_gate = torch::sigmoid(gates[1]);
  	auto candidate_cell = torch::elu(gates[2], /*alpha=*/1.0);

  	auto new_cell = old_cell + candidate_cell * input_gate;
  	auto new_h = torch::tanh(new_cell) * output_gate;
    
    return {new_h, new_cell, input_gate, output_gate, candidate_cell, X, gate_weights};
}
```



**Backward Pass**

C++ extension API 是不提供自动生产backward function的。因此，我们要手动生成，根据每个输入的前向传播计算损失的倒数。在编写完backward后，将两个函数都放到`torch.autograd.Functin`中生成Python binding。

```c++
// tanh'(z) = 1 - tanh^2(z)
torch::Tensor d_tanh(torch::Tensor z) {
  return 1 - z.tanh().pow(2);
}

// elu'(z) = relu'(z) + { alpha * exp(z) if (alpha * (exp(z) - 1)) < 0, else 0}
torch::Tensor d_elu(torch::Tensor z, torch::Scalar alpha = 1.0) {
  auto e = z.exp();
  auto mask = (alpha * (e - 1)) < 0;
  return (z > 0).type_as(z) + mask.type_as(z) * (alpha * e);
}

std::vector<torch::Tensor> lltm_backward(torch::Tensor grad_h, torch::Tensor grad_cell,
           torch::Tensor new_cell, torch::Tensor input_gate,
           torch::Tensor output_gate, torch::Tensor candidate_cell,
           torch::Tensor X, torch::Tensor gate_weights, 
           torch::Tensor weights)
{
    auto d_output_gate = torch::tanh(new_cell) * grad_h;
  	auto d_tanh_new_cell = output_gate * grad_h;
  	auto d_new_cell = d_tanh(new_cell) * d_tanh_new_cell + grad_cell;

  	auto d_old_cell = d_new_cell;
  	auto d_candidate_cell = input_gate * d_new_cell;
  	auto d_input_gate = candidate_cell * d_new_cell;

  	auto gates = gate_weights.chunk(3, /*dim=*/1);
  	d_input_gate *= d_sigmoid(gates[0]);
  	d_output_gate *= d_sigmoid(gates[1]);
  	d_candidate_cell *= d_elu(gates[2]);

  	auto d_gates =
      torch::cat({d_input_gate, d_output_gate, d_candidate_cell}, /*dim=*/1);

  	auto d_weights = d_gates.t().mm(X);
  	auto d_bias = d_gates.sum(/*dim=*/0, /*keepdim=*/true);

  	auto d_X = d_gates.mm(weights);
  	const auto state_size = grad_h.size(1);
  	auto d_old_h = d_X.slice(/*dim=*/1, 0, state_size);
  	auto d_input = d_X.slice(/*dim=*/1, state_size);

  	return {d_old_h, d_input, d_weights, d_bias, d_old_cell};
}
```



**Binding to Python**

Use pybind11 to bind your C++ functions or classes into Python.

```c++
PYBIND11_MODULE(TORCH_EXTENSION_NAME, m){
    m.def("forward", &lltm_forward, "LLTM forward");
    m.def("backward", &lltm_backward, "LLTM backward");
}
```



此处忽略一部分性能对比和使用方法，直接看下面的Reference



** Performance on GPU Devices**

ATen的原生方法都是可以直接利用python中的

- .device()

- .cuda()

- .cpu() 

  来迁移到不同设备进行计算的。







#### Reference

https://pytorch.org/tutorials/advanced/cpp_extension.html