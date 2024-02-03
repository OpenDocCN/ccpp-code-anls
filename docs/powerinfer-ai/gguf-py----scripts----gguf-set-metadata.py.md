# `PowerInfer\gguf-py\scripts\gguf-set-metadata.py`

```cpp
#!/usr/bin/env python3
import argparse  # 导入命令行参数解析模块
import os  # 导入操作系统模块
import sys  # 导入系统模块
from pathlib import Path  # 从路径模块中导入 Path 类

# 如果环境变量中没有设置 NO_LOCAL_GGUF，并且 gguf-py 包存在于父目录的父目录中，则将其路径插入到系统路径中
if "NO_LOCAL_GGUF" not in os.environ and (Path(__file__).parent.parent.parent / 'gguf-py').exists():
    sys.path.insert(0, str(Path(__file__).parent.parent))

from gguf import GGUFReader  # noqa: E402  # 从 gguf 模块中导入 GGUFReader 类


def minimal_example(filename: str) -> None:
    # 创建 GGUFReader 对象
    reader = GGUFReader(filename, 'r+')
    # 获取字段 'tokenizer.ggml.bos_token_id'
    field = reader.fields['tokenizer.ggml.bos_token_id']
    # 如果字段不存在，则返回
    if field is None:
        return
    # 获取字段数据的第一个元素
    part_index = field.data[0]
    # 设置字段 'tokenizer.ggml.bos_token_id' 的值为 2
    field.parts[part_index][0] = 2
    #
    # So what's this field.data thing? It's helpful because field.parts contains
    # _every_ part of the GGUF field. For example, tokenizer.ggml.bos_token_id consists
    # of:
    #
    #  Part index 0: Key length (27)
    #  Part index 1: Key data ("tokenizer.ggml.bos_token_id")
    #  Part index 2: Field type (4, the id for GGUFValueType.UINT32)
    #  Part index 3: Field value
    #
    # Note also that each part is an NDArray slice, so even a part that
    # is only a single value like the key length will be a NDArray of
    # the key length type (numpy.uint32).
    #
    # The .data attribute in the Field is a list of relevant part indexes
    # and doesn't contain internal GGUF details like the key length part.
    # In this case, .data will be [3] - just the part index of the
    # field value itself.


def set_metadata(reader: GGUFReader, args: argparse.Namespace) -> None:
    # 获取指定字段
    field = reader.get_field(args.key)
    # 如果字段不存在，则打印错误信息并退出程序
    if field is None:
        print(f'! Field {repr(args.key)} not found', file = sys.stderr)
        sys.exit(1)
    # 注意，field.types 是一个类型列表。这是因为 GGUF 格式支持数组。例如，一个 UINT32 数组看起来像 [GGUFValueType.ARRAY, GGUFValueType.UINT32]
    handler = reader.gguf_scalar_to_np.get(field.types[0]) if field.types else None
    # 如果处理程序为空，则打印错误信息并退出程序
    if handler is None:
        print(
            f'! This tool only supports changing simple values, {repr(args.key)} has unsupported type {field.types}',
            file = sys.stderr,
        )
        sys.exit(1)
    # 获取当前字段的值
    current_value = field.parts[field.data[0]][0]
    # 使用处理程序处理新值
    new_value = handler(args.value)
    # 打印准备修改字段的信息
    print(f'* Preparing to change field {repr(args.key)} from {current_value} to {new_value}')
    # 如果当前值等于新值，则打印信息并退出程序
    if current_value == new_value:
        print(f'- Key {repr(args.key)} already set to requested value {current_value}')
        sys.exit(0)
    # 如果是模拟运行，则退出程序
    if args.dry_run:
        sys.exit(0)
    # 如果不是强制执行，则打印警告信息，并根据用户输入决定是否继续
    if not args.force:
        print('*** Warning *** Warning *** Warning **')
        print('* Changing fields in a GGUF file can make it unusable. Proceed at your own risk.')
        print('* Enter exactly YES if you are positive you want to proceed:')
        response = input('YES, I am sure> ')
        if response != 'YES':
            print("You didn't enter YES. Okay then, see ya!")
            sys.exit(0)
    # 修改字段的值为新值
    field.parts[field.data[0]][0] = new_value
    # 打印字段修改成功的信息
    print('* Field changed. Successful completion.')
# 定义主函数，不返回任何结果
def main() -> None:
    # 创建参数解析器对象，设置程序描述
    parser = argparse.ArgumentParser(description="Set a simple value in GGUF file metadata")
    # 添加模型文件名参数，类型为字符串，帮助信息为GGUF格式模型文件名
    parser.add_argument("model",     type=str,            help="GGUF format model filename")
    # 添加元数据键参数，类型为字符串，帮助信息为要设置的元数据键
    parser.add_argument("key",       type=str,            help="Metadata key to set")
    # 添加元数据值参数，类型为字符串，帮助信息为要设置的元数据值
    parser.add_argument("value",     type=str,            help="Metadata value to set")
    # 添加可选参数--dry-run，动作为存储布尔值，帮助信息为不实际改变任何内容
    parser.add_argument("--dry-run", action="store_true", help="Don't actually change anything")
    # 添加可选参数--force，动作为存储布尔值，帮助信息为在不确认的情况下更改字段
    parser.add_argument("--force",   action="store_true", help="Change the field without confirmation")
    # 解析命令行参数，如果没有参数则显示帮助信息
    args = parser.parse_args(None if len(sys.argv) > 1 else ["--help"])
    # 打印加载模型文件的提示信息
    print(f'* Loading: {args.model}')
    # 创建GGUFReader对象，打开模型文件以供读取或读取和写入
    reader = GGUFReader(args.model, 'r' if args.dry_run else 'r+')
    # 调用set_metadata函数，传入reader对象和命令行参数args
    set_metadata(reader, args)

# 如果当前脚本被直接执行，则调用主函数
if __name__ == '__main__':
    main()
```