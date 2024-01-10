# `PowerInfer\run_with_preset.py`

```
# 导入 argparse 模块，用于解析命令行参数
import argparse
# 导入 os 模块，用于提供与操作系统交互的功能
import os
# 导入 subprocess 模块，用于生成子进程
import subprocess
# 导入 sys 模块，用于提供对 Python 解释器的访问
import sys
# 导入 yaml 模块，用于读写 YAML 文件
import yaml

# 定义三个不同的 CLI 参数列表
CLI_ARGS_MAIN_PERPLEXITY = [
    "batch-size", "cfg-negative-prompt", "cfg-scale", "chunks", "color", "ctx-size", "escape",
    "export", "file", "frequency-penalty", "grammar", "grammar-file", "hellaswag",
    "hellaswag-tasks", "ignore-eos", "in-prefix", "in-prefix-bos", "in-suffix", "instruct",
    "interactive", "interactive-first", "keep", "logdir", "logit-bias", "lora", "lora-base",
    "low-vram", "main-gpu", "memory-f32", "mirostat", "mirostat-ent", "mirostat-lr", "mlock",
    "model", "multiline-input", "n-gpu-layers", "n-predict", "no-mmap", "no-mul-mat-q",
    "np-penalize-nl", "numa", "ppl-output-type", "ppl-stride", "presence-penalty", "prompt",
    "prompt-cache", "prompt-cache-all", "prompt-cache-ro", "random-prompt", "repeat-last-n",
    "repeat-penalty", "reverse-prompt", "rope-freq-base", "rope-freq-scale", "rope-scale", "seed",
    "simple-io", "tensor-split", "threads", "temp", "tfs", "top-k", "top-p", "typical",
    "verbose-prompt"
]

CLI_ARGS_LLAMA_BENCH = [
    "batch-size", "memory-f32", "low-vram", "model", "mul-mat-q", "n-gen", "n-gpu-layers",
    "n-prompt", "output", "repetitions", "tensor-split", "threads", "verbose"
]

CLI_ARGS_SERVER = [
    "alias", "batch-size", "ctx-size", "embedding", "host", "memory-f32", "lora", "lora-base",
    "low-vram", "main-gpu", "mlock", "model", "n-gpu-layers", "n-probs", "no-mmap", "no-mul-mat-q",
    "numa", "path", "port", "rope-freq-base", "timeout", "rope-freq-scale", "tensor-split",
    "threads", "verbose"
]

# 定义描述信息
description = """Run llama.cpp binaries with presets from YAML file(s).
To specify which binary should be run, specify the "binary" property (main, perplexity, llama-bench, and server are supported).
To get a preset file template, run a llama.cpp binary with the "--logdir" CLI argument.

Formatting considerations:
# 定义一些关于 YAML 属性和 CLI 参数对应关系的规则
- YAML 属性的名称与相应二进制文件的 CLI 参数名称相同。
- 属性必须使用 llama.cpp CLI 参数的长名称。
- 与 llama.cpp 二进制文件一样，属性名称不区分连字符和下划线。
- 标志必须定义为 "<PROPERTY_NAME>: true" 才能生效。
- 要定义 logit_bias 属性，期望的格式是在 "logit_bias" 命名空间中的 "<TOKEN_ID>: <BIAS>"。
- 同时定义多个 "reverse_prompt" 属性的期望格式是一个字符串列表。
- 要定义张量分割，传递一个浮点数列表。

# 定义用法和结尾说明
usage = "run_with_preset.py [-h] [yaml_files ...] [--<ARG_NAME> <ARG_VALUE> ...]"
epilog = ("  --<ARG_NAME> specify additional CLI ars to be passed to the binary (override all preset files). "
          "Unknown args will be ignored.")

# 创建解析器对象
parser = argparse.ArgumentParser(
    description=description, usage=usage, epilog=epilog, formatter_class=argparse.RawTextHelpFormatter)
# 添加命令行参数
parser.add_argument("-bin", "--binary", help="The binary to run.")
parser.add_argument("yaml_files", nargs="*",
                    help="Arbitrary number of YAML files from which to read preset values. "
                    "If two files specify the same values the later one will be used.")

# 解析已知和未知的命令行参数
known_args, unknown_args = parser.parse_known_args()

# 如果没有已知的 YAML 文件和未知的参数，则打印帮助信息并退出
if not known_args.yaml_files and not unknown_args:
    parser.print_help()
    sys.exit(0)

# 创建属性字典
props = dict()

# 遍历已知的 YAML 文件，读取预设值并更新属性字典
for yaml_file in known_args.yaml_files:
    with open(yaml_file, "r") as f:
        props.update(yaml.load(f, yaml.SafeLoader))

# 将属性名称中的下划线替换为连字符
props = {prop.replace("_", "-"): val for prop, val in props.items()}

# 获取二进制文件名并设置默认值
binary = props.pop("binary", "main")
if known_args.binary:
    binary = known_args.binary

# 如果二进制文件存在，则设置二进制文件路径
if os.path.exists(f"./{binary}"):
    binary = f"./{binary}"

# 根据二进制文件名的后缀选择 CLI 参数
if binary.lower().endswith("main") or binary.lower().endswith("perplexity"):
    cli_args = CLI_ARGS_MAIN_PERPLEXITY
elif binary.lower().endswith("llama-bench"):
    # 从常量 CLI_ARGS_LLAMA_BENCH 中获取命令行参数
    cli_args = CLI_ARGS_LLAMA_BENCH
# 如果给定的二进制文件名以"server"结尾，则使用服务器命令行参数
elif binary.lower().endswith("server"):
    cli_args = CLI_ARGS_SERVER
# 否则，打印未知的二进制文件名并退出程序
else:
    print(f"Unknown binary: {binary}")
    sys.exit(1)

# 创建命令列表，初始值为给定的二进制文件名
command_list = [binary]

# 遍历命令行参数列表
for cli_arg in cli_args:
    # 从属性字典中弹出对应的值
    value = props.pop(cli_arg, None)

    # 如果值为空或为-1，则跳过当前循环
    if not value or value == -1:
        continue

    # 如果命令行参数为"logit-bias"
    if cli_arg == "logit-bias":
        # 遍历值中的token和bias，将它们添加到命令列表中
        for token, bias in value.items():
            command_list.append("--logit-bias")
            command_list.append(f"{token}{bias:+}")
        continue

    # 如果命令行参数为"reverse-prompt"且值不是字符串
    if cli_arg == "reverse-prompt" and not isinstance(value, str):
        # 遍历值中的每个reverse-prompt，将它们添加到命令列表中
        for rp in value:
            command_list.append("--reverse-prompt")
            command_list.append(str(rp))
        continue

    # 将命令行参数添加到命令列表中
    command_list.append(f"--{cli_arg}")

    # 如果命令行参数为"tensor-split"
    if cli_arg == "tensor-split":
        # 将值转换为字符串并用逗号连接，然后添加到命令列表中
        command_list.append(",".join([str(v) for v in value]))
        continue

    # 将值转换为字符串并添加到命令列表中
    value = str(value)
    if value != "True":
        command_list.append(str(value))

# 计算未使用的属性数量
num_unused = len(props)
# 如果未使用的属性数量大于10，则打印提示信息
if num_unused > 10:
    print(f"The preset file contained a total of {num_unused} unused properties.")
# 如果未使用的属性数量大于0，则打印未使用的属性列表
elif num_unused > 0:
    print("The preset file contained the following unused properties:")
    for prop, value in props.items():
        print(f"  {prop}: {value}")

# 将未知的参数添加到命令列表中
command_list += unknown_args

# 使用命令列表创建子进程
sp = subprocess.Popen(command_list)

# 当子进程的返回码不为None时，循环等待子进程结束
while sp.returncode is None:
    try:
        sp.wait()
    except KeyboardInterrupt:
        pass

# 退出程序，返回子进程的返回码
sys.exit(sp.returncode)
```