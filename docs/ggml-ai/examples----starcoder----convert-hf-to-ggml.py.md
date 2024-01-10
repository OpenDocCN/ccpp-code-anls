# `ggml\examples\starcoder\convert-hf-to-ggml.py`

```
# Convert HF models to ggml format
#

import sys  # 导入 sys 模块
import struct  # 导入 struct 模块
import json  # 导入 json 模块
import torch  # 导入 torch 模块
import numpy as np  # 导入 numpy 模块，并使用别名 np
import re  # 导入 re 模块
import os  # 导入 os 模块
import argparse  # 导入 argparse 模块

from transformers import AutoModelForCausalLM  # 从 transformers 模块中导入 AutoModelForCausalLM 类
from transformers import AutoTokenizer, AutoModelForCausalLM, AutoConfig, BloomForCausalLM  # 从 transformers 模块中导入其他类

# ref: https://github.com/openai/gpt-2/blob/master/src/encoder.py
def bytes_to_unicode():
    """
    Returns list of utf-8 byte and a corresponding list of unicode strings.
    The reversible bpe codes work on unicode strings.
    This means you need a large # of unicode characters in your vocab if you want to avoid UNKs.
    When you're at something like a 10B token dataset you end up needing around 5K for decent coverage.
    This is a signficant percentage of your normal, say, 32K bpe vocab.
    To avoid that, we want lookup tables between utf-8 bytes and unicode strings.
    And avoids mapping to whitespace/control characters the bpe code barfs on.
    """
    bs = list(range(ord("!"), ord("~")+1))+list(range(ord("¡"), ord("¬")+1))+list(range(ord("®"), ord("ÿ")+1))  # 创建 utf-8 字节列表
    cs = bs[:]  # 复制 utf-8 字节列表
    n = 0  # 初始化 n 为 0
    for b in range(2**8):  # 遍历 0 到 255 的范围
        if b not in bs:  # 如果 b 不在 utf-8 字节列表中
            bs.append(b)  # 将 b 添加到 utf-8 字节列表中
            cs.append(2**8+n)  # 将 2**8+n 添加到 cs 列表中
            n += 1  # n 自增 1
    cs = [chr(n) for n in cs]  # 将 cs 列表中的每个元素转换为对应的 Unicode 字符
    return dict(zip(bs, cs))  # 返回 utf-8 字节和对应的 Unicode 字符的字典

parser = argparse.ArgumentParser(description='Convert starcoder HF model to GGML')  # 创建参数解析器
parser.add_argument('model_name_or_path', type=str, help='Name of model on HF hub, or local model folder')  # 添加位置参数 model_name_or_path
parser.add_argument('--outfile', type=str, default='ggml-model.bin', help='Path of GGML file to write.')  # 添加可选参数 outfile，默认值为 'ggml-model.bin'
parser.add_argument('--use_f32', action="store_true", help='Save GGML file in fp32')  # 添加可选参数 use_f32，表示是否使用 32 位浮点数

args = parser.parse_args()  # 解析命令行参数

# use 16-bit or 32-bit floats
use_f16 = not args.use_f32  # 根据参数 use_f32 的值确定是否使用 16 位浮点数

fname_out = args.outfile  # 获取输出文件名
fname_dir = os.path.dirname(fname_out)  # 获取输出文件所在目录
if fname_dir:  # 如果输出文件所在目录不为空
    os.makedirs(fname_dir, exist_ok=True)  # 创建输出文件所在目录，如果目录已存在则不报错

print("Loading model: ", args.model_name_or_path)  # 打印加载模型的信息
tokenizer = AutoTokenizer.from_pretrained(args.model_name_or_path)  # 根据模型名称或路径加载对应的 tokenizer
# 从预训练模型名称或路径中加载自动配置对象，设置信任远程代码为True
config = AutoConfig.from_pretrained(args.model_name_or_path, trust_remote_code=True)
# 将配置对象转换为字典
hparams = config.to_dict()
# 从预训练模型名称或路径中加载自动模型对象，设置相关参数，如torch数据类型、低CPU内存使用等
model = AutoModelForCausalLM.from_pretrained(args.model_name_or_path, config=config, torch_dtype=torch.float16 if use_f16 else torch.float32, low_cpu_mem_usage=True, trust_remote_code=True, offload_state_dict=True)
# 打印模型加载信息
print("Model loaded: ", args.model_name_or_path)

# 获取模型的状态字典
list_vars = model.state_dict()

# 获取tokenizer的词汇表
encoder = tokenizer.vocab
# 将特殊标记（添加的特殊标记）添加到词汇表中
encoder.update(tokenizer.get_added_vocab())
# 打印模型参数字典
print(hparams)

# 打印保存模型的文件路径
print("Saving ggml model to: ", fname_out)
# 以二进制写模式打开文件
fout = open(fname_out, "wb")

# 写入文件头部信息，如魔数、词汇表大小等
fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
vocab_size = hparams["vocab_size"]
fout.write(struct.pack("i", vocab_size))
fout.write(struct.pack("i", hparams["n_positions"]))
fout.write(struct.pack("i", hparams["n_embd"]))
fout.write(struct.pack("i", hparams["n_head"]))
fout.write(struct.pack("i", hparams["n_layer"]))
fout.write(struct.pack("i", use_f16))

# 将编码器转换为字节流
byte_encoder = bytes_to_unicode()
byte_decoder = {v:k for k, v in byte_encoder.items()}

# 写入词汇表大小
fout.write(struct.pack("i", vocab_size))

counter = 0
# 按值排序编码器
for key in sorted(encoder, key=encoder.get):
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)
    counter += 1

# TODO: 重复最后一个标记直到词汇表大小
while counter < vocab_size:
    fout.write(struct.pack("i", len(text)))
    fout.write(text)
    counter += 1
# assert counter == config.vocab_size

# 遍历模型状态字典的键
for name in list_vars.keys():
    # 获取变量数据并转换为numpy数组
    data = list_vars[name].squeeze().numpy()
    # 打印处理的变量信息
    print("Processing variable: " + name + " with shape: ", data.shape)

    # 重命名头部以保持兼容性
    if name == "transformer.ln_f.weight":
        name = "model/ln_f/g"
    elif name == "transformer.ln_f.bias":
        name = "model/ln_f/b"
    elif name == "transformer.wte.weight":
        name = "model/wte"
    # 如果变量名为"transformer.wpe.weight"，则修改为"model/wpe"
    elif name == "transformer.wpe.weight":
        name = "model/wpe"
    # 如果变量名为"lm_head.weight"，则修改为"model/lm_head"
    elif name == "lm_head.weight":
        name = "model/lm_head"
    # 如果变量名匹配"transformer.h.\d+.ln_1.weight"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h\.\d+\.ln_1\.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_1/g"
    # 如果变量名匹配"transformer.h.\d+.ln_1.bias"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h\.\d+\.ln_1\.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_1/b"
    # 如果变量名匹配"transformer.h.\d+.attn.c_attn.weight"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h\.\d+\.attn\.c_attn\.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_attn/w"
    # 如果变量名匹配"transformer.h.\d+.attn.c_attn.bias"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h\.\d+\.attn\.c_attn\.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_attn/b"
    # 如果变量名匹配"transformer.h.\d+.attn.c_proj.weight"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h.\d+.attn.c_proj.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_proj/w"
    # 如果变量名匹配"transformer.h.\d+.attn.c_proj.bias"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h.\d+.attn.c_proj.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_proj/b"
    # 如果变量名匹配"transformer.h.\d+.ln_2.weight"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h.\d+.ln_2.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_2/g"
    # 如果变量名匹配"transformer.h.\d+.ln_2.bias"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h.\d+.ln_2.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_2/b"
    # 如果变量名匹配"transformer.h.\d+.mlp.c_fc.weight"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h.\d+.mlp.c_fc.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_fc/w"
    # 如果变量名匹配"transformer.h.\d+.mlp.c_fc.bias"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h.\d+.mlp.c_fc.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_fc/b"
    # 如果变量名匹配"transformer.h.\d+.mlp.c_proj.weight"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h.\d+.mlp.c_proj.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_proj/w"
    # 如果变量名匹配"transformer.h.\d+.mlp.c_proj.bias"的格式，则修改为对应格式的模型路径
    elif re.match(r"transformer.h.\d+.mlp.c_proj.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_proj/b"
    # 如果变量名不匹配以上任何格式，则打印未识别的变量名
    else:
        print("Unrecognized variable name. %s", name)

    # 我们不需要这些
    # 如果文件名以"attn.masked_bias"或".attn.bias"结尾，则跳过当前变量并打印提示信息
    if name.endswith("attn.masked_bias") or name.endswith(".attn.bias"):
        print("  Skipping variable: " + name)
        continue

    # 获取数据的维度
    n_dims = len(data.shape);

    # 设置默认的数据类型为float32
    ftype = 0;
    # 如果使用float16，并且满足条件，则将数据类型设置为float16
    if use_f16:
        if (name == "model/wte" or name == "model/lm_head" or name[-2:] == "/g" or name[-2:] == "/w") and n_dims == 2:
            print("  Converting to float16")
            data = data.astype(np.float16)
            ftype = 1
        else:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype = 0

    # 如果文件名满足条件，则进行特定操作
    if name[-14:] == "/attn/c_attn/w" or name[-14:] == "/attn/c_attn/b":
        print("  Duplicate K,V heads to use MHA instead of MQA")

        # 计算embed_dim和head_dim
        embed_dim = hparams["n_embd"]
        head_dim = embed_dim // hparams["n_head"]

        # 将数据分割成q, k, v
        q, k ,v = np.split(data, (hparams["n_head"] * head_dim, (hparams["n_head"] + 1) * head_dim), axis=0)
        # 如果k和v的维度为2，则进行复制操作
        if len(k.shape) == 2:
            k = np.tile(k, (hparams["n_head"], 1))
            v = np.tile(v, (hparams["n_head"], 1))
        # 如果k和v的维度为1，则进行复制操作
        elif len(k.shape) == 1:
            k = np.tile(k, (hparams["n_head"]))
            v = np.tile(v, (hparams["n_head"]))
        # 沿着第一个轴连接q, k, v
        data = np.concatenate((q, k, v), axis=0)

    # 将文件名转换为字节流并写入文件
    str = name.encode('utf-8')
    fout.write(struct.pack("iii", n_dims, len(str), ftype))
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    fout.write(str);

    # 将数据写入文件
    data.tofile(fout)
# 关闭输出文件
fout.close()

# 打印完成消息和输出文件名
print("Done. Output file: " + fname_out)
print("")
```