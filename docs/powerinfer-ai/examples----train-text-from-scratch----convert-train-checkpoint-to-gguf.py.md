# `PowerInfer\examples\train-text-from-scratch\convert-train-checkpoint-to-gguf.py`

```
#!/usr/bin/env python3
# 设置脚本的解释器为 Python 3

import argparse  # 导入用于解析命令行参数的模块
import os  # 导入操作系统相关的模块
import struct  # 导入用于处理字节数据的模块
import sys  # 导入系统相关的模块
import numpy as np  # 导入用于科学计算的模块，并将其命名为 np
from pathlib import Path  # 从 pathlib 模块中导入 Path 类

if 'NO_LOCAL_GGUF' not in os.environ:
    # 如果环境变量中没有定义 NO_LOCAL_GGUF，则将 gguf-py 目录添加到系统路径中
    sys.path.insert(1, str(Path(__file__).parent / '..' / '..' / 'gguf-py'))
import gguf  # 导入 gguf 模块

# gguf constants
LLM_KV_OPTIMIZER_TYPE = "optimizer.type"  # 定义优化器类型的常量
LLM_KV_OPTIMIZER_TYPE_ADAM  = "adam"  # 定义 Adam 优化器类型的常量
LLM_KV_OPTIMIZER_TYPE_LBFGS = "lbfgs"  # 定义 LBFGS 优化器类型的常量
LLM_KV_OPTIMIZER_FILE_VERSION = "optimizer.file_version"  # 定义优化器文件版本的常量
LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT = "optimizer.convergence_past_count"  # 定义优化器收敛过去计数的常量
# 定义优化器参数计数的键
LLM_KV_OPTIMIZER_PARAMETER_COUNT = "optimizer.parameter_count"
# 定义优化器迭代次数的键
LLM_KV_OPTIMIZER_ITERATION_COUNT = "optimizer.iteration_count"
# 定义优化器是否刚初始化的键
LLM_KV_OPTIMIZER_JUST_INITIALIZED = "optimizer.just_initialized"
# 定义Adam优化器最佳损失的键
LLM_KV_OPTIMIZER_ADAM_BEST_LOSS = "optimizer.adam.best_loss"
# 定义Adam优化器先前损失的键
LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS = "optimizer.adam.previous_loss"
# 定义Adam优化器无改善计数的键
LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT = "optimizer.adam.no_improvement_count"
# 定义LBFGS优化器近似Hessian矩阵计数的键
LLM_KV_OPTIMIZER_LBFGS_APPROX_HESSIAN_COUNT = "optimizer.lbfgs.approx_hessian_count"
# 定义LBFGS优化器最佳损失的键
LLM_KV_OPTIMIZER_LBFGS_BEST_LOSS = "optimizer.lbfgs.best_loss"
# 定义LBFGS优化器线搜索步长的键
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_STEP = "optimizer.lbfgs.line_search_step"
# 定义LBFGS优化器线搜索j的键
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_J = "optimizer.lbfgs.line_search_j"
# 定义LBFGS优化器线搜索k的键
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_K = "optimizer.lbfgs.line_search_k"
# 定义LBFGS优化器线搜索结束的键
LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_END = "optimizer.lbfgs.line_search_end"
# 定义LBFGS优化器无改善计数的键
LLM_KV_OPTIMIZER_LBFGS_NO_IMPROVEMENT_COUNT = "optimizer.lbfgs.no_improvement_count"

# 定义Adam优化器第一矩张量的键
LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS = "optimizer.adam.first_moments"
# 定义Adam优化器第二矩张量的键
LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS = "optimizer.adam.second_moments"
# 定义Adam优化器先前损失值的键
LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES = "optimizer.adam.past_loss_values"

# 定义LBFGS优化器当前参数的键
LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS = "optimizer.lbfgs.current_parameters"
# 定义LBFGS优化器先前参数的键
LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS = "optimizer.lbfgs.previous_parameters"
# 定义一系列常量，用于表示优化器的不同参数和训练类型的键值
LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS   = "optimizer.lbfgs.current_gradients"
LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS  = "optimizer.lbfgs.previous_gradients"
LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION    = "optimizer.lbfgs.search_direction"
LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES    = "optimizer.lbfgs.past_loss_values"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA        = "optimizer.lbfgs.memory_alpha"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS           = "optimizer.lbfgs.memory_ys"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S            = "optimizer.lbfgs.memory_s"
LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y            = "optimizer.lbfgs.memory_y"

LLM_KV_TRAINING_TYPE_TRAIN_MODEL   = "train_model"
LLM_KV_TRAINING_TYPE_FINETUNE_LORA = "finetune_lora"
LLM_KV_TRAINING_TYPE               = "training.type"
LLM_KV_TRAINING_FILE_VERSION       = "training.file_version"
LLM_KV_TRAINING_ITERATION_COUNT    = "training.iteration_count"
LLM_KV_TRAINING_SAMPLE_COUNT       = "training.sample_count"
LLM_KV_TRAINING_TOKEN_COUNT        = "training.token_count"

# 定义一个名为Tensor的类
class Tensor:
    # 初始化方法，可以接受dtype和ne两个参数
    def __init__(self, dtype='f', ne=None):
        # 如果ne参数为None，则执行以下操作
# 创建一个空列表ne
ne = []
# 设置对象的数据类型为传入的dtype
self.dtype = dtype
# 设置对象的ne属性为传入的ne
self.ne = ne
# 初始化对象的字节数为0
self.nbytes = 0
# 如果数据类型为'f'
if self.dtype == 'f':
    # 如果ne列表为空
    if len(self.ne) == 0:
        # 字节数为0
        self.nbytes = 0
    else:
        # 否则，根据ne列表中元素的乘积计算字节数
        self.nbytes = int(np.product(self.ne)) * 4
# 如果数据类型不是'f'，抛出数值错误异常
else:
    raise ValueError(f"Unhandled data type '{self.dtype}'")

# 加载数据的方法，传入数据和偏移量
def load(self, data, offset):
    # 从数据中解包出nd，表示维度的数量；然后更新偏移量
    nd = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
    # 从数据中解包出namelen，表示名称长度；然后更新偏移量
    namelen = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
    # 从数据中解包出dtype，表示数据类型；然后更新偏移量
    dtype = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4

    # 断言nd等于ne列表的长度
    assert(nd == len(self.ne))
    # 创建一个空列表ne
    ne = []
    # 遍历nd次
    for d in range(nd):
# 从数据中解析出一个无符号整数，使用小端字节顺序，更新偏移量
n = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
# 将解析出的整数添加到列表中
ne.append(n)

# 断言解析出的整数列表与对象属性中的整数元组相等
assert(tuple(ne) == tuple(self.ne))

# 如果对象的数据类型为 'f'，则断言解析出的数据类型为 0
if self.dtype == 'f':
    assert(dtype == 0)
# 否则，抛出数值错误，指明未处理的数据类型
else:
    raise ValueError(f"Unhandled data type '{self.dtype}'")

# 从数据中解析出名称字节流，更新偏移量
self.name = bytes(data[offset:offset+namelen]); offset += namelen
# 32 字节对齐
offset += (0 - offset) & 31
# 从数据中解析出数据，更新偏移量
self.data = data[offset:offset+self.nbytes]
offset += self.nbytes
# 返回更新后的偏移量
return offset

# 计算对象的最大存储大小
def max_storage_size(self):
    result = 0
    # 增加 4 字节的存储大小，表示 nd
    result += 4 # nd
        # 增加4，表示namelen的长度
        result += 4 # namelen
        # 增加4，表示dtype的长度
        result += 4 # dtype
        # 增加self.ne列表长度乘以8，表示ne的长度
        result += len(self.ne)*8 # ne
        # 增加48，表示name的长度（截至到提交3b5515bbe0e2224425986ba24f1f5d84aa38dce9时的最大值）
        result += 48 # name (maximum as of commit 3b5515bbe0e2224425986ba24f1f5d84aa38dce9)
        # 增加31，表示32字节对齐
        result += 31 # 32-byte alignment
        # 增加self.nbytes，表示数据的长度
        result += self.nbytes
        # 返回结果
        return result

    # 保存gguf数据
    def save_gguf(self, gguf_writer, name):
        # 将张量添加到gguf_writer中
        gguf_writer.add_tensor(
            name=name,
            tensor=self.data,
            raw_shape=np.array(list(reversed(self.ne))),
            raw_dtype=gguf.GGMLQuantizationType.F32)

# 优化参数类
class OptimizationParamsV0:
    # 初始化方法
    def __init__(self):
        pass

    # 加载数据的方法
    def load(self, data, offset):
# 从数据中解析出一个无符号整数，存入self.type中，并更新偏移量
self.type = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个有符号整数，存入self.n_threads中，并更新偏移量
self.n_threads = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个有符号整数，存入self.past中，并更新偏移量
self.past = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.delta中，并更新偏移量
self.delta = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个布尔值，存入self.print_forward_graph中，并更新偏移量
self.print_forward_graph = struct.unpack('<?', bytes(data[offset:offset + 1]))[0]; offset += 4 # 32bit-aligned
# 从数据中解析出一个布尔值，存入self.print_backward_graph中，并更新偏移量
self.print_backward_graph = struct.unpack('<?', bytes(data[offset:offset + 1]))[0]; offset += 4 # 32bit-aligned
# 从数据中解析出一个有符号整数，存入self.adam_n_iter中，并更新偏移量
self.adam_n_iter = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.adam_sched中，并更新偏移量
self.adam_sched = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.adam_decay中，并更新偏移量
self.adam_decay = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.adam_alpha中，并更新偏移量
self.adam_alpha = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.adam_beta1中，并更新偏移量
self.adam_beta1 = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.adam_beta2中，并更新偏移量
self.adam_beta2 = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.adam_eps中，并更新偏移量
self.adam_eps = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.adam_eps_f中，并更新偏移量
self.adam_eps_f = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.adam_eps_g中，并更新偏移量
self.adam_eps_g = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个有符号整数，存入self.lbfgs_m中，并更新偏移量
self.lbfgs_m = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个有符号整数，存入self.lbfgs_n_iter中，并更新偏移量
self.lbfgs_n_iter = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个有符号整数，存入self.lbfgs_max_linesearch中，并更新偏移量
self.lbfgs_max_linesearch = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.lbfgs_eps中，并更新偏移量
self.lbfgs_eps = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解析出一个浮点数，存入self.lbfgs_ftol中，并更新偏移量
self.lbfgs_ftol = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中按照指定格式解析出 lbfgs_wolfe 的值，并更新偏移量
self.lbfgs_wolfe = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中按照指定格式解析出 lbfgs_min_step 的值，并更新偏移量
self.lbfgs_min_step = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中按照指定格式解析出 lbfgs_max_step 的值，并更新偏移量
self.lbfgs_max_step = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中按照指定格式解析出 lbfgs_linesearch 的值，并更新偏移量
self.lbfgs_linesearch = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
# 返回更新后的偏移量
return offset

# 定义 OptimizationContext 类
class OptimizationContext:
    # 初始化方法
    def __init__(self):
        pass

    # 加载数据的方法
    def load(self, data, offset):
        # 从数据中按照指定格式解析出 version 的值，并更新偏移量
        self.version = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]
        offset += 4

        # 如果 version 为 0
        if self.version == 0:
            # 创建 OptimizationParamsV0 对象
            params = OptimizationParamsV0()
            # 调用 params 对象的 load 方法，更新偏移量
            offset = params.load(data, offset)
            # 更新 self.past 的值
            self.past = params.past
            # 更新 self.lbfgs_m 的值
            self.lbfgs_m = params.lbfgs_m
            # 从数据中按照指定格式解析出 nx 的值，并更新偏移量
            self.nx = struct.unpack('N', bytes(data[offset:offset + 8]))[0];  offset += 8
# 从数据中解析出迭代次数，使用小端字节序，4个字节
self.iter = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
# 从数据中解析出是否刚初始化，使用小端字节序，4个字节，然后转换为布尔值
self.just_initialized = bool(struct.unpack('<i', bytes(data[offset:offset + 4]))[0]);  offset += 4
# 设置参数类型
self.type = params.type

# 初始化各种张量
self.adam_m  = Tensor('f', [self.nx])
self.adam_v  = Tensor('f', [self.nx])
self.adam_pf = Tensor('f', [self.past] if self.past > 0 else [])
self.lbfgs_x    = Tensor('f', [self.nx])
self.lbfgs_xp   = Tensor('f', [self.nx])
self.lbfgs_g    = Tensor('f', [self.nx])
self.lbfgs_gp   = Tensor('f', [self.nx])
self.lbfgs_d    = Tensor('f', [self.nx])
self.lbfgs_pf   = Tensor('f', [self.past] if self.past > 0 else [])
self.lbfgs_lmal = Tensor('f', [self.lbfgs_m])
self.lbfgs_lmys = Tensor('f', [self.lbfgs_m])
self.lbfgs_lms  = Tensor('f', [self.nx, self.lbfgs_m])
self.lbfgs_lmy  = Tensor('f', [self.nx, self.lbfgs_m])

# 如果参数类型为0，则执行以下操作
if self.type == 0:
# 创建存储数据的张量，但不需要它们的数据
x  = Tensor('f', [self.nx])
g  = Tensor('f', [self.nx])
g2 = Tensor('f', [self.nx])
mh = Tensor('f', [self.nx])
vh = Tensor('f', [self.nx])

# 从数据中加载偏移量，并更新偏移量
offset = x.load(data, offset)
offset = g.load(data, offset)
offset = g2.load(data, offset)
offset = self.adam_m.load(data, offset)
offset = self.adam_v.load(data, offset)
offset = mh.load(data, offset)
offset = vh.load(data, offset)
offset = self.adam_pf.load(data, offset)

# 从数据中解包并加载 Adam 优化器的最佳值、上一次的值和未改进次数
self.adam_fx_best          = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
self.adam_fx_prev          = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
self.adam_n_no_improvement = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
# 如果类型为1，依次加载lbfgs_x、lbfgs_xp、lbfgs_g、lbfgs_gp、lbfgs_d、lbfgs_pf、lbfgs_lmal、lbfgs_lmys、lbfgs_lms、lbfgs_lmy，并更新偏移量
elif self.type == 1:
    offset = self.lbfgs_x.load(data, offset)
    offset = self.lbfgs_xp.load(data, offset)
    offset = self.lbfgs_g.load(data, offset)
    offset = self.lbfgs_gp.load(data, offset)
    offset = self.lbfgs_d.load(data, offset)
    offset = self.lbfgs_pf.load(data, offset)
    offset = self.lbfgs_lmal.load(data, offset)
    offset = self.lbfgs_lmys.load(data, offset)
    offset = self.lbfgs_lms.load(data, offset)
    offset = self.lbfgs_lmy.load(data, offset)

    # 依次解包数据，更新偏移量，并赋值给对应的属性
    self.lbfgs_fx_best          = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
    self.lbfgs_step             = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
    self.lbfgs_j                = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
    self.lbfgs_k                = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
    self.lbfgs_end              = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
    self.lbfgs_n_no_improvement = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4

# 如果类型不为1，执行其他操作
else:
# 如果版本号为1，则解析数据并初始化各个变量
elif self.version == 1:
    # 解析数据中的过去值
    self.past    = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
    # 解析数据中的 LBFGS_m 值
    self.lbfgs_m = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
    # 解析数据中的 nx 值
    self.nx      = struct.unpack('N',  bytes(data[offset:offset + 8]))[0];  offset += 8
    # 解析数据中的迭代次数
    self.iter    = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4
    # 解析数据中的初始化标志
    self.just_initialized = bool(struct.unpack('<i', bytes(data[offset:offset + 4]))[0]);  offset += 4

    # 初始化 adam_m, adam_v, adam_pf, lbfgs_x, lbfgs_xp, lbfgs_g, lbfgs_gp, lbfgs_d, lbfgs_pf 变量
    self.adam_m  = Tensor('f', [self.nx])
    self.adam_v  = Tensor('f', [self.nx])
    self.adam_pf = Tensor('f', [self.past] if self.past > 0 else [])

    self.lbfgs_x    = Tensor('f', [self.nx])
    self.lbfgs_xp   = Tensor('f', [self.nx])
    self.lbfgs_g    = Tensor('f', [self.nx])
    self.lbfgs_gp   = Tensor('f', [self.nx])
    self.lbfgs_d    = Tensor('f', [self.nx])
    self.lbfgs_pf   = Tensor('f', [self.past] if self.past > 0 else [])
# 创建 lbfgs_lmal 张量，数据类型为 'f'，大小为 lbfgs_m
self.lbfgs_lmal = Tensor('f', [self.lbfgs_m])
# 创建 lbfgs_lmys 张量，数据类型为 'f'，大小为 lbfgs_m
self.lbfgs_lmys = Tensor('f', [self.lbfgs_m])
# 创建 lbfgs_lms 张量，数据类型为 'f'，大小为 nx * lbfgs_m
self.lbfgs_lms  = Tensor('f', [self.nx, self.lbfgs_m])
# 创建 lbfgs_lmy 张量，数据类型为 'f'，大小为 nx * lbfgs_m
self.lbfgs_lmy  = Tensor('f', [self.nx, self.lbfgs_m])

# 在版本 1 中忘记保存类型：
# 从剩余字节数推测 self.type
size_type_0 = 12 + sum([t.max_storage_size() for t in
                        [self.adam_m, self.adam_v]
                        +([self.adam_pf] if (self.past > 0) else [])])
size_type_1 = 24 + sum([t.max_storage_size() for t in
                        [self.lbfgs_x, self.lbfgs_xp, self.lbfgs_g,
                         self.lbfgs_gp, self.lbfgs_d, self.lbfgs_pf,
                         self.lbfgs_lmal, self.lbfgs_lmys,
                         self.lbfgs_lms, self.lbfgs_lmy]
                         +([self.lbfgs_pf] if (self.past > 0) else [])])
# 由于对齐填充，大小可能不是精确的
# 但两种类型的大小差异很大，
# 所以我们可以使用最接近的那个
remaining = len(data) - offset
# 如果 type_0 和 type_1 两种类型的大小差距与 remaining 的差距比较，选择较小的那个作为 self.type 的值
if abs(remaining - size_type_0) < abs(remaining - size_type_1):
    self.type = 0
else:
    self.type = 1

# 如果 self.type 的值为 0，加载相应的数据到对应的对象，并更新偏移量
if self.type == 0:
    offset = self.adam_m.load(data, offset)
    offset = self.adam_v.load(data, offset)
    offset = self.adam_pf.load(data,offset)

    # 从数据中解析出浮点数，并更新偏移量
    self.adam_fx_best          = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
    self.adam_fx_prev          = struct.unpack('<f', bytes(data[offset:offset + 4]))[0];  offset += 4
    self.adam_n_no_improvement = struct.unpack('<i', bytes(data[offset:offset + 4]))[0];  offset += 4

# 如果 self.type 的值为 1，加载相应的数据到对应的对象，并更新偏移量
elif self.type == 1:
    offset = self.lbfgs_x.load(data, offset)
    offset = self.lbfgs_xp.load(data, offset)
    offset = self.lbfgs_g.load(data, offset)
    offset = self.lbfgs_gp.load(data, offset)
    offset = self.lbfgs_d.load(data, offset)
# 从数据中加载 lbfgs_pf，并更新偏移量
offset = self.lbfgs_pf.load(data, offset)
# 从数据中加载 lbfgs_lmal，并更新偏移量
offset = self.lbfgs_lmal.load(data, offset)
# 从数据中加载 lbfgs_lmys，并更新偏移量
offset = self.lbfgs_lmys.load(data, offset)
# 从数据中加载 lbfgs_lms，并更新偏移量
offset = self.lbfgs_lms.load(data, offset)
# 从数据中加载 lbfgs_lmy，并更新偏移量
offset = self.lbfgs_lmy.load(data, offset)

# 从数据中解包 lbfgs_fx_best，并更新偏移量
self.lbfgs_fx_best = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解包 lbfgs_step，并更新偏移量
self.lbfgs_step = struct.unpack('<f', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解包 lbfgs_j，并更新偏移量
self.lbfgs_j = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解包 lbfgs_k，并更新偏移量
self.lbfgs_k = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解包 lbfgs_end，并更新偏移量
self.lbfgs_end = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从数据中解包 lbfgs_n_no_improvement，并更新偏移量
self.lbfgs_n_no_improvement = struct.unpack('<i', bytes(data[offset:offset + 4]))[0]; offset += 4

# 如果不满足条件，则抛出数值错误异常
else:
    raise ValueError('Invalid version of checkpoint file')

# 返回更新后的偏移量
return offset

# 将 gguf_writer 添加 LLM_KV_OPTIMIZER_FILE_VERSION 和 0
def save_gguf(self, gguf_writer):
    gguf_writer.add_uint32(LLM_KV_OPTIMIZER_FILE_VERSION, 0)
# 将整数值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT作为键
gguf_writer.add_uint32(LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT, self.past)
# 将整数值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_PARAMETER_COUNT作为键
gguf_writer.add_uint64(LLM_KV_OPTIMIZER_PARAMETER_COUNT, self.nx)
# 将整数值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_ITERATION_COUNT作为键
gguf_writer.add_uint32(LLM_KV_OPTIMIZER_ITERATION_COUNT, self.iter)
# 将布尔值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_JUST_INITIALIZED作为键
gguf_writer.add_bool(LLM_KV_OPTIMIZER_JUST_INITIALIZED, self.just_initialized)

# 如果self.type等于0，则执行以下操作
if self.type == 0:
    # 将字符串值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_TYPE作为键
    gguf_writer.add_string(LLM_KV_OPTIMIZER_TYPE, LLM_KV_OPTIMIZER_TYPE_ADAM)
    # 将浮点数值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_ADAM_BEST_LOSS作为键
    gguf_writer.add_float32(LLM_KV_OPTIMIZER_ADAM_BEST_LOSS, self.adam_fx_best)
    # 将浮点数值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS作为键
    gguf_writer.add_float32(LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS, self.adam_fx_prev)
    # 将整数值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT作为键
    gguf_writer.add_uint32(LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT, self.adam_n_no_improvement)

    # 调用adam_m对象的save_gguf方法，将数据保存到gguf_writer中
    self.adam_m.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS)
    # 调用adam_v对象的save_gguf方法，将数据保存到gguf_writer中
    self.adam_v.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS)
    # 如果self.past大于0，则执行以下操作
    if self.past > 0:
        # 调用adam_pf对象的save_gguf方法，将数据保存到gguf_writer中
        self.adam_pf.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES)

# 如果self.type等于1，则执行以下操作
elif self.type == 1:
    # 将字符串值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_TYPE作为键
    gguf_writer.add_string(LLM_KV_OPTIMIZER_TYPE, LLM_KV_OPTIMIZER_TYPE_LBFGS)
    # 将整数值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_LBFGS_APPROX_HESSIAN_COUNT作为键
    gguf_writer.add_uint32(LLM_KV_OPTIMIZER_LBFGS_APPROX_HESSIAN_COUNT, self.lbfgs_m)
    # 将浮点数值添加到gguf_writer中，使用LLM_KV_OPTIMIZER_LBFGS_BEST_LOSS作为键
    gguf_writer.add_float32(LLM_KV_OPTIMIZER_LBFGS_BEST_LOSS, self.lbfgs_fx_best)
# 将浮点数添加到gguf_writer中，键为LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_STEP，值为self.lbfgs_step
gguf_writer.add_float32(LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_STEP, self.lbfgs_step)
# 将整数添加到gguf_writer中，键为LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_J，值为self.lbfgs_j
gguf_writer.add_int32(LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_J, self.lbfgs_j)
# 将整数添加到gguf_writer中，键为LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_K，值为self.lbfgs_k
gguf_writer.add_int32(LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_K, self.lbfgs_k)
# 将整数添加到gguf_writer中，键为LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_END，值为self.lbfgs_end
gguf_writer.add_int32(LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_END, self.lbfgs_end)
# 将无符号整数添加到gguf_writer中，键为LLM_KV_OPTIMIZER_LBFGS_NO_IMPROVEMENT_COUNT，值为self.lbfgs_n_no_improvement
gguf_writer.add_uint32(LLM_KV_OPTIMIZER_LBFGS_NO_IMPROVEMENT_COUNT, self.lbfgs_n_no_improvement)

# 将lbfgs_x对象保存到gguf_writer中，键为LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS
self.lbfgs_x.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS)
# 将lbfgs_xp对象保存到gguf_writer中，键为LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS
self.lbfgs_xp.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS)
# 将lbfgs_g对象保存到gguf_writer中，键为LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS
self.lbfgs_g.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS)
# 将lbfgs_gp对象保存到gguf_writer中，键为LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS
self.lbfgs_gp.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS)
# 将lbfgs_d对象保存到gguf_writer中，键为LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION
self.lbfgs_d.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION)
# 如果self.past大于0，则将lbfgs_pf对象保存到gguf_writer中，键为LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES
if self.past > 0:
    self.lbfgs_pf.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES)
# 将lbfgs_lmal对象保存到gguf_writer中，键为LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA
self.lbfgs_lmal.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA)
# 将lbfgs_lmys对象保存到gguf_writer中，键为LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS
self.lbfgs_lmys.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS)
# 将lbfgs_lms对象保存到gguf_writer中，键为LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S
self.lbfgs_lms.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S)
# 将lbfgs_lmy对象保存到gguf_writer中，键为LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y
self.lbfgs_lmy.save_gguf(gguf_writer, name=LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y)
# 如果不满足以上条件，则抛出值错误异常，提示未知的优化器类型
else:
    raise ValueError('Unknown optimizer type')
# 定义一个模型参数类
class ModelParams:
    def __init__(self):
        pass

    # 从给定数据和偏移量加载模型参数
    def load(self, data, offset):
        # 从数据中解析出词汇量大小，并更新偏移量
        self.n_vocab = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中解析出嵌入维度大小，并更新偏移量
        self.n_embd  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中解析出多头注意力的数量，并更新偏移量
        self.n_mult  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中解析出注意力头的数量，并更新偏移量
        self.n_head  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中解析出层数，并更新偏移量
        self.n_layer = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        # 从数据中解析出旋转数量，并更新偏移量
        self.n_rot   = struct.unpack('<I', bytes(data[offset:offset + 4]))[0];  offset += 4
        return offset

    # 计算前馈神经网络的隐藏层大小
    def get_n_ff(self):
        # 根据公式计算前馈神经网络的隐藏层大小
        return ((2*(4*self.n_embd)//3 + self.n_mult - 1)//self.n_mult)*self.n_mult

    # 保存 GGUF（待补充）
    def save_gguf(self, gguf_writer):
        # 不保存 self.n_vocab
        gguf_writer.add_embedding_length(self.n_embd)
# 将头部数量添加到gguf_writer中
gguf_writer.add_head_count(self.n_head)
# 将层数量添加到gguf_writer中
gguf_writer.add_block_count(self.n_layer)
# 将绳索维度数量添加到gguf_writer中
gguf_writer.add_rope_dimension_count(self.n_rot)
# 将前馈长度添加到gguf_writer中
gguf_writer.add_feed_forward_length(self.get_n_ff())

# 根据键和块ID返回张量名称
def tensor_name(key, bid=None):
    return gguf.TENSOR_NAMES[key].format(bid=bid) + ".weight"

# 定义Layer类
class Layer:
    # 初始化方法
    def __init__(self, params, bid):
        # 设置属性bid
        self.bid = bid
        # 初始化属性att_norm
        self.att_norm = Tensor('f', [params.n_embd])
        # 初始化属性wq
        self.wq       = Tensor('f', [params.n_embd, params.n_embd])
        # 初始化属性wk
        self.wk       = Tensor('f', [params.n_embd, params.n_embd])
        # 初始化属性wv
        self.wv       = Tensor('f', [params.n_embd, params.n_embd])
        # 初始化属性wo
        self.wo       = Tensor('f', [params.n_embd, params.n_embd])
        # 初始化属性ffn_norm
        self.ffn_norm = Tensor('f', [params.n_embd])
        # 初始化属性w1
        self.w1       = Tensor('f', [params.n_embd, params.get_n_ff()])
        # 初始化属性w2
        self.w2       = Tensor('f', [params.get_n_ff(), params.n_embd])
        # 初始化属性w3
        self.w3       = Tensor('f', [params.n_embd, params.get_n_ff()])
# 从给定数据和偏移量加载模型参数，更新偏移量并返回
def load(self, data, offset):
    # 加载注意力层的参数
    offset = self.att_norm.load(data, offset)
    # 加载查询权重参数
    offset = self.wq.load(data, offset)
    # 加载键权重参数
    offset = self.wk.load(data, offset)
    # 加载值权重参数
    offset = self.wv.load(data, offset)
    # 加载输出权重参数
    offset = self.wo.load(data, offset)
    # 加载前馈神经网络层的参数
    offset = self.ffn_norm.load(data, offset)
    # 加载第一个全连接层的权重参数
    offset = self.w1.load(data, offset)
    # 加载第二个全连接层的权重参数
    offset = self.w2.load(data, offset)
    # 加载第三个全连接层的权重参数
    offset = self.w3.load(data, offset)
    # 返回更新后的偏移量
    return offset

# 将模型参数保存为GGUF格式
def save_gguf(self, gguf_writer):
    # 保存注意力层的参数
    self.att_norm.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_NORM, self.bid))
    # 保存查询权重参数
    self.wq.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_Q, self.bid))
    # 保存键权重参数
    self.wk.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_K, self.bid))
    # 保存值权重参数
    self.wv.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_V, self.bid))
    # 保存输出权重参数
    self.wo.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.ATTN_OUT, self.bid))
    # 保存前馈神经网络层的参数
    self.ffn_norm.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_NORM, self.bid))
# 调用self.w1对象的save_gguf方法，将gguf_writer和tensor_name(gguf.MODEL_TENSOR.FFN_GATE, self.bid)作为参数传入
self.w1.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_GATE, self.bid))
# 调用self.w2对象的save_gguf方法，将gguf_writer和tensor_name(gguf.MODEL_TENSOR.FFN_DOWN, self.bid)作为参数传入
self.w2.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_DOWN, self.bid))
# 调用self.w3对象的save_gguf方法，将gguf_writer和tensor_name(gguf.MODEL_TENSOR.FFN_UP, self.bid)作为参数传入
self.w3.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.FFN_UP, self.bid))

# 定义Model类
class Model:
    # 初始化方法
    def __init__(self):
        # 创建ModelParams对象并赋值给self.params
        self.params = ModelParams()
        # 创建空列表并赋值给self.layers
        self.layers = []

    # 加载方法，接受data和offset作为参数
    def load(self, data, offset):
        # 调用self.params对象的load方法，传入data和offset，并将返回值赋给offset
        offset = self.params.load(data, offset)

        # 创建名为tok_embd的Tensor对象，传入参数'f'和[self.params.n_embd, self.params.n_vocab]
        self.tok_embd = Tensor('f', [self.params.n_embd, self.params.n_vocab])
        # 创建名为norm的Tensor对象，传入参数'f'和[self.params.n_embd]
        self.norm = Tensor('f', [self.params.n_embd])
        # 创建名为output的Tensor对象，传入参数'f'和[self.params.n_embd, self.params.n_vocab]
        self.output = Tensor('f', [self.params.n_embd, self.params.n_vocab])

        # 调用self.tok_embd对象的load方法，传入data和offset，并将返回值赋给offset
        offset = self.tok_embd.load(data, offset)
        # 调用self.norm对象的load方法，传入data和offset，并将返回值赋给offset
        offset = self.norm.load(data, offset)
        # 调用self.output对象的load方法，传入data和offset，并将返回值赋给offset
        offset = self.output.load(data, offset)
# 清空图层列表
self.layers.clear()
# 遍历每一层，加载数据并添加到图层列表中
for bid in range(self.params.n_layer):
    layer = Layer(self.params, bid)
    offset = layer.load(data, offset)
    self.layers.append(layer)
# 返回偏移量
return offset

# 保存 GGUF 文件
def save_gguf(self, gguf_writer):
    # 保存参数到 GGUF 文件
    self.params.save_gguf(gguf_writer)
    # 保存 token_embd 到 GGUF 文件
    self.tok_embd.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.TOKEN_EMBD))
    # 保存 norm 到 GGUF 文件
    self.norm.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT_NORM))
    # 保存 output 到 GGUF 文件
    self.output.save_gguf(gguf_writer, name=tensor_name(gguf.MODEL_TENSOR.OUTPUT))
    # 遍历每一层，保存到 GGUF 文件
    for layer in self.layers:
        layer.save_gguf(gguf_writer)

# 检查点类的初始化方法
class Checkpoint:
    def __init__(self):
# 创建一个 Model 对象并赋值给 self.model
self.model = Model()
# 创建一个 OptimizationContext 对象并赋值给 self.opt_ctx
self.opt_ctx = OptimizationContext()

# 从给定的数据和偏移量开始，读取4个字节作为 magic 值，并将偏移量向后移动4个字节
magic   = bytes(reversed(data[offset:offset + 4])); offset += 4
# 如果 magic 值不等于 b'ggcp'，则抛出数值错误异常
if magic != b'ggcp':
    raise ValueError(f"File header magic indicates, that this is no checkpoint file. Expected 'ggcp', Got '{str(magic)}'")

# 从给定的数据和偏移量开始，按小端序解析4个字节作为版本号，并将偏移量向后移动4个字节
self.version = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
# 如果版本号不等于0，则抛出数值错误异常
if self.version != 0:
    raise ValueError('Invalid version of checkpoint file')

# 从给定的数据和偏移量开始，按小端序解析4个字节作为训练迭代次数，并将偏移量向后移动4个字节
self.train_its     = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从给定的数据和偏移量开始，按小端序解析4个字节作为训练样本数，并将偏移量向后移动4个字节
self.train_samples = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4
# 从给定的数据和偏移量开始，按小端序解析4个字节作为训练标记数，并将偏移量向后移动4个字节
self.train_tokens  = struct.unpack('<I', bytes(data[offset:offset + 4]))[0]; offset += 4

# 调用 Model 对象的 load 方法，传入数据和偏移量，并将返回的新偏移量赋值给 offset
offset = self.model.load(data, offset)
# 调用 OptimizationContext 对象的 load 方法，传入数据和偏移量，并将返回的新偏移量赋值给 offset
offset = self.opt_ctx.load(data, offset)

# 返回最终的偏移量
return offset
# 保存 GGUF 文件，向 GGUF 写入文件类型
def save_gguf(self, gguf_writer):
    gguf_writer.add_file_type(gguf.GGMLQuantizationType.F32)
    gguf_writer.add_layer_norm_rms_eps(1e-5)
    gguf_writer.add_uint32(LLM_KV_TRAINING_FILE_VERSION,    0)
    gguf_writer.add_string(LLM_KV_TRAINING_TYPE,            LLM_KV_TRAINING_TYPE_TRAIN_MODEL)
    gguf_writer.add_uint32(LLM_KV_TRAINING_ITERATION_COUNT, self.train_its)
    gguf_writer.add_uint32(LLM_KV_TRAINING_SAMPLE_COUNT,    self.train_samples)
    gguf_writer.add_uint32(LLM_KV_TRAINING_TOKEN_COUNT,     self.train_tokens)
    # 保存模型的 GGUF
    self.model.save_gguf(gguf_writer)
    # 保存优化器上下文的 GGUF
    self.opt_ctx.save_gguf(gguf_writer)

# 处理命令行参数
def handle_args():
    parser = argparse.ArgumentParser(description = 'Convert train-text-from-scratch checkpoints to GGUF')
    parser.add_argument('--input',  '-i', type = Path, help = 'Input train checkpoint filename', required=True)
    parser.add_argument('--output', '-o', type = Path, help ='Output GGUF filename', required=True)
    return parser.parse_args()

# 主函数
def main():
    # 处理命令行参数
    cfg = handle_args()
# 从指定的文件中创建一个内存映射，以只读模式打开
data = np.memmap(cfg.input, mode = 'r')
# 创建一个检查点对象
chk = Checkpoint()
# 设置偏移量为0
offset = 0
# 从内存映射中加载数据到检查点对象中，并返回新的偏移量
offset = chk.load(data, offset)
# 断言偏移量是否等于数据的长度，以确保已经读取了所有可用的数据
assert(offset == len(data))

# 创建一个GGUFWriter对象，用于写入GGUF文件
gguf_writer = gguf.GGUFWriter(cfg.output, gguf.MODEL_ARCH_NAMES[gguf.MODEL_ARCH.LLAMA], use_temp_file = False)
# 将检查点对象保存到GGUFWriter对象中
chk.save_gguf(gguf_writer)
# 打印信息
print("    gguf: write header")
# 将头部信息写入文件
gguf_writer.write_header_to_file()
# 打印信息
print("    gguf: write metadata")
# 将元数据写入文件
gguf_writer.write_kv_data_to_file()
# 打印信息
print("    gguf: write tensors")
# 将张量数据写入文件
gguf_writer.write_tensors_to_file()
# 关闭GGUFWriter对象
gguf_writer.close()

# 如果当前脚本为主程序，则执行main函数
if __name__ == '__main__':
    main()
```