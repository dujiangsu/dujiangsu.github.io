---
layout: post
title: "CUDA Pinned Memory & Memory Manage Strategy"
tag: cuda, gpu
---

在Nvidia Jetson上编程，遇到了共享内存架构，希望能降低内存拷贝的开销，学习一下。对于CUDA架构而言，主机端内存被分为两种，一种是可分页内存（pageable memory）和页锁定内存（page-lock / pinned memory）。

<!--more-->

#### Pinned Memory

可分页内存是由操作系统API `malloc()`在主机上分配的，页锁定内存是由CUDA函数`cudaHostAlloc()`在主机内存上分配的。页锁定内存的重要属性是主机的操作系统将不会对这块内存进行分页和交换操作，确保该内存始终驻留在物理内存中。

**GPU知道页锁定内存的物理地址，可以通过“直接内存访问（Direct Memory Access，DMA）”技术直接在主机和GPU之间复制数据，速率更快**。由于每个页锁定内存都需要分配物理内存，并且这些内存不能交换到磁盘上，所以页锁定内存比使用标准malloc()分配的可分页内存更消耗内存空间。





#### About Memroy Management Strategies 

前提是CPU/GPU是physically unified memory）

**Traditional memory：**在这种策略下CUDA程序必须显式的将数据在CPU与GPU之间传递。

**Zero-copy memory：**CPU和GPU可以访问相同的内存区域，避免GPU内存分配以及CPU与GPU之间的内存拷贝。但是奇怪的是这种策略下，GPU会自动禁用所有的缓存，然而在CUDA官方文档里缺少相关描述，只在Nvidia的报告中出现过（due to their consistency model and concerns about maintaining cache coherence）。

**Unified memory:** 与zero-copy很像，CPU和GPU之间使用同一指针。但是在程序的表现上，像一个封装后的zero-copy，而且启用了cache。

综上，目前想要利用CPU/GPU共享内存特点，用Unified Memory比较好.

```c++
# Explicit Memory Manage
void * data, *d_data;
data = malloc(N);
cudaMalloc(&d_data, N);
cpu_func1(data, N);

cudaMemcpy(d_data, data, N, ...);
gpu_func2<<<...>>>(d_data, N);
cudaMemcpy(data, d_data, N, ...);

cudaFree(d_data);
free(data);
```

```c++
# Unified Memory
void * data;
data = malloc(N);

cpu_func1(data, N);

gpu_func2<<<...>>>(data, N);
cudaDeviceSynchronize();

free(data);
```



[一篇很详细的讲 Jetson × Shared Memory的blog](https://www.fastcompression.com/blog/jetson-zero-copy.htm)





