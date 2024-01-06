# `PowerInfer\powerinfer-py\powerinfer\export_split.py`

```
# 导入必要的库
import argparse  # 用于解析命令行参数
import pickle  # 用于序列化和反序列化Python对象
import gguf  # 导入gguf模块
from gguf.constants import GGMLQuantizationType  # 从gguf.constants模块中导入GGMLQuantizationType常量
from gguf.gguf_writer import GGUFWriter  # 从gguf.gguf_writer模块中导入GGUFWriter类
import torch  # 导入PyTorch库
from pathlib import Path  # 从pathlib模块中导入Path类
import os  # 导入os模块
import struct  # 用于处理字节数据的结构化模块
import numpy as np  # 导入NumPy库

# 加载激活权重
def load_activation_weights(models_base: Path):
    # TODO: 可能需要一个规范文件来指示加载哪些模型。
    # 但现在，让我们假设它是一个激活_{0, ... , n_layers - 1}.pt的普通目录。
    *_, files = next(os.walk(models_base))  # 获取目录下的文件列表
    return [torch.load(models_base / f"activation_{i}.pt") for i in range(len(files))]  # 加载激活权重文件并返回列表

# 添加GPU索引
def append_gpu_idx(gguf: GGUFWriter, i_layer: int, activation, select_count) -> None:
    _, indices = torch.topk(activation, k=int(select_count))  # 获取激活值中前k个最大值的索引
    gpu_idx = torch.zeros_like(activation)  # 创建与激活值相同形状的全零张量作为GPU索引
    # 将指定索引位置的元素设置为1
    gpu_idx[indices] = 1
    # 将 GPU 索引转换为 numpy 数组，并将数据类型转换为 np.int32
    gpu_idx = gpu_idx.numpy().astype(np.int32)
    # 创建用于标识 GPU 索引的键
    key = f"blk.{i_layer}.gpu_idx"
    # 打印 GPU 索引的相关信息
    print(
        f"{key} => {key} {gpu_idx.shape} {gpu_idx.dtype} {gpu_idx.nbytes/1024/1024} MiB"
    )
    # 将 GPU 索引添加到 GGUF 中
    gguf.add_tensor(
        name=key,
        tensor=gpu_idx,
        raw_shape=gpu_idx.shape[::-1],
        raw_dtype=GGMLQuantizationType.I32,
    )

    # 将索引转换为 numpy 数组，并将数据类型转换为 np.int32
    indices = indices.numpy().astype(np.int32)
    # 对 GPU 索引进行排序
    gpu_bucket = np.sort(indices)
    # 创建用于标识 GPU 存储桶的键
    key = f"blk.{i_layer}.gpu_bucket"
    # 打印 GPU 存储桶的相关信息
    print(
        f"{key} => {key} {gpu_bucket.shape} {gpu_bucket.dtype} {gpu_bucket.nbytes/1024/1024} MiB"
    )
    # 将 GPU 存储桶添加到 GGUF 中
    gguf.add_tensor(
# 将参数传递给 GGUFWriter 对象的构造函数，设置名称、张量、原始形状和数据类型
name=key,
tensor=gpu_bucket,
raw_shape=gpu_bucket.shape[::-1],
raw_dtype=GGMLQuantizationType.I32,
)

# 导出分割后的激活数据
def export_split(activations_path: str, output_path: str, solved_list: list[int], vram_capacity: int):
    # 加载激活权重数据，返回预测器列表
    predictors = load_activation_weights(Path(activations_path)) # predictor => activation acount
    # 创建 GGUFWriter 对象，用于写入分割后的数据
    gguf_out = GGUFWriter(output_path, "generic.gpu_index")
    # 遍历预测器列表和解决列表，将激活数据和选择的数量添加到 GGUFWriter 对象中
    for i, (activation, selected_count) in enumerate(zip(predictors, solved_list)):
        append_gpu_idx(gguf_out, i, activation, selected_count)

    # 设置键值对
    gguf_out.add_block_count(len(predictors))
    # TODO: 最好保存分割神经元所需的实际容量
    gguf_out.add_uint64(gguf.Keys.Split.VRAM_CAPACITY, vram_capacity)

    # 将头部信息写入文件
    gguf_out.write_header_to_file()
    # 将键值对数据写入文件
    gguf_out.write_kv_data_to_file()
    # 将张量数据写入文件
    gguf_out.write_tensors_to_file()
    gguf_out.close()  # 关闭 gguf_out 文件流

    # 后处理：写入另一个唯一的文件头，以区别于原始的 GGUF 文件
    with open(output_path, "r+b") as fout:  # 以读写二进制模式打开 output_path 文件
        POWERINFER_MAGIC = int.from_bytes(b"PWRI", "little")  # 将字节序列 b"PWRI" 转换为小端整数
        fout.write(struct.pack("<I", POWERINFER_MAGIC))  # 将小端整数写入文件
        fout.write(struct.pack("<I", 3))  # 将整数 3 写入文件
    print(f"exported GPU index to {output_path}")  # 打印输出 GPU 索引到 output_path
```