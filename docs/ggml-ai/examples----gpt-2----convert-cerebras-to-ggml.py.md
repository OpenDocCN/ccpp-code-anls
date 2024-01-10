# `ggml\examples\gpt-2\convert-cerebras-to-ggml.py`

```
# Convert Cerebras models to ggml format
#
# ref: https://www.cerebras.net/blog/cerebras-gpt-a-family-of-open-compute-efficient-large-language-models/
#

import sys  # 导入 sys 模块，用于处理命令行参数
import struct  # 导入 struct 模块，用于处理字节流
import json  # 导入 json 模块，用于处理 JSON 格式数据
import torch  # 导入 torch 模块，用于深度学习
import numpy as np  # 导入 numpy 模块，用于科学计算
import re  # 导入 re 模块，用于正则表达式匹配

from transformers import AutoModelForCausalLM  # 从 transformers 模块中导入 AutoModelForCausalLM 类

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
    bs = list(range(ord("!"), ord("~")+1))+list(range(ord("¡"), ord("¬")+1))+list(range(ord("®"), ord("ÿ")+1))
    cs = bs[:]
    n = 0
    for b in range(2**8):
        if b not in bs:
            bs.append(b)
            cs.append(2**8+n)
            n += 1
    cs = [chr(n) for n in cs]
    return dict(zip(bs, cs))  # 返回 utf-8 字节和对应的 unicode 字符的字典

if len(sys.argv) < 2:  # 如果命令行参数少于 2 个
    print("Usage: convert-cerebras-to-ggml.py dir-model [use-f32]\n")  # 打印使用说明
    sys.exit(1)  # 退出程序

# output in the same directory as the model
dir_model = sys.argv[1]  # 获取模型目录
fname_out = sys.argv[1] + "/ggml-model-f16.bin"  # 设置输出文件名为模型目录下的 ggml-model-f16.bin

with open(dir_model + "/vocab.json", "r", encoding="utf-8") as f:  # 打开 vocab.json 文件
    encoder = json.load(f)  # 加载 JSON 数据到 encoder 变量

with open(dir_model + "/config.json", "r", encoding="utf-8") as f:  # 打开 config.json 文件
    hparams = json.load(f)  # 加载 JSON 数据到 hparams 变量

# use 16-bit or 32-bit floats
use_f16 = True  # 默认使用 16 位浮点数
if len(sys.argv) > 2:  # 如果命令行参数大于 2 个
    use_f16 = False  # 设置使用 32 位浮点数
    fname_out = sys.argv[1] + "/ggml-model-f32.bin"  # 设置输出文件名为模型目录下的 ggml-model-f32.bin

model = AutoModelForCausalLM.from_pretrained(dir_model, low_cpu_mem_usage=True)  # 从预训练模型中加载 AutoModelForCausalLM 对象
#print (model)

list_vars = model.state_dict()  # 获取模型的状态字典
#print (list_vars)

print(hparams)  # 打印模型参数
# 打开一个文件以写入二进制数据
fout = open(fname_out, "wb")

# 写入一个整数，表示文件的魔数，用16进制表示为 ggml
fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex

# 写入整数，表示词汇表大小
fout.write(struct.pack("i", hparams["vocab_size"]))

# 写入整数，表示位置编码的长度
fout.write(struct.pack("i", hparams["n_positions"]))

# 写入整数，表示嵌入层的维度
fout.write(struct.pack("i", hparams["n_embd"]))

# 写入整数，表示注意力头的数量
fout.write(struct.pack("i", hparams["n_head"]))

# 写入整数，表示层数
fout.write(struct.pack("i", hparams["n_layer"]))

# 写入整数，表示是否使用 f16
fout.write(struct.pack("i", use_f16))

# 创建字节到 Unicode 的编码器
byte_encoder = bytes_to_unicode()

# 创建字节到 Unicode 的解码器
byte_decoder = {v:k for k, v in byte_encoder.items()}

# 写入整数，表示编码器的长度
fout.write(struct.pack("i", len(encoder)))

# 遍历编码器中的键
for key in encoder:
    # 将编码器中的键转换为字节，并写入文件
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

# 遍历变量列表中的键
for name in list_vars.keys():
    # 获取变量的数据并转换为 numpy 数组
    data = list_vars[name].squeeze().numpy()
    # 打印变量的名称和形状
    print("Processing variable: " + name + " with shape: ", data.shape)

    # 重命名头部以保持兼容性
    if name == "transformer.ln_f.weight":
        name = "model/ln_f/g"
    elif name == "transformer.ln_f.bias":
        name = "model/ln_f/b"
    elif name == "transformer.wte.weight":
        name = "model/wte"
    elif name == "transformer.wpe.weight":
        name = "model/wpe"
    elif name == "lm_head.weight":
        name = "model/lm_head"
    elif re.match(r"transformer.h\.\d+\.ln_1\.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_1/g"
    elif re.match(r"transformer.h\.\d+\.ln_1\.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_1/b"
    elif re.match(r"transformer.h\.\d+\.attn\.c_attn\.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_attn/w"
    elif re.match(r"transformer.h\.\d+\.attn\.c_attn\.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_attn/b"
    elif re.match(r"transformer.h\.\d+\.attn\.c_proj\.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_proj/w"
    # 如果变量名匹配特定模式，则进行相应的重命名操作
    elif re.match(r"transformer.h.\d+.attn.c_proj.bias", name):
        # 从变量名中提取数字
        i = re.findall("\d+", name)[0]
        # 重命名变量
        name = f"model/h{i}/attn/c_proj/b"
    elif re.match(r"transformer.h.\d+.ln_2.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_2/g"
    elif re.match(r"transformer.h.\d+.ln_2.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_2/b"
    elif re.match(r"transformer.h.\d+.mlp.c_fc.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_fc/w"
    elif re.match(r"transformer.h.\d+.mlp.c_fc.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_fc/b"
    elif re.match(r"transformer.h.\d+.mlp.c_proj.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_proj/w"
    elif re.match(r"transformer.h.\d+.mlp.c_proj.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_proj/b"
    else:
        # 如果变量名不匹配任何模式，则打印未识别的变量名
        print("Unrecognized variable name. %s", name)

    # 如果变量名以特定后缀结尾，则跳过该变量
    if name.endswith("attn.masked_bias") or name.endswith(".attn.bias"):
        print("  Skipping variable: " + name)
        continue

    # 获取数据的维度
    n_dims = len(data.shape);

    # ftype == 0 -> float32, ftype == 1 -> float16
    ftype = 0;
    # 如果使用 float16 类型，并且满足特定条件，则将数据类型转换为 float16
    if use_f16:
        if (name == "model/wte" or name == "model/lm_head" or name[-2:] == "/g" or name[-2:] == "/w") and n_dims == 2:
            print("  Converting to float16")
            data = data.astype(np.float16)
            ftype = 1
        else:
            # 否则将数据类型转换为 float32
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype = 0

    # 为了效率，对投影矩阵进行转置操作
    # 匹配特定模式的变量名，并且数据维度为2，则进行转换操作
    # "model/h.*/attn/c_attn/w"
    # "model/h.*/attn/c_proj/w"
    # "model/h.*/mlp/c_fc/w"
    # "model/h.*/mlp/c_proj/w"
    # 检查文件名是否符合特定条件，如果符合则进行数据转置操作
    if name[-14:] == "/attn/c_attn/w" or \
       name[-14:] == "/attn/c_proj/w" or \
       name[-11:] == "/mlp/c_fc/w" or \
       name[-13:] == "/mlp/c_proj/w":
        # 打印提示信息
        print("  Transposing")
        # 对数据进行转置操作
        data = data.transpose()

    # 写入文件头部信息
    # 将文件名转换成 UTF-8 编码
    str = name.encode('utf-8')
    # 将文件头部信息写入输出文件
    fout.write(struct.pack("iii", n_dims, len(str), ftype))
    # 遍历维度信息，写入输出文件
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    # 将文件名写入输出文件
    fout.write(str);

    # 写入数据
    # 将数据以二进制形式写入输出文件
    data.tofile(fout)
# 关闭输出文件
fout.close()

# 打印完成消息和输出文件名
print("Done. Output file: " + fname_out)
print("")
```