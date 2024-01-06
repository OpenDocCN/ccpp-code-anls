# `PowerInfer\gguf-py\scripts\gguf-convert-endian.py`

```
#!/usr/bin/env python3
# 指定脚本解释器为 Python 3

from __future__ import annotations
# 导入未来版本的特性，用于支持类型注解

import argparse
# 导入用于解析命令行参数的模块
import os
# 导入用于与操作系统交互的模块
import sys
# 导入用于与 Python 解释器交互的模块
from pathlib import Path
# 从 pathlib 模块中导入 Path 类

import numpy as np
# 导入 NumPy 库，并使用 np 别名

# 如果环境变量中没有设置 NO_LOCAL_GGUF，并且 gguf-py 包存在于指定路径下，则将该路径添加到模块搜索路径中
if "NO_LOCAL_GGUF" not in os.environ and (Path(__file__).parent.parent.parent / 'gguf-py').exists():
    sys.path.insert(0, str(Path(__file__).parent.parent))

import gguf
# 导入 gguf 模块

def convert_byteorder(reader: gguf.GGUFReader, args: argparse.Namespace) -> None:
    # 如果无符号 32 位整数 1 的字节顺序改为小端序后仍然等于 1
    if np.uint32(1) == np.uint32(1).newbyteorder("<"):
        # 主机是小端序
# 设置主机字节顺序为小端，交换字节顺序为大端
host_endian = "little"
swapped_endian = "big"
# 如果不是小端系统，则设置主机字节顺序为大端，交换字节顺序为小端
else:
    host_endian = "big"
    swapped_endian = "little"
# 如果读取器的字节顺序为交换，则文件字节顺序为交换字节顺序，否则为主机字节顺序
if reader.byte_order == "S":
    file_endian = swapped_endian
else:
    file_endian = host_endian
# 如果命令行参数中的顺序为本机，则使用主机字节顺序，否则使用命令行参数中的顺序
order = host_endian if args.order == "native" else args.order
# 打印主机字节顺序和文件字节顺序
print(f"* Host is {host_endian.upper()} endian, GGUF file seems to be {file_endian.upper()} endian")
# 如果文件字节顺序和命令行参数中的顺序相同，则打印文件已经是目标字节顺序，无需操作，然后退出程序
if file_endian == order:
    print(f"* File is already {order.upper()} endian. Nothing to do.")
    sys.exit(0)
# 打印检查张量是否兼容转换
print("* Checking tensors for conversion compatibility")
# 遍历读取器的张量
for tensor in reader.tensors:
    # 如果张量类型不是F32或F16，则进行下一次循环
    if tensor.tensor_type not in (
        gguf.GGMLQuantizationType.F32,
        gguf.GGMLQuantizationType.F16,
# 检查是否为指定的量化类型，如果不是则抛出数值错误
if tensor.tensor_type.name not in (gguf.GGMLQuantizationType.Q8_0,):
    raise ValueError(f"Cannot handle type {tensor.tensor_type.name} for tensor {repr(tensor.name)}")
# 打印文件转换的起始和目标字节序
print(f"* Preparing to convert from {file_endian.upper()} to {order.upper()}")
# 如果是 dry run 模式，则直接返回
if args.dry_run:
    return
# 打印警告信息，提醒可能会损坏文件，需要备份
print("\n*** Warning *** Warning *** Warning **")
print("* This conversion process may damage the file. Ensure you have a backup.")
# 如果请求的字节序与主机字节序不同，则打印警告信息
if order != host_endian:
    print("* Requested endian differs from host, you will not be able to load the model on this machine.")
# 提示用户确认是否继续进行文件转换
print("* The file will be modified immediately, so if conversion fails or is interrupted")
print("* the file will be corrupted. Enter exactly YES if you are positive you want to proceed:")
response = input("YES, I am sure> ")
# 如果用户输入不是 YES，则退出程序
if response != "YES":
    print("You didn't enter YES. Okay then, see ya!")
    sys.exit(0)
# 打印转换字段的数量
print(f"\n* Converting fields ({len(reader.fields)})")
# 遍历字段并打印转换信息
for idx, field in enumerate(reader.fields.values()):
    print(f"- {idx:4}: Converting field {repr(field.name)}, part count: {len(field.parts)}")
    # 遍历字段的各个部分
    for part in field.parts:
# 对部分数据进行字节交换，修改原数据
part.byteswap(inplace=True)

# 打印转换张量的信息
print(f"\n* Converting tensors ({len(reader.tensors)})")

# 遍历张量列表，打印每个张量的信息
for idx, tensor in enumerate(reader.tensors):
    print(
        f"  - {idx:4}: Converting tensor {repr(tensor.name)}, type={tensor.tensor_type.name}, "
        f"elements={tensor.n_elements}... ",
        end="",
    )
    # 获取张量类型
    tensor_type = tensor.tensor_type
    # 对张量的每个部分进行字节交换
    for part in tensor.field.parts:
        part.byteswap(inplace=True)
    # 如果张量类型不是 Q8_0，则对张量数据进行字节交换并打印信息
    if tensor_type != gguf.GGMLQuantizationType.Q8_0:
        tensor.data.byteswap(inplace=True)
        print()
        continue
    # 一个 Q8_0 块由一个 f16 delta 后跟 32 个 int8 quants 组成，所以是 34 个字节
    block_size = 34
    # 计算张量数据中 Q8_0 块的数量
    n_blocks = len(tensor.data) // block_size
    # 遍历每个 Q8_0 块
    for block_num in range(n_blocks):
        block_offs = block_num * block_size
# 定义一个函数，用于将 GGUF 文件的字节顺序转换
def convert_byte_order(model: str, order: str) -> None:
    # 创建参数解析器
    parser = argparse.ArgumentParser(description="Convert GGUF file byte order")
    # 添加模型文件名参数
    parser.add_argument(
        "model", type=str,
        help="GGUF format model filename",
    )
    # 添加字节顺序参数，可选值为'big', 'little', 'native'
    parser.add_argument(
        "order", type=str, choices=['big', 'little', 'native'],
        help="Requested byte order",
    )
# 添加一个命令行参数，--dry-run，如果设置了该参数则执行该动作，否则不执行
parser.add_argument(
    "--dry-run", action="store_true",
    help="Don't actually change anything",
)
# 解析命令行参数，如果没有传入参数则显示帮助信息
args = parser.parse_args(None if len(sys.argv) > 1 else ["--help"])
# 打印加载信息
print(f'* Loading: {args.model}')
# 根据参数创建一个GGUFReader对象，根据--dry-run参数选择只读或读写模式
reader = gguf.GGUFReader(args.model, 'r' if args.dry_run else 'r+')
# 调用convert_byteorder函数，传入reader和args作为参数
convert_byteorder(reader, args)

# 如果当前脚本为主程序，则执行main函数
if __name__ == "__main__":
    main()
```