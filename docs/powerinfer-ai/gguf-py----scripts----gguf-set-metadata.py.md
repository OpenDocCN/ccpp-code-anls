# `PowerInfer\gguf-py\scripts\gguf-set-metadata.py`

```
#!/usr/bin/env python3
# 指定脚本解释器为 Python3

import argparse
import os
import sys
from pathlib import Path

# 如果环境变量中没有设置 NO_LOCAL_GGUF，并且 gguf-py 包存在于父目录的父目录中，则将其路径添加到系统路径中
if "NO_LOCAL_GGUF" not in os.environ and (Path(__file__).parent.parent.parent / 'gguf-py').exists():
    sys.path.insert(0, str(Path(__file__).parent.parent))

from gguf import GGUFReader  # noqa: E402
# 导入 GGUFReader 类

def minimal_example(filename: str) -> None:
    # 创建 GGUFReader 对象，以读写模式打开指定文件
    reader = GGUFReader(filename, 'r+')
    # 获取字段 'tokenizer.ggml.bos_token_id'
    field = reader.fields['tokenizer.ggml.bos_token_id']
    # 如果字段不存在，则返回
    if field is None:
        return
    # 获取字段数据的第一个元素作为 part_index
    part_index = field.data[0]
    # 将字段的 parts 中的第一个元素设置为 2
    field.parts[part_index][0] = 2  # Set tokenizer.ggml.bos_token_id to 2
# 这个 field.data 是什么？它很有用，因为 field.parts 包含了 GGUF 字段的每一个部分。例如，tokenizer.ggml.bos_token_id 包括：
# 部分索引 0：键长度（27）
# 部分索引 1：键数据（"tokenizer.ggml.bos_token_id"）
# 部分索引 2：字段类型（4，GGUFValueType.UINT32 的 id）
# 部分索引 3：字段值
# 还要注意，每个部分都是一个 NDArray 切片，所以即使像键长度这样的单个值也是一个 NDArray 类型（numpy.uint32）。
# Field 中的 .data 属性是相关部分索引的列表，不包含像键长度部分这样的内部 GGUF 细节。在这种情况下，.data 将是 [3] - 只包含字段值本身的部分索引。
# 设置元数据，接受一个GGUFReader对象和一个argparse.Namespace对象作为参数，不返回任何结果
def set_metadata(reader: GGUFReader, args: argparse.Namespace) -> None:
    # 从reader对象中获取指定key对应的字段
    field = reader.get_field(args.key)
    # 如果字段不存在，则打印错误信息并退出程序
    if field is None:
        print(f'! Field {repr(args.key)} not found', file = sys.stderr)
        sys.exit(1)
    # 注意，field.types是一个类型列表，因为GGUF格式支持数组。例如，一个UINT32类型的数组看起来像[GGUFValueType.ARRAY, GGUFValueType.UINT32]
    # 根据字段类型获取对应的处理器
    handler = reader.gguf_scalar_to_np.get(field.types[0]) if field.types else None
    # 如果处理器不存在，则打印错误信息并退出程序
    if handler is None:
        print(
            f'! This tool only supports changing simple values, {repr(args.key)} has unsupported type {field.types}',
            file = sys.stderr,
        )
        sys.exit(1)
    # 获取当前字段的值
    current_value = field.parts[field.data[0]][0]
    # 将输入的值转换为处理器对应的类型
    new_value = handler(args.value)
    # 打印准备修改字段的信息
    print(f'* Preparing to change field {repr(args.key)} from {current_value} to {new_value}')
    # 如果当前值等于新值，则打印信息表示已经设置过了
    if current_value == new_value:
        print(f'- Key {repr(args.key)} already set to requested value {current_value}')
# 退出程序，返回状态码 0
sys.exit(0)
# 如果是 dry run 模式，退出程序，返回状态码 0
if args.dry_run:
    sys.exit(0)
# 如果不是强制执行模式，提示警告信息，需要用户确认是否继续操作
if not args.force:
    print('*** Warning *** Warning *** Warning **')
    print('* Changing fields in a GGUF file can make it unusable. Proceed at your own risk.')
    print('* Enter exactly YES if you are positive you want to proceed:')
    response = input('YES, I am sure> ')
    # 如果用户输入不是 YES，提示用户并退出程序，返回状态码 0
    if response != 'YES':
        print("You didn't enter YES. Okay then, see ya!")
        sys.exit(0)
# 修改字段的值
field.parts[field.data[0]][0] = new_value
# 打印字段修改成功的提示信息
print('* Field changed. Successful completion.')

# 主函数
def main() -> None:
    # 创建参数解析器
    parser = argparse.ArgumentParser(description="Set a simple value in GGUF file metadata")
    # 添加命令行参数
    parser.add_argument("model",     type=str,            help="GGUF format model filename")
    parser.add_argument("key",       type=str,            help="Metadata key to set")
    parser.add_argument("value",     type=str,            help="Metadata value to set")
# 添加一个名为--dry-run的命令行参数，如果存在则设置为True，用于指示不实际改变任何内容
parser.add_argument("--dry-run", action="store_true", help="Don't actually change anything")
# 添加一个名为--force的命令行参数，如果存在则设置为True，用于指示在不确认的情况下改变字段
parser.add_argument("--force",   action="store_true", help="Change the field without confirmation")
# 解析命令行参数，如果没有参数则显示帮助信息
args = parser.parse_args(None if len(sys.argv) > 1 else ["--help"])
# 打印加载模型的信息
print(f'* Loading: {args.model}')
# 根据参数设置读取模式，如果--dry-run存在则为只读模式，否则为读写模式
reader = GGUFReader(args.model, 'r' if args.dry_run else 'r+')
# 设置元数据
set_metadata(reader, args)

# 如果是主程序入口
if __name__ == '__main__':
    # 执行主程序
    main()
```