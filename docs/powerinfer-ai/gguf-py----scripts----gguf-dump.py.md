# `PowerInfer\gguf-py\scripts\gguf-dump.py`

```
#!/usr/bin/env python3
# 指定脚本使用的 Python 解释器

from __future__ import annotations
# 导入未来版本的特性，用于支持类型注解

import argparse
# 用于解析命令行参数

import os
# 提供与操作系统交互的功能

import sys
# 提供与 Python 解释器交互的功能

from pathlib import Path
# 用于操作文件路径的类

from typing import Any
# 用于类型提示

import numpy as np
# 导入 NumPy 库

# 如果环境变量中没有设置 NO_LOCAL_GGUF，并且 gguf-py 包存在于指定路径，则将其添加到模块搜索路径中
if "NO_LOCAL_GGUF" not in os.environ and (Path(__file__).parent.parent.parent / 'gguf-py').exists():
    sys.path.insert(0, str(Path(__file__).parent.parent))

from gguf import GGUFReader, GGUFValueType  # noqa: E402
# 导入自定义的 GGUFReader 和 GGUFValueType 类，禁止 E402 错误提示

def get_file_host_endian(reader: GGUFReader) -> tuple[str, str]:
    # 判断主机的字节序，返回结果为元组，包含主机字节序和网络字节序
    host_endian = 'LITTLE' if np.uint32(1) == np.uint32(1).newbyteorder("<") else 'BIG'
# 如果读取器的字节顺序为 'S'，则文件字节顺序为大端字节序（BIG），否则为主机字节顺序
if reader.byte_order == 'S':
    file_endian = 'BIG' if host_endian == 'LITTLE' else 'LITTLE'
else:
    file_endian = host_endian
# 返回主机字节顺序和文件字节顺序的元组
return (host_endian, file_endian)

# 获取文件和主机的字节顺序，并打印相关信息
def dump_metadata(reader: GGUFReader, args: argparse.Namespace) -> None:
    host_endian, file_endian = get_file_host_endian(reader)
    print(f'* File is {file_endian} endian, script is running on a {host_endian} endian host.')
    print(f'\n* Dumping {len(reader.fields)} key/value pair(s)')
    # 遍历字段字典，打印字段类型信息
    for n, field in enumerate(reader.fields.values(), 1):
        # 如果字段类型为空，则打印 'N/A'
        if not field.types:
            pretty_type = 'N/A'
        # 如果字段类型为数组，则打印数组类型信息
        elif field.types[0] == GGUFValueType.ARRAY:
            nest_count = len(field.types) - 1
            pretty_type = '[' * nest_count + str(field.types[-1].name) + ']' * nest_count
        else:
            # 其他情况暂时未给出注释
# 获取字段的类型并转换成字符串
pretty_type = str(field.types[-1].name)
# 打印字段的名称、类型、数据长度和字段名
print(f'  {n:5}: {pretty_type:10} | {len(field.data):8} | {field.name}', end = '')
# 如果字段类型只有一个，获取当前类型
if len(field.types) == 1:
    curr_type = field.types[0]
    # 如果当前类型是字符串，打印字段的部分数据
    if curr_type == GGUFValueType.STRING:
        print(' = {0}'.format(repr(str(bytes(field.parts[-1]), encoding='utf8')[:60])), end = '')
    # 如果当前类型在 gguf_scalar_to_np 中，打印字段的部分数据
    elif field.types[0] in reader.gguf_scalar_to_np:
        print(' = {0}'.format(field.parts[-1][0]), end = '')
# 打印换行
print()
# 如果参数中指定不打印张量，直接返回
if args.no_tensors:
    return
# 打印张量的数量
print(f'\n* Dumping {len(reader.tensors)} tensor(s)')
# 遍历并打印每个张量的元素数量、维度、张量类型和名称
for n, tensor in enumerate(reader.tensors, 1):
    prettydims = ', '.join('{0:5}'.format(d) for d in list(tensor.shape) + [1] * (4 - len(tensor.shape)))
    print(f'  {n:5}: {tensor.n_elements:10} | {prettydims} | {tensor.tensor_type.name:7} | {tensor.name}')

# 定义一个函数，用于将元数据转换成 JSON 格式并打印
def dump_metadata_json(reader: GGUFReader, args: argparse.Namespace) -> None:
    import json
    # 获取文件的主机字节序和文件字节序
    host_endian, file_endian = get_file_host_endian(reader)
# 创建一个空的元数据字典
metadata: dict[str, Any] = {}
# 创建一个空的张量字典
tensors: dict[str, Any] = {}
# 创建一个结果字典，包括文件名、字节顺序、元数据和张量
result = {
    "filename": args.model,
    "endian": file_endian,
    "metadata": metadata,
    "tensors": tensors,
}
# 遍历读取器字段的值
for idx, field in enumerate(reader.fields.values()):
    # 创建当前字段的字典，包括索引、类型和偏移量
    curr: dict[str, Any] = {
        "index": idx,
        "type": field.types[0].name if field.types else 'UNKNOWN',
        "offset": field.offset,
    }
    # 将当前字段的字典添加到元数据中
    metadata[field.name] = curr
    # 如果字段类型是数组，将数组类型添加到当前字段的字典中
    if field.types[:1] == [GGUFValueType.ARRAY]:
        curr["array_types"] = [t.name for t in field.types][1:]
        # 如果不需要 JSON 数组，跳过当前字段
        if not args.json_array:
            continue
        # 获取字段类型的最后一个类型
        itype = field.types[-1]
# 如果字段类型为字符串，则将字段数据转换为字符串列表，并存储在当前字典的"value"键下
if itype == GGUFValueType.STRING:
    curr["value"] = [str(bytes(field.parts[idx]), encoding="utf-8") for idx in field.data]
# 如果字段类型不为字符串，则将字段数据转换为列表，并存储在当前字典的"value"键下
else:
    curr["value"] = [pv for idx in field.data for pv in field.parts[idx].tolist()]
# 如果字段的第一个类型为字符串，则将字段的最后一个部分转换为字符串，并存储在当前字典的"value"键下
elif field.types[0] == GGUFValueType.STRING:
    curr["value"] = str(bytes(field.parts[-1]), encoding="utf-8")
# 如果字段的第一个类型不为字符串，则将字段的最后一个部分转换为列表，并存储在当前字典的"value"键下
else:
    curr["value"] = field.parts[-1].tolist()[0]

# 遍历读取器的张量列表，将张量的名称作为键，张量的索引、形状、类型和偏移量作为值存储在tensors字典中
for idx, tensor in enumerate(reader.tensors):
    tensors[tensor.name] = {
        "index": idx,
        "shape": tensor.shape.tolist(),
        "type": tensor.tensor_type.name,
        "offset": tensor.field.offset,
    }

# 将结果字典以JSON格式写入到标准输出
json.dump(result, sys.stdout)

# 定义主函数
def main() -> None:
    # 创建参数解析器对象，设置描述信息为"Dump GGUF file metadata"
    parser = argparse.ArgumentParser(description="Dump GGUF file metadata")
# 添加一个名为 "model" 的位置参数，类型为字符串，帮助信息为 "GGUF format model filename"
parser.add_argument("model", type=str, help="GGUF format model filename")
# 添加一个名为 "no-tensors" 的可选参数，如果设置则为True，帮助信息为 "Don't dump tensor metadata"
parser.add_argument("--no-tensors", action="store_true", help="Don't dump tensor metadata")
# 添加一个名为 "json" 的可选参数，如果设置则为True，帮助信息为 "Produce JSON output"
parser.add_argument("--json", action="store_true", help="Produce JSON output")
# 添加一个名为 "json-array" 的可选参数，如果设置则为True，帮助信息为 "Include full array values in JSON output (long)"
parser.add_argument("--json-array", action="store_true", help="Include full array values in JSON output (long)")
# 解析命令行参数，如果没有参数则显示帮助信息
args = parser.parse_args(None if len(sys.argv) > 1 else ["--help"])
# 如果不是 JSON 输出，则打印加载模型的信息
if not args.json:
    print(f'* Loading: {args.model}')
# 创建一个 GGUFReader 对象，读取模型文件
reader = GGUFReader(args.model, 'r')
# 如果是 JSON 输出，则以 JSON 格式输出模型的元数据
if args.json:
    dump_metadata_json(reader, args)
# 否则以普通格式输出模型的元数据
else:
    dump_metadata(reader, args)

# 如果是主程序入口
if __name__ == '__main__':
    # 调用主函数
    main()
```