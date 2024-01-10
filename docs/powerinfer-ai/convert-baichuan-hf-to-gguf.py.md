# `PowerInfer\convert-baichuan-hf-to-gguf.py`

```
#!/usr/bin/env python3
# 指定脚本解释器为 Python 3

# HF baichuan --> gguf conversion
# HuggingFace baichuan 转换为 gguf

from __future__ import annotations

import argparse  # 导入命令行参数解析模块
import json  # 导入 JSON 模块
import os  # 导入操作系统模块
import struct  # 导入 struct 模块
import sys  # 导入系统模块
from pathlib import Path  # 从路径模块中导入 Path 类
from typing import TYPE_CHECKING, Any  # 从类型提示模块中导入类型

import itertools  # 导入迭代工具模块
import numpy as np  # 导入 NumPy 模块，并使用别名 np
import torch  # 导入 PyTorch 模块
from sentencepiece import SentencePieceProcessor  # 从 sentencepiece 模块中导入 SentencePieceProcessor 类

if 'NO_LOCAL_GGUF' not in os.environ:
    sys.path.insert(1, str(Path(__file__).parent / 'gguf-py'))
import gguf  # 导入 gguf 模块

if TYPE_CHECKING:
    from typing import TypeAlias  # 如果是类型检查，从类型提示模块中导入 TypeAlias 类型别名

NDArray: TypeAlias = 'np.ndarray[Any, Any]'  # 定义类型别名 NDArray 为字符串 'np.ndarray[Any, Any]'

# reverse HF permute back to original pth layout
# 将 HuggingFace 的排列逆转回原始的 pth 布局

def reverse_hf_permute(weights: NDArray, n_head: int, n_kv_head: int | None = None) -> NDArray:
    # 如果 n_kv_head 不为 None 且 n_head 不等于 n_kv_head
    if n_kv_head is not None and n_head != n_kv_head:
        n_head //= n_kv_head

    # 重新排列权重张量，并返回结果
    return (weights.reshape(n_head, 2, weights.shape[0] // n_head // 2, *weights.shape[1:])
            .swapaxes(1, 2)
            .reshape(weights.shape))

def reverse_hf_permute_part(weights: NDArray, n_part: int, n_head: int, n_head_kv: int| None = None) -> NDArray:
    # 计算每个部分的大小
    r = weights.shape[0] // 3
    # 调用 reverse_hf_permute 函数，返回结果
    return (reverse_hf_permute(weights[r * n_part : r * n_part + r, ...], n_head, n_head_kv))

def reverse_hf_part(weights: NDArray, n_part: int) -> NDArray:
    # 计算每个部分的大小
    r = weights.shape[0] // 3
    # 返回指定部分的权重张量
    return weights[r * n_part : r * n_part + r, ...]

def count_model_parts(dir_model: str) -> int:
    # 初始化模型部分数量为 0
    num_parts = 0

    # 遍历模型目录下的文件
    for filename in os.listdir(dir_model):
        # 如果文件名以 "pytorch_model-" 开头
        if filename.startswith("pytorch_model-"):
            # 模型部分数量加一
            num_parts += 1

    # 如果模型部分数量大于 0，则打印找到的模型部分数量
    if num_parts > 0:
        print("gguf: found " + str(num_parts) + " model parts")

    # 返回模型部分数量
    return num_parts

def parse_args() -> argparse.Namespace:
    # 创建参数解析器对象，设置描述信息
    parser = argparse.ArgumentParser(description="Convert a HuggingFace LLaMA model to a GGML compatible file")
    # 添加参数选项 --vocab-only，用于提取词汇表
    parser.add_argument(
        "--vocab-only", action="store_true",
        help="extract only the vocab",
    )
    # 添加一个名为“--outfile”的命令行参数，类型为Path，用于指定输出文件的路径，默认值为基于输入的路径
    parser.add_argument(
        "--outfile", type=Path,
        help="path to write to; default: based on input",
    )
    # 添加一个名为“model”的位置参数，类型为Path，用于指定包含模型文件的目录，或者模型文件本身（*.bin）
    parser.add_argument(
        "model", type=Path,
        help="directory containing model file, or model file itself (*.bin)",
    )
    # 添加一个名为“ftype”的位置参数，类型为int，可选值为0或1，默认值为1，可选参数个数为1，用于指定输出格式，0表示float32，1表示float16
    parser.add_argument(
        "ftype", type=int, choices=[0, 1], default=1, nargs='?',
        help="output format - use 0 for float32, 1 for float16",
    )
    # 添加一个名为“--bigendian”的命令行参数，类型为布尔值，用于指定模型在大端机器上执行
    parser.add_argument("--bigendian",   action="store_true",    help="model is executed on big endian machine")
    # 解析命令行参数并返回结果
    return parser.parse_args()
# 解析命令行参数
args = parse_args()

# 获取模型目录和文件类型
dir_model = args.model
ftype = args.ftype

# 如果模型目录不存在，则输出错误信息并退出程序
if not dir_model.is_dir():
    print(f'Error: {args.model} is not a directory', file = sys.stderr)
    sys.exit(1)

# 根据命令行参数设置字节序
endianess = gguf.GGUFEndian.LITTLE
if args.bigendian:
    endianess = gguf.GGUFEndian.BIG
endianess_str = "Big Endian" if args.bigendian else "Little Endian"
print(f"gguf: Conversion Endianess {endianess}")

# 定义可能的张量数据类型
#   ftype == 0 -> float32
#   ftype == 1 -> float16
# 将文件类型映射为字符串
ftype_str = ["f32", "f16"]

# 如果输出文件名不为空，则使用命令行参数中指定的输出文件名，否则使用默认文件名
if args.outfile is not None:
    fname_out = args.outfile
else:
    fname_out = dir_model / f'ggml-model-{ftype_str[ftype]}.gguf'

# 输出加载模型的信息
print("gguf: loading model "+dir_model.name)

# 读取模型配置文件
with open(dir_model / "config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)
print("hello print: ",hparams["architectures"][0])

# 如果模型架构不受支持，则输出错误信息并退出程序
if hparams["architectures"][0] != "BaichuanForCausalLM" and hparams["architectures"][0] != "BaiChuanForCausalLM":
    print("Model architecture not supported: " + hparams["architectures"][0])
    sys.exit()

# 获取模型部件的数量
num_parts = count_model_parts(dir_model)
print(f"num_parts:{num_parts}\n")

# 设置模型架构和创建 GGUFWriter 对象
ARCH=gguf.MODEL_ARCH.BAICHUAN
gguf_writer = gguf.GGUFWriter(fname_out, gguf.MODEL_ARCH_NAMES[ARCH], endianess=endianess)

# 输出获取模型元数据的信息
print("gguf: get model metadata")

# 获取模型块数和头数
block_count = hparams["num_hidden_layers"]
head_count = hparams["num_attention_heads"]

# 如果模型中存在键值头的数量，则使用该数量，否则使用注意力头的数量
if "num_key_value_heads" in hparams:
    head_count_kv = hparams["num_key_value_heads"]
else:
    head_count_kv = head_count

# 如果模型中存在"_name_or_path"字段，则使用该字段的值，否则为空字符串
if "_name_or_path" in hparams:
    hf_repo = hparams["_name_or_path"]
else:
    hf_repo = ""

# 获取上下文长度
if "max_sequence_length" in hparams:
    ctx_length = hparams["max_sequence_length"]
elif "max_position_embeddings" in hparams:
    ctx_length = hparams["max_position_embeddings"]
elif "model_max_length" in hparams:
    ctx_length = hparams["model_max_length"]
else:
    # 打印错误信息，指示找不到 ctx 长度参数
    print("gguf: can not find ctx length parameter.")
    # 退出程序
    sys.exit()
# 将目录模型的名称添加到gguf_writer中
gguf_writer.add_name(dir_model.name)
# 将hf_repo添加到gguf_writer中
gguf_writer.add_source_hf_repo(hf_repo)
# 将数据布局添加到gguf_writer中
gguf_writer.add_tensor_data_layout("Meta AI original pth")
# 将上下文长度添加到gguf_writer中
gguf_writer.add_context_length(ctx_length)
# 将嵌入长度添加到gguf_writer中
gguf_writer.add_embedding_length(hparams["hidden_size"])
# 将块数量添加到gguf_writer中
gguf_writer.add_block_count(block_count)
# 将前馈长度添加到gguf_writer中
gguf_writer.add_feed_forward_length(hparams["intermediate_size"])
# 将绳索维度数量添加到gguf_writer中
gguf_writer.add_rope_dimension_count(hparams["hidden_size"] // hparams["num_attention_heads"])
# 将头数量添加到gguf_writer中
gguf_writer.add_head_count(head_count)
# 将键值头数量添加到gguf_writer中
gguf_writer.add_head_count_kv(head_count_kv)
# 将层归一化rms_eps添加到gguf_writer中
gguf_writer.add_layer_norm_rms_eps(hparams["rms_norm_eps"])

# 如果hparams中包含rope_scaling并且不为None，并且包含factor
if "rope_scaling" in hparams and hparams["rope_scaling"] != None and "factor" in hparams["rope_scaling"]:
    # 如果hparams中包含rope_scaling的type，并且type为linear
    if "type" in hparams["rope_scaling"]:
        if hparams["rope_scaling"]["type"] == "linear":
            # 将线性绳索缩放类型添加到gguf_writer中
            gguf_writer.add_rope_scaling_type(gguf.RopeScalingType.LINEAR)
            # 将绳索缩放因子添加到gguf_writer中
            gguf_writer.add_rope_scaling_factor(hparams["rope_scaling"]["factor"])

# TOKENIZATION

# 打印获取标记器元数据的消息
print("gguf: get tokenizer metadata")

# 定义存储标记、分数和标记类型的列表
tokens: list[bytes] = []
scores: list[float] = []
toktypes: list[int] = []

# 获取标记器模型文件路径
tokenizer_model_file = dir_model / 'tokenizer.model'
# 如果标记器模型文件不存在
if not tokenizer_model_file.is_file():
    # 打印错误消息并退出程序
    print(f'Error: Missing {tokenizer_model_file}', file = sys.stderr)
    sys.exit(1)

# 打印获取sentencepiece标记器词汇、分数和标记类型的消息
print("gguf: get sentencepiece tokenizer vocab, scores and token types")

# 创建SentencePieceProcessor对象
tokenizer = SentencePieceProcessor(str(tokenizer_model_file))
# 获取vocab_size，如果不存在则使用标记器的词汇大小
vocab_size = hparams.get('vocab_size')
if vocab_size is None:
    vocab_size = tokenizer.vocab_size()

# 遍历词汇大小
for i in range(vocab_size):
    text: bytes
    score: float

    # 获取id对应的标记和分数
    piece = tokenizer.id_to_piece(i)
    text = piece.encode("utf-8")
    score = tokenizer.get_score(i)

    # 默认标记类型为1
    toktype = 1  # defualt to normal token type
    # 如果是未知标记，则标记类型为2
    if tokenizer.is_unknown(i):
        toktype = 2
    # 如果是控制标记，则标记类型为3
    if tokenizer.is_control(i):
        toktype = 3

    # 如果是未使用的标记，则标记类型为5
    if tokenizer.is_unused(i):
        toktype = 5
    # 如果当前的标记是一个字节，则将标记类型设置为6
    if tokenizer.is_byte(i):
        toktype = 6

    # 将文本添加到tokens列表中
    tokens.append(text)
    # 将分数添加到scores列表中
    scores.append(score)
    # 将标记类型添加到toktypes列表中
    toktypes.append(toktype)
# 拼接文件路径和文件名，得到添加的标记文件的完整路径
added_tokens_file = dir_model / 'added_tokens.json'
# 如果添加的标记文件存在
if added_tokens_file.is_file():
    # 以只读方式打开添加的标记文件
    with open(added_tokens_file, "r", encoding="utf-8") as f:
        # 从文件中加载 JSON 数据
        addtokens_json = json.load(f)

        # 打印信息
        print("gguf: get added tokens")

        # 遍历添加的标记 JSON 数据
        for key in addtokens_json:
            # 将键编码为 UTF-8 格式并添加到 tokens 列表中
            tokens.append( key.encode("utf-8") )
            # 添加一个极小的分数到 scores 列表中
            scores.append(-1000.0)
            # 添加用户定义的标记类型到 toktypes 列表中
            toktypes.append(4) # user-defined token type

# 向 gguf_writer 添加标记器模型
gguf_writer.add_tokenizer_model("llama")
# 向 gguf_writer 添加标记列表
gguf_writer.add_token_list(tokens)
# 向 gguf_writer 添加标记分数
gguf_writer.add_token_scores(scores)
# 向 gguf_writer 添加标记类型
gguf_writer.add_token_types(toktypes)

# 创建特殊词汇对象
special_vocab = gguf.SpecialVocab(dir_model, n_vocab = len(tokens))
# 将特殊词汇添加到 gguf_writer
special_vocab.add_to_gguf(gguf_writer)

# 获取张量名称映射
tensor_map = gguf.get_tensor_name_map(ARCH,block_count)

# 打印信息
print("gguf: get tensor metadata")

# 如果模型部分数量为 0
if num_parts == 0:
    # 创建一个迭代器，包含一个元素为 "pytorch_model.bin" 的元组
    part_names = iter(("pytorch_model.bin",))
# 否则
else:
    # 创建一个生成器，生成模型部分的文件名
    part_names = (
        f"pytorch_model-{n:05}-of-{num_parts:05}.bin" for n in range(1, num_parts + 1)
    )

# 遍历模型部分的文件名
for part_name in part_names:
    # 如果设置了仅词汇标记的参数
    if args.vocab_only:
        # 跳出循环
        break
    # 打印信息
    print("gguf: loading model part '" + part_name + "'")
    # 加载模型部分到内存中
    model_part = torch.load(f"{dir_model}/{part_name}", map_location="cpu")

    # 临时变量赋值为模型部分
    tmp=model_part
    # 遍历每个块
    for i in range(block_count):
        # 如果模型部分包含指定的自注意力权重
        if f"model.layers.{i}.self_attn.W_pack.weight" in model_part:
            # 打印信息
            print(f"Unpacking and permuting layer {i}")
            # 对权重进行逆序排列和重排列
            tmp[f"model.layers.{i}.self_attn.q_proj.weight"]=reverse_hf_permute_part(model_part[f"model.layers.{i}.self_attn.W_pack.weight"],0,head_count,head_count_kv)
            tmp[f"model.layers.{i}.self_attn.k_proj.weight"]=reverse_hf_permute_part(model_part[f"model.layers.{i}.self_attn.W_pack.weight"],1,head_count,head_count_kv)
            tmp[f"model.layers.{i}.self_attn.v_proj.weight"]=reverse_hf_part(model_part[f"model.layers.{i}.self_attn.W_pack.weight"],2)
            # 删除权重
            del tmp[f"model.layers.{i}.self_attn.W_pack.weight"]
    # 遍历模型部分的所有键（即张量名）
    for name in model_part.keys():
        # 获取当前张量的数据
        data = model_part[name]
        # 如果张量名以".rotary_emb.inv_freq"结尾，则跳过当前循环
        if name.endswith(".rotary_emb.inv_freq"):
            continue

        # 保存当前数据的数据类型
        old_dtype = data.dtype

        # 将不支持的数据类型转换为float32
        if data.dtype != torch.float16 and data.dtype != torch.float32:
            data = data.to(torch.float32)

        # 压缩数据，去除维度为1的维度，并转换为numpy数组
        data = data.squeeze().numpy()

        # 根据映射表映射张量名
        new_name = tensor_map.get_name(name, try_suffixes = (".weight", ".bias"))
        # 如果无法映射，则打印错误信息并退出程序
        if new_name is None:
            print("Can not map tensor '" + name + "'")
            sys.exit()

        # 获取数据的维度和数据类型
        n_dims = len(data.shape)
        data_dtype = data.dtype

        # 如果ftype为0且数据类型为np.float16，则将数据类型转换为np.float32
        if ftype == 0 and data_dtype == np.float16:
            data = data.astype(np.float32)

        # 如果ftype为1且数据类型为np.float16且维度为1，则将数据类型转换为np.float32
        if ftype == 1 and data_dtype == np.float16 and n_dims == 1:
            data = data.astype(np.float32)

        # 如果ftype为1且数据类型为np.float32且张量名以".weight"结尾且维度为2，则将数据类型转换为np.float16
        if ftype == 1 and data_dtype == np.float32 and name.endswith(".weight") and n_dims == 2:
            data = data.astype(np.float16)

        # 打印张量名、映射后的张量名、数据维度、旧数据类型和新数据类型
        print(name + " -> " +  new_name + ", n_dims = " + str(n_dims) + ", " + str(old_dtype) + " --> " + str(data.dtype))
        # 将张量名和数据添加到gguf_writer中
        gguf_writer.add_tensor(new_name, data)
# 打印提示信息，表示正在写入头部数据
print("gguf: write header")
# 调用 gguf_writer 对象的方法，将头部数据写入文件
gguf_writer.write_header_to_file()
# 打印提示信息，表示正在写入元数据
print("gguf: write metadata")
# 调用 gguf_writer 对象的方法，将键值对数据写入文件
gguf_writer.write_kv_data_to_file()
# 如果不仅仅是词汇表，则执行下面的代码块
if not args.vocab_only:
    # 打印提示信息，表示正在写入张量数据
    print("gguf: write tensors")
    # 调用 gguf_writer 对象的方法，将张量数据写入文件
    gguf_writer.write_tensors_to_file()
# 关闭 gguf_writer 对象
gguf_writer.close()
# 打印提示信息，表示模型成功导出到指定文件
print(f"gguf: model successfully exported to '{fname_out}'")
# 打印空行
print("")
```