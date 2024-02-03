# `PowerInfer\gguf-py\scripts\gguf-convert-endian.py`

```cpp
#!/usr/bin/env python3
from __future__ import annotations

import argparse  # 导入用于解析命令行参数的模块
import os  # 导入操作系统相关的模块
import sys  # 导入系统相关的模块
from pathlib import Path  # 导入用于处理文件路径的模块

import numpy as np  # 导入用于科学计算的模块

# 如果环境变量中没有设置 NO_LOCAL_GGUF，并且 gguf-py 包存在于指定路径，则将该路径添加到模块搜索路径中
if "NO_LOCAL_GGUF" not in os.environ and (Path(__file__).parent.parent.parent / 'gguf-py').exists():
    sys.path.insert(0, str(Path(__file__).parent.parent))

import gguf  # 导入自定义的 gguf 模块


def convert_byteorder(reader: gguf.GGUFReader, args: argparse.Namespace) -> None:
    # 检查系统的字节序，确定主机的字节序和交换后的字节序
    if np.uint32(1) == np.uint32(1).newbyteorder("<"):
        # 主机是小端字节序
        host_endian = "little"
        swapped_endian = "big"
    else:
        # 其他系统，如 PDP 或其他不使用大端或小端字节序的系统
        host_endian = "big"
        swapped_endian = "little"
    # 根据 GGUF 文件的字节序确定文件的字节序
    if reader.byte_order == "S":
        file_endian = swapped_endian
    else:
        file_endian = host_endian
    # 根据命令行参数确定要转换的字节序
    order = host_endian if args.order == "native" else args.order
    print(f"* Host is {host_endian.upper()} endian, GGUF file seems to be {file_endian.upper()} endian")
    # 如果文件的字节序和要转换的字节序相同，则无需转换
    if file_endian == order:
        print(f"* File is already {order.upper()} endian. Nothing to do.")
        sys.exit(0)
    print("* Checking tensors for conversion compatibility")
    # 检查张量的类型是否支持转换
    for tensor in reader.tensors:
        if tensor.tensor_type not in (
            gguf.GGMLQuantizationType.F32,
            gguf.GGMLQuantizationType.F16,
            gguf.GGMLQuantizationType.Q8_0,
        ):
            raise ValueError(f"Cannot handle type {tensor.tensor_type.name} for tensor {repr(tensor.name)}")
    print(f"* Preparing to convert from {file_endian.upper()} to {order.upper()}")
    # 如果是 dry run 模式，则直接返回，不执行转换
    if args.dry_run:
        return
    print("\n*** Warning *** Warning *** Warning **")
    print("* This conversion process may damage the file. Ensure you have a backup.")
    # 如果要转换的字节序和主机字节序不同，则给出警告
    if order != host_endian:
        print("* Requested endian differs from host, you will not be able to load the model on this machine.")
    # 打印警告信息，提醒用户文件将立即被修改，如果转换失败或中断，文件将损坏
    print("* The file will be modified immediately, so if conversion fails or is interrupted")
    print("* the file will be corrupted. Enter exactly YES if you are positive you want to proceed:")
    # 获取用户输入，确认是否要继续进行文件转换
    response = input("YES, I am sure> ")
    # 如果用户输入不是YES，则打印提示信息并退出程序
    if response != "YES":
        print("You didn't enter YES. Okay then, see ya!")
        sys.exit(0)
    # 打印正在转换字段的信息
    print(f"\n* Converting fields ({len(reader.fields)})")
    # 遍历字段并打印转换信息
    for idx, field in enumerate(reader.fields.values()):
        print(f"- {idx:4}: Converting field {repr(field.name)}, part count: {len(field.parts)}")
        # 对字段的每个部分进行字节交换
        for part in field.parts:
            part.byteswap(inplace=True)
    # 打印正在转换张量的信息
    print(f"\n* Converting tensors ({len(reader.tensors)})")
    # 遍历张量并打印转换信息
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
        # 如果张量类型不是Q8_0，则对张量数据进行字节交换
        if tensor_type != gguf.GGMLQuantizationType.Q8_0:
            tensor.data.byteswap(inplace=True)
            print()
            continue
        # 一个Q8_0块由一个f16增量后跟32个int8量化值组成，所以总共34个字节
        block_size = 34
        n_blocks = len(tensor.data) // block_size
        # 遍历每个块并对其中的数据进行字节交换
        for block_num in range(n_blocks):
            block_offs = block_num * block_size
            # 获取增量值并进行字节交换
            delta = tensor.data[block_offs:block_offs + 2].view(dtype=np.uint16)
            delta.byteswap(inplace=True)
            # 每处理100000个块打印进度信息
            if block_num % 100000 == 0:
                print(f"[{(n_blocks - block_num) // 1000}K]", end="")
                sys.stdout.flush()
        print()
    # 打印转换完成信息
    print("* Completion")
# 定义主函数，不返回任何结果
def main() -> None:
    # 创建参数解析器对象，设置描述信息
    parser = argparse.ArgumentParser(description="Convert GGUF file byte order")
    # 添加模型文件名参数，类型为字符串
    parser.add_argument(
        "model", type=str,
        help="GGUF format model filename",
    )
    # 添加字节顺序参数，类型为字符串，可选值为'big', 'little', 'native'
    parser.add_argument(
        "order", type=str, choices=['big', 'little', 'native'],
        help="Requested byte order",
    )
    # 添加可选参数'--dry-run'，设置动作为存储为True
    parser.add_argument(
        "--dry-run", action="store_true",
        help="Don't actually change anything",
    )
    # 解析命令行参数，如果没有参数则显示帮助信息
    args = parser.parse_args(None if len(sys.argv) > 1 else ["--help"])
    # 打印加载模型文件的提示信息
    print(f'* Loading: {args.model}')
    # 创建 GGUFReader 对象，根据参数设置读取或读写模式
    reader = gguf.GGUFReader(args.model, 'r' if args.dry_run else 'r+')
    # 调用函数，转换字节顺序
    convert_byteorder(reader, args)

# 如果当前脚本为主程序，则执行主函数
if __name__ == "__main__":
    main()
```