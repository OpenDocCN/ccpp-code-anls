# `PowerInfer\run_with_preset.py`

```
#!/usr/bin/env python3
# 指定脚本的解释器为 Python 3

import argparse
# 导入用于解析命令行参数的模块
import os
# 导入用于与操作系统交互的模块
import subprocess
# 导入用于在子进程中执行命令的模块
import sys
# 导入用于与 Python 解释器交互的模块

import yaml
# 导入用于解析 YAML 格式的模块

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
]
# 定义一个包含命令行参数的列表
# 定义不同的命令行参数列表，用于不同的 llama.cpp 二进制文件
CLI_ARGS_MAIN = [
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

# 定义描述信息，用于说明 llama.cpp 二进制文件的预设参数
description = """Run llama.cpp binaries with presets from YAML file(s).
To specify which binary should be run, specify the "binary" property (main, perplexity, llama-bench, and server are supported).
To get a preset file template, run a llama.cpp binary with the "--logdir" CLI argument.

Formatting considerations:
# 定义了一些关于 YAML 属性和 CLI 参数的规则
- YAML 属性的名称与相应二进制文件的 CLI 参数名称相同。
- 属性必须使用其对应 llama.cpp CLI 参数的长名称。
- 与 llama.cpp 二进制文件一样，属性名称不区分连字符和下划线。
- 标志必须定义为 "<PROPERTY_NAME>: true" 才能生效。
- 要定义 logit_bias 属性，期望的格式是在 "logit_bias" 命名空间中的 "<TOKEN_ID>: <BIAS>"。
- 要同时定义多个 "reverse_prompt" 属性，期望的格式是一个字符串列表。
- 要定义张量分割，传递一个浮点数列表。

# 定义了程序的使用说明和结尾说明
usage = "run_with_preset.py [-h] [yaml_files ...]"
epilog = ("  --<ARG_NAME> 指定要传递给二进制文件的额外 CLI 参数（覆盖所有预设文件）。"
          "未知参数将被忽略。")

# 创建解析器对象
parser = argparse.ArgumentParser(
    description=description, usage=usage, epilog=epilog, formatter_class=argparse.RawTextHelpFormatter)
# 添加命令行参数
parser.add_argument("-bin", "--binary", help="要运行的二进制文件。")
parser.add_argument("yaml_files", nargs="*",
                    help="要读取预设值的任意数量 YAML 文件。"
                    "如果两个文件指定相同的值，将使用后一个文件。")

# 解析已知和未知的命令行参数
known_args, unknown_args = parser.parse_known_args()
# 如果没有指定yaml_files和unknown_args，则打印帮助信息并退出程序
if not known_args.yaml_files and not unknown_args:
    parser.print_help()
    sys.exit(0)

# 创建一个空字典
props = dict()

# 遍历已知的yaml文件，读取文件内容并更新到props字典中
for yaml_file in known_args.yaml_files:
    with open(yaml_file, "r") as f:
        props.update(yaml.load(f, yaml.SafeLoader))

# 将props字典中的属性名中的下划线替换为连字符
props = {prop.replace("_", "-"): val for prop, val in props.items()}

# 从props字典中弹出"binary"属性，如果没有则使用"main"
binary = props.pop("binary", "main")

# 如果已知参数中有binary属性，则使用已知参数中的值
if known_args.binary:
    binary = known_args.binary

# 如果当前目录下存在binary文件，则使用该文件
if os.path.exists(f"./{binary}"):
    binary = f"./{binary}"
# 如果文件名以"main"或"perplexity"结尾，则使用CLI_ARGS_MAIN_PERPLEXITY参数
if binary.lower().endswith("main") or binary.lower().endswith("perplexity"):
    cli_args = CLI_ARGS_MAIN_PERPLEXITY
# 如果文件名以"llama-bench"结尾，则使用CLI_ARGS_LLAMA_BENCH参数
elif binary.lower().endswith("llama-bench"):
    cli_args = CLI_ARGS_LLAMA_BENCH
# 如果文件名以"server"结尾，则使用CLI_ARGS_SERVER参数
elif binary.lower().endswith("server"):
    cli_args = CLI_ARGS_SERVER
# 如果文件名不符合以上条件，则打印未知文件名并退出程序
else:
    print(f"Unknown binary: {binary}")
    sys.exit(1)

# 创建命令列表，将文件名添加到列表中
command_list = [binary]

# 遍历参数列表
for cli_arg in cli_args:
    # 从参数字典中取出对应的值
    value = props.pop(cli_arg, None)

    # 如果值为空或为-1，则跳过当前循环
    if not value or value == -1:
        continue

    # 如果参数为"logit-bias"，则遍历值中的token和bias
    if cli_arg == "logit-bias":
        for token, bias in value.items():
# 将"--logit-bias"添加到命令列表中
command_list.append("--logit-bias")
# 将带有偏置值的字符串添加到命令列表中
command_list.append(f"{token}{bias:+}")
# 继续下一次循环

# 如果命令行参数是"reverse-prompt"且值不是字符串类型
if cli_arg == "reverse-prompt" and not isinstance(value, str):
    # 遍历值列表，将"--reverse-prompt"和值添加到命令列表中
    for rp in value:
        command_list.append("--reverse-prompt")
        command_list.append(str(rp))
    # 继续下一次循环

# 将"--{cli_arg}"添加到命令列表中
command_list.append(f"--{cli_arg}")

# 如果命令行参数是"tensor-split"
if cli_arg == "tensor-split":
    # 将值列表中的元素转换为字符串并用逗号连接，然后添加到命令列表中
    command_list.append(",".join([str(v) for v in value])
    # 继续下一次循环

# 将值转换为字符串
value = str(value)

# 如果值不等于"True"
if value != "True":
    # 将值添加到命令列表中
    command_list.append(str(value))
# 计算未使用的属性的数量
num_unused = len(props)
# 如果未使用的属性数量大于10，打印提示信息
if num_unused > 10:
    print(f"The preset file contained a total of {num_unused} unused properties.")
# 如果未使用的属性数量大于0但小于等于10，打印未使用的属性列表
elif num_unused > 0:
    print("The preset file contained the following unused properties:")
    for prop, value in props.items():
        print(f"  {prop}: {value}")

# 将未知参数添加到命令列表中
command_list += unknown_args

# 使用 subprocess 模块执行命令列表中的命令
sp = subprocess.Popen(command_list)

# 循环等待子进程返回结果
while sp.returncode is None:
    try:
        sp.wait()
    except KeyboardInterrupt:
        pass

# 退出程序并返回子进程的返回码
sys.exit(sp.returncode)
抱歉，我无法为您提供代码注释，因为您没有提供任何代码。如果您有任何需要帮助的代码，请随时告诉我。我会尽力帮助您。
```