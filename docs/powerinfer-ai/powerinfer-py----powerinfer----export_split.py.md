# `PowerInfer\powerinfer-py\powerinfer\export_split.py`

```cpp
import argparse  # 导入命令行参数解析模块
import pickle  # 导入 pickle 序列化模块
import gguf  # 导入 gguf 模块
from gguf.constants import GGMLQuantizationType  # 从 gguf.constants 模块导入 GGMLQuantizationType 常量
from gguf.gguf_writer import GGUFWriter  # 从 gguf.gguf_writer 模块导入 GGUFWriter 类
import torch  # 导入 PyTorch 深度学习框架
from pathlib import Path  # 从 pathlib 模块导入 Path 类
import os  # 导入操作系统相关模块
import struct  # 导入 struct 模块，用于处理字节数据
import numpy as np  # 导入 NumPy 数学计算库

def load_activation_weights(models_base: Path):
    # TODO: might need a specification file to indicate which models to load.
    # But for now, let's assume it is a plain directory of activation_{0, ... , n_layers - 1}.pt
    *_, files = next(os.walk(models_base))  # 获取指定目录下的文件列表
    return [torch.load(models_base / f"activation_{i}.pt") for i in range(len(files))]  # 返回加载的模型激活数据列表

def append_gpu_idx(gguf: GGUFWriter, i_layer: int, activation, select_count) -> None:
    _, indices = torch.topk(activation, k=int(select_count))  # 获取激活数据中前 k 个最大值的索引
    gpu_idx = torch.zeros_like(activation)  # 创建与激活数据相同形状的全零张量
    gpu_idx[indices] = 1  # 将前 k 个最大值的索引位置置为 1
    gpu_idx = gpu_idx.numpy().astype(np.int32)  # 将张量转换为 NumPy 数组，并转换数据类型为 int32
    key = f"blk.{i_layer}.gpu_idx"  # 设置键名
    print(
        f"{key} => {key} {gpu_idx.shape} {gpu_idx.dtype} {gpu_idx.nbytes/1024/1024} MiB"
    )  # 打印输出信息
    gguf.add_tensor(
        name=key,  # 张量名称
        tensor=gpu_idx,  # 张量数据
        raw_shape=gpu_idx.shape[::-1],  # 原始形状
        raw_dtype=GGMLQuantizationType.I32,  # 原始数据类型
    )

    indices = indices.numpy().astype(np.int32)  # 将索引数组转换为 int32 类型的 NumPy 数组
    gpu_bucket = np.sort(indices)  # 对索引数组进行排序
    key = f"blk.{i_layer}.gpu_bucket"  # 设置键名
    print(
        f"{key} => {key} {gpu_bucket.shape} {gpu_bucket.dtype} {gpu_bucket.nbytes/1024/1024} MiB"
    )  # 打印输出信息
    gguf.add_tensor(
        name=key,  # 张量名称
        tensor=gpu_bucket,  # 张量数据
        raw_shape=gpu_bucket.shape[::-1],  # 原始形状
        raw_dtype=GGMLQuantizationType.I32,  # 原始数据类型
    )

def export_split(activations_path: str, output_path: str, solved_list: list[int], vram_capacity: int):
    predictors = load_activation_weights(Path(activations_path))  # 加载激活数据
    gguf_out = GGUFWriter(output_path, "generic.gpu_index")  # 创建 GGUFWriter 对象
    for i, (activation, selected_count) in enumerate(zip(predictors, solved_list)):  # 遍历激活数据和选择数量
        append_gpu_idx(gguf_out, i, activation, selected_count)  # 调用函数添加 GPU 索引数据

    # set kvs
    gguf_out.add_block_count(len(predictors))  # 添加块数量信息
    # 将分割神经元所需的实际容量保存到 gguf_out 对象中
    gguf_out.add_uint64(gguf.Keys.Split.VRAM_CAPACITY, vram_capacity)

    # 将头部信息写入文件
    gguf_out.write_header_to_file()
    # 将键值数据写入文件
    gguf_out.write_kv_data_to_file()
    # 将张量数据写入文件
    gguf_out.write_tensors_to_file()
    # 关闭 gguf_out 对象
    gguf_out.close()

    # 后处理：写入另一个唯一文件头，以区分原始的 GGUF 文件
    with open(output_path, "r+b") as fout:
        # 将字符串 "PWRI" 转换为小端字节序的整数，并写入文件
        POWERINFER_MAGIC = int.from_bytes(b"PWRI", "little")
        fout.write(struct.pack("<I", POWERINFER_MAGIC))
        # 写入一个整数 3，用于区分不同类型的文件
        fout.write(struct.pack("<I", 3))

    # 打印导出的 GPU 索引到 output_path
    print(f"exported GPU index to {output_path}")
```