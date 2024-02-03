# `ggml\examples\whisper\convert-pt-to-ggml.py`

```cpp
# 将Whisper转换器模型从PyTorch转换为ggml格式
#
# 用法：python convert-pt-to-ggml.py ~/.cache/whisper/medium.pt ~/path/to/repo/whisper/ ./models/whisper-medium
#
# 您需要在~/path/to/repo/whisper/中克隆原始存储库
#
#  git clone https://github.com/openai/whisper ~/path/to/repo/whisper/
#
# 它用于算法需要的各种资产：
#
#  - 分词器
#  - 梅尔滤波器
#
# 此外，您需要在~/.cache/whisper/中拥有原始模型
# 有关更多详细信息，请参阅原始存储库。
#
# 此脚本加载指定的模型和whisper资产，并以ggml格式保存它们。
# 输出是一个包含以下信息的单个二进制文件：
#
#  - hparams
#  - 梅尔滤波器
#  - 分词器词汇表
#  - 模型变量
#
# 对于每个变量，写入以下内容：
#
#  - 维度数量（int）
#  - 名称长度（int）
#  - 维度（int[n_dims]）
#  - 名称（char[name_length]）
#  - 数据（float[n_dims]）
#

import io
import os
import sys
import struct
import json
import code
import torch
import numpy as np
import base64
from pathlib import Path
#from transformers import GPTJForCausalLM
#from transformers import GPT2TokenizerFast

# 参考：https://github.com/openai/whisper/blob/8cf36f3508c9acd341a45eb2364239a3d81458b9/whisper/tokenizer.py#L10-L110
# 语言字典
#LANGUAGES = {
#    "en": "english",
#    "zh": "chinese",
#    "de": "german",
#    "es": "spanish",
#    "ru": "russian",
#    "ko": "korean",
#    "fr": "french",
#    "ja": "japanese",
#    "pt": "portuguese",
#    "tr": "turkish",
#    "pl": "polish",
#    "ca": "catalan",
#    "nl": "dutch",
#    "ar": "arabic",
#    "sv": "swedish",
#    "it": "italian",
#    "id": "indonesian",
#    "hi": "hindi",
#    "fi": "finnish",
#    "vi": "vietnamese",
#    "iw": "hebrew",
#    "uk": "ukrainian",
#    "el": "greek",
#    "ms": "malay",
#    "cs": "czech",
#    "ro": "romanian",
#    "da": "danish",
#    "hu": "hungarian",
#    "ta": "tamil",
#    "no": "norwegian",
#    "th": "thai",
# 定义一个包含语言缩写和语言名称的字典
LANGUAGES = {
    "ur": "urdu",
    "hr": "croatian",
    "bg": "bulgarian",
    "lt": "lithuanian",
    "la": "latin",
    "mi": "maori",
    "ml": "malayalam",
    "cy": "welsh",
    "sk": "slovak",
    "te": "telugu",
    "fa": "persian",
    "lv": "latvian",
    "bn": "bengali",
    "sr": "serbian",
    "az": "azerbaijani",
    "sl": "slovenian",
    "kn": "kannada",
    "et": "estonian",
    "mk": "macedonian",
    "br": "breton",
    "eu": "basque",
    "is": "icelandic",
    "hy": "armenian",
    "ne": "nepali",
    "mn": "mongolian",
    "bs": "bosnian",
    "kk": "kazakh",
    "sq": "albanian",
    "sw": "swahili",
    "gl": "galician",
    "mr": "marathi",
    "pa": "punjabi",
    "si": "sinhala",
    "km": "khmer",
    "sn": "shona",
    "yo": "yoruba",
    "so": "somali",
    "af": "afrikaans",
    "oc": "occitan",
    "ka": "georgian",
    "be": "belarusian",
    "tg": "tajik",
    "sd": "sindhi",
    "gu": "gujarati",
    "am": "amharic",
    "yi": "yiddish",
    "lo": "lao",
    "uz": "uzbek",
    "fo": "faroese",
    "ht": "haitian creole",
    "ps": "pashto",
    "tk": "turkmen",
    "nn": "nynorsk",
    "mt": "maltese",
    "sa": "sanskrit",
    "lb": "luxembourgish",
    "my": "myanmar",
    "bo": "tibetan",
    "tl": "tagalog",
    "mg": "malagasy",
    "as": "assamese",
    "tt": "tatar",
    "haw": "hawaiian",
    "ln": "lingala",
    "ha": "hausa",
    "ba": "bashkir",
    "jw": "javanese",
    "su": "sundanese",
}

# 引用了一个外部的代码库，构建一个分词器
# 定义了一个函数，用于构建分词器
def build_tokenizer(path_to_whisper_repo: str, name: str = "gpt2"):
    # 设置环境变量，禁用分词器的并行处理
    os.environ["TOKENIZERS_PARALLELISM"] = "false"
    # 拼接路径，指向分词器所在的位置
    path = os.path.join(path_to_whisper_repo, "whisper/assets", name)
    # 从预训练模型中加载 GPT2 分词器
    tokenizer = GPT2TokenizerFast.from_pretrained(path)

    # 定义特殊标记
    specials = [
        "<|startoftranscript|>",
        # 使用列表推导式，为每种语言添加特殊标记
        *[f"<|{lang}|>" for lang in LANGUAGES.keys()],
# 引入必要的库
import sys
from pathlib import Path
import io
import torch

# 如果命令行参数少于4个，打印使用说明并退出程序
if len(sys.argv) < 4:
    print("Usage: convert-pt-to-ggml.py model.pt path-to-whisper-repo dir-output [use-f32]\n")
    sys.exit(1)

# 从命令行参数中获取输入文件名、whisper仓库路径和输出目录路径
fname_inp   = Path(sys.argv[1])
dir_whisper = Path(sys.argv[2])
dir_out     = Path(sys.argv[3])

# 尝试加载PyTorch二进制数据
try:
    # 读取模型文件的二进制数据
    model_bytes = open(fname_inp, "rb").read()
    # 将二进制数据封装成字节流
    with io.BytesIO(model_bytes) as fp:
        # 使用torch.load加载模型数据
        checkpoint = torch.load(fp, map_location="cpu")
except Exception:
    # 如果加载失败，打印错误信息并退出程序
    print("Error: failed to load PyTorch model file:" , fname_inp)
    sys.exit(1)

# 从checkpoint中获取模型参数
hparams = checkpoint["dims"]
# 打印模型参数
print("hparams:", hparams)

# 从checkpoint中获取模型的状态字典
list_vars = checkpoint["model_state_dict"]

# 打印特定变量的值
# print(list_vars['encoder.positional_embedding'])
# print(list_vars['encoder.conv1.weight'])
# 打印变量 'encoder.conv1.weight' 的形状
#print(list_vars['encoder.conv1.weight'].shape)

# 加载 mel 滤波器
n_mels = hparams["n_mels"]
with np.load(dir_whisper / "whisper" / "assets" / "mel_filters.npz") as f:
    filters = torch.from_numpy(f[f"mel_{n_mels}"])
    #print (filters)

#code.interact(local=locals())

# 加载分词器
# 为了向后兼容，还要检查旧的 hf_transformers 格式的分词器文件
# 旧格式: dir_whisper/whisper/assets/[multilingual/gpt2]/vocab.json
# 新格式: dir_whisper/whisper/assets/[multilingual/gpt2].tiktoken
multilingual = hparams["n_vocab"] == 51865
tokenizer = dir_whisper / "whisper" / "assets" / (multilingual and "multilingual.tiktoken" or "gpt2.tiktoken")
tokenizer_type = "tiktoken"
if not tokenizer.is_file():
    tokenizer = dir_whisper / "whisper" / "assets" / (multilingual and "multilingual" or "gpt2") / "vocab.json"
    tokenizer_type = "hf_transformers"
    if not tokenizer.is_file():
        print("Error: failed to find either tiktoken or hf_transformers tokenizer file:", tokenizer)
        sys.exit(1)

# 创建字节编码器和字节解码器
byte_encoder = bytes_to_unicode()
byte_decoder = {v:k for k, v in byte_encoder.items()}

# 根据分词器类型加载 tokens
if tokenizer_type == "tiktoken":
    with open(tokenizer, "rb") as f:
        contents = f.read()
        tokens = {base64.b64decode(token): int(rank) for token, rank in (line.split() for line in contents.splitlines() if line)}
elif tokenizer_type == "hf_transformers":
    with open(tokenizer, "r", encoding="utf8") as f:
        _tokens_raw = json.load(f)
        if '
    # 创建一个文件名，指向 dir_out 目录下的 ggml-model-f32.bin 文件
    fname_out = dir_out / "ggml-model-f32.bin"
# 以二进制写入模式打开输出文件
fout = fname_out.open("wb")

# 写入魔数，表示文件类型
fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex

# 写入模型参数的数量
fout.write(struct.pack("i", hparams["n_vocab"]))
fout.write(struct.pack("i", hparams["n_audio_ctx"]))
fout.write(struct.pack("i", hparams["n_audio_state"]))
fout.write(struct.pack("i", hparams["n_audio_head"]))
fout.write(struct.pack("i", hparams["n_audio_layer"]))
fout.write(struct.pack("i", hparams["n_text_ctx"]))
fout.write(struct.pack("i", hparams["n_text_state"]))
fout.write(struct.pack("i", hparams["n_text_head"]))
fout.write(struct.pack("i", hparams["n_text_layer"]))
fout.write(struct.pack("i", hparams["n_mels"]))
fout.write(struct.pack("i", use_f16))

# 写入梅尔滤波器的形状信息
fout.write(struct.pack("i", filters.shape[0]))
fout.write(struct.pack("i", filters.shape[1]))

# 遍历梅尔滤波器的值，并写入文件
for i in range(filters.shape[0]):
    for j in range(filters.shape[1]):
        fout.write(struct.pack("f", filters[i][j]))

# 写入标记器的信息
fout.write(struct.pack("i", len(tokens)))

# 遍历标记器的键值对，并写入文件
for key in tokens:
    fout.write(struct.pack("i", len(key)))
    fout.write(key)

# 遍历模型变量的键值对
for name in list_vars.keys():
    # 获取变量的数据并打印其形状信息
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " , name ,  " with shape: ", data.shape)

    # 如果变量名在指定的列表中，则将其形状从 [n] 调整为 [n, 1]
    if name in ["encoder.conv1.bias", "encoder.conv2.bias"]:
        data = data.reshape(data.shape[0], 1)
        print(f"  Reshaped variable: {name} to shape: ", data.shape)

    # 获取数据的维度
    n_dims = len(data.shape)

    # 设置数据类型为 float16
    ftype = 1
    # 如果 use_f16 为真，则执行以下代码块
    if use_f16:
        # 如果维度小于2或者名称为指定的几个之一，则执行以下代码块
        if n_dims < 2 or \
                name == "encoder.conv1.bias"   or \
                name == "encoder.conv2.bias"   or \
                name == "encoder.positional_embedding" or \
                name == "decoder.positional_embedding":
            # 打印提示信息
            print("  Converting to float32")
            # 将数据类型转换为 np.float32
            data = data.astype(np.float32)
            # 设置 ftype 为 0
            ftype = 0
    # 如果 use_f16 为假，则执行以下代码块
    else:
        # 将数据类型转换为 np.float32
        data = data.astype(np.float32)
        # 设置 ftype 为 0
        ftype = 0

    # 创建名称的 UTF-8 编码
    str_ = name.encode('utf-8')
    # 将头部信息写入输出文件
    fout.write(struct.pack("iii", n_dims, len(str_), ftype))
    # 遍历维度，将每个维度的大小写入输出文件
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    # 将名称写入输出文件
    fout.write(str_)

    # 将数据写入输出文件
    data.tofile(fout)
# 关闭输出文件
fout.close()

# 打印完成消息和输出文件名
print("Done. Output file: " , fname_out)
print("")
```