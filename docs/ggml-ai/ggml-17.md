# GGML源码解析 17

# `examples/whisper/convert-pt-to-ggml.py`

这段代码的作用是将PyTorch中的Whisper模型转换为GGML格式。它将./models/whisper-medium文件中的Whisper模型转换为JSON格式，以便在其它地方使用。转换后的模型需要从~/.cache/whisper/中克隆原始仓库。

首先，这段代码检查本地是否已经存在Whisper模型，如果不存在，它会从 GitHub 上克隆原始仓库。然后，它会将Whisper模型中的tokenizer和mel过滤器从模型中提取出来，并将它们存储在一个名为whisper-medium的文件中。这个文件需要在运行这段代码之前，从~/.cache/whisper/中下载并将其解压缩。

最后，这段代码将./models/whisper-medium文件中的Whisper模型转换为JSON格式，以便在其它地方使用。


```cpp
# Convert Whisper transformer model from PyTorch to ggml format
#
# Usage: python convert-pt-to-ggml.py ~/.cache/whisper/medium.pt ~/path/to/repo/whisper/ ./models/whisper-medium
#
# You need to clone the original repo in ~/path/to/repo/whisper/
#
#  git clone https://github.com/openai/whisper ~/path/to/repo/whisper/
#
# It is used to various assets needed by the algorithm:
#
#  - tokenizer
#  - mel filters
#
# Also, you need to have the original models in ~/.cache/whisper/
# See the original repo for more details.
```

这段代码的作用是加载指定的训练模型（hparams）、whisper资产（whisper_assets）并将其保存为ggml格式，然后输出一个单一的二进制文件。ggml格式是一种包含文本、图像、语音和其他类型数据的标准格式。

具体来说，这段代码对于每个变量（hparams、mel filters、tokenizer vocab、model variables等），都会将其值写入一个ggml文件中。ggml文件包含了一个二进制数据流，每个数据流都被写入了一个不同的ggml记录中。这些ggml记录包含了不同类型的数据，包括文本、图像、语音等，以及一些元数据，如样本大小、采样率等。

这段代码的作用是将指定的训练模型及其相关的元数据（如mel过滤器、词向量等）和whisper资产保存为ggml格式，并输出一个二进制文件。这个文件可以在需要时被用于训练或分析机器学习模型。


```cpp
#
# This script loads the specified model and whisper assets and saves them in ggml format.
# The output is a single binary file containing the following information:
#
#  - hparams
#  - mel filters
#  - tokenizer vocab
#  - model variables
#
# For each variable, write the following:
#
#  - Number of dimensions (int)
#  - Name length (int)
#  - Dimensions (int[n_dims])
#  - Name (char[name_length])
```

这段代码的作用是实现了一个数据读取、转换和处理的函数。具体来说，它包括以下几个步骤：

1. 读取输入数据：从用户输入或文件中读取数据，可以是文本、图像或其他类型的数据。
2. 转换数据类型：将数据转换为浮点数向量，以便适用于后续计算。
3. 处理数据：对数据进行处理，例如将数据按照指定维度进行切分、分组或合并等操作。
4. 将数据存储：将处理后的数据存储到指定的目标文件中，可以是本地文件、服务器或远程服务器等。

具体实现过程可能会根据具体的应用场景而有所不同，但总体来说，这段代码的核心目的是为了实现数据的中间处理和存储，方便后续的计算和分析。


```cpp
#  - Data (float[n_dims])
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

```

这段代码定义了一个名为 "LANGUAGES" 的字典，它包含了 20 个不同的语言名称，包括英语、汉语、德语、西班牙语、法语、意大利语、德语、韩语、日语、韩语、波兰语、葡萄牙语、土耳其语、荷兰语和 Catalan 语等。这个字典被用于定义一个名为 "whisper.py" 的模块，它可能是一个机器学习模型或应用程序，用于从文本数据中提取语言信息。


```cpp
# ref: https://github.com/openai/whisper/blob/8cf36f3508c9acd341a45eb2364239a3d81458b9/whisper/tokenizer.py#L10-L110
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
```

这是一段Python代码，它定义了一个包含多个国家和其相应的缩写代码的字典。这个字典的目的是让用户能够通过输入一个字符串，来查找该字符串对应的国家名称。

例如，如果您输入 "arabic"，则程序将返回 "arabic" 的缩写代码，而如果您输入 "hi"，则程序将返回 "hindi"。


```cpp
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
```

这是一段Python代码，它定义了一个包含多个国家和相应的货币代码的的字典。这个字典的目的是为了存储来自不同国家的示例文本，以便在需要时进行查找和理解。

具体来说，这个字典包含了来自印度、挪威、泰国、土耳其、乌克兰、匈牙利、波斯尼亚和德国等国家的货币代码。这些货币代码分别对应着“tamil”、“norwegian”、“thai”、“urdu”、“croatian”、“bulgarian”、“lithuanian”、“latvian”、“welsh”、“slovak”、“telugu”和“persian”等不同的字符串。


```cpp
#    "ta": "tamil",
#    "no": "norwegian",
#    "th": "thai",
#    "ur": "urdu",
#    "hr": "croatian",
#    "bg": "bulgarian",
#    "lt": "lithuanian",
#    "la": "latin",
#    "mi": "maori",
#    "ml": "malayalam",
#    "cy": "welsh",
#    "sk": "slovak",
#    "te": "telugu",
#    "fa": "persian",
#    "lv": "latvian",
```



以上代码是一个Python字符串列表，其中包含了一些国家/地区名称。每个条目是一个字典，其中包含一个键值对，键是一个字符串，值为该国家的语言代码。 

这个列表是一个预定义的Python字符串列表，通常用于在Python脚本中导入并定义这些国家/地区名称。在Python中，可以使用这些名称来描述不同国家/地区的语言，可以帮助编写易于阅读和理解的代码。


```cpp
#    "bn": "bengali",
#    "sr": "serbian",
#    "az": "azerbaijani",
#    "sl": "slovenian",
#    "kn": "kannada",
#    "et": "estonian",
#    "mk": "macedonian",
#    "br": "breton",
#    "eu": "basque",
#    "is": "icelandic",
#    "hy": "armenian",
#    "ne": "nepali",
#    "mn": "mongolian",
#    "bs": "bosnian",
#    "kk": "kazakh",
```

以上代码是一个Python字符串列表，其中包含多个键值对，每个键都是一个翻译名称，值为该名称对应的语言名称。

这个列表的作用是提供一个Map，它将每种语言的名称映射到一个翻译名称。例如，如果你使用这个列表，你可以键入一个翻译名称，然后程序将返回与该名称相对应的语言名称。例如，如果你想将"albanian"翻译成"阿尔巴尼亚语"，你可以在Map中键入"sq"这个键，然后程序将返回"albanian"。


```cpp
#    "sq": "albanian",
#    "sw": "swahili",
#    "gl": "galician",
#    "mr": "marathi",
#    "pa": "punjabi",
#    "si": "sinhala",
#    "km": "khmer",
#    "sn": "shona",
#    "yo": "yoruba",
#    "so": "somali",
#    "af": "afrikaans",
#    "oc": "occitan",
#    "ka": "georgian",
#    "be": "belarusian",
#    "tg": "tajik",
```

这是一段Python代码，它定义了一个包含多个翻译字典的字典。每个键都是一个字符串，表示一种语言，而值则是该语言的名称。

这个字典的目的是提供一个简单的映射，让开发者在学习其他语言时，可以使用这个字典来查找自己需要翻译的语言。例如，如果你正在学习西班牙语，你可以使用这个字典查找 "sd"（某种印度语言）的翻译。


```cpp
#    "sd": "sindhi",
#    "gu": "gujarati",
#    "am": "amharic",
#    "yi": "yiddish",
#    "lo": "lao",
#    "uz": "uzbek",
#    "fo": "faroese",
#    "ht": "haitian creole",
#    "ps": "pashto",
#    "tk": "turkmen",
#    "nn": "nynorsk",
#    "mt": "maltese",
#    "sa": "sanskrit",
#    "lb": "luxembourgish",
#    "my": "myanmar",
```

这段代码定义了一个名为 `build_tokenizer` 的函数，它的功能是构建一个词表。这个词表是一个字典，其中包含了许多不同的语言和对应的单词。

字段名包含：

* `"bo"`：藏语
* `"tl"`： Tagalog
* `"mg"`： Malagasy
* `"as"`： Assamese
* `"tt"`： Tatar
* `"haw"`： Hawaiian
* `"ln"`： Linguala
* `"ha"`： Hausa
* `"ba"`： Bashkir
* `"jw"`： Javanese
* `"su"`： Sundanese

这个词表是通过 `whisper.py` 中的 `tokenizer.py` 文件中的 `load_whisper_tokenizer` 函数加载出来的。


```cpp
#    "bo": "tibetan",
#    "tl": "tagalog",
#    "mg": "malagasy",
#    "as": "assamese",
#    "tt": "tatar",
#    "haw": "hawaiian",
#    "ln": "lingala",
#    "ha": "hausa",
#    "ba": "bashkir",
#    "jw": "javanese",
#    "su": "sundanese",
#}

## ref: https://github.com/openai/whisper/blob/8cf36f3508c9acd341a45eb2364239a3d81458b9/whisper/tokenizer.py#L273-L292
#def build_tokenizer(path_to_whisper_repo: str, name: str = "gpt2"):
```

这段代码的作用是设置一个名为 "TOKENIZERS_PARALLELISM" 的环境变量为 "false"，然后将一个名为 "whisper/assets/<PASCAL_NamedEntityType-token|tokenized_text|segmentation_mask>.txt" 的文件夹路径加入到了 "path_to_whisper_repo" 这个变量中。接着，从 "whisper/assets/<PASCAL_NamedEntityType-token|tokenized_text|segmentation_mask>.txt" 中读取一个名为 GPT2TokenizerFast 的预训练模型，并将其存储在名为 "tokenizer" 的变量中。最后，创建了一个名为 "specials" 的列表，其中包含了一些特殊标记，如开始转录、翻译、转录、开始oflm 和开始ofprev等，这些标记将在预处理文本时用于处理。


```cpp
#    os.environ["TOKENIZERS_PARALLELISM"] = "false"
#    path = os.path.join(path_to_whisper_repo, "whisper/assets", name)
#    tokenizer = GPT2TokenizerFast.from_pretrained(path)
#
#    specials = [
#        "<|startoftranscript|>",
#        *[f"<|{lang}|>" for lang in LANGUAGES.keys()],
#        "<|translate|>",
#        "<|transcribe|>",
#        "<|startoflm|>",
#        "<|startofprev|>",
#        "<|nocaptions|>",
#        "<|notimestamps|>",
#    ]
#
```

这段代码定义了一个名为 `bytes_to_unicode` 的函数，它的作用是将一个字节序列转换为 Unicode 序列，并返回一个二者的映射。

具体来说，这个函数接收一个字节序列（也就是一个文本数据中的所有字节）和一个特殊标记的字典（这个字典包含了一些特殊标记，比如 `specials`）。函数首先将所有这些特殊标记转换为 Unicode 字符，并将它们与字节列表一起存储在一个字典中，然后将这个字典的键（也就是这些特殊标记）映射到它们对应的 Unicode 字符上，最后返回这个字典。

这个函数的实现依赖于一个名为 `bytes_to_unicode` 的函数，这个函数接收一个字节序列和一个特殊标记的字典，返回一个二者的映射，这个映射将特殊标记映射为 Unicode 字符。这里定义的 `bytes_to_unicode` 函数接收一个字节序列和一个特殊标记的字典，并将特殊标记的字典中的每个特殊标记映射为 Unicode 字符，然后将特殊标记的字典和它们对应的 Unicode 字符一起存储在一个新的字典中，最后返回这个新的字典。


```cpp
#    tokenizer.add_special_tokens(dict(additional_special_tokens=specials))
#    return tokenizer

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
    return dict(zip(bs, cs))


```

这段代码的作用是运行一个名为“convert-pt-to-ggml.py”的Python脚本，它需要三个参数：一个用于存储模型文件的路径和一个用于存储Whisper Repo文件的路径。如果存储模型文件的路径和一个Whisper Repo文件的路径中有一个是空字符串，则脚本会输出一条使用说明，并终止脚本。

具体来说，这段代码首先读取使用者的参数列表，如果其中的参数数量少于4个，则输出一条使用说明，并返回一个错误代码。接下来，它将尝试加载一个PyTorch二进制数据文件，如果加载失败，则输出一条错误消息，并终止脚本。如果成功加载，它将读取存储的模型文件并将其加载到内存中，然后使用`torch.load`函数将其保存到指定的输出文件中。


```cpp
if len(sys.argv) < 4:
    print("Usage: convert-pt-to-ggml.py model.pt path-to-whisper-repo dir-output [use-f32]\n")
    sys.exit(1)

fname_inp   = Path(sys.argv[1])
dir_whisper = Path(sys.argv[2])
dir_out     = Path(sys.argv[3])

# try to load PyTorch binary data
try:
    model_bytes = open(fname_inp, "rb").read()
    with io.BytesIO(model_bytes) as fp:
        checkpoint = torch.load(fp, map_location="cpu")
except Exception:
    print("Error: failed to load PyTorch model file:" , fname_inp)
    sys.exit(1)

```

这段代码的主要作用是加载预训练的模型参数和Mel频率文件。

首先，它将一个名为"checkpoint"的参数存储在名为"hparams"的变量中。这个参数包含模型的大小和架构信息。

然后，它打印出"hparams"变量的值。

接下来，它打印出从"model_state_dict"中获取的编码器的状态字典。这个字典包含了编码器中的各种参数和它们的位置信息。

然后，它打印出编码器中位置编码的Mel频率数组。"positional_embedding"在这里是指编码器中位置编码的Mel频率数组。

接着，它使用"with"语句加载Mel频率文件。"whisper"是一个文件夹，里面包含许多Mel频率文件。它从每个文件中读取Mel频率数组，并将其转换为PyTorch张量。

最后，它将Mel频率数组存储在一个名为"filters"的PyTorch张量中，并打印出这个张量的值。


```cpp
hparams = checkpoint["dims"]
print("hparams:", hparams)

list_vars = checkpoint["model_state_dict"]

#print(list_vars['encoder.positional_embedding'])
#print(list_vars['encoder.conv1.weight'])
#print(list_vars['encoder.conv1.weight'].shape)

# load mel filters
n_mels = hparams["n_mels"]
with np.load(dir_whisper / "whisper" / "assets" / "mel_filters.npz") as f:
    filters = torch.from_numpy(f[f"mel_{n_mels}"])
    #print (filters)

```

这段代码是一个交互式代码，它尝试加载一个预训练的语言模型，以便对文本进行分析和摘要。这个语言模型的加载取决于几个参数。

首先，它定义了一个名为 "locals" 的变量，但是这个变量在代码的上下文中并没有被定义。

然后，它加载了一个tokenizer，这个tokenizer的加载取决于一个名为 "hparams" 的参数。hparams中包含一个名为 "n_vocab" 的参数，它的值决定了要加载的词汇表的大小。

接下来，定义了一个名为 "tokenizer" 的变量，它是上面加载的tokenizer的别名，可以是 "tiktoken" 或 "hf_transformers"。

接着，定义了一个名为 "multilingual" 的变量，它的值是一个布尔值，表示要加载的语言模型是否支持多语言。

然后，定义了一个名为 "dir_whisper" 的目录，它包含了一个名为 "whisper" 的目录，它包含了一个名为 "assets" 的目录，这个目录包含预训练的模型和tokenizer。

接着，定义了一个名为 "hf_transformers" 的变量，它的值是一个布尔值，表示要加载的tokenizer类型是HfTransformers。

接下来，定义了一个名为 "tokenizer_type" 的变量，它的值是一个字符串，表示要加载的tokenizer类型。这个变量根据hparams["n_vocab"]的值来决定，如果n_vocab等于51865，那么使用旧格式的tokenizer，否则使用新格式的tokenizer。

最后，如果tokenizer存在的话，就加载它，否则就加载 multilingual 中定义的文件，它是这个语言模型的 tokenizer 文件。


```cpp
#code.interact(local=locals())

# load tokenizer
# for backwards compatibility, also check for older hf_transformers format tokenizer files
# old format: dir_whisper/whisper/assets/[multilingual/gpt2]/vocab.json
# new format: dir_whisper/whisper/assets/[multilingual/gpt2].tiktoken
multilingual = hparams["n_vocab"] == 51865
tokenizer = dir_whisper / "whisper" / "assets" / (multilingual and "multilingual.tiktoken" or "gpt2.tiktoken")
tokenizer_type = "tiktoken"
if not tokenizer.is_file():
    tokenizer = dir_whisper / "whisper" / "assets" / (multilingual and "multilingual" or "gpt2") / "vocab.json"
    tokenizer_type = "hf_transformers"
    if not tokenizer.is_file():
        print("Error: failed to find either tiktoken or hf_transformers tokenizer file:", tokenizer)
        sys.exit(1)

```

这段代码的作用是实现一个字节编码到unicode编码的byte_encoder和一个将byte_encoder中的键值对映射到整数的byte_decoder。然后根据一个特定的tokenizer_type，读取文件中的内容并将其解码为相应的token。

如果tokenizer_type == "tiktoken"，则代码将读取一个名为tokenizer的文件，并将其内容读取到一个名为byte_decoder的map中，其中key是每个line中的基64编码的token,value是该token对应的整数 rank。

如果tokenizer_type == "hf_transformers"，则代码将读取一个名为tokenizer的文件，并将其内容解码为两个map，一个存储了所有实际的token，另一个存储了每个token对应的排名。在token中，如果存在"<|endoftext|>"这样的标记，则从该文件中删除该标记。最后，将解码后的token映射到整数上，并将解码后的token存储到一个名为tokens的map中。


```cpp
byte_encoder = bytes_to_unicode()
byte_decoder = {v:k for k, v in byte_encoder.items()}

if tokenizer_type == "tiktoken":
    with open(tokenizer, "rb") as f:
        contents = f.read()
        tokens = {base64.b64decode(token): int(rank) for token, rank in (line.split() for line in contents.splitlines() if line)}
elif tokenizer_type == "hf_transformers":
    with open(tokenizer, "r", encoding="utf8") as f:
        _tokens_raw = json.load(f)
        if '<|endoftext|>' in _tokens_raw:
            # ensures exact same model as tokenizer_type == tiktoken
            # details: https://github.com/ggerganov/whisper.cpp/pull/725
            del _tokens_raw['<|endoftext|>']
        tokens = {bytes([byte_decoder[c] for c in token]): int(idx) for token, idx in _tokens_raw.items()}

```

这段代码的作用是输出一个名为 "ggml-model.bin" 的文件，该文件内容是一个GGML模型，支持16位或32位浮点数。文件输出目录为 "output"，因此文件名为 "ggml-model.bin"。

首先，判断命令行参数数量是否大于4个。如果是，则执行以下操作：

1. 如果使用16位浮点数，则创建名为 "ggml-model-f32.bin" 的文件，并输出到文件输出目录中。
2. 如果使用32位浮点数，则创建名为 "ggml-model.bin" 的文件，并输出到文件输出目录中。

这段代码的逻辑是：首先定义一个变量 "use_f16"，值为True或False。然后，使用len(sys.argv)获取命令行参数的数量，如果数量大于4个，则执行以下操作：如果使用16位浮点数，则创建名为 "ggml-model-f32.bin" 的文件并输出到文件输出目录中；如果使用32位浮点数，则创建名为 "ggml-model.bin" 的文件并输出到文件输出目录中。


```cpp
# output in the same directory as the model
fname_out = dir_out / "ggml-model.bin"

# use 16-bit or 32-bit floats
use_f16 = True
if len(sys.argv) > 4:
    use_f16 = False
    fname_out = dir_out / "ggml-model-f32.bin"

fout = fname_out.open("wb")

fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["n_vocab"]))
fout.write(struct.pack("i", hparams["n_audio_ctx"]))
fout.write(struct.pack("i", hparams["n_audio_state"]))
```

这段代码是 writteni 中的一个函数，它的作用是向文件 fout 中写入一个结构体 hparams，然后向 fout 写入一系列整数，这些整数都是 hparams 中的各个成员的值。

首先，fout 中的第一个写入是 `struct.pack("i", hparams["n_audio_head"])`，这个函数将 hparams 中的 "n_audio_head" 成员的值封装成一个整数类型的变量，并使用 "i" 格式进行打包，最终 fout 会输出一个整数，这个整数就是 "n_audio_head" 的值。

接着，fout 中的第二个写入是 `struct.pack("i", hparams["n_audio_layer"])`，这个函数将 hparams 中的 "n_audio_layer" 成员的值封装成一个整数类型的变量，并使用 "i" 格式进行打包，最终 fout 会输出一个整数，这个整数就是 "n_audio_layer" 的值。

然后，fout 中的第三个写入是 `struct.pack("i", hparams["n_text_ctx"])`，这个函数将 hparams 中的 "n_text_ctx" 成员的值封装成一个整数类型的变量，并使用 "i" 格式进行打包，最终 fout 会输出一个整数，这个整数就是 "n_text_ctx" 的值。

接下来，fout 中的第四个写入是 `struct.pack("i", hparams["n_text_state"])`，这个函数将 hparams 中的 "n_text_state" 成员的值封装成一个整数类型的变量，并使用 "i" 格式进行打包，最终 fout 会输出一个整数，这个整数就是 "n_text_state" 的值。

接着，fout 中的第五个写入是 `struct.pack("i", hparams["n_text_head"])`，这个函数将 hparams 中的 "n_text_head" 成员的值封装成一个整数类型的变量，并使用 "i" 格式进行打包，最终 fout 会输出一个整数，这个整数就是 "n_text_head" 的值。

然后，fout 中的第六个写入是 `struct.pack("i", hparams["n_text_layer"])`，这个函数将 hparams 中的 "n_text_layer" 成员的值封装成一个整数类型的变量，并使用 "i" 格式进行打包，最终 fout 会输出一个整数，这个整数就是 "n_text_layer" 的值。

接下来，fout 中的第七个写入是 `struct.pack("f", filters[i][j])`，这个函数会将 filters 中的第 i 行第 j 列的值封装成一个浮点数类型的变量，并使用 "f" 格式进行打包，最终 fout 会输出一个浮点数，这个浮点数就是 filters[i][j] 的值。

最后，fout 中的第八个写入是 `use_f16`，这个函数会将 use_f16 的值输出，最终 fout 会输出一个整数，这个整数就是 `use_f16` 的值。


```cpp
fout.write(struct.pack("i", hparams["n_audio_head"]))
fout.write(struct.pack("i", hparams["n_audio_layer"]))
fout.write(struct.pack("i", hparams["n_text_ctx"]))
fout.write(struct.pack("i", hparams["n_text_state"]))
fout.write(struct.pack("i", hparams["n_text_head"]))
fout.write(struct.pack("i", hparams["n_text_layer"]))
fout.write(struct.pack("i", hparams["n_mels"]))
fout.write(struct.pack("i", use_f16))

# write mel filters
fout.write(struct.pack("i", filters.shape[0]))
fout.write(struct.pack("i", filters.shape[1]))
for i in range(filters.shape[0]):
    for j in range(filters.shape[1]):
        fout.write(struct.pack("f", filters[i][j]))

```

This code appears to be processing and storing the input data, `data`, in a file named `data.bin`. The input data is expected to be a tensor of shape `(batch_size, sequence_length, input_dim)`, where `batch_size` is the number of input sequences in a batch, `sequence_length` is the length of each sequence, and `input_dim` is the dimensionality of the input data.

The code first reshapes the convolutional bias (`data.shape[1]`) to have a shape of `(batch_size, 1)` before writing it to the file.

Then it checks whether the input data is intended to be in 16-bit format (`use_f16`) or in f32-bit format (`use_f32`). If the input data is intended to be in f16 format, the code converts it to f32 format until it reaches the first non-empty dimension of the input tensor.

The code then writes the input tensor to the file, along with some additional information in the format of `(batch_size, sequence_length, input_dim, ftype)`.

The data is then written to the file, along with the header information.


```cpp
# write tokenizer
fout.write(struct.pack("i", len(tokens)))

for key in tokens:
    fout.write(struct.pack("i", len(key)))
    fout.write(key)

for name in list_vars.keys():
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " , name ,  " with shape: ", data.shape)

    # reshape conv bias from [n] to [n, 1]
    if name in ["encoder.conv1.bias", "encoder.conv2.bias"]:
        data = data.reshape(data.shape[0], 1)
        print(f"  Reshaped variable: {name} to shape: ", data.shape)

    n_dims = len(data.shape)

    # looks like the whisper models are in f16 by default
    # so we need to convert the small tensors to f32 until we fully support f16 in ggml
    # ftype == 0 -> float32, ftype == 1 -> float16
    ftype = 1
    if use_f16:
        if n_dims < 2 or \
                name == "encoder.conv1.bias"   or \
                name == "encoder.conv2.bias"   or \
                name == "encoder.positional_embedding" or \
                name == "decoder.positional_embedding":
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype = 0
    else:
        data = data.astype(np.float32)
        ftype = 0

    #if name.startswith("encoder"):
    #    if name.endswith("mlp.0.weight") or \
    #       name.endswith("mlp.2.weight"):
    #        print("  Transposing")
    #        data = data.transpose()

    # header
    str_ = name.encode('utf-8')
    fout.write(struct.pack("iii", n_dims, len(str_), ftype))
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    fout.write(str_)

    # data
    data.tofile(fout)

```

这段代码的作用是关闭名为 "fout" 的文件，并输出文件的名称 "fname_out"。

具体来说，第一行使用 `fout.close()` 方法关闭文件输出流，防止文件被其他程序或脚本读取或修改。第二行使用 `print()` 函数输出一个空行，第三行再次使用 `print()` 函数输出文件的名称 "fname_out"。


```cpp
fout.close()

print("Done. Output file: " , fname_out)
print("")

```

# `examples/whisper/main.cpp`

这段代码是一个C语言程序，它包括两个头文件：whisper.h和common.h。whisper.h和common.h可能包含一些通用的函数和数据结构，whisper.h可能包含一些用于whisper库的函数和数据结构。

以下是可能的代码作用：

1. 引入whisper库的头文件，这样就可以使用whisper库中的函数和数据结构了。
2. 定义了一些变量，包括一个整型变量x，一个浮点型变量y和一个字符型变量name。
3. 包含一个数学库，使用<cmath>来包含一些数学函数，例如sin和cos函数。
4. 包含一个文件输入流库，使用<fstream>来读取文件中的数据。
5. 包含一个文件输出流库，使用<cstdio>来输出数据到文件中。
6. 包含一个字符串库，使用<cstring>来包含字符串操作的函数。
7. 包含一个多维数组，使用<vector>来表示。
8. 如果定义(_MSC_VER)，会禁用一些安全功能，以避免可能的loss of data。


```cpp
#include "common.h"

#include "whisper.h"

#include <cmath>
#include <fstream>
#include <cstdio>
#include <string>
#include <thread>
#include <vector>
#include <cstring>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

```

这段代码定义了一个终端颜色映射，将颜色分为 ranges [0.0, 0.1, ..., 0.9]，并确定了每个 range 对应的颜色。这些颜色有明确的表示，从红色到绿色，分别对应于 `"\033[38;5;196m"`，`"\033[38;5;202m"`，`"\033[38;5;208m"`，`"\033[38;5;220m"`，`"\033[38;5;190m"`，`"\033[38;5;154m"`，`"\033[38;5;82m"` 和 `"\033[38;5;82m"`。

此外，还定义了一个 `to_timestamp()` 函数，它将一个时间戳（`int64_t` 类型）转换为一个时间字符串（`std::string` 类型），例如：
```cppcpp
// 500 -> 00:05.000
// 6000 -> 01:00.000
std::string to_timestamp(int64_t t, bool comma = false) {
   int64_t msec = t * 10;
   int64_t hr = msec / (1000 * 60 * 60);
   msec = msec - hr * (1000 * 60 * 60);
   int64_t min = msec / (1000 * 60);
   msec = msec - min * (1000 * 60);
   int64_t sec = msec / 1000;
   msec = msec - sec * 1000;

   char buf[32];
   snprintf(buf, sizeof(buf), "%02d:%02d:%02d%s%03d", (int) hr, (int) min, (int) sec, comma ? "," : ".", (int) msec);

   return std::string(buf);
}
```
这个函数接受一个时间戳（`int64_t` 类型）和一个布尔参数 `comma`（false 时输出时间字符串，否则输出时间数字），它将时间戳转换为一个时间字符串，类似于 MySQL 中的 `to_string()` 函数。


```cpp
// Terminal color map. 10 colors grouped in ranges [0.0, 0.1, ..., 0.9]
// Lowest is red, middle is yellow, highest is green.
const std::vector<std::string> k_colors = {
    "\033[38;5;196m", "\033[38;5;202m", "\033[38;5;208m", "\033[38;5;214m", "\033[38;5;220m",
    "\033[38;5;226m", "\033[38;5;190m", "\033[38;5;154m", "\033[38;5;118m", "\033[38;5;82m",
};

//  500 -> 00:05.000
// 6000 -> 01:00.000
std::string to_timestamp(int64_t t, bool comma = false) {
    int64_t msec = t * 10;
    int64_t hr = msec / (1000 * 60 * 60);
    msec = msec - hr * (1000 * 60 * 60);
    int64_t min = msec / (1000 * 60);
    msec = msec - min * (1000 * 60);
    int64_t sec = msec / 1000;
    msec = msec - sec * 1000;

    char buf[32];
    snprintf(buf, sizeof(buf), "%02d:%02d:%02d%s%03d", (int) hr, (int) min, (int) sec, comma ? "," : ".", (int) msec);

    return std::string(buf);
}

```

这两段代码定义了两个函数。

第一个函数 `timestamp_to_sample` 接收两个整数参数 `t` 和 `n_samples`，并返回一个整数。这个函数的作用是将 `t` 转换为采样率（以每秒多少个采样为例），然后再将 `n_samples` 减去 1，取整之后将这个整数作为返回值。

第二个函数 `replace_all` 接收一个字符串 `s`、一个要查找的字符串 `search` 和一个要替换的字符串 `replace`，并返回替换后的字符串。这个函数遍历 `s` 字符串中 `search` 子串的位置，如果位置为 `std::string::npos`，则表示找到了匹配的位置，函数便开始替换。


```cpp
int timestamp_to_sample(int64_t t, int n_samples) {
    return std::max(0, std::min((int) n_samples - 1, (int) ((t*WHISPER_SAMPLE_RATE)/100)));
}

// helper function to replace substrings
void replace_all(std::string & s, const std::string & search, const std::string & replace) {
    for (size_t pos = 0; ; pos += replace.length()) {
        pos = s.find(search, pos);
        if (pos == std::string::npos) break;
        s.erase(pos, search.length());
        s.insert(pos, replace);
    }
}

// command-line parameters
```

This is a C++ class for speech synthesis tasks, specifically for TDRZ speaker turn tasks. It includes various parameters for controlling the speech synthesis process, such as the language to use, the prompt to display before each turn, the font for the text, and the model to use for the synthesis. It also has variables for tracking the turn duration, the maximum length of the text, the maximum number of concurrent synths, and the maximum number of scores that can be processed. Additionally, it has a parameter for enabling or disabling the speed up, debug mode, and detect language.

It also has a parameter for the word threshold, entropy threshold, log probability threshold, and float for word, entropy, log probability, beam size.

This class can be leveraged to create a Speech synthesis Task class that can be used to perform various tasks, such as generating text or synthesizing speech. And it can be used to perform various operations such as text formatting,score normalization, and many more.


```cpp
struct whisper_params {
    int32_t n_threads    = std::min(4, (int32_t) std::thread::hardware_concurrency());
    int32_t n_processors =  1;
    int32_t offset_t_ms  =  0;
    int32_t offset_n     =  0;
    int32_t duration_ms  =  0;
    int32_t progress_step =  5;
    int32_t max_context  = -1;
    int32_t max_len      =  0;
    int32_t best_of      =  2;
    int32_t beam_size    = -1;

    float word_thold    =  0.01f;
    float entropy_thold =  2.40f;
    float logprob_thold = -1.00f;

    bool speed_up        = false;
    bool debug_mode      = false;
    bool translate       = false;
    bool detect_language = false;
    bool diarize         = false;
    bool tinydiarize     = false;
    bool split_on_word   = false;
    bool no_fallback     = false;
    bool output_txt      = false;
    bool output_vtt      = false;
    bool output_srt      = false;
    bool output_wts      = false;
    bool output_csv      = false;
    bool output_jsn      = false;
    bool output_jsn_full = false;
    bool output_lrc      = false;
    bool print_special   = false;
    bool print_colors    = false;
    bool print_progress  = false;
    bool no_timestamps   = false;
    bool log_score       = false;

    std::string language  = "en";
    std::string prompt;
    std::string font_path = "/System/Library/Fonts/Supplemental/Courier New Bold.ttf";
    std::string model     = "models/ggml-base.en.bin";

    // [TDRZ] speaker turn string
    std::string tdrz_speaker_turn = " [SPEAKER_TURN]"; // TODO: set from command line

    std::string openvino_encode_device = "CPU";

    std::vector<std::string> fname_inp = {};
    std::vector<std::string> fname_out = {};
};

```

This is a program that appears to parse command-line arguments and set default values for different command-line options.

It takes command-line arguments using the `-h` or `--help` option to see the full list of available options and their default values.

For example, if the user enters `数码签 -h`, the program will print out all the available options with their default values:

```cpp 
数码签命令行工具

数码签版本：1.0
编码：utf-8

所有选项：
  -h,--help            显示帮助并退出
  -o,--output [文件名],--output-json-full,--output-file        输出JSON文件或文件
```

If the user enters a specific option, the program will set the corresponding default value for the user.

For example, if the user enters `数码签 -o --output-json-full`, the program will automatically set the `output_jsn_full` parameter to `true` when `-o` or `--output` is used.

Overall, the program appears to be well-structured and easy to use.


```cpp
void whisper_print_usage(int argc, char ** argv, const whisper_params & params);

bool whisper_params_parse(int argc, char ** argv, whisper_params & params) {
    for (int i = 1; i < argc; i++) {
        std::string arg = argv[i];

        if (arg == "-"){
            params.fname_inp.push_back(arg);
            continue;
        }

        if (arg[0] != '-') {
            params.fname_inp.push_back(arg);
            continue;
        }

        if (arg == "-h" || arg == "--help") {
            whisper_print_usage(argc, argv, params);
            exit(0);
        }
        else if (arg == "-t"    || arg == "--threads")         { params.n_threads       = std::stoi(argv[++i]); }
        else if (arg == "-p"    || arg == "--processors")      { params.n_processors    = std::stoi(argv[++i]); }
        else if (arg == "-ot"   || arg == "--offset-t")        { params.offset_t_ms     = std::stoi(argv[++i]); }
        else if (arg == "-on"   || arg == "--offset-n")        { params.offset_n        = std::stoi(argv[++i]); }
        else if (arg == "-d"    || arg == "--duration")        { params.duration_ms     = std::stoi(argv[++i]); }
        else if (arg == "-mc"   || arg == "--max-context")     { params.max_context     = std::stoi(argv[++i]); }
        else if (arg == "-ml"   || arg == "--max-len")         { params.max_len         = std::stoi(argv[++i]); }
        else if (arg == "-bo"   || arg == "--best-of")         { params.best_of         = std::stoi(argv[++i]); }
        else if (arg == "-bs"   || arg == "--beam-size")       { params.beam_size       = std::stoi(argv[++i]); }
        else if (arg == "-wt"   || arg == "--word-thold")      { params.word_thold      = std::stof(argv[++i]); }
        else if (arg == "-et"   || arg == "--entropy-thold")   { params.entropy_thold   = std::stof(argv[++i]); }
        else if (arg == "-lpt"  || arg == "--logprob-thold")   { params.logprob_thold   = std::stof(argv[++i]); }
        // else if (arg == "-su"   || arg == "--speed-up")        { params.speed_up        = true; }
        else if (arg == "-debug"|| arg == "--debug-mode")      { params.debug_mode      = true; }
        else if (arg == "-tr"   || arg == "--translate")       { params.translate       = true; }
        else if (arg == "-di"   || arg == "--diarize")         { params.diarize         = true; }
        else if (arg == "-tdrz" || arg == "--tinydiarize")     { params.tinydiarize     = true; }
        else if (arg == "-sow"  || arg == "--split-on-word")   { params.split_on_word   = true; }
        else if (arg == "-nf"   || arg == "--no-fallback")     { params.no_fallback     = true; }
        else if (arg == "-otxt" || arg == "--output-txt")      { params.output_txt      = true; }
        else if (arg == "-ovtt" || arg == "--output-vtt")      { params.output_vtt      = true; }
        else if (arg == "-osrt" || arg == "--output-srt")      { params.output_srt      = true; }
        else if (arg == "-owts" || arg == "--output-words")    { params.output_wts      = true; }
        else if (arg == "-olrc" || arg == "--output-lrc")      { params.output_lrc      = true; }
        else if (arg == "-fp"   || arg == "--font-path")       { params.font_path       = argv[++i]; }
        else if (arg == "-ocsv" || arg == "--output-csv")      { params.output_csv      = true; }
        else if (arg == "-oj"   || arg == "--output-json")     { params.output_jsn      = true; }
        else if (arg == "-ojf"  || arg == "--output-json-full"){ params.output_jsn_full = params.output_jsn = true; }
        else if (arg == "-of"   || arg == "--output-file")     { params.fname_out.emplace_back(argv[++i]); }
        else if (arg == "-ps"   || arg == "--print-special")   { params.print_special   = true; }
        else if (arg == "-pc"   || arg == "--print-colors")    { params.print_colors    = true; }
        else if (arg == "-pp"   || arg == "--print-progress")  { params.print_progress  = true; }
        else if (arg == "-nt"   || arg == "--no-timestamps")   { params.no_timestamps   = true; }
        else if (arg == "-l"    || arg == "--language")        { params.language        = argv[++i]; }
        else if (arg == "-dl"   || arg == "--detect-language") { params.detect_language = true; }
        else if (                  arg == "--prompt")          { params.prompt          = argv[++i]; }
        else if (arg == "-m"    || arg == "--model")           { params.model           = argv[++i]; }
        else if (arg == "-f"    || arg == "--file")            { params.fname_inp.emplace_back(argv[++i]); }
        else if (arg == "-oved" || arg == "--ov-e-device")     { params.openvino_encode_device = argv[++i]; }
        else if (arg == "-ls"   || arg == "--log-score")       { params.log_score = true; }
        else {
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            whisper_print_usage(argc, argv, params);
            exit(0);
        }
    }

    return true;
}

```

这是一段用于配置选项的C代码。它定义了一系列命令行选项以及它们的含义。这些选项包括：

-p, --print-percentage, 打印百分比信息。
-pc, --print-colors, 打印颜色信息。
-pp, --print-progress, 打印进度信息。
-nt, --no-timestamps, 不打印时间戳信息。
-l, --language, 指定所选的语言。
-dl, --detect-language, 在自动检测到其他语言时退出。
-prompt, --initial-prompt, 初始提示信息。
-m, --model, 指定训练的语音识别模型。
-f, --file, 指定输入的WAV文件路径。
-oved, --ov-e-device, 指定用于 encode inference 的OpenVINO 设备。
-ls, --log-score, 打印模型分数。

每当我们运行程序时，它将读取用户提供的选项，并根据这些选项加载对应的配置文件。


```cpp
void whisper_print_usage(int /*argc*/, char ** argv, const whisper_params & params) {
    fprintf(stderr, "\n");
    fprintf(stderr, "usage: %s [options] file0.wav file1.wav ...\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h,        --help              [default] show this help message and exit\n");
    fprintf(stderr, "  -t N,      --threads N         [%-7d] number of threads to use during computation\n",    params.n_threads);
    fprintf(stderr, "  -p N,      --processors N      [%-7d] number of processors to use during computation\n", params.n_processors);
    fprintf(stderr, "  -ot N,     --offset-t N        [%-7d] time offset in milliseconds\n",                    params.offset_t_ms);
    fprintf(stderr, "  -on N,     --offset-n N        [%-7d] segment index offset\n",                           params.offset_n);
    fprintf(stderr, "  -d  N,     --duration N        [%-7d] duration of audio to process in milliseconds\n",   params.duration_ms);
    fprintf(stderr, "  -mc N,     --max-context N     [%-7d] maximum number of text context tokens to store\n", params.max_context);
    fprintf(stderr, "  -ml N,     --max-len N         [%-7d] maximum segment length in characters\n",           params.max_len);
    fprintf(stderr, "  -sow,      --split-on-word     [%-7s] split on word rather than on token\n",             params.split_on_word ? "true" : "false");
    fprintf(stderr, "  -bo N,     --best-of N         [%-7d] number of best candidates to keep\n",              params.best_of);
    fprintf(stderr, "  -bs N,     --beam-size N       [%-7d] beam size for beam search\n",                      params.beam_size);
    fprintf(stderr, "  -wt N,     --word-thold N      [%-7.2f] word timestamp probability threshold\n",         params.word_thold);
    fprintf(stderr, "  -et N,     --entropy-thold N   [%-7.2f] entropy threshold for decoder fail\n",           params.entropy_thold);
    fprintf(stderr, "  -lpt N,    --logprob-thold N   [%-7.2f] log probability threshold for decoder fail\n",   params.logprob_thold);
    // fprintf(stderr, "  -su,       --speed-up          [%-7s] speed up audio by x2 (reduced accuracy)\n",        params.speed_up ? "true" : "false");
    fprintf(stderr, "  -debug,    --debug-mode        [%-7s] enable debug mode (eg. dump log_mel)\n",           params.debug_mode ? "true" : "false");
    fprintf(stderr, "  -tr,       --translate         [%-7s] translate from source language to english\n",      params.translate ? "true" : "false");
    fprintf(stderr, "  -di,       --diarize           [%-7s] stereo audio diarization\n",                       params.diarize ? "true" : "false");
    fprintf(stderr, "  -tdrz,     --tinydiarize       [%-7s] enable tinydiarize (requires a tdrz model)\n",     params.tinydiarize ? "true" : "false");
    fprintf(stderr, "  -nf,       --no-fallback       [%-7s] do not use temperature fallback while decoding\n", params.no_fallback ? "true" : "false");
    fprintf(stderr, "  -otxt,     --output-txt        [%-7s] output result in a text file\n",                   params.output_txt ? "true" : "false");
    fprintf(stderr, "  -ovtt,     --output-vtt        [%-7s] output result in a vtt file\n",                    params.output_vtt ? "true" : "false");
    fprintf(stderr, "  -osrt,     --output-srt        [%-7s] output result in a srt file\n",                    params.output_srt ? "true" : "false");
    fprintf(stderr, "  -olrc,     --output-lrc        [%-7s] output result in a lrc file\n",                    params.output_lrc ? "true" : "false");
    fprintf(stderr, "  -owts,     --output-words      [%-7s] output script for generating karaoke video\n",     params.output_wts ? "true" : "false");
    fprintf(stderr, "  -fp,       --font-path         [%-7s] path to a monospace font for karaoke video\n",     params.font_path.c_str());
    fprintf(stderr, "  -ocsv,     --output-csv        [%-7s] output result in a CSV file\n",                    params.output_csv ? "true" : "false");
    fprintf(stderr, "  -oj,       --output-json       [%-7s] output result in a JSON file\n",                   params.output_jsn ? "true" : "false");
    fprintf(stderr, "  -ojf,      --output-json-full  [%-7s] include more information in the JSON file\n",      params.output_jsn_full ? "true" : "false");
    fprintf(stderr, "  -of FNAME, --output-file FNAME [%-7s] output file path (without file extension)\n",      "");
    fprintf(stderr, "  -ps,       --print-special     [%-7s] print special tokens\n",                           params.print_special ? "true" : "false");
    fprintf(stderr, "  -pc,       --print-colors      [%-7s] print colors\n",                                   params.print_colors ? "true" : "false");
    fprintf(stderr, "  -pp,       --print-progress    [%-7s] print progress\n",                                 params.print_progress ? "true" : "false");
    fprintf(stderr, "  -nt,       --no-timestamps     [%-7s] do not print timestamps\n",                        params.no_timestamps ? "true" : "false");
    fprintf(stderr, "  -l LANG,   --language LANG     [%-7s] spoken language ('auto' for auto-detect)\n",       params.language.c_str());
    fprintf(stderr, "  -dl,       --detect-language   [%-7s] exit after automatically detecting language\n",    params.detect_language ? "true" : "false");
    fprintf(stderr, "             --prompt PROMPT     [%-7s] initial prompt\n",                                 params.prompt.c_str());
    fprintf(stderr, "  -m FNAME,  --model FNAME       [%-7s] model path\n",                                     params.model.c_str());
    fprintf(stderr, "  -f FNAME,  --file FNAME        [%-7s] input WAV file path\n",                            "");
    fprintf(stderr, "  -oved D,   --ov-e-device DNAME [%-7s] the OpenVINO device used for encode inference\n",  params.openvino_encode_device.c_str());
    fprintf(stderr, "  -ls,       --log-score         [%-7s] log best decoder scores of tokens\n",              params.log_score?"true":"false");
    fprintf(stderr, "\n");
}

```

这是一个 C++ 结构体，名为 `whisper_print_user_data`。这个结构体包含以下成员：

1. `params`，是一个指向 `whisper_params` 类型的指针。
2. `pcmf32s`，是一个指向 `std::vector<std::vector<float>>` 类型的指针。这个类型可能包含多个 32 位浮点数组，用于存储数据。
3. `progress_prev`，是一个整数类型，表示最近的进度。

函数 `estimate_diarization_speaker` 接受一个 `std::vector<std::vector<float>>` 类型的参数 `pcmf32s`，以及两个时间戳 `t0` 和 `t1`，布尔参数 `id_only` 是否仅输出说话人而不输出时间戳。函数返回说话人。

以下是 `estimate_diarization_speaker` 的实现细节：

1. 计算能量：对于每个时间戳 `j`，计算两个浮点数组 `energy0` 和 `energy1`。这里我们假设输入数据是 32 位浮点数，因此将绝对值计算出来。
2. 比较能量：计算两个浮点数的能量，如果 `energy0` 大于 1.1 倍的 `energy1`，则说话人是 "0"（静音），否则说话人是 "1"（发音）。
3. 输出说话人：如果 `id_only` 为真，则只输出说话人而不输出时间戳。否则，在说话人前加上 "(" 和 ")"，使其更易于阅读。
4. 去除背景噪声：对于说话人，去除背景噪声。由于没有具体实现，因此这里只是一个简单的示例，没有对数据进行预处理，也没有对背景噪声进行任何处理。


```cpp
struct whisper_print_user_data {
    const whisper_params * params;

    const std::vector<std::vector<float>> * pcmf32s;
    int progress_prev;
};

std::string estimate_diarization_speaker(std::vector<std::vector<float>> pcmf32s, int64_t t0, int64_t t1, bool id_only = false) {
    std::string speaker = "";
    const int64_t n_samples = pcmf32s[0].size();

    const int64_t is0 = timestamp_to_sample(t0, n_samples);
    const int64_t is1 = timestamp_to_sample(t1, n_samples);

    double energy0 = 0.0f;
    double energy1 = 0.0f;

    for (int64_t j = is0; j < is1; j++) {
        energy0 += fabs(pcmf32s[0][j]);
        energy1 += fabs(pcmf32s[1][j]);
    }

    if (energy0 > 1.1*energy1) {
        speaker = "0";
    } else if (energy1 > 1.1*energy0) {
        speaker = "1";
    } else {
        speaker = "?";
    }

    //printf("is0 = %lld, is1 = %lld, energy0 = %f, energy1 = %f, speaker = %s\n", is0, is1, energy0, energy1, speaker.c_str());

    if (!id_only) {
        speaker.insert(0, "(speaker ");
        speaker.append(")");
    }

    return speaker;
}
```

This is a C-like language program that appears to perform the task of processing and displaying NLP data. It appears to support a variety of parameters and options, such as the ability to change the font for displaying text, the font size, and whether to display the text in colors or not. It also appears to support the ability to perform timestamps or speaker diarization, and the option to print out the text on a new line.

The program first sets the塌陷-related parameters and then enters a loop that processes each segment of text in the NLP data. For each segment, the program performs various operations such as getting the token ID, text, and speaker ID, estimating the speaker's turn, and displaying the text.

If the program is set to perform timestamps, it will display the text with the timestamp for each segment on a new line. If the program is set to perform speaker diarization, it will display the text with the speaker's turn for each segment on a new line.


```cpp
void whisper_print_progress_callback(struct whisper_context * /*ctx*/, struct whisper_state * /*state*/, int progress, void * user_data) {
    int progress_step = ((whisper_print_user_data *) user_data)->params->progress_step;
    int * progress_prev  = &(((whisper_print_user_data *) user_data)->progress_prev);
    if (progress >= *progress_prev + progress_step) {
        *progress_prev += progress_step;
        fprintf(stderr, "%s: progress = %3d%%\n", __func__, progress);
    }
}

void whisper_print_segment_callback(struct whisper_context * ctx, struct whisper_state * /*state*/, int n_new, void * user_data) {
    const auto & params  = *((whisper_print_user_data *) user_data)->params;
    const auto & pcmf32s = *((whisper_print_user_data *) user_data)->pcmf32s;

    const int n_segments = whisper_full_n_segments(ctx);

    std::string speaker = "";

    int64_t t0 = 0;
    int64_t t1 = 0;

    // print the last n_new segments
    const int s0 = n_segments - n_new;

    if (s0 == 0) {
        printf("\n");
    }

    for (int i = s0; i < n_segments; i++) {
        if (!params.no_timestamps || params.diarize) {
            t0 = whisper_full_get_segment_t0(ctx, i);
            t1 = whisper_full_get_segment_t1(ctx, i);
        }

        if (!params.no_timestamps) {
            printf("[%s --> %s]  ", to_timestamp(t0).c_str(), to_timestamp(t1).c_str());
        }

        if (params.diarize && pcmf32s.size() == 2) {
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        if (params.print_colors) {
            for (int j = 0; j < whisper_full_n_tokens(ctx, i); ++j) {
                if (params.print_special == false) {
                    const whisper_token id = whisper_full_get_token_id(ctx, i, j);
                    if (id >= whisper_token_eot(ctx)) {
                        continue;
                    }
                }

                const char * text = whisper_full_get_token_text(ctx, i, j);
                const float  p    = whisper_full_get_token_p   (ctx, i, j);

                const int col = std::max(0, std::min((int) k_colors.size() - 1, (int) (std::pow(p, 3)*float(k_colors.size()))));

                printf("%s%s%s%s", speaker.c_str(), k_colors[col].c_str(), text, "\033[0m");
            }
        } else {
            const char * text = whisper_full_get_segment_text(ctx, i);

            printf("%s%s", speaker.c_str(), text);
        }

        if (params.tinydiarize) {
            if (whisper_full_get_segment_speaker_turn_next(ctx, i)) {
                printf("%s", params.tdrz_speaker_turn.c_str());
            }
        }

        // with timestamps or speakers: each segment on new line
        if (!params.no_timestamps || params.diarize) {
            printf("\n");
        }

        fflush(stdout);
    }
}

```

该函数的作用是创建并输出一个文本文件，该文件包含由 Whisper SDK 中的 whisper_context 和 whisper_params 结构体定义的音频数据。

函数的实现包括以下步骤：

1. 打开输出文件，如果失败则输出错误信息并返回 false。
2. 输出一条消息说明要保存的文件名。
3. 读取 Whisper SDK 中 speakers 的结构体并将它们存储到变量 speaker 中。
4. 通过调用 Whisper SDK 中 whisper_full_n_segments 和 whisper_full_get_segment_text 函数，读取完整的音频数据和每个子段落的文本，并将它们存储到 std::vector<std::vector<float>> pcmf32s 容器中。
5. 通过循环和 Whisper SDK中 whisper_full_get_segment_t0 和 whisper_full_get_segment_t1 函数，确定存储在 pcmf32s 中的音频数据中的说话者的 Diarization。
6. 通过循环和 Whisper SDK中的 ffmpeg 函数，将 speaker 和文本写入到输出文件中。
7. 返回 true，表示函数成功执行并成功保存了输出文件。


```cpp
bool output_txt(struct whisper_context * ctx, const char * fname, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    std::ofstream fout(fname);
    if (!fout.is_open()) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        return false;
    }

    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    const int n_segments = whisper_full_n_segments(ctx);
    for (int i = 0; i < n_segments; ++i) {
        const char * text = whisper_full_get_segment_text(ctx, i);
        std::string speaker = "";

        if (params.diarize && pcmf32s.size() == 2)
        {
            const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
            const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        fout << speaker << text << "\n";
    }

    return true;
}

```

该函数的作用是输出一个名为“WEBVTT”的文本格式，并保存到指定文件中。它主要实现了以下几个步骤：

1. 打开输出文件并输出预置信息。
2. 读取 Whisper 中的文本数据，并按给定的分段和参数进行相应的处理。
3. 在文本中插入说话人信息。
4. 将文本数据按时间戳顺序保存到文件中。

它依赖于一个 Whisper 上下文结构体以及一个 Whisper 参数对象，并使用 estimate_diarization_speaker 函数对说话人进行估计。


```cpp
bool output_vtt(struct whisper_context * ctx, const char * fname, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    std::ofstream fout(fname);
    if (!fout.is_open()) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        return false;
    }

    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    fout << "WEBVTT\n\n";

    const int n_segments = whisper_full_n_segments(ctx);
    for (int i = 0; i < n_segments; ++i) {
        const char * text = whisper_full_get_segment_text(ctx, i);
        const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
        const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
        std::string speaker = "";

        if (params.diarize && pcmf32s.size() == 2)
        {
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1, true);
            speaker.insert(0, "<v Speaker");
            speaker.append(">");
        }

        fout << to_timestamp(t0) << " --> " << to_timestamp(t1) << "\n";
        fout << speaker << text << "\n\n";
    }

    return true;
}

```

该函数的作用是输出一个SRT文本文件，其中包含由传入的whisper_context结构体中的文本内容。以下是函数的详细解释：

1. 函数参数说明：
  -   const char * fname: 输出文件的文件名，包括扩展名。
  -   const whisper_params & params: 包含whisper_params结构体的参数，用于控制输出的格式和语义。
  -   std::vector<std::vector<float>> pcmf32s: 存储PCM数据向量的向量。

2. 函数实现：
  - 如果文件无法打开，则输出错误信息并返回false。
  - 打开文件并输出当前时间戳。
  - 读取输入文本中的每个文本段落，并将其存储在向量pcmf32s中。
  - 如果whisper_params中 Diarize参数为true且pcmf32s大小为2，则使用估计的说话者来连接文本段落。
  - 循环输出每个文本段落的编号，并将其存储在向量pcmf32s中。

3. 函数提醒：
  - 在函数头部，使用fprintf函数输出一条消息，指明正在执行的操作。
  - 在函数内部，使用fprintf函数输出错误信息，以便在程序中捕获并处理任何错误。


```cpp
bool output_srt(struct whisper_context * ctx, const char * fname, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    std::ofstream fout(fname);
    if (!fout.is_open()) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        return false;
    }

    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    const int n_segments = whisper_full_n_segments(ctx);
    for (int i = 0; i < n_segments; ++i) {
        const char * text = whisper_full_get_segment_text(ctx, i);
        const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
        const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
        std::string speaker = "";

        if (params.diarize && pcmf32s.size() == 2)
        {
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        fout << i + 1 + params.offset_n << "\n";
        fout << to_timestamp(t0, true) << " --> " << to_timestamp(t1, true) << "\n";
        fout << speaker << text << "\n\n";
    }

    return true;
}

```

这段代码的目的是对传入的字符串进行转义，将其中的双引号和反斜杠转换为实际字符。

具体来说，代码首先检查传入的字符串是否为空，如果是，则直接返回一个空字符串。然后，对于字符串中的每个字符，判断它是否为双引号或反斜杠，如果是，就在escaped_length中加1。

接下来，代码根据双引号或反斜杠来遍历字符串，并将escaped_length加1，以便在需要时能够将转义字符串正确复制回原始字符串中。

最后，代码使用calloc()函数来分配一个足够大的内存区域，用于存储转义后的字符。不过，在分配内存之前，需要先检查该内存区域是否已经被分配给其他的calloc()函数。如果没有，则说明已经内存泄漏，程序可能会崩溃或泄漏内存。


```cpp
char *escape_double_quotes_and_backslashes(const char *str) {
    if (str == NULL) {
        return NULL;
    }

    size_t escaped_length = strlen(str) + 1;

    for (size_t i = 0; str[i] != '\0'; i++) {
        if (str[i] == '"' || str[i] == '\\') {
            escaped_length++;
        }
    }

    char *escaped = (char *)calloc(escaped_length, 1); // pre-zeroed
    if (escaped == NULL) {
        return NULL;
    }

    size_t pos = 0;
    for (size_t i = 0; str[i] != '\0'; i++) {
        if (str[i] == '"' || str[i] == '\\') {
            escaped[pos++] = '\\';
        }
        escaped[pos++] = str[i];
    }

    // no need to set zero due to calloc() being used prior

    return escaped;
}

```

这是一个C++函数，名为`output_csv`，它用于将whisper SDK中的音频数据保存为CSV文件。它需要三个参数：`ctx`表示whisper SDK的上下文，`fname`是要保存的CSV文件的文件名，`params`是存储在`whisper_params`中的whisper参数，`pcmf32s`是一个存储在内存中的2D PCM音频数据。

函数首先创建一个输出文件，如果失败的话会输出错误信息。然后输出要保存的CSV文件的格式，包括起始时间、结束时间、说话人和文本内容。接着循环包含在whisper SDK中的每个音频分段，并提取出每个分段的文本内容。对于每个分段，函数会先将文本内容进行 escape\_double\_quotes\_and\_backslashes() 处理，然后将其输出到文件中。在循环的过程中，函数会根据`params.diarize`的值决定是否将每个文本内容进行声学建模，并使用`estimate_diarization_speaker()`函数来估计说话人的声学模型。


```cpp
bool output_csv(struct whisper_context * ctx, const char * fname, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    std::ofstream fout(fname);
    if (!fout.is_open()) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        return false;
    }

    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    const int n_segments = whisper_full_n_segments(ctx);
    fout << "start,end,";
    if (params.diarize && pcmf32s.size() == 2)
    {
        fout << "speaker,";
    }
    fout << "text\n";

    for (int i = 0; i < n_segments; ++i) {
        const char * text = whisper_full_get_segment_text(ctx, i);
        const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
        const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
        char * text_escaped = escape_double_quotes_and_backslashes(text);

        //need to multiply times returned from whisper_full_get_segment_t{0,1}() by 10 to get milliseconds.
        fout << 10 * t0 << "," << 10 * t1 << ",";
        if (params.diarize && pcmf32s.size() == 2)
        {
            fout << estimate_diarization_speaker(pcmf32s, t0, t1, true) << ",";
        }
        fout << "\"" << text_escaped << "\"\n";
    }

    return true;
}

```

该函数的作用是读取一个后缀文件的输出分数，并输出到指定的后缀文件中。函数接收一个whisper_context结构体、一个文件名和一个whisper_params结构体作为参数。函数内部通过std::ofstream fout对象写入到文件中，每次写入一个整段落，并包含一个格式字符串，其中%s表示要保存的文件名。在格式字符串中，使用%d表示要保存的整数，得到整数n_segments，即要保存的段落数。然后使用for循环，在每一段落中，使用whisper_full_n_tokens函数获取该段落中的词数，使用whisper_full_get_token_p函数获取该段落对应的概率，并将获取到的token文本和概率写入到输出文件中，其中token文本使用%s表示，概率使用%f表示。函数的返回值是true，表示函数执行成功。


```cpp
bool output_score(struct whisper_context * ctx, const char * fname, const whisper_params & /*params*/, std::vector<std::vector<float>> /*pcmf32s*/) {
    std::ofstream fout(fname);
    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    const int n_segments = whisper_full_n_segments(ctx);
    // fprintf(stderr,"segments: %d\n",n_segments);
    for (int i = 0; i < n_segments; ++i) {
        const int n_tokens = whisper_full_n_tokens(ctx, i);
        // fprintf(stderr,"tokens: %d\n",n_tokens);
        for (int j = 0; j < n_tokens; j++) {
            auto token = whisper_full_get_token_text(ctx, i, j);
            auto probability = whisper_full_get_token_p(ctx, i, j);
            fout << token << '\t' << probability << std::endl;
            // fprintf(stderr,"token: %s %f\n",token,probability);
	    }
    }
    return true;
}

```

This function appears to be a part of a larger program that performs text-to-speech synthesis. It takes a segmentation object as input and outputs a synthesized speech.

The function takes an integer `i` and outputs a segmented speech by first getting the segmentation object `i` and then using it to get a set of tokenized words. It then loops through each token and performs various operations on them, such as converting the token ID to a string, getting the start and end time of the token, and writing out the timestamps if `params.diarize` is true.

If `params.diarize` is true, it also performs a simple speaker turn operation. If `params.tinydiarize` is true, it also performs a speaker turn operation that turns the speaker to the next segment.

It ends with a return statement indicating whether the function completed successfully or not.


```cpp
bool output_json(
             struct whisper_context * ctx,
                         const char * fname,
               const whisper_params & params,
    std::vector<std::vector<float>>   pcmf32s,
                               bool   full) {
    std::ofstream fout(fname);
    int indent = 0;

    auto doindent = [&]() {
        for (int i = 0; i < indent; i++) fout << "\t";
    };

    auto start_arr = [&](const char *name) {
        doindent();
        fout << "\"" << name << "\": [\n";
        indent++;
    };

    auto end_arr = [&](bool end) {
        indent--;
        doindent();
        fout << (end ? "]\n" : "],\n");
    };

    auto start_obj = [&](const char *name) {
        doindent();
        if (name) {
            fout << "\"" << name << "\": {\n";
        } else {
            fout << "{\n";
        }
        indent++;
    };

    auto end_obj = [&](bool end) {
        indent--;
        doindent();
        fout << (end ? "}\n" : "},\n");
    };

    auto start_value = [&](const char *name) {
        doindent();
        fout << "\"" << name << "\": ";
    };

    auto value_s = [&](const char *name, const char *val, bool end) {
        start_value(name);
        char * val_escaped = escape_double_quotes_and_backslashes(val);
        fout << "\"" << val_escaped << (end ? "\"\n" : "\",\n");
        free(val_escaped);
    };

    auto end_value = [&](bool end) {
        fout << (end ? "\n" : ",\n");
    };

    auto value_i = [&](const char *name, const int64_t val, bool end) {
        start_value(name);
        fout << val;
        end_value(end);
    };

    auto value_f = [&](const char *name, const float val, bool end) {
        start_value(name);
        fout << val;
        end_value(end);
    };

    auto value_b = [&](const char *name, const bool val, bool end) {
        start_value(name);
        fout << (val ? "true" : "false");
        end_value(end);
    };

    auto times_o = [&](int64_t t0, int64_t t1, bool end) {
        start_obj("timestamps");
        value_s("from", to_timestamp(t0, true).c_str(), false);
        value_s("to", to_timestamp(t1, true).c_str(), true);
        end_obj(false);
        start_obj("offsets");
        value_i("from", t0 * 10, false);
        value_i("to", t1 * 10, true);
        end_obj(end);
    };

    if (!fout.is_open()) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        return false;
    }

    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);
    start_obj(nullptr);
        value_s("systeminfo", whisper_print_system_info(), false);
        start_obj("model");
            value_s("type", whisper_model_type_readable(ctx), false);
            value_b("multilingual", whisper_is_multilingual(ctx), false);
            value_i("vocab", whisper_model_n_vocab(ctx), false);
            start_obj("audio");
                value_i("ctx", whisper_model_n_audio_ctx(ctx), false);
                value_i("state", whisper_model_n_audio_state(ctx), false);
                value_i("head", whisper_model_n_audio_head(ctx), false);
                value_i("layer", whisper_model_n_audio_layer(ctx), true);
            end_obj(false);
            start_obj("text");
                value_i("ctx", whisper_model_n_text_ctx(ctx), false);
                value_i("state", whisper_model_n_text_state(ctx), false);
                value_i("head", whisper_model_n_text_head(ctx), false);
                value_i("layer", whisper_model_n_text_layer(ctx), true);
            end_obj(false);
            value_i("mels", whisper_model_n_mels(ctx), false);
            value_i("ftype", whisper_model_ftype(ctx), true);
        end_obj(false);
        start_obj("params");
            value_s("model", params.model.c_str(), false);
            value_s("language", params.language.c_str(), false);
            value_b("translate", params.translate, true);
        end_obj(false);
        start_obj("result");
            value_s("language", whisper_lang_str(whisper_full_lang_id(ctx)), true);
        end_obj(false);
        start_arr("transcription");

            const int n_segments = whisper_full_n_segments(ctx);
            for (int i = 0; i < n_segments; ++i) {
                const char * text = whisper_full_get_segment_text(ctx, i);

                const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
                const int64_t t1 = whisper_full_get_segment_t1(ctx, i);

                start_obj(nullptr);
                    times_o(t0, t1, false);
                    value_s("text", text, !params.diarize && !params.tinydiarize && !full);

                    if (full) {
                        start_arr("tokens");
                        const int n = whisper_full_n_tokens(ctx, i);
                        for (int j = 0; j < n; ++j) {
                            auto token = whisper_full_get_token_data(ctx, i, j);
                            start_obj(nullptr);
                                value_s("text", whisper_token_to_str(ctx, token.id), false);
                                if(token.t0 > -1 && token.t1 > -1) {
                                    // If we have per-token timestamps, write them out
                                    times_o(token.t0, token.t1, false);
                                }
                                value_i("id", token.id, false);
                                value_f("p", token.p, true);
                            end_obj(j == (n - 1));
                        }
                        end_arr(!params.diarize && !params.tinydiarize);
                    }

                    if (params.diarize && pcmf32s.size() == 2) {
                        value_s("speaker", estimate_diarization_speaker(pcmf32s, t0, t1, true).c_str(), true);
                    }

                    if (params.tinydiarize) {
                        value_b("speaker_turn_next", whisper_full_get_segment_speaker_turn_next(ctx, i), true);
                    }
                end_obj(i == (n_segments - 1));
            }

        end_arr(true);
    end_obj(true);
    return true;
}

```

This is a C language function that generates karaoke text for an input video file. The text is displayed in the foreground and underlined for better readability.

The function takes two arguments: a text background color and a text font. The text font is set to use a specific font, and the font size and color are set as well. The text background color is set to a specified range of the width and height of the video frame, and is combined with the text color to create the final look.

The function first checks if the first text already exists in the video file, and if it does, it adds the text to the background and font color. If the first text does not exist, it adds the text to the background and font color, and then combines it with the text color to create the final look.

After that, the function generates the text for each frame of the video file, combines them together, and saves the resulting video file.

Finally, the function prints some information about the video file, such as the name of the output file and the information about the video quality.


```cpp
// karaoke video generation
// outputs a bash script that uses ffmpeg to generate a video with the subtitles
// TODO: font parameter adjustments
bool output_wts(struct whisper_context * ctx, const char * fname, const char * fname_inp, const whisper_params & params, float t_sec, std::vector<std::vector<float>> pcmf32s) {
    std::ofstream fout(fname);

    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    static const char * font = params.font_path.c_str();

    std::ifstream fin(font);
    if (!fin.is_open()) {
        fprintf(stderr, "%s: font not found at '%s', please specify a monospace font with -fp\n", __func__, font);
        return false;
    }

    fout << "#!/bin/bash" << "\n";
    fout << "\n";

    fout << "ffmpeg -i " << fname_inp << " -f lavfi -i color=size=1200x120:duration=" << t_sec << ":rate=25:color=black -vf \"";

    for (int i = 0; i < whisper_full_n_segments(ctx); i++) {
        const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
        const int64_t t1 = whisper_full_get_segment_t1(ctx, i);

        const int n = whisper_full_n_tokens(ctx, i);

        std::vector<whisper_token_data> tokens(n);
        for (int j = 0; j < n; ++j) {
            tokens[j] = whisper_full_get_token_data(ctx, i, j);
        }

        if (i > 0) {
            fout << ",";
        }

        // background text
        fout << "drawtext=fontfile='" << font << "':fontsize=24:fontcolor=gray:x=(w-text_w)/2:y=h/2:text='':enable='between(t," << t0/100.0 << "," << t0/100.0 << ")'";

        bool is_first = true;
        std::string speaker = "";

        if (params.diarize && pcmf32s.size() == 2) {
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        for (int j = 0; j < n; ++j) {
            const auto & token = tokens[j];

            if (tokens[j].id >= whisper_token_eot(ctx)) {
                continue;
            }

            std::string txt_bg = "";
            std::string txt_fg = ""; // highlight token
            std::string txt_ul = ""; // underline

            if (params.diarize && pcmf32s.size() == 2) {
                txt_bg = speaker;
                txt_fg = speaker;
                txt_ul = "\\ \\ \\ \\ \\ \\ \\ \\ \\ \\ \\ ";
            }

            txt_bg.append("> ");
            txt_fg.append("> ");
            txt_ul.append("\\ \\ ");

            {
                for (int k = 0; k < n; ++k) {
                    const auto & token2 = tokens[k];

                    if (tokens[k].id >= whisper_token_eot(ctx)) {
                        continue;
                    }

                    const std::string txt = whisper_token_to_str(ctx, token2.id);

                    txt_bg += txt;

                    if (k == j) {
                        for (int l = 0; l < (int) txt.size(); ++l) {
                            txt_fg += txt[l];
                            txt_ul += "_";
                        }
                        txt_fg += "|";
                    } else {
                        for (int l = 0; l < (int) txt.size(); ++l) {
                            txt_fg += "\\ ";
                            txt_ul += "\\ ";
                        }
                    }
                }

                ::replace_all(txt_bg, "'", "\u2019");
                ::replace_all(txt_bg, "\"", "\\\"");
                ::replace_all(txt_fg, "'", "\u2019");
                ::replace_all(txt_fg, "\"", "\\\"");
            }

            if (is_first) {
                // background text
                fout << ",drawtext=fontfile='" << font << "':fontsize=24:fontcolor=gray:x=(w-text_w)/2:y=h/2:text='" << txt_bg << "':enable='between(t," << t0/100.0 << "," << t1/100.0 << ")'";
                is_first = false;
            }

            // foreground text
            fout << ",drawtext=fontfile='" << font << "':fontsize=24:fontcolor=lightgreen:x=(w-text_w)/2+8:y=h/2:text='" << txt_fg << "':enable='between(t," << token.t0/100.0 << "," << token.t1/100.0 << ")'";

            // underline
            fout << ",drawtext=fontfile='" << font << "':fontsize=24:fontcolor=lightgreen:x=(w-text_w)/2+8:y=h/2+16:text='" << txt_ul << "':enable='between(t," << token.t0/100.0 << "," << token.t1/100.0 << ")'";
        }
    }

    fout << "\" -c:v libx264 -pix_fmt yuv420p -y " << fname_inp << ".mp4" << "\n";

    fout << "\n\n";
    fout << "echo \"Your video has been saved to " << fname_inp << ".mp4\"" << "\n";
    fout << "\n";
    fout << "echo \"  ffplay " << fname_inp << ".mp4\"\n";
    fout << "\n";

    fout.close();

    fprintf(stderr, "%s: run 'source %s' to generate karaoke video\n", __func__, fname);

    return true;
}

```

This is a C++ program that takes output from a research diary and outputs segments of the diary as CSV files. It uses the Whisper library to parse the diary data and estimate the speaker. The program takes command-line options to specify the input and output diaries, the output format, and other parameters.

Here is an example usage of the program:
```cppcss
$ python3 whisper_script.py -h -r -o output_file.csv input_file.csv --diarize true --speaker_estimation true --num_threads 4
```
This command will read the input diaries and estimate the speaker for each segment and save the output to a CSV file named output_file.csv and input_file.csv. The output format is a CSV file with columns for the date and time of each segment, the speaker, and the segment text. The program also takes an optional flag `--diarize` to enable the diarization speaker estimation.

The program uses the Whisper library to parse the diary data and estimate the speaker. The `estimate_diarization_speaker` function in the Whisper library is used to estimate the speaker for each segment. The `snprintf` function in the C++ `std::string` class is used to format the timestamp string.


```cpp
bool output_lrc(struct whisper_context * ctx, const char * fname, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    std::ofstream fout(fname);
    if (!fout.is_open()) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        return false;
    }

    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    fout << "[by:whisper.cpp]\n";

    const int n_segments = whisper_full_n_segments(ctx);
    for (int i = 0; i < n_segments; ++i) {
        const char * text = whisper_full_get_segment_text(ctx, i);
        const int64_t t = whisper_full_get_segment_t0(ctx, i);

        int64_t msec = t * 10;
        int64_t min = msec / (1000 * 60);
        msec = msec - min * (1000 * 60);
        int64_t sec = msec / 1000;
        msec = msec - sec * 1000;

        char buf[16];
        snprintf(buf, sizeof(buf), "%02d:%02d.%02d", (int) min, (int) sec, (int) ( msec / 10));
        std::string timestamp_lrc = std::string(buf);
        std::string speaker = "";

        if (params.diarize && pcmf32s.size() == 2)
        {
            const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
            const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        fout <<  '[' << timestamp_lrc << ']' << speaker << text << "\n";
    }

    return true;
}

```

This is a C++ program that generates a score for a music piece based on the input parameters. It uses the Whisper library to output the score to different file formats, such asSRT, WTS, CSV, JSON, and LRC. It also outputs the score to a score file if the `output_score` parameter is set to `true`.

The program takes an input score file (which can be specified by the user) and a set of output file formats. It then reads the input score and outputs it to the specified format. If the `output_score` parameter is set to `true`, it outputs the score to a score file.

The program includes a number of helper functions for generating the output in different formats, such as `output_srt`, `output_wts`, `output_csv`, `output_json`, and `output_score`. These functions take in various parameters and return the output file name and format.

The program also includes a number of error handling functions for things like file not found or invalid input.

Overall, this program looks like a score generator for music pieces that can be easily customized to generate the output in different formats.


```cpp
int main(int argc, char ** argv) {
    whisper_params params;

    if (whisper_params_parse(argc, argv, params) == false) {
        whisper_print_usage(argc, argv, params);
        return 1;
    }

    if (params.fname_inp.empty()) {
        fprintf(stderr, "error: no input files specified\n");
        whisper_print_usage(argc, argv, params);
        return 2;
    }

    if (params.language != "auto" && whisper_lang_id(params.language.c_str()) == -1) {
        fprintf(stderr, "error: unknown language '%s'\n", params.language.c_str());
        whisper_print_usage(argc, argv, params);
        exit(0);
    }

    if (params.diarize && params.tinydiarize) {
        fprintf(stderr, "error: cannot use both --diarize and --tinydiarize\n");
        whisper_print_usage(argc, argv, params);
        exit(0);
    }

    // whisper init

    struct whisper_context * ctx = whisper_init_from_file(params.model.c_str());

    if (ctx == nullptr) {
        fprintf(stderr, "error: failed to initialize whisper context\n");
        return 3;
    }

    // initialize openvino encoder. this has no effect on whisper.cpp builds that don't have OpenVINO configured
    whisper_ctx_init_openvino_encoder(ctx, nullptr, params.openvino_encode_device.c_str(), nullptr);

    for (int f = 0; f < (int) params.fname_inp.size(); ++f) {
        const auto fname_inp = params.fname_inp[f];
		const auto fname_out = f < (int) params.fname_out.size() && !params.fname_out[f].empty() ? params.fname_out[f] : params.fname_inp[f];

        std::vector<float> pcmf32;               // mono-channel F32 PCM
        std::vector<std::vector<float>> pcmf32s; // stereo-channel F32 PCM

        if (!::read_wav(fname_inp, pcmf32, pcmf32s, params.diarize)) {
            fprintf(stderr, "error: failed to read WAV file '%s'\n", fname_inp.c_str());
            continue;
        }

        // print system information
        {
            fprintf(stderr, "\n");
            fprintf(stderr, "system_info: n_threads = %d / %d | %s\n",
                    params.n_threads*params.n_processors, std::thread::hardware_concurrency(), whisper_print_system_info());
        }

        // print some info about the processing
        {
            fprintf(stderr, "\n");
            if (!whisper_is_multilingual(ctx)) {
                if (params.language != "en" || params.translate) {
                    params.language = "en";
                    params.translate = false;
                    fprintf(stderr, "%s: WARNING: model is not multilingual, ignoring language and translation options\n", __func__);
                }
            }
            if (params.detect_language) {
                params.language = "auto";
            }
            fprintf(stderr, "%s: processing '%s' (%d samples, %.1f sec), %d threads, %d processors, lang = %s, task = %s, %stimestamps = %d ...\n",
                    __func__, fname_inp.c_str(), int(pcmf32.size()), float(pcmf32.size())/WHISPER_SAMPLE_RATE,
                    params.n_threads, params.n_processors,
                    params.language.c_str(),
                    params.translate ? "translate" : "transcribe",
                    params.tinydiarize ? "tdrz = 1, " : "",
                    params.no_timestamps ? 0 : 1);

            fprintf(stderr, "\n");
        }

        // run the inference
        {
            whisper_full_params wparams = whisper_full_default_params(WHISPER_SAMPLING_GREEDY);

            wparams.strategy = params.beam_size > 1 ? WHISPER_SAMPLING_BEAM_SEARCH : WHISPER_SAMPLING_GREEDY;

            wparams.print_realtime   = false;
            wparams.print_progress   = params.print_progress;
            wparams.print_timestamps = !params.no_timestamps;
            wparams.print_special    = params.print_special;
            wparams.translate        = params.translate;
            wparams.language         = params.language.c_str();
            wparams.detect_language  = params.detect_language;
            wparams.n_threads        = params.n_threads;
            wparams.n_max_text_ctx   = params.max_context >= 0 ? params.max_context : wparams.n_max_text_ctx;
            wparams.offset_ms        = params.offset_t_ms;
            wparams.duration_ms      = params.duration_ms;

            wparams.token_timestamps = params.output_wts || params.output_jsn_full || params.max_len > 0;
            wparams.thold_pt         = params.word_thold;
            wparams.max_len          = params.output_wts && params.max_len == 0 ? 60 : params.max_len;
            wparams.split_on_word    = params.split_on_word;

            wparams.speed_up         = params.speed_up;
            wparams.debug_mode       = params.debug_mode;

            wparams.tdrz_enable      = params.tinydiarize; // [TDRZ]

            wparams.initial_prompt   = params.prompt.c_str();

            wparams.greedy.best_of        = params.best_of;
            wparams.beam_search.beam_size = params.beam_size;

            wparams.temperature_inc  = params.no_fallback ? 0.0f : wparams.temperature_inc;
            wparams.entropy_thold    = params.entropy_thold;
            wparams.logprob_thold    = params.logprob_thold;

            whisper_print_user_data user_data = { &params, &pcmf32s, 0 };

            // this callback is called on each new segment
            if (!wparams.print_realtime) {
                wparams.new_segment_callback           = whisper_print_segment_callback;
                wparams.new_segment_callback_user_data = &user_data;
            }

            if (wparams.print_progress) {
                wparams.progress_callback           = whisper_print_progress_callback;
                wparams.progress_callback_user_data = &user_data;
            }

            // examples for abort mechanism
            // in examples below, we do not abort the processing, but we could if the flag is set to true

            // the callback is called before every encoder run - if it returns false, the processing is aborted
            {
                static bool is_aborted = false; // NOTE: this should be atomic to avoid data race

                wparams.encoder_begin_callback = [](struct whisper_context * /*ctx*/, struct whisper_state * /*state*/, void * user_data) {
                    bool is_aborted = *(bool*)user_data;
                    return !is_aborted;
                };
                wparams.encoder_begin_callback_user_data = &is_aborted;
            }

            // the callback is called before every computation - if it returns true, the computation is aborted
            {
                static bool is_aborted = false; // NOTE: this should be atomic to avoid data race

                wparams.abort_callback = [](void * user_data) {
                    bool is_aborted = *(bool*)user_data;
                    return is_aborted;
                };
                wparams.abort_callback_user_data = &is_aborted;
            }

            if (whisper_full_parallel(ctx, wparams, pcmf32.data(), pcmf32.size(), params.n_processors) != 0) {
                fprintf(stderr, "%s: failed to process audio\n", argv[0]);
                return 10;
            }
        }

        // output stuff
        {
            printf("\n");

            // output to text file
            if (params.output_txt) {
                const auto fname_txt = fname_out + ".txt";
                output_txt(ctx, fname_txt.c_str(), params, pcmf32s);
            }

            // output to VTT file
            if (params.output_vtt) {
                const auto fname_vtt = fname_out + ".vtt";
                output_vtt(ctx, fname_vtt.c_str(), params, pcmf32s);
            }

            // output to SRT file
            if (params.output_srt) {
                const auto fname_srt = fname_out + ".srt";
                output_srt(ctx, fname_srt.c_str(), params, pcmf32s);
            }

            // output to WTS file
            if (params.output_wts) {
                const auto fname_wts = fname_out + ".wts";
                output_wts(ctx, fname_wts.c_str(), fname_inp.c_str(), params, float(pcmf32.size() + 1000)/WHISPER_SAMPLE_RATE, pcmf32s);
            }

            // output to CSV file
            if (params.output_csv) {
                const auto fname_csv = fname_out + ".csv";
                output_csv(ctx, fname_csv.c_str(), params, pcmf32s);
            }

            // output to JSON file
            if (params.output_jsn) {
                const auto fname_jsn = fname_out + ".json";
                output_json(ctx, fname_jsn.c_str(), params, pcmf32s, params.output_jsn_full);
            }

            // output to LRC file
            if (params.output_lrc) {
                const auto fname_lrc = fname_out + ".lrc";
                output_lrc(ctx, fname_lrc.c_str(), params, pcmf32s);
            }

            // output to score file
            if (params.log_score) {
                const auto fname_score = fname_out + ".score.txt";
                output_score(ctx, fname_score.c_str(), params, pcmf32s);
            }
        }
    }

    whisper_print_timings(ctx);
    whisper_free(ctx);

    return 0;
}

```