# `PowerInfer\powerinfer-py\powerinfer\solver.py`

```
#!/usr/bin/env python
# 指定解释器为 Python
# coding=utf-8
# 设置文件编码格式为 UTF-8

import argparse
# 导入命令行参数解析模块
from cvxopt.glpk import ilp
# 从 cvxopt.glpk 模块中导入整数线性规划求解器
import numpy as np
# 导入 NumPy 数学计算库
from cvxopt import matrix
# 从 cvxopt 模块中导入矩阵处理工具
import torch
# 导入 PyTorch 深度学习框架
import pickle
# 导入 pickle 序列化模块

def solve_gpu_split(
    activation_path: str,
    neuron: int,
    capacity: int,
    layer: int,
    batch: int,
    threshold: int,
):
    # 解决 GPU 分割问题的函数，接受激活数据路径、神经元数量、容量、层级、批次和阈值作为参数
    # Processing activation data
    # 处理激活数据
    values = []
    # 创建空列表用于存储处理后的数据
    for i in range(layer):
    # 遍历层级数量的次数
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
    # 创建一个空列表用于存储约束条件
    h = []

    # 约束条件1：总神经元激活约束
    lst = []
    # 为每个神经元的激活添加约束值为1
    for i in range(neuron * layer):
        lst.append(1)
    # 为每一层添加约束值为0
    for i in range(layer):
        lst.append(0)
    # 将约束条件添加到系数列表中
    coeff.append(lst)
    # 将总神经元激活约束值添加到约束值列表中
    h.append(CAP)

    # 约束条件2：每层GPU分割的阈值约束
    for i in range(layer):
        # 创建一个长度为神经元数乘以层数加上层数的零列表
        lst = [0] * (neuron * layer + layer)
        # 将当前层的神经元的系数设为-1
        for j in range(neuron):
            lst[i * neuron + j] = -1
        # 将当前层的阈值系数设为阈值除以批次大小
        lst[neuron * layer + i] = int(threshold / batch)
        # 将约束条件添加到系数列表中
        coeff.append(lst)
        # 将约束值添加到约束值列表中
        h.append(0)
    # Constraint 3: Upper bound on neuron activations
    # 对神经元激活的上限进行约束
    for i in range(layer):
        # 创建一个长度为神经元数量乘以层数加上层数的列表，并初始化为0
        lst = [0] * (neuron * layer + layer)
        for j in range(neuron):
            # 将特定位置的值设为1，表示神经元激活的上限
            lst[i * neuron + j] = 1
        # 将神经元激活的上限设为一个极大的负数
        lst[neuron * layer + i] = -1000000  # Arbitrary large negative number as an upper bound
        # 将约束条件添加到系数列表中
        coeff.append(lst)
        # 将h列表中添加一个0，表示约束条件的结果
        h.append(0)

    # Convert lists to matrix format for ILP solver
    # 将列表转换为矩阵格式，以便ILP求解器处理
    coeff = np.array(coeff, dtype=float)
    G = matrix(coeff)
    h = np.array(h, dtype=float)
    h = matrix(h)

    # Define the set of integer and binary variables
    # 定义整数和二进制变量的集合
    I = set(range(neuron * layer + layer)
    B = set()

    # Solving the ILP problem
    # 解决ILP问题
    # 调用 ILP 函数求解整数线性规划问题，传入参数 c, G, h, B, I，并设置超时时间为30秒
    (status, x) = ilp(c, G, h, None, None, B, I, options={'tm_lim' : 30000}) # with 30s timeout
    # 打印 ILP 求解的状态
    print(f"ILP Status: {status}")
    # 将 x 转换为列表
    ans = list(x)
    # 打印总激活单元的数量
    print(f"Total Activation Units: {sum(ans)}")

    # 初始化一个空列表用于存储对齐后的结果
    aligned_lst = []
    # 遍历每一层的神经元
    for i in range(layer):
        # 计算每一层的激活单元总数，并乘以批次大小，然后添加到 aligned_lst 中
        aligned_lst.append(sum(ans[i * neuron:i * neuron + neuron] * batch))

    # 返回对齐后的结果列表
    return aligned_lst
```