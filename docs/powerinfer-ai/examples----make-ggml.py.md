# `PowerInfer\examples\make-ggml.py`

```
#!/usr/bin/env python3
"""
This script converts Hugging Face Llama, StarCoder, Falcon, Baichuan, and GPT-NeoX models to GGUF and quantizes them.
"""

"""
Usage:
python make-ggml.py {model_dir_or_hf_repo_name} --model_type {model_type} [--outname {output_name} (Optional)] [--outdir {output_directory} (Optional)] [--quants {quant_types} (Optional)] [--keep_fp16 (Optional)]
"""

"""
Arguments:
- model: (Required) The directory of the downloaded Hugging Face model or the name of the Hugging Face model repository. If the model directory does not exist, it will be downloaded from the Hugging Face model hub.
- --model_type: (Required) The type of the model to be converted. Choose from llama, starcoder, falcon, baichuan, or gptneox.
- --outname: (Optional) The name of the output model. If not specified, the last part of the model directory path or the Hugging Face model repo name will be used.
- --outdir: (Optional) The directory where the output model(s) will be stored. If not specified, '../models/{outname}' will be used.
- --quants: (Optional) The types of quantization to apply. This should be a space-separated list. The default is 'Q4_K_M Q5_K_S'.
- --keep_fp16: (Optional) If specified, the FP16 model will not be deleted after the quantized models are created.
"""

"""
Old quant types (some base model types require these):
- Q4_0: small, very high quality loss - legacy, prefer using Q3_K_M
- Q4_1: small, substantial quality loss - legacy, prefer using Q3_K_L
- Q5_0: medium, balanced quality - legacy, prefer using Q4_K_M
- Q5_1: medium, low quality loss - legacy, prefer using Q5_K_M
"""

"""
New quant types (recommended):
- Q2_K: smallest, extreme quality loss - not recommended
- Q3_K: alias for Q3_K_M
- Q3_K_S: very small, very high quality loss
- Q3_K_M: very small, very high quality loss
- Q3_K_L: small, substantial quality loss
- Q4_K: alias for Q4_K_M
- Q4_K_S: small, significant quality loss
- Q4_K_M: medium, balanced quality - recommended
- Q5_K: alias for Q5_K_M
"""
# 定义不同的压缩质量和大小
- Q5_K_S: large, low quality loss - recommended
- Q5_K_M: large, very low quality loss - recommended
- Q6_K: very large, extremely low quality loss
- Q8_0: very large, extremely low quality loss - not recommended
- F16: extremely large, virtually no quality loss - not recommended
- F32: absolutely huge, lossless - not recommended

# 导入subprocess模块，用于运行外部命令
import subprocess
# 安装指定版本的huggingface-hub
subprocess.run(f"pip install huggingface-hub==0.16.4", shell=True, check=True)

# 导入argparse模块，用于解析命令行参数
import argparse
# 导入os模块，用于与操作系统交互
import os
# 从huggingface_hub模块中导入snapshot_download函数
from huggingface_hub import snapshot_download

# 定义主函数，接收模型、模型类型、输出名称、输出目录、压缩参数和是否保持fp16的参数
def main(model, model_type, outname, outdir, quants, keep_fp16):
    # 如果模型目录不存在，则下载模型
    if not os.path.isdir(model):
        print(f"Model not found at {model}. Downloading...")
        try:
            # 如果未指定输出名称，则使用模型路径中的最后一部分作为输出名称
            if outname is None:
                outname = model.split('/')[-1]
            # 下载模型到指定缓存目录
            model = snapshot_download(repo_id=model, cache_dir='../models/hf_cache')
        except Exception as e:
            raise Exception(f"Could not download the model: {e}")

    # 如果未指定输出目录，则使用默认输出目录
    if outdir is None:
        outdir = f'../models/{outname}'

    # 如果模型目录中不存在config.json文件，则抛出异常
    if not os.path.isfile(f"{model}/config.json"):
        raise Exception(f"Could not find config.json in {model}")

    # 创建输出目录（如果不存在）
    os.makedirs(outdir, exist_ok=True)

    # 编译llama.cpp
    print("Building llama.cpp")
    subprocess.run(f"cd .. && make quantize", shell=True, check=True)

    # 定义fp16文件路径
    fp16 = f"{outdir}/{outname}.gguf.fp16.bin"

    # 生成未压缩的GGUF文件
    print(f"Making unquantised GGUF at {fp16}")
    if not os.path.isfile(fp16):
        # 如果模型类型不是llama，则使用对应的转换脚本进行转换
        if model_type != "llama":
            subprocess.run(f"python3 ../convert-{model_type}-hf-to-gguf.py {model} 1 --outfile {fp16}", shell=True, check=True)
        else:
            subprocess.run(f"python3 ../convert.py {model} --outtype f16 --outfile {fp16}", shell=True, check=True)
    else:
        print(f"Unquantised GGML already exists at: {fp16}")

    # 生成压缩文件
    print("Making quants")
    for type in quants:
        outfile = f"{outdir}/{outname}.gguf.{type}.bin"
        print(f"Making {type} : {outfile}")
        subprocess.run(f"../quantize {fp16} {outfile} {type}", shell=True, check=True)
    # 如果不需要保留 fp16 文件，则删除该文件
    if not keep_fp16:
        os.remove(fp16)
# 如果当前脚本被直接执行，则执行以下代码
if __name__ == "__main__":
    # 创建一个参数解析器对象，设置描述信息
    parser = argparse.ArgumentParser(description='Convert/Quantize HF models to GGUF. If you have the HF model downloaded already, pass the path to the model dir. Otherwise, pass the Hugging Face model repo name. You need to be in the /examples folder for it to work.')
    # 添加一个位置参数，用于接收下载的模型目录或 Hugging Face 模型仓库名称
    parser.add_argument('model', help='Downloaded model dir or Hugging Face model repo name')
    # 添加一个必选参数，用于指定模型类型，可选值为 llama, starcoder, falcon, baichuan, or gptneox
    parser.add_argument('--model_type', required=True, choices=['llama', 'starcoder', 'falcon', 'baichuan', 'gptneox'], help='Type of the model to be converted. Choose from llama, starcoder, falcon, baichuan, or gptneox.')
    # 添加一个可选参数，用于指定输出模型的名称
    parser.add_argument('--outname', default=None, help='Output model(s) name')
    # 添加一个可选参数，用于指定输出目录
    parser.add_argument('--outdir', default=None, help='Output directory')
    # 添加一个可选参数，用于指定量化类型，默认值为["Q4_K_M", "Q5_K_S"]
    parser.add_argument('--quants', nargs='*', default=["Q4_K_M", "Q5_K_S"], help='Quant types')
    # 添加一个布尔类型的可选参数，用于指定是否保留 fp16 模型，默认值为 False
    parser.add_argument('--keep_fp16', action='store_true', help='Keep fp16 model', default=False)

    # 解析命令行参数
    args = parser.parse_args()

    # 调用 main 函数，传入解析后的参数
    main(args.model, args.model_type, args.outname, args.outdir, args.quants, args.keep_fp16)
```