# `PowerInfer\convert-baichuan-hf-to-gguf.py`

```
#!/usr/bin/env python3
# 指定脚本解释器为 Python 3

# HF baichuan --> gguf conversion
# 用于将 HF baichuan 转换为 gguf

from __future__ import annotations
# 导入未来的注解特性

import argparse
import json
import os
import struct
import sys
from pathlib import Path
from typing import TYPE_CHECKING, Any
import itertools
import numpy as np
import torch
from sentencepiece import SentencePieceProcessor  # type: ignore[import]
# 导入所需的模块和库

if 'NO_LOCAL_GGUF' not in os.environ:
    # 如果环境变量中没有设置 NO_LOCAL_GGUF
    sys.path.insert(1, str(Path(__file__).parent / 'gguf-py'))
    # 将 gguf-py 目录添加到系统路径中
import gguf
# 导入 gguf 模块
# 如果类型检查开启，从 typing 模块中导入 TypeAlias 类型别名
if TYPE_CHECKING:
    from typing import TypeAlias

# 定义类型别名 NDArray 为 'np.ndarray[Any, Any]'
NDArray: TypeAlias = 'np.ndarray[Any, Any]'

# 将 HF 排列逆转回原始的 pth 排列
def reverse_hf_permute(weights: NDArray, n_head: int, n_kv_head: int | None = None) -> NDArray:
    # 如果 n_kv_head 不为 None 且 n_head 不等于 n_kv_head，则将 n_head 除以 n_kv_head
    if n_kv_head is not None and n_head != n_kv_head:
        n_head //= n_kv_head

    # 重塑权重矩阵，交换轴，再次重塑，返回结果
    return (weights.reshape(n_head, 2, weights.shape[0] // n_head // 2, *weights.shape[1:])
            .swapaxes(1, 2)
            .reshape(weights.shape))

# 将 HF 排列的一部分逆转回原始的 pth 排列
def reverse_hf_permute_part(weights: NDArray, n_part: int, n_head: int, n_head_kv: int| None = None) -> NDArray:
    # 计算权重矩阵的形状
    r = weights.shape[0] // 3
# 返回反向 HF 部分权重的排列
def reverse_hf_permute(weights: NDArray, n_head: int, n_head_kv: int) -> NDArray:
    # 计算每个部分的大小
    r = weights.shape[0] // 3
    # 返回反向 HF 部分的权重
    return (reverse_hf_part(weights[r * n_part : r * n_part + r, ...], n_head, n_head_kv))

# 返回反向 HF 部分的权重
def reverse_hf_part(weights: NDArray, n_part: int) -> NDArray:
    # 计算每个部分的大小
    r = weights.shape[0] // 3
    # 返回反向 HF 部分的权重
    return weights[r * n_part : r * n_part + r, ...]

# 计算模型部分的数量
def count_model_parts(dir_model: str) -> int:
    num_parts = 0

    # 遍历模型目录下的文件
    for filename in os.listdir(dir_model):
        # 如果文件名以 "pytorch_model-" 开头，则部分数量加一
        if filename.startswith("pytorch_model-"):
            num_parts += 1

    # 如果找到模型部分，则打印部分数量
    if num_parts > 0:
        print("gguf: found " + str(num_parts) + " model parts")

    # 返回部分数量
    return num_parts
# 解析命令行参数，返回命名空间对象
def parse_args() -> argparse.Namespace:
    # 创建参数解析器，设置描述信息
    parser = argparse.ArgumentParser(description="Convert a HuggingFace LLaMA model to a GGML compatible file")
    # 添加参数：--vocab-only，用于提取词汇表
    parser.add_argument(
        "--vocab-only", action="store_true",
        help="extract only the vocab",
    )
    # 添加参数：--outfile，用于指定输出文件路径，默认基于输入文件
    parser.add_argument(
        "--outfile", type=Path,
        help="path to write to; default: based on input",
    )
    # 添加参数：model，用于指定模型文件夹或模型文件本身（*.bin）
    parser.add_argument(
        "model", type=Path,
        help="directory containing model file, or model file itself (*.bin)",
    )
    # 添加参数：ftype，用于指定输出格式，0 为 float32，1 为 float16
    parser.add_argument(
        "ftype", type=int, choices=[0, 1], default=1, nargs='?',
        help="output format - use 0 for float32, 1 for float16",
    )
    # 添加参数：--bigendian，用于指定模型在大端机器上执行
    parser.add_argument("--bigendian",   action="store_true",    help="model is executed on big endian machine")
    # 解析并返回命令行参数
    return parser.parse_args()
# 解析命令行参数
args = parse_args()

# 获取模型目录和文件类型
dir_model = args.model
ftype = args.ftype

# 如果模型目录不存在，则打印错误信息并退出程序
if not dir_model.is_dir():
    print(f'Error: {args.model} is not a directory', file = sys.stderr)
    sys.exit(1)

# 设置默认的字节序为小端
endianess = gguf.GGUFEndian.LITTLE

# 如果命令行参数指定了大端字节序，则将字节序设置为大端
if args.bigendian:
    endianess = gguf.GGUFEndian.BIG

# 根据字节序设置相应的字符串
endianess_str = "Big Endian" if args.bigendian else "Little Endian"
print(f"gguf: Conversion Endianess {endianess}")

# 可能的张量数据类型
#   ftype == 0 -> float32
#   ftype == 1 -> float16

# 将文件类型映射为字符串
ftype_str = ["f32", "f16"]
# 如果命令行参数中指定了输出文件名，则使用指定的文件名，否则使用默认文件名
if args.outfile is not None:
    fname_out = args.outfile
else:
    # 默认情况下在与模型相同的目录中输出文件
    fname_out = dir_model / f'ggml-model-{ftype_str[ftype]}.gguf'

# 打印加载模型的信息
print("gguf: loading model "+dir_model.name)

# 以只读方式打开模型目录下的config.json文件，并读取其中的内容
with open(dir_model / "config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

# 打印config.json文件中的内容
print("hello print: ",hparams["architectures"][0])

# 如果模型的架构不是BaichuanForCausalLM或BaiChuanForCausalLM，则打印不支持的模型架构，并退出程序
if hparams["architectures"][0] != "BaichuanForCausalLM" and hparams["architectures"][0] != "BaiChuanForCausalLM":
    print("Model architecture not supported: " + hparams["architectures"][0])
    sys.exit()

# 获取模型部件的数量
num_parts = count_model_parts(dir_model)
# 打印模型部件的数量
print(f"num_parts:{num_parts}\n")
# 设置模型架构为BAICHUAN
ARCH=gguf.MODEL_ARCH.BAICHUAN
# 创建一个GGUFWriter对象，指定输出文件名、模型架构名称和字节序
gguf_writer = gguf.GGUFWriter(fname_out, gguf.MODEL_ARCH_NAMES[ARCH], endianess=endianess)

# 打印提示信息
print("gguf: get model metadata")

# 从hparams中获取隐藏层的数量
block_count = hparams["num_hidden_layers"]
# 从hparams中获取注意力头的数量
head_count = hparams["num_attention_heads"]

# 如果hparams中包含"num_key_value_heads"，则从hparams中获取key-value头的数量，否则使用注意力头的数量
if "num_key_value_heads" in hparams:
    head_count_kv = hparams["num_key_value_heads"]
else:
    head_count_kv = head_count

# 如果hparams中包含"_name_or_path"，则从hparams中获取模型的repo名称，否则使用空字符串
if "_name_or_path" in hparams:
    hf_repo = hparams["_name_or_path"]
else:
    hf_repo = ""

# 如果hparams中包含"max_sequence_length"，则从hparams中获取最大序列长度
ctx_length = hparams["max_sequence_length"]
# 如果参数中包含"max_position_embeddings"，则将上下文长度设置为对应的值
elif "max_position_embeddings" in hparams:
    ctx_length = hparams["max_position_embeddings"]
# 如果参数中包含"model_max_length"，则将上下文长度设置为对应的值
elif "model_max_length" in hparams:
    ctx_length = hparams["model_max_length"]
# 如果以上两个条件都不满足，则打印错误信息并退出程序
else:
    print("gguf: can not find ctx length parameter.")
    sys.exit()

# 将模型名称添加到gguf_writer中
gguf_writer.add_name(dir_model.name)
# 将源代码的hf_repo添加到gguf_writer中
gguf_writer.add_source_hf_repo(hf_repo)
# 将数据布局信息添加到gguf_writer中
gguf_writer.add_tensor_data_layout("Meta AI original pth")
# 将上下文长度添加到gguf_writer中
gguf_writer.add_context_length(ctx_length)
# 将嵌入长度添加到gguf_writer中
gguf_writer.add_embedding_length(hparams["hidden_size"])
# 将块数量添加到gguf_writer中
gguf_writer.add_block_count(block_count)
# 将前馈网络长度添加到gguf_writer中
gguf_writer.add_feed_forward_length(hparams["intermediate_size"])
# 将绳索维度数量添加到gguf_writer中
gguf_writer.add_rope_dimension_count(hparams["hidden_size"] // hparams["num_attention_heads"])
# 将头部数量添加到gguf_writer中
gguf_writer.add_head_count(head_count)
# 将键值头部数量添加到gguf_writer中
gguf_writer.add_head_count_kv(head_count_kv)
# 调用gguf_writer对象的add_layer_norm_rms_eps方法，传入hparams["rms_norm_eps"]作为参数
gguf_writer.add_layer_norm_rms_eps(hparams["rms_norm_eps"])

# 检查hparams中是否包含rope_scaling，并且不为None，同时包含factor
if "rope_scaling" in hparams and hparams["rope_scaling"] != None and "factor" in hparams["rope_scaling"]:
    # 检查hparams["rope_scaling"]中是否包含type
    if "type" in hparams["rope_scaling"]:
        # 如果hparams["rope_scaling"]["type"]为"linear"，则调用gguf_writer对象的add_rope_scaling_type方法，传入gguf.RopeScalingType.LINEAR作为参数
        if hparams["rope_scaling"]["type"] == "linear":
            gguf_writer.add_rope_scaling_type(gguf.RopeScalingType.LINEAR)
            # 调用gguf_writer对象的add_rope_scaling_factor方法，传入hparams["rope_scaling"]["factor"]作为参数
            gguf_writer.add_rope_scaling_factor(hparams["rope_scaling"]["factor"])

# 打印提示信息
print("gguf: get tokenizer metadata")

# 初始化tokens、scores、toktypes变量
tokens: list[bytes] = []
scores: list[float] = []
toktypes: list[int] = []

# 设置tokenizer_model_file为dir_model目录下的'tokenizer.model'文件
tokenizer_model_file = dir_model / 'tokenizer.model'
# 如果tokenizer_model_file文件不存在，则打印错误信息
if not tokenizer_model_file.is_file():
    print(f'Error: Missing {tokenizer_model_file}', file = sys.stderr)
# 退出程序并返回状态码1
sys.exit(1)

# 打印提示信息
print("gguf: get sentencepiece tokenizer vocab, scores and token types")

# 使用SentencePieceProcessor加载tokenizer_model_file
tokenizer = SentencePieceProcessor(str(tokenizer_model_file))

# 获取vocab_size，如果没有指定则使用tokenizer的vocab_size
vocab_size = hparams.get('vocab_size')
if vocab_size is None:
    vocab_size = tokenizer.vocab_size()

# 遍历vocab_size，获取每个token的文本和分数
for i in range(vocab_size):
    text: bytes
    score: float

    # 获取token对应的文本和编码
    piece = tokenizer.id_to_piece(i)
    text = piece.encode("utf-8")
    # 获取token的分数
    score = tokenizer.get_score(i)

    # 默认token类型为1
    toktype = 1  # defualt to normal token type
    # 如果token是未知的，则更新token类型
    if tokenizer.is_unknown(i):
# 设置默认的 token 类型为 2
toktype = 2
# 如果当前 token 是控制字符，则将 token 类型设置为 3
if tokenizer.is_control(i):
    toktype = 3

# 如果 token 类型为 4，则表示用户自定义的 token，来自 added_tokens.json 文件

# 如果当前 token 是未使用的，则将 token 类型设置为 5
if tokenizer.is_unused(i):
    toktype = 5
# 如果当前 token 是字节，则将 token 类型设置为 6
if tokenizer.is_byte(i):
    toktype = 6

# 将 token 添加到 tokens 列表中
tokens.append(text)
# 将分数添加到 scores 列表中
scores.append(score)
# 将 token 类型添加到 toktypes 列表中
toktypes.append(toktype)

# 检查是否存在 added_tokens.json 文件
added_tokens_file = dir_model / 'added_tokens.json'
if added_tokens_file.is_file():
    # 如果文件存在，则打开并加载其中的 JSON 数据
    with open(added_tokens_file, "r", encoding="utf-8") as f:
        addtokens_json = json.load(f)
# 打印提示信息
print("gguf: get added tokens")

# 遍历 addtokens_json 中的键
for key in addtokens_json:
    # 将键编码为 UTF-8 格式并添加到 tokens 列表中
    tokens.append( key.encode("utf-8") )
    # 添加一个初始分数为 -1000.0 的分数到 scores 列表中
    scores.append(-1000.0)
    # 添加一个用户定义的标记类型到 toktypes 列表中
    toktypes.append(4) # user-defined token type

# 向 gguf_writer 中添加 tokenizer 模型
gguf_writer.add_tokenizer_model("llama")
# 向 gguf_writer 中添加 token 列表
gguf_writer.add_token_list(tokens)
# 向 gguf_writer 中添加 token 分数列表
gguf_writer.add_token_scores(scores)
# 向 gguf_writer 中添加 token 类型列表
gguf_writer.add_token_types(toktypes)

# 创建一个特殊的词汇对象
special_vocab = gguf.SpecialVocab(dir_model, n_vocab = len(tokens))
# 将特殊词汇对象添加到 gguf_writer 中
special_vocab.add_to_gguf(gguf_writer)

# 获取张量名称映射
tensor_map = gguf.get_tensor_name_map(ARCH,block_count)
# 打印信息，表示获取张量元数据
print("gguf: get tensor metadata")

# 如果部分数量为0，则设置部分名称为"pytorch_model.bin"，否则设置部分名称为一系列文件名
if num_parts == 0:
    part_names = iter(("pytorch_model.bin",))
else:
    part_names = (
        f"pytorch_model-{n:05}-of-{num_parts:05}.bin" for n in range(1, num_parts + 1)
    )

# 遍历部分名称
for part_name in part_names:
    # 如果参数中指定了仅加载词汇表，则跳出循环
    if args.vocab_only:
        break
    # 打印信息，表示正在加载模型部分
    print("gguf: loading model part '" + part_name + "'")
    # 加载模型部分数据
    model_part = torch.load(f"{dir_model}/{part_name}", map_location="cpu")

    # 临时变量，用于存储模型部分数据
    tmp=model_part
    # 遍历块数量
    for i in range(block_count):
        # 如果指定的模型层存在于模型部分中
        if f"model.layers.{i}.self_attn.W_pack.weight" in model_part:
# 打印信息，表示正在解压和排列第i层的数据
print(f"Unpacking and permuting layer {i}")

# 对应位置的权重矩阵进行逆序排列
tmp[f"model.layers.{i}.self_attn.q_proj.weight"]=reverse_hf_permute_part(model_part[f"model.layers.{i}.self_attn.W_pack.weight"],0,head_count,head_count)

# 对应位置的权重矩阵进行逆序排列
tmp[f"model.layers.{i}.self_attn.k_proj.weight"]=reverse_hf_permute_part(model_part[f"model.layers.{i}.self_attn.W_pack.weight"],1,head_count,head_count_kv)

# 对应位置的权重矩阵进行逆序排列
tmp[f"model.layers.{i}.self_attn.v_proj.weight"]=reverse_hf_part(model_part[f"model.layers.{i}.self_attn.W_pack.weight"],2)

# 删除指定的权重矩阵
del tmp[f"model.layers.{i}.self_attn.W_pack.weight"]

# 遍历模型部分的所有键
for name in model_part.keys():
    # 获取对应键的数据
    data = model_part[name]
    
    # 如果键以".rotary_emb.inv_freq"结尾，则跳过
    if name.endswith(".rotary_emb.inv_freq"):
        continue

    # 保存原始数据类型
    old_dtype = data.dtype

    # 将不支持的数据类型转换为float32
    if data.dtype != torch.float16 and data.dtype != torch.float32:
        data = data.to(torch.float32)

    # 压缩数据并转换为numpy数组
    data = data.squeeze().numpy()
# 映射张量名称
# 获取新的张量名称，如果没有找到，则尝试使用指定的后缀（".weight", ".bias"）
new_name = tensor_map.get_name(name, try_suffixes = (".weight", ".bias"))
if new_name is None:
    print("Can not map tensor '" + name + "'")
    sys.exit()

# 获取数据的维度和数据类型
n_dims = len(data.shape)
data_dtype = data.dtype

# 如果需要 f32 类型的数据，将所有 float16 类型的数据转换为 float32
if ftype == 0 and data_dtype == np.float16:
    data = data.astype(np.float32)

# TODO: 为什么不能直接使用 float16 类型的数据？应该没有理由将 float16 存储为 float32
if ftype == 1 and data_dtype == np.float16 and n_dims == 1:
    data = data.astype(np.float32)

# 如果需要 f16 类型的数据，将所有 2 维权重张量的 float32 数据转换为 float16
if ftype == 1 and data_dtype == np.float32 and name.endswith(".weight") and n_dims == 2:
    data = data.astype(np.float16)
# 打印变量 name、new_name、n_dims、old_dtype 和 data.dtype 的值
print(name + " -> " +  new_name + ", n_dims = " + str(n_dims) + ", " + str(old_dtype) + " --> " + str(data.dtype))
# 将数据添加到 gguf_writer 对象中
gguf_writer.add_tensor(new_name, data)

# 打印信息
print("gguf: write header")
# 将头部信息写入文件
gguf_writer.write_header_to_file()
# 打印信息
print("gguf: write metadata")
# 将元数据写入文件
gguf_writer.write_kv_data_to_file()
# 如果不仅仅是词汇表，则执行以下操作
if not args.vocab_only:
    # 打印信息
    print("gguf: write tensors")
    # 将张量写入文件
    gguf_writer.write_tensors_to_file()

# 关闭 gguf_writer 对象
gguf_writer.close()

# 打印成功导出模型的信息
print(f"gguf: model successfully exported to '{fname_out}'")
# 打印空行
print("")
```