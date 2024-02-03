# `whisper.cpp\models\convert-h5-to-ggml.py`

```cpp
# 将 Hugging Face 微调的模型转换为 ggml 格式
#
# 用法：
#
#   git clone https://github.com/openai/whisper
#   git clone https://github.com/ggerganov/whisper.cpp
#   git clone https://huggingface.co/openai/whisper-medium
#
#   python3 ./whisper.cpp/models/convert-h5-to-ggml.py ./whisper-medium/ ./whisper .
#
# 该脚本类似于 "convert-pt-to-ggml.py"
#
# 更多信息请查看：
#
#   https://github.com/ggerganov/whisper.cpp/issues/157
#

import io
import os
import sys
import struct
import json
import code
import torch
import numpy as np
from pathlib import Path

from transformers import WhisperForConditionalGeneration

# 定义转换映射表，用于将模型参数名称转换为 ggml 格式
conv_map = {
        'self_attn.k_proj'              : 'attn.key',
        'self_attn.q_proj'              : 'attn.query',
        'self_attn.v_proj'              : 'attn.value',
        'self_attn.out_proj'            : 'attn.out',
        'self_attn_layer_norm'          : 'attn_ln',
        'encoder_attn.q_proj'           : 'cross_attn.query',
        'encoder_attn.v_proj'           : 'cross_attn.value',
        'encoder_attn.out_proj'         : 'cross_attn.out',
        'encoder_attn_layer_norm'       : 'cross_attn_ln',
        'fc1'                           : 'mlp.0',
        'fc2'                           : 'mlp.2',
        'final_layer_norm'              : 'mlp_ln',
        'encoder.layer_norm.bias'       : 'encoder.ln_post.bias',
        'encoder.layer_norm.weight'     : 'encoder.ln_post.weight',
        'encoder.embed_positions.weight': 'encoder.positional_embedding',
        'decoder.layer_norm.bias'       : 'decoder.ln.bias',
        'decoder.layer_norm.weight'     : 'decoder.ln.weight',
        'decoder.embed_positions.weight': 'decoder.positional_embedding',
        'decoder.embed_tokens.weight'   : 'decoder.token_embedding.weight',
        'proj_out.weight'               : 'decoder.proj.weight',
        }

# 参考：https://github.com/openai/gpt-2/blob/master/src/encoder.py
def bytes_to_unicode():
    #
    # 返回一个包含 utf-8 字节和相应的 Unicode 字符串列表
    # 可逆的 BPE 编码适用于 Unicode 字符串
    # 这意味着如果要避免 UNKs，需要在词汇表中包含大量的 Unicode 字符
    # 当处理大约 100 亿个标记的数据集时，需要大约 5K 个字符以获得良好的覆盖率
    # 这占据了正常情况下 32K bpe 词汇表的相当大比例
    # 为了避免这种情况，我们需要在 utf-8 字节和 Unicode 字符串之间建立查找表
    # 并避免将 bpe 编码映射到空格/控制字符，以免出现错误
    """
    # 初始化 utf-8 字节列表，包含可打印字符和一些特殊字符
    bs = list(range(ord("!"), ord("~")+1))+list(range(ord("¡"), ord("¬")+1))+list(range(ord("®"), ord("ÿ")+1))
    # 复制 utf-8 字节列表，用于存储对应的 Unicode 字符
    cs = bs[:]
    # 初始化计数器
    n = 0
    # 遍历 0 到 255 的所有字节
    for b in range(2**8):
        # 如果字节不在 utf-8 字节列表中
        if b not in bs:
            # 将字节添加到 utf-8 字节列表
            bs.append(b)
            # 将对应的 Unicode 字符添加到 cs 列表
            cs.append(2**8+n)
            # 更新计数器
            n += 1
    # 将 cs 列表中的数字转换为对应的 Unicode 字符
    cs = [chr(n) for n in cs]
    # 返回 utf-8 字节和 Unicode 字符之间的映射关系字典
    return dict(zip(bs, cs))
# 检查命令行参数是否足够，如果不足则打印用法信息并退出程序
if len(sys.argv) < 4:
    print("Usage: convert-h5-to-ggml.py dir_model path-to-whisper-repo dir-output [use-f32]\n")
    sys.exit(1)

# 从命令行参数中获取目录路径
dir_model   = Path(sys.argv[1])
dir_whisper = Path(sys.argv[2])
dir_out     = Path(sys.argv[3])

# 从文件中加载编码器、添加的编码器、超参数和模型
encoder = json.load((dir_model / "vocab.json").open("r", encoding="utf8"))
encoder_added = json.load((dir_model / "added_tokens.json").open( "r", encoding="utf8"))
hparams = json.load((dir_model / "config.json").open("r", encoding="utf8") )
model = WhisperForConditionalGeneration.from_pretrained(dir_model)

# 从文件中加载 mel_filters 数据
n_mels = hparams["num_mel_bins"]
with np.load(os.path.join(dir_whisper, "whisper/assets", "mel_filters.npz")) as f:
    filters = torch.from_numpy(f[f"mel_{n_mels}"])

# 设置 tokenizer 目录路径和输出文件名
dir_tokenizer = dir_model
fname_out = dir_out / "ggml-model.bin"

# 从文件中加载 tokens 数据
tokens = json.load(open(dir_tokenizer / "vocab.json", "r", encoding="utf8"))

# 根据命令行参数决定是否使用 16 位或 32 位浮点数
use_f16 = True
if len(sys.argv) > 4:
    use_f16 = False
    fname_out = dir_out / "ggml-model-f32.bin"

# 打开输出文件，写入 GGML 文件头信息
fout = open(fname_out, "wb")
fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("i", hparams["max_source_positions"]))
fout.write(struct.pack("i", hparams["d_model"]))
fout.write(struct.pack("i", hparams["encoder_attention_heads"]))
fout.write(struct.pack("i", hparams["encoder_layers"]))
fout.write(struct.pack("i", hparams["max_length"]))
fout.write(struct.pack("i", hparams["d_model"]))
fout.write(struct.pack("i", hparams["decoder_attention_heads"]))
fout.write(struct.pack("i", hparams["decoder_layers"]))
fout.write(struct.pack("i", hparams["num_mel_bins"]))
fout.write(struct.pack("i", use_f16))

# 写入 mel_filters 数据到输出文件
fout.write(struct.pack("i", filters.shape[0]))
fout.write(struct.pack("i", filters.shape[1]))
for i in range(filters.shape[0]):
    for j in range(filters.shape[1]):
        fout.write(struct.pack("f", filters[i][j]))

# 将字符串编码为 Unicode 字节
byte_encoder = bytes_to_unicode()
# 创建一个字典，用于将字节编码器的键值对进行反转
byte_decoder = {v:k for k, v in byte_encoder.items()}

# 将tokens的长度打包为整数并写入文件
fout.write(struct.pack("i", len(tokens)))

# 对tokens按值进行排序，并遍历每个键值对
for key in tokens:
    # 将每个键转换为文本形式的字节数组
    text = bytearray([byte_decoder[c] for c in key[0]])
    # 将文本长度打包为整数并写入文件
    fout.write(struct.pack("i", len(text)))
    # 将文本数据写入文件
    fout.write(text)

# 获取模型的状态字典
list_vars = model.state_dict()
# 遍历状态字典的键
for name in list_vars.keys():
    # 如果键为"proj_out.weight"，则跳过
    if name == "proj_out.weight":
        print('Skipping', name)
        continue

    # 复制键名
    src = name

    # 如果键名不是"proj_out.weight"，则根据"."拆分键名
    nn = name
    if name != "proj_out.weight":
        nn = nn.split(".")[1:]
    else:
        nn = nn.split(".")

    # 根据特定规则对键名进行转换
    if nn[1] == "layers":
        # 将"layers"替换为"blocks"
        nn[1] = "blocks"
        # 根据特定规则映射键名
        if ".".join(nn[3:-1]) == "encoder_attn.k_proj":
            mapped = "attn.key" if nn[0] == "encoder" else "cross_attn.key"
        else:
            mapped = conv_map[".".join(nn[3:-1])]
        name = ".".join(nn[:3] + [mapped] + nn[-1])
    else:
        name = ".".join(nn)
        # 如果键名在conv_map中，则替换为对应值，否则保持不变
        name = conv_map[name] if name in conv_map else name

    print(src, ' -> ', name)
    # 获取键对应的值，并转换为numpy数组
    data = list_vars[src].squeeze().numpy()
    # 将数据类型转换为float16
    data = data.astype(np.float16)

    # 如果键名为"encoder.conv1.bias"或"encoder.conv2.bias"，则将数据形状从[n]改为[n, 1]
    if name in ["encoder.conv1.bias", "encoder.conv2.bias"]:
        data = data.reshape(data.shape[0], 1)
        print("  Reshaped variable: " , name , " to shape: ", data.shape)

    # 获取数据维度和形状
    n_dims = len(data.shape)
    print(name, n_dims, data.shape)

    # 设置数据类型为float32
    ftype = 1
    # 如果使用 f16 数据类型
    if use_f16:
        # 如果维度小于2或者是指定的几个特殊名称的参数，则转换为 float32 类型
        if n_dims < 2 or \
                name == "encoder.conv1.bias"   or \
                name == "encoder.conv2.bias"   or \
                name == "encoder.positional_embedding" or \
                name == "decoder.positional_embedding":
            print("  Converting to float32")
            # 将数据类型转换为 float32
            data = data.astype(np.float32)
            # 设置数据类型标志为0
            ftype = 0
    else:
        # 将数据类型转换为 float32
        data = data.astype(np.float32)
        # 设置数据类型标志为0

    # 将参数名称编码为 utf-8 格式
    str_ = name.encode('utf-8')
    # 写入头部信息到输出文件
    fout.write(struct.pack("iii", n_dims, len(str_), ftype))
    # 遍历维度信息，写入到输出文件
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    # 写入参数名称到输出文件
    fout.write(str_)

    # 将数据写入到输出文件
    data.tofile(fout)
# 关闭输出文件
fout.close()

# 打印完成消息以及输出文件名
print("Done. Output file: " , fname_out)
print("")
```