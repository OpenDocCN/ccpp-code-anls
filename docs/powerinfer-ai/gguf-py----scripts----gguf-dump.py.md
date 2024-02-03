# `PowerInfer\gguf-py\scripts\gguf-dump.py`

```cpp
#!/usr/bin/env python3
# 指定脚本使用的 Python 解释器

from __future__ import annotations
# 导入未来版本的特性

import argparse
# 导入用于解析命令行参数的模块
import os
# 导入操作系统相关的模块
import sys
# 导入与 Python 解释器交互的模块
from pathlib import Path
# 导入处理文件路径的模块
from typing import Any
# 导入类型提示相关的模块

import numpy as np
# 导入数值计算库 NumPy

# 如果环境变量中没有设置 NO_LOCAL_GGUF，并且 gguf-py 包存在于指定路径下，则将该路径添加到模块搜索路径中
if "NO_LOCAL_GGUF" not in os.environ and (Path(__file__).parent.parent.parent / 'gguf-py').exists():
    sys.path.insert(0, str(Path(__file__).parent.parent))

from gguf import GGUFReader, GGUFValueType  # noqa: E402
# 从 gguf 模块中导入 GGUFReader 和 GGUFValueType 类

def get_file_host_endian(reader: GGUFReader) -> tuple[str, str]:
    # 判断主机的字节序是小端还是大端
    host_endian = 'LITTLE' if np.uint32(1) == np.uint32(1).newbyteorder("<") else 'BIG'
    # 如果文件的字节序是 'S'，则文件的字节序与主机的字节序相反，否则与主机的字节序相同
    if reader.byte_order == 'S':
        file_endian = 'BIG' if host_endian == 'LITTLE' else 'LITTLE'
    else:
        file_endian = host_endian
    return (host_endian, file_endian)
    # 返回主机字节序和文件字节序的元组

# 有关 field.parts 和 field.data 代表的更多信息，请参见 modify_gguf.py 示例中的注释
def dump_metadata(reader: GGUFReader, args: argparse.Namespace) -> None:
    # 获取文件字节序和主机字节序
    host_endian, file_endian = get_file_host_endian(reader)
    print(f'* File is {file_endian} endian, script is running on a {host_endian} endian host.')
    print(f'\n* Dumping {len(reader.fields)} key/value pair(s)')
    # 打印文件字节序和主机字节序的信息
    # 遍历读取器字段的值，并使用enumerate函数获取索引和字段值
    for n, field in enumerate(reader.fields.values(), 1):
        # 如果字段类型为空，则将pretty_type设置为'N/A'
        if not field.types:
            pretty_type = 'N/A'
        # 如果字段类型为数组类型，则计算嵌套层数并生成对应的pretty_type
        elif field.types[0] == GGUFValueType.ARRAY:
            nest_count = len(field.types) - 1
            pretty_type = '[' * nest_count + str(field.types[-1].name) + ']' * nest_count
        # 否则将pretty_type设置为字段类型的最后一个名称
        else:
            pretty_type = str(field.types[-1].name)
        # 打印字段的索引、pretty_type、数据长度和字段名称，并以空格结尾
        print(f'  {n:5}: {pretty_type:10} | {len(field.data):8} | {field.name}', end = '')
        # 如果字段类型长度为1，则获取当前类型并根据类型进行不同的打印
        if len(field.types) == 1:
            curr_type = field.types[0]
            # 如果当前类型为字符串类型，则打印字段的前60个字符
            if curr_type == GGUFValueType.STRING:
                print(' = {0}'.format(repr(str(bytes(field.parts[-1]), encoding='utf8')[:60])), end = '')
            # 如果字段类型在reader.gguf_scalar_to_np中，则打印字段的第一个元素
            elif field.types[0] in reader.gguf_scalar_to_np:
                print(' = {0}'.format(field.parts[-1][0]), end = '')
        # 打印换行符
        print()
    # 如果参数中包含no_tensors，则直接返回
    if args.no_tensors:
        return
    # 打印换行符和tensor的数量
    print(f'\n* Dumping {len(reader.tensors)} tensor(s)')
    # 遍历读取器的tensors，并使用enumerate函数获取索引和tensor值
    for n, tensor in enumerate(reader.tensors, 1):
        # 将tensor的shape转换为字符串，并打印tensor的索引、元素数量、prettydims、tensor类型和tensor名称
        prettydims = ', '.join('{0:5}'.format(d) for d in list(tensor.shape) + [1] * (4 - len(tensor.shape)))
        print(f'  {n:5}: {tensor.n_elements:10} | {prettydims} | {tensor.tensor_type.name:7} | {tensor.name}')
# 定义一个函数，用于将 GGUF 文件的元数据转换为 JSON 格式并输出到标准输出
def dump_metadata_json(reader: GGUFReader, args: argparse.Namespace) -> None:
    import json
    # 获取文件的主机字节序和文件字节序
    host_endian, file_endian = get_file_host_endian(reader)
    # 创建空的元数据和张量字典
    metadata: dict[str, Any] = {}
    tensors: dict[str, Any] = {}
    # 创建结果字典，包括文件名、字节序、元数据和张量
    result = {
        "filename": args.model,
        "endian": file_endian,
        "metadata": metadata,
        "tensors": tensors,
    }
    # 遍历 GGUF 文件的字段
    for idx, field in enumerate(reader.fields.values()):
        # 创建当前字段的字典，包括索引、类型和偏移量
        curr: dict[str, Any] = {
            "index": idx,
            "type": field.types[0].name if field.types else 'UNKNOWN',
            "offset": field.offset,
        }
        # 将当前字段的字典添加到元数据字典中
        metadata[field.name] = curr
        # 如果字段类型为数组
        if field.types[:1] == [GGUFValueType.ARRAY]:
            # 添加数组类型到当前字段的字典中
            curr["array_types"] = [t.name for t in field.types][1:]
            # 如果不需要输出 JSON 数组，则跳过当前字段
            if not args.json_array:
                continue
            # 获取数组元素类型
            itype = field.types[-1]
            # 如果数组元素类型为字符串
            if itype == GGUFValueType.STRING:
                # 将数组元素转换为 UTF-8 编码的字符串，并添加到当前字段的字典中
                curr["value"] = [str(bytes(field.parts[idx]), encoding="utf-8") for idx in field.data]
            else:
                # 将数组元素转换为列表，并添加到当前字段的字典中
                curr["value"] = [pv for idx in field.data for pv in field.parts[idx].tolist()]
        # 如果字段类型为字符串
        elif field.types[0] == GGUFValueType.STRING:
            # 将字段值转换为 UTF-8 编码的字符串，并添加到当前字段的字典中
            curr["value"] = str(bytes(field.parts[-1]), encoding="utf-8")
        else:
            # 将字段值转换为列表，并添加到当前字段的字典中
            curr["value"] = field.parts[-1].tolist()[0]
    # 遍历 GGUF 文件的张量
    for idx, tensor in enumerate(reader.tensors):
        # 将张量的索引、形状、类型和偏移量添加到张量字典中
        tensors[tensor.name] = {
            "index": idx,
            "shape": tensor.shape.tolist(),
            "type": tensor.tensor_type.name,
            "offset": tensor.field.offset,
        }
    # 将结果字典转换为 JSON 格式并输出到标准输出
    json.dump(result, sys.stdout)


# 定义主函数
def main() -> None:
    # 创建参数解析器
    parser = argparse.ArgumentParser(description="Dump GGUF file metadata")
    # 添加模型文件名参数
    parser.add_argument("model", type=str, help="GGUF format model filename")
    # 添加是否不输出张量元数据的选项
    parser.add_argument("--no-tensors", action="store_true", help="Don't dump tensor metadata")
    # 添加是否输出 JSON 格式的选项
    parser.add_argument("--json", action="store_true", help="Produce JSON output")
    # 添加一个命令行参数，用于在 JSON 输出中包含完整的数组值（长格式）
    parser.add_argument("--json-array", action="store_true", help="Include full array values in JSON output (long)")
    # 解析命令行参数，如果没有参数则打印帮助信息
    args = parser.parse_args(None if len(sys.argv) > 1 else ["--help"])
    # 如果没有设置 json 参数，则打印加载模型的信息
    if not args.json:
        print(f'* Loading: {args.model}')
    # 创建一个 GGUFReader 对象，用于读取模型数据
    reader = GGUFReader(args.model, 'r')
    # 如果设置了 json 参数，则以 JSON 格式输出元数据
    if args.json:
        dump_metadata_json(reader, args)
    # 否则以普通格式输出元数据
    else:
        dump_metadata(reader, args)
# 如果当前模块被直接执行，则调用 main() 函数
if __name__ == '__main__':
    main()
```