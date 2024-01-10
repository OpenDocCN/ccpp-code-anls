# `PowerInfer\powerinfer-py\powerinfer\solver.py`

```
#!/usr/bin/env python
# coding=utf-8
# 导入必要的库
import argparse
from cvxopt.glpk import ilp
import numpy as np
from cvxopt import matrix
import torch
import pickle

# 定义解决 GPU 分割问题的函数
def solve_gpu_split(
    activation_path: str,
    neuron: int,
    capacity: int,
    layer: int,
    batch: int,
    threshold: int,
):
    # 处理激活数据
    values = []
    for i in range(layer):
        # 加载并对每一层的激活数据进行排序
        freq = torch.load(f"{activation_path}/activation_{i}.pt")
        freq, _ = torch.sort(freq, descending=True)
        freq = freq * -1.0
        freq = freq.view(-1, batch)
        freq = freq.sum(dim=1)
        freq = freq.tolist()
        values += freq

    # 为额外的约束填充零值
    for i in range(layer):
        values += [0.0]
    c = np.array(values, dtype=float)
    c = matrix(c)

    # 设置容量和每批次的神经元数量
    CAP = capacity
    CAP = int(CAP / batch)
    neuron = int(neuron / batch)
    coeff = []
    h = []

    # 约束 1: 总神经元激活约束
    lst = []
    for i in range(neuron * layer):
        lst.append(1)
    for i in range(layer):
        lst.append(0)
    coeff.append(lst)
    h.append(CAP)

    # 约束 2: 每层 GPU 分割的阈值约束
    for i in range(layer):
        lst = [0] * (neuron * layer + layer)
        for j in range(neuron):
            lst[i * neuron + j] = -1
        lst[neuron * layer + i] = int(threshold / batch)
        coeff.append(lst)
        h.append(0)

    # 约束 3: 神经元激活的上限约束
    for i in range(layer):
        lst = [0] * (neuron * layer + layer)
        for j in range(neuron):
            lst[i * neuron + j] = 1
        lst[neuron * layer + i] = -1000000  # 作为上限的任意大负数
        coeff.append(lst)
        h.append(0)

    # 将列表转换为 ILP 求解器的矩阵格式
    coeff = np.array(coeff, dtype=float)
    G = matrix(coeff)
    # 将数组 h 转换为浮点数类型的 NumPy 数组
    h = np.array(h, dtype=float)
    # 将 NumPy 数组 h 转换为矩阵
    h = matrix(h)

    # 定义整数和二进制变量的集合
    I = set(range(neuron * layer + layer))
    B = set()

    # 解决整数线性规划问题
    (status, x) = ilp(c, G, h, None, None, B, I, options={'tm_lim' : 30000}) # 设置30秒的超时时间
    # 打印整数线性规划问题的求解状态
    print(f"ILP Status: {status}")
    # 将解转换为列表并打印总激活单元的数量
    ans = list(x)
    print(f"Total Activation Units: {sum(ans)}")

    # 创建一个空列表 aligned_lst
    aligned_lst = []
    # 遍历每一层，计算每层的激活单元总数并添加到 aligned_lst 中
    for i in range(layer):
        aligned_lst.append(sum(ans[i * neuron:i * neuron + neuron] * batch))

    # 返回 aligned_lst 列表作为结果
    return aligned_lst
```