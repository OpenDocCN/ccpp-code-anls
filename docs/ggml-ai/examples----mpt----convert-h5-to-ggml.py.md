# `ggml\examples\mpt\convert-h5-to-ggml.py`

```cpp
import os  # 导入操作系统模块
import struct  # 导入 struct 模块
import sys  # 导入系统模块

import torch  # 导入 torch 模块
from transformers import AutoConfig, AutoTokenizer  # 从 transformers 模块中导入 AutoConfig 和 AutoTokenizer 类


# ref: https://github.com/openai/gpt-2/blob/master/src/encoder.py
def bytes_to_unicode():  # 定义函数 bytes_to_unicode
    """
    Returns list of utf-8 byte and a corresponding list of unicode strings.
    The reversible bpe codes work on unicode strings.
    This means you need a large # of unicode characters in your vocab if you want to avoid UNKs.
    When you're at something like a 10B token dataset you end up needing around 5K for decent coverage.
    This is a signficant percentage of your normal, say, 32K bpe vocab.
    To avoid that, we want lookup tables between utf-8 bytes and unicode strings.
    And avoids mapping to whitespace/control characters the bpe code barfs on.
    """
    bs = (
        list(range(ord("!"), ord("~") + 1))  # 创建 utf-8 字节范围列表
        + list(range(ord("¡"), ord("¬") + 1))  # 创建 utf-8 字节范围列表
        + list(range(ord("®"), ord("ÿ") + 1))  # 创建 utf-8 字节范围列表
    )
    cs = bs[:]  # 复制 bs 列表
    n = 0  # 初始化 n 为 0
    for b in range(2**8):  # 遍历 0 到 255 的范围
        if b not in bs:  # 如果 b 不在 bs 列表中
            bs.append(b)  # 将 b 添加到 bs 列表
            cs.append(2**8 + n)  # 将 2**8 + n 添加到 cs 列表
            n += 1  # n 自增 1

    cs = [chr(n) for n in cs]  # 将 cs 列表中的每个元素转换为对应的 Unicode 字符

    return dict(zip(bs, cs))  # 返回 bs 和 cs 对应的字典


def count_model_parts(dir_model: str) -> int:  # 定义函数 count_model_parts，参数为 dir_model，返回类型为 int
    """Returns the number of model parts in the model directory."""
    num_parts = 0  # 初始化 num_parts 为 0
    for filename in os.listdir(dir_model):  # 遍历指定目录下的文件
        if filename.startswith("pytorch_model-"):  # 如果文件名以 "pytorch_model-" 开头
            num_parts += 1  # num_parts 自增 1

    if num_parts > 0:  # 如果 num_parts 大于 0
        print(f"Found {num_parts} model parts in {dir_model}")  # 打印找到的模型部分数量
    return num_parts  # 返回 num_parts


if len(sys.argv) < 3:  # 如果命令行参数个数小于 3
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")  # 打印使用说明
    print("  ftype == 0 -> float32")  # 打印说明
    print("  ftype == 1 -> float16")  # 打印说明
    sys.exit(1)  # 退出程序，返回状态码 1


# output in the same directory as the model
dir_model = sys.argv[1]  # 获取命令行参数中的第一个参数作为模型目录
# get number of model parts
num_parts = count_model_parts(dir_model)  # 调用 count_model_parts 函数获取模型部分数量

# possible data types
#   ftype == 0 -> float32
#   ftype == 1 -> float16
#
# map from ftype to string
ftype_str = ["f32", "f16"]  # 定义 ftype 对应的字符串列表

ftype = 1  # 初始化 ftype 为 1
if len(sys.argv) > 2:  # 如果命令行参数个数大于 2
    # 从命令行参数中获取文件类型，并转换为整数
    ftype = int(sys.argv[2])
    # 如果文件类型小于0或大于1，则打印错误信息并退出程序
    if ftype < 0 or ftype > 1:
        print("Invalid ftype: " + str(ftype))
        sys.exit(1)
    # 根据文件类型和目录模型拼接输出文件名
    fname_out = dir_model + "/ggml-model-" + ftype_str[ftype] + ".bin"
# 从指定目录加载预训练模型的分词器
tokenizer = AutoTokenizer.from_pretrained(dir_model, trust_remote_code=True)
# 从指定目录加载预训练模型的配置
config = AutoConfig.from_pretrained(dir_model, trust_remote_code=True)
# 将配置转换为字典格式
hparams = config.to_dict()

# 打开一个文件以进行写入操作，以二进制格式
fout = open(fname_out, "wb")

# 写入文件头部信息，包括魔数和模型参数等
fout.write(struct.pack("i", 0x67676D6C))  # magic: ggml in hex
fout.write(struct.pack("i", hparams["d_model"]))
fout.write(struct.pack("i", hparams["max_seq_len"]))
fout.write(struct.pack("i", hparams["n_heads"]))
fout.write(struct.pack("i", hparams["n_layers"]))
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("f", hparams["attn_config"]["alibi_bias_max"]))
fout.write(struct.pack("f", hparams["attn_config"]["clip_qkv"] or 0.0))
fout.write(struct.pack("i", ftype))

# 获取词汇表大小
vocab_size = hparams["vocab_size"]

# 获取编码器的词汇表
encoder = tokenizer.vocab
# 将特殊标记（添加的标记）添加到编码器中
encoder.update(tokenizer.get_added_vocab())

# 创建字节到Unicode的映射
byte_encoder = bytes_to_unicode()
# 创建Unicode到字节的映射
byte_decoder = {v: k for k, v in byte_encoder.items()}

# 计数器初始化
counter = 0
# 按值对编码器进行排序
for key in sorted(encoder, key=encoder.get):
    # 为了解决当c未找到时的键错误
    text = ""
    for c in key:
        if c not in byte_decoder:
            text += c
        else:
            text += chr(byte_decoder[c])
    # 将文本转换为UTF-8编码的字节数组，并写入文件
    text = bytearray(text, encoding="utf-8")
    fout.write(struct.pack("i", len(text)))
    fout.write(text)
    counter += 1

# 直到计数器达到词汇表大小，重复最后一个标记
while counter < vocab_size:
    fout.write(struct.pack("i", len(text)))
    fout.write(text)
    counter += 1

# 如果num_parts为0，则设置part_names为("pytorch_model.bin",)；否则，设置为对应的文件名生成器
if num_parts == 0:
    part_names = ("pytorch_model.bin",)
else:
    part_names = (
        f"pytorch_model-{n:05}-of-{num_parts:05}.bin" for n in range(1, num_parts + 1)
    )

# 遍历part_names中的文件名
for part_name in part_names:
    # 打印加载的部分模型文件名
    print(f"\n* Loading part: {part_name}")
    # 加载模型的部分文件
    model_part = torch.load(f"{dir_model}/{part_name}", map_location="cpu")
    # 遍历模型部分的所有键值对
    for name in model_part.keys():
        # 从模型部分获取数据，并去除维度为1的维度
        data = model_part[name].squeeze()
        # 获取数据的维度数
        n_dims = len(data.shape)

        # 判断数据类型，ftype == 0 -> float32, ftype == 1 -> float16，默认类型是fp32
        ftype_cur = 0
        # 如果ftype为1且数据名称以".weight"结尾且维度大于1，则数据类型为float16
        if ftype == 1 and name[-7:] == ".weight" and n_dims > 1:
            ftype_cur = 1
        # 将数据转换为指定类型的Tensor，并转换为numpy数组
        data = data.to(dtype=torch.float16 if ftype_cur == 1 else torch.float32).numpy()

        # 打印处理的变量名称、形状和数据类型
        print(
            "Processing variable: " + name + " with shape: ",
            data.shape,
            "->",
            data.dtype,
        )

        # 写入头部信息
        # 将变量名编码为utf-8格式
        str = name.encode("utf-8")
        # 写入维度数、变量名长度、数据类型到文件
        fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
        # 写入每个维度的大小到文件
        for i in range(n_dims):
            fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
        # 写入变量名到文件
        fout.write(str)

        # 写入数据
        data.tofile(fout)

    # 释放模型部分的内存
    del model_part
# 关闭输出文件
fout.close()

# 打印完成消息和输出文件名
print("Done. Output file: " + fname_out)
print("")
```