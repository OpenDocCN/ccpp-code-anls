# `PowerInfer\examples\make-ggml.py`

```
#!/usr/bin/env python3
"""
这个脚本将 Hugging Face Llama、StarCoder、Falcon、Baichuan 和 GPT-NeoX 模型转换为 GGUF 并对其进行量化。

用法：

参数：
- --model_type:（必需）要转换的模型类型。可选择 llama、starcoder、falcon、baichuan 或 gptneox。
- --outname:（可选）输出模型的名称。如果未指定，则将使用模型目录路径的最后一部分或 Hugging Face 模型仓库的名称。
- --outdir:（可选）存储输出模型的目录。如果未指定，将使用'../models/{outname}'。
- --quants:（可选）要应用的量化类型。这应该是一个以空格分隔的列表。默认值为'Q4_K_M Q5_K_S'。
- --keep_fp16:（可选）如果指定，则在创建量化模型后不会删除 FP16 模型。

旧的量化类型（一些基础模型类型需要这些）：
- Q4_0: 小型，非常高质量的损失 - 传统，建议使用 Q3_K_M
- Q4_1: 小型，实质性的质量损失 - 传统，建议使用 Q3_K_L
- Q5_0: 中型，平衡的质量 - 传统，建议使用 Q4_K_M
- Q5_1: 中型，低质量损失 - 传统，建议使用 Q5_K_M

新的量化类型（推荐）：
# 定义不同压缩质量级别的标识和描述
- Q2_K: 最小，极端质量损失 - 不推荐使用
- Q3_K: Q3_K_M 的别名
- Q3_K_S: 非常小，非常高的质量损失
- Q3_K_M: 非常小，非常高的质量损失
- Q3_K_L: 小，质量损失较大
- Q4_K: Q4_K_M 的别名
- Q4_K_S: 小，质量损失显著
- Q4_K_M: 中等，平衡的质量 - 推荐使用
- Q5_K: Q5_K_M 的别名
- Q5_K_S: 大，低质量损失 - 推荐使用
- Q5_K_M: 大，非常低的质量损失 - 推荐使用
- Q6_K: 非常大，极低的质量损失
- Q8_0: 非常大，极低的质量损失 - 不推荐使用
- F16: 极大，几乎没有质量损失 - 不推荐使用
- F32: 绝对巨大，无损 - 不推荐使用

# 使用 subprocess 模块运行命令行安装指定版本的 huggingface-hub
import subprocess
subprocess.run(f"pip install huggingface-hub==0.16.4", shell=True, check=True)

# 导入 argparse 模块
import argparse
# 导入 os 模块和 huggingface_hub 模块中的 snapshot_download 函数
import os
from huggingface_hub import snapshot_download

# 定义主函数，接受模型名称、模型类型、输出文件名、输出目录、量化参数和是否保持 FP16 作为参数
def main(model, model_type, outname, outdir, quants, keep_fp16):
    # 如果指定的模型路径不存在
    if not os.path.isdir(model):
        # 打印提示信息
        print(f"Model not found at {model}. Downloading...")
        # 尝试下载模型
        try:
            # 如果未指定输出文件名，则使用模型路径中的最后一部分作为输出文件名
            if outname is None:
                outname = model.split('/')[-1]
            # 使用 snapshot_download 函数下载模型到指定缓存目录
            model = snapshot_download(repo_id=model, cache_dir='../models/hf_cache')
        # 捕获异常并抛出新的异常
        except Exception as e:
            raise Exception(f"Could not download the model: {e}")

    # 如果未指定输出目录
    if outdir is None:
        # 使用默认输出目录路径
        outdir = f'../models/{outname}'

    # 如果模型目录中不存在 config.json 文件
    if not os.path.isfile(f"{model}/config.json"):
        # 抛出异常
        raise Exception(f"Could not find config.json in {model}")

    # 创建输出目录，如果目录已存在则不做任何操作
    os.makedirs(outdir, exist_ok=True)
# 打印信息，表示正在构建 llama.cpp
print("Building llama.cpp")
# 在上级目录执行 make quantize 命令，使用 shell=True 表示在 shell 中执行，check=True 表示如果返回值不为 0 则抛出异常
subprocess.run(f"cd .. && make quantize", shell=True, check=True)

# 设置 fp16 变量为输出目录和输出名称拼接而成的路径
fp16 = f"{outdir}/{outname}.gguf.fp16.bin"

# 打印信息，表示正在生成未量化的 GGUF 文件
print(f"Making unquantised GGUF at {fp16}")
# 如果 fp16 文件不存在
if not os.path.isfile(fp16):
    # 如果模型类型不是 llama，则执行相应的转换脚本，否则执行默认的转换脚本
    if model_type != "llama":
        subprocess.run(f"python3 ../convert-{model_type}-hf-to-gguf.py {model} 1 --outfile {fp16}", shell=True, check=True)
    else:
        subprocess.run(f"python3 ../convert.py {model} --outtype f16 --outfile {fp16}", shell=True, check=True)
# 如果 fp16 文件已经存在
else:
    print(f"Unquantised GGML already exists at: {fp16}")

# 打印信息，表示正在生成量化文件
print("Making quants")
# 遍历 quants 列表中的每个类型
for type in quants:
    # 设置 outfile 变量为输出目录和输出名称以及类型拼接而成的路径
    outfile = f"{outdir}/{outname}.gguf.{type}.bin"
    # 打印信息，表示正在生成指定类型的文件
    print(f"Making {type} : {outfile}")
    # 执行 quantize 命令，将 fp16 文件转换为指定类型的文件，使用 shell=True 表示在 shell 中执行，check=True 表示如果返回值不为 0 则抛出异常
    subprocess.run(f"../quantize {fp16} {outfile} {type}", shell=True, check=True)
# 如果不保留 fp16 模型，则删除该模型
if not keep_fp16:
    os.remove(fp16)

# 如果作为主程序运行
if __name__ == "__main__":
    # 添加命令行参数：模型目录或 Hugging Face 模型库名称
    parser.add_argument('model', help='Downloaded model dir or Hugging Face model repo name')
    # 添加命令行参数：输出模型名称
    parser.add_argument('--outname', default=None, help='Output model(s) name')
    # 添加命令行参数：输出目录
    parser.add_argument('--outdir', default=None, help='Output directory')
    # 添加命令行参数：量化类型
    parser.add_argument('--quants', nargs='*', default=["Q4_K_M", "Q5_K_S"], help='Quant types')
    # 添加命令行参数：保留 fp16 模型
    parser.add_argument('--keep_fp16', action='store_true', help='Keep fp16 model', default=False)

    # 解析命令行参数
    args = parser.parse_args()

    # 调用主函数，传入参数
    main(args.model, args.model_type, args.outname, args.outdir, args.quants, args.keep_fp16)
```