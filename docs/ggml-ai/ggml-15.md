# GGML源码解析 15

# SAM.cpp

Inference of Meta's [Segment Anything Model](https://github.com/facebookresearch/segment-anything/) in pure C/C++

## Description

The example currently supports only the [ViT-B SAM model checkpoint](https://huggingface.co/facebook/sam-vit-base).

## Next steps

- [X] Reduce memory usage by utilizing the new ggml-alloc
- [X] Remove redundant graph nodes
- [ ] Make inference faster
- [X] Fix the difference in output masks compared to the PyTorch implementation
- [X] Filter masks based on stability score
- [ ] Add support for user input
- [ ] Support F16 for heavy F32 ops
- [ ] Test quantization
- [X] Support bigger model checkpoints
- [ ] GPU support

## Quick start
```cppbash
git clone https://github.com/ggerganov/ggml
cd ggml

# Install Python dependencies
python3 -m pip install -r requirements.txt

# Download PTH model
wget -P examples/sam/ https://dl.fbaipublicfiles.com/segment_anything/sam_vit_b_01ec64.pth

# Convert PTH model to ggml
python examples/sam/convert-pth-to-ggml.py examples/sam/sam_vit_b_01ec64.pth examples/sam/ 1

# Build ggml + examples
mkdir build && cd build
cmake .. && make -j4

# run inference
./bin/sam -t 16 -i ../examples/sam/example.jpg -m ../examples/sam/ggml-model-f16.bin
```

## Downloading and converting the model checkpoints

You can download a [model checkpoint](https://github.com/facebookresearch/segment-anything/tree/main#model-checkpoints) and convert it to `ggml` format using the script `convert-pth-to-ggml.py`:

## Example output on M2 Ultra
```cpp
 $ ▶ make -j sam && time ./bin/sam -t 8 -i img.jpg
[ 28%] Built target common
[ 71%] Built target ggml
[100%] Built target sam
main: seed = 1693224265
main: loaded image 'img.jpg' (680 x 453)
sam_image_preprocess: scale = 0.664062
main: preprocessed image (1024 x 1024)
sam_model_load: loading model from 'models/sam-vit-b/ggml-model-f16.bin' - please wait ...
sam_model_load: n_enc_state      = 768
sam_model_load: n_enc_layer      = 12
sam_model_load: n_enc_head       = 12
sam_model_load: n_enc_out_chans  = 256
sam_model_load: n_pt_embd        = 4
sam_model_load: ftype            = 1
sam_model_load: qntvr            = 0
operator(): ggml ctx size = 202.32 MB
sam_model_load: ...................................... done
sam_model_load: model size =   185.05 MB / num tensors = 304
embd_img
dims: 64 64 256 1 f32
First & Last 10 elements:
-0.05117 -0.06408 -0.07154 -0.06991 -0.07212 -0.07690 -0.07508 -0.07281 -0.07383 -0.06779
0.01589 0.01775 0.02250 0.01675 0.01766 0.01661 0.01811 0.02051 0.02103 0.03382
sum:  12736.272313

Skipping mask 0 with iou 0.705935 below threshold 0.880000
Skipping mask 1 with iou 0.762136 below threshold 0.880000
Mask 2: iou = 0.947081, stability_score = 0.955437, bbox (371, 436), (144, 168)


main:     load time =    51.28 ms
main:    total time =  2047.49 ms

real	0m2.068s
user	0m16.343s
sys	0m0.214s
```

Input point is (414.375, 162.796875) (currently hardcoded)

Input image:

![llamas](https://user-images.githubusercontent.com/8558655/261301565-37b7bf4b-bf91-40cf-8ec1-1532316e1612.jpg)

Output mask (mask_out_2.png in build folder):

![mask_glasses](https://user-images.githubusercontent.com/8558655/263706800-47eeea30-1457-4c87-938b-8f11536c5aa7.png)

## References

- [ggml](https://github.com/ggerganov/ggml)
- [SAM](https://segment-anything.com/)
- [SAM demo](https://segment-anything.com/demo)


# `examples/starcoder/convert-hf-to-ggml.py`

这段代码的作用是將一個名為HF的模型轉換為ggml格式。HF模型是一個質量管制模型，ggml格式是一種用於描述地理空間信息系統（GIS）數據的格式。

具体來說，这段代码执行以下操作：

1. 导入必要的模块（包括System，struct，json，torch，numpy，re，os，argparse和transformers）。
2. 从argparse模块中获取输入参数。
3. 从transformers模块中获取AutoModelForCausalLM，AutoTokenizer，AutoModelForCausalLM和BloomForCausalLM对象。
4. 使用这些模块将HF模型转换为ggml格式。
5. 将转换后的ggml格式保存为文件。

值得注意的是，这段代码使用了一些第三方模块和第三方库，如transformers和PyTorch。如果您不想使用这些模块和库，您需要自行编写代码来完成任务。


```cpp
# Convert HF models to ggml format
#

import sys
import struct
import json
import torch
import numpy as np
import re
import os
import argparse

from transformers import AutoModelForCausalLM
from transformers import AutoTokenizer, AutoModelForCausalLM, AutoConfig, BloomForCausalLM

```

这段代码定义了一个名为 `bytes_to_unicode` 的函数，它将一个字节序列（即 UTF-8 编码）转换为相应的 Unicode 字符串列表。

函数接收一个字节序列（也就是一个字符串），并返回一个字典，其中键是 UTF-8 编码的字节，值是相应的 Unicode 字符串。

函数内部首先定义了一个包含一系列字节和对应 Unicode 字符的列表（也就是一个词法表），然后遍历这个列表，对于每个字节，如果该字节不在列表中，就将其添加到列表中，并将该列表的索引（也就是 Unicode 字符的数量）增加到 1。

接下来，函数内部将这个列表转换为一个字符表（也就是一个 Chr() 函数返回的结果），然后再遍历这个字符表，将其转换为相应的 Unicode 字符串列表。

最后，函数返回一个字典，其中键是 UTF-8 编码的字节，值是相应的 Unicode 字符串。


```cpp
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

这段代码定义了一个 `argparse.ArgumentParser` 对象，用于从命令行参数中读取用户输入的参数。

该 `argparse.ArgumentParser` 对象接受了以下参数：

- `description`：描述了该命令行工具栏程序的用途。
- `model_name_or_path`：指定模型名称或路径。该参数是命令行参数的必填字段。
- `--outfile`：指定写入的 GGML 文件的路径。该参数是命令行参数的必填字段，默认值为 `"ggml-model.bin"`。
- `--use_f32`：如果设置了此参数，则会将 GGML 文件保存为 fp32 浮点数，否则会保存为单精度浮点数。该参数是一个可选项，默认为 False。

在程序的其余部分中，该 `argparse.ArgumentParser` 对象接受了命令行参数并返回了一个 `Namespace` 对象，其中包含了所有读取的参数。这些参数可以用于指定模型、输出文件类型以及是否使用 16 位或 32 位浮点数。


```cpp
parser = argparse.ArgumentParser(description='Convert starcoder HF model to GGML')
parser.add_argument('model_name_or_path', type=str, help='Name of model on HF hub, or local model folder')
parser.add_argument('--outfile', type=str, default='ggml-model.bin', help='Path of GGML file to write.')
parser.add_argument('--use_f32', action="store_true", help='Save GGML file in fp32')

args = parser.parse_args()

# use 16-bit or 32-bit floats
use_f16 = not args.use_f32

fname_out = args.outfile
fname_dir = os.path.dirname(fname_out)
if fname_dir:
    os.makedirs(fname_dir, exist_ok=True)

```

这段代码的主要作用是加载一个预训练的语言模型（CausalLM），并将其保存为ggml格式。它主要包含以下步骤：

1. 加载参数：首先，它将加载用户提供的模型名称或路径，然后定义一个名为args.model_name_or_path的参数。

2. 加载预训练模型：使用用户提供的模型名称或路径，从Hugging Face库中加载预训练的模型，并将其保存到名为args.model_name_or_path的模型对象中。

3. 设置模型参数：通过从预训练模型中获取配置文件，设置模型的架构、训练设置等参数。

4. 保存模型配置：将模型配置对象保存到文件args.model_name_or_path_config.txt中，以便稍后加载。

5. 加载模型变量：通过从预训练模型中获取state_dict，加载模型的可导变量，包括词向量等。

6. 保存模型：将模型保存为ggml格式，文件名为args.model_name_or_path。


```cpp
print("Loading model: ", args.model_name_or_path)
tokenizer = AutoTokenizer.from_pretrained(args.model_name_or_path)
config = AutoConfig.from_pretrained(args.model_name_or_path, trust_remote_code=True)
hparams = config.to_dict()
model = AutoModelForCausalLM.from_pretrained(args.model_name_or_path, config=config, torch_dtype=torch.float16 if use_f16 else torch.float32, low_cpu_mem_usage=True, trust_remote_code=True, offload_state_dict=True)
print("Model loaded: ", args.model_name_or_path)

list_vars = model.state_dict()

encoder = tokenizer.vocab
# Add added_tokens (special tokens) to the encoder
encoder.update(tokenizer.get_added_vocab())
print(hparams)

print("Saving ggml model to: ", fname_out)
```

这段代码的作用是创建一个文件输出流（fout），打开一个名为fname_out的文件，并写入一个结构体数组ggml，ggml是一个二进制序列，由0x67676d6c组成。

接着，这段代码读取hparams["vocab_size"]，将其转换成int类型，并将其写入fout。

然后，这段代码读取hparams["n_positions"]，将其转换成int类型，并将其写入fout。

接下来，这段代码读取hparams["n_embd"]，将其转换成int类型，并将其写入fout。

然后，这段代码读取hparams["n_head"]，将其转换成int类型，并将其写入fout。

接着，这段代码读取hparams["n_layer"]，将其转换成int类型，并将其写入fout。

然后，这段代码读取使用的主张牌（use_f16），并将其写入fout。

接着，这段代码将bytes_to_unicode()函数中的bytes字节数组转换成unicode字符串，并将其写入fout。

然后，这段代码创建了两个字典byte_decoder，并将它们中的键值对分别写入fout。

最后，这段代码没有做其他事情，直接返回。


```cpp
fout = open(fname_out, "wb")

fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
vocab_size = hparams["vocab_size"]
fout.write(struct.pack("i", vocab_size))
# fout.write(struct.pack("i", len(encoder)))
fout.write(struct.pack("i", hparams["n_positions"]))
fout.write(struct.pack("i", hparams["n_embd"]))
fout.write(struct.pack("i", hparams["n_head"]))
fout.write(struct.pack("i", hparams["n_layer"]))
fout.write(struct.pack("i", use_f16))

byte_encoder = bytes_to_unicode()
byte_decoder = {v:k for k, v in byte_encoder.items()}

```

这段代码的主要作用是读取一个文件中的文本，并对文本进行分词处理，然后将分词后的文本写入到另一个文件中。

具体来说，代码首先读取一个名为 fout 的文件，并定义一个变量 counter 用于跟踪已读取的文本行数。接着，代码使用 sorted 函数对传入的 encoder 对象进行排序，根据键的自然顺序(即字典中键的顺序)进行排序。

在循环中，代码遍历每个已经排好序的键(即已经分好词的文本行)，并将该键对应的 bytearray 对象写入到 fout 中。为了方便，代码将 bytearray 对象中的每个元素遍历并转化为整数，然后将其打包成一个 i 类型，并将其写入到 fout 中。

代码还会在循环的最后，重复遍历整个词汇表，直到达到给定的词汇表大小。这样，代码就可以在需要时重复读取同一个文件中的文本，从而实现分词处理并多次读取同一个文件中的文本。


```cpp
fout.write(struct.pack("i", vocab_size))

counter = 0
# sort by value
for key in sorted(encoder, key=encoder.get):
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)
    counter += 1

# TODO: Repeat last token until vocab_size
while counter < vocab_size:
    fout.write(struct.pack("i", len(text)))
    fout.write(text)
    counter += 1
```

This is a Python implementation of a function called `convert_dir.py` that converts a directory containing CSV files to a file containing the data in CSV format. The files in the directory are assumed to have the same format as input to the function, which is a common CSV file containing sensor data, such as temperature, humidity, and so on.

The function takes as input the directory containing the CSV files and the file header (CSV file name without extension). It then reads the header file and creates a header string with the header information.

Next, the function reads the data from each CSV file in the directory and concatenates them along the first dimension to form a 2D array. The header information is then appended to the 2D array using the `np.concatenate()` function.

The function then writes the header information along with the length of the data and the data type to a file object. The data is then written to the file object.

Finally, the function creates a new CSV file from the 2D array of data using the `concat` function from the pandas library and writes the new file to the same file handle.

Note that this implementation assumes that the input files are in the same format and contain only one column (the data) and that the header file is in CSV format. The implementation also assumes that the header file is named differently from the input files. This implementation can be easily modified to suit different file formats and naming conventions.


```cpp
# assert counter == config.vocab_size

for name in list_vars.keys():
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " + name + " with shape: ", data.shape)

    # rename headers to keep compatibility
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
    elif re.match(r"transformer.h.\d+.attn.c_proj.bias", name):
        i = re.findall("\d+", name)[0]
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
        print("Unrecognized variable name. %s", name)

    # we don't need these
    if name.endswith("attn.masked_bias") or name.endswith(".attn.bias"):
        print("  Skipping variable: " + name)
        continue

    n_dims = len(data.shape);

    # ftype == 0 -> float32, ftype == 1 -> float16
    ftype = 0;
    if use_f16:
        if (name == "model/wte" or name == "model/lm_head" or name[-2:] == "/g" or name[-2:] == "/w") and n_dims == 2:
            print("  Converting to float16")
            data = data.astype(np.float16)
            ftype = 1
        else:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype = 0

    "model/h.*/attn/c_attn/w"
    "model/h.*/attn/c_proj/w"
    "model/h.*/mlp/c_fc/w"
    "model/h.*/mlp/c_proj/w"
    if name[-14:] == "/attn/c_attn/w" or name[-14:] == "/attn/c_attn/b":
        print("  Duplicate K,V heads to use MHA instead of MQA")

        embed_dim = hparams["n_embd"]
        head_dim = embed_dim // hparams["n_head"]

        # ((n_heads + 2) * head_dim, hidden_dim) -> (3 * n_heads * head_dim, hidden_dim)
        q, k ,v = np.split(data, (hparams["n_head"] * head_dim, (hparams["n_head"] + 1) * head_dim), axis=0)
        # duplicate k, v along the first axis (head_dim, hidden_dim) -> (n_heads * head_dim, hidden_dim)
        if len(k.shape) == 2:
            k = np.tile(k, (hparams["n_head"], 1))
            v = np.tile(v, (hparams["n_head"], 1))
        elif len(k.shape) == 1:
            k = np.tile(k, (hparams["n_head"]))
            v = np.tile(v, (hparams["n_head"]))
        # concat q, k, v along the first axis (n_heads * head_dim, hidden_dim) -> (3 * n_heads * head_dim, hidden_dim)
        data = np.concatenate((q, k, v), axis=0)

    # header
    str = name.encode('utf-8')
    fout.write(struct.pack("iii", n_dims, len(str), ftype))
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    fout.write(str);

    # data
    data.tofile(fout)

```

这段代码的作用是关闭名为 fout 的输出文件，并输出一个消息表明操作已经完成。

具体来说，首先使用 fout.close() 方法关闭 fout 文件，这将阻塞文件输出流，防止写出文件内容。然后使用 print() 函数输出一条消息，其中 fname_out 参数被用来将文件名输出到屏幕上。最后，使用 print() 函数输出一个空行，通常是用于在代码中留下一个空行，使得代码更易于阅读。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/starcoder/main.cpp`

这段代码是一个C++程序，它包括两个头文件和两个函数。它们以下是：

```cpp
#include "ggml/ggml.h"

#include "common.h"
#include "common-ggml.h"
```

这两个头文件可能是ggml库和common库的定义。ggml库和common库可能是一些通用的库，比如数据结构或者图形数据结构，ggml.h 和 common.h 是ggml库和common库的头文件。

接下来的两个函数可能是ggml库和common库的具体实现：

```cpp
#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>
```

这些函数可能是实现ggml库和common库的一些具体操作，比如数据类型的转换，函数的实现，等等。

然后，接下来的代码可能是main函数，它调用前面的函数，使用它们来处理数据：

```cpp
int main()
{
   // 在这里调用ggml库和common库的具体函数
   // 比如，使用ggml库的ggml.h文件实现数据读取和转换，以及common库的common.h文件实现数据类型的转换和函数的调用

   // 在这里输出结果，比如打印数据

   return 0;
}
```

由于我对这个程序的实现并不了解，所以无法提供更具体的解释。


```cpp
#include "ggml/ggml.h"

#include "common.h"
#include "common-ggml.h"

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>

#if defined(_MSC_VER)
```

这段代码是一个C++代码，它定义了一个名为`starcoder_hparams`的结构体，用于指定GPT-2模型的参数。

具体来说，这个结构体包含以下参数：

- `n_vocab`：GPT-2模型的词汇表大小，即模型的词典大小。这个参数在代码中被定义为49280。
- `n_ctx`：GPT-2模型的当前状态大小，即模型的输入长度。这个参数在代码中被定义为2048。
- `n_embd`：GPT-2模型的嵌入层大小，即模型中每个头部的参数个数。这个参数在代码中被定义为2048。
- `n_head`：GPT-2模型的头数，即模型中每个输入头的大小。这个参数在代码中被定义为16。
- `n_layer`：GPT-2模型的层数，即模型中每个头部的层数。这个参数在代码中被定义为24。
- `ftype`：GPT-2模型的类型，可以是0表示"养生"，1表示"恶劣"。这个参数在代码中被定义为1。
- `eps`：GPT-2模型的下的安全性惩罚因子，用于防止梯度消失。这个参数在代码中被定义为1e-5f。


```cpp
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// default hparams (GPT-2 117M)
// https://huggingface.co/bigcode/gpt_bigcode-santacoder/blob/main/config.json
struct starcoder_hparams {
    int32_t n_vocab = 49280;
    int32_t n_ctx   = 2048;
    int32_t n_embd  = 2048;
    int32_t n_head  = 16;
    int32_t n_layer = 24;
    int32_t ftype   = 1;
    float   eps     = 1e-5f;
};

```

这段代码定义了一个名为 `starcoder_layer` 的结构体，表示层与层的连接。这个结构体包含了一些输入和输出张量，以及一些注意力相关的参数。

具体来说，这个结构体包含了以下内容：

* 一个名为 `ln_1_g` 的张量，它的类型是 `ggml_tensor`，表示输入层（layer 1）的第一个神经元的输出。
* 一个名为 `ln_1_b` 的张量，它的类型是 `ggml_tensor`，表示输入层（layer 1）的第二个神经元的输出。
* 一个名为 `ln_2_g` 的张量，它的类型是 `ggml_tensor`，表示注意力机制的注意力权重。
* 一个名为 `ln_2_b` 的张量，它的类型是 `ggml_tensor`，表示注意力机制的注意力强度。
* 一个名为 `c_attn_attn_w` 的张量，它的类型是 `ggml_tensor`，表示卷积神经网络（CNN）的注意力权重。
* 一个名为 `c_attn_attn_b` 的张量，它的类型是 `ggml_tensor`，表示卷积神经网络（CNN）的注意力强度。
* 一个名为 `c_attn_proj_w` 的张量，它的类型是 `ggml_tensor`，表示卷积神经网络（CNN）的注意力前馈（API）。
* 一个名为 `c_attn_proj_b` 的张量，它的类型是 `ggml_tensor`，表示卷积神经网络（CNN）的注意力前馈（API）。
* 一个名为 `c_mlp_fc_w` 的张量，它的类型是 `ggml_tensor`，表示多层感知器（MLP）的前馈（API）。
* 一个名为 `c_mlp_fc_b` 的张量，它的类型是 `ggml_tensor`，表示多层感知器（MLP）的输出。

这个结构体定义了 `starcoder_layer` 的输出，它是一个 `ggml_tensor` 数组，每个元素都是一个神经元的输出，包括注意力相关的参数。


```cpp
struct starcoder_layer {
    // normalization
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;

    // attention
    struct ggml_tensor * c_attn_attn_w;
    struct ggml_tensor * c_attn_attn_b;

    struct ggml_tensor * c_attn_proj_w;
    struct ggml_tensor * c_attn_proj_b;

    // mlp
    struct ggml_tensor * c_mlp_fc_w;
    struct ggml_tensor * c_mlp_fc_b;

    struct ggml_tensor * c_mlp_proj_w;
    struct ggml_tensor * c_mlp_proj_b;
};

```

这是一个使用固体进化（starcoder）模型的结构体定义。它包括以下部分：

1. `starcoder_hparams`：这是一个结构体，定义了模型的超参数。
2. `ln_f_g` 和 `ln_f_b`：这两个结构体存储了自然语言中的词汇和词汇分布。
3. `wte`、`wpe` 和 `lm_head`：这三个结构体存储了位置嵌入、token embedding 和语言模型头。
4. `layers`：这是一个容器，存储了整数表示的层。
5. `memory_k` 和 `memory_v`：这两个结构体存储了用于存储层参数的内存。
6. `ctx`：这是一个指向`ggml_context`结构体的指针。
7. `tensors`：这是一个哈希表，存储了模型的参数。

`starcoder_model`的作用是创建一个用于训练和推理的`starcoder_model`实例。它用于存储模型的参数和结构体，以便在训练和推理时使用。


```cpp
struct starcoder_model {
    starcoder_hparams hparams;

    // normalization
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte;     // position embedding
    struct ggml_tensor * wpe;     //    token embedding
    struct ggml_tensor * lm_head; // language model head

    std::vector<starcoder_layer> layers;

    // key + value memory
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    //
    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

```

This is a function that reads a model file and checks its quality. It takes a function name and a model file path as input and returns a boolean value indicating whether the file is valid.

The function reads the model file and parses its content. It checks whether the file has a header (the first 4 lines), which describes the model's architecture. If the file does not have a header, the function returns `false`.

The function then reads the model's data from the file. It checks whether the data has a header and, if it does not, the function finishes reading the data. The function then calculates the total size of the data and prints it out in a human-readable format.

The function also checks whether the model is a GPT-2 model. If it is, the function ignores the LM head and returns the LM head as if it were part of the model. If it is not a GPT-2 model, the function reads the LM head and returns it.

Finally, the function closes the file.

Note: This function assumes that the model file is well-formed and follows the schema described in the GPT-2 documentation.


```cpp
// load the model's weights from a file
bool starcoder_model_load(const std::string & fname, starcoder_model & model, gpt_vocab & vocab) {
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());

    auto fin = std::ifstream(fname, std::ios::binary);
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // verify magic
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // load hparams
    {
        auto & hparams = model.hparams;

        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // load vocab
    {
        int32_t n_vocab = 0;
        fin.read((char *) &n_vocab, sizeof(n_vocab));

        if (n_vocab != model.hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
            return false;
        }

        std::string word;
        std::vector<char> buf(128);

        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;

            // if (i < 10) fprintf(stderr, "%.s: vocab[%d] = '%s'\n", __func__, i, word.c_str());
        }

        // Add StarChat special tokens.
        for (std::string token : {
                "<|system|>",
                "<|user|>",
                "<|assistant|>",
                "<|end|>",
                "<fim-prefix>",
                "<fim-middle>",
                "<fim-suffix>",
                "<fim-pad>",
                "<|end_of_turn|>"
            }) {
            if (vocab.token_to_id.find(token) != vocab.token_to_id.end()) {
                vocab.add_special_token(token);
            }
        }
    }

    // for the big tensors, we have the option to store the data in 16-bit floats or quantized
    // in order to save memory and also to speed up the computation
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    auto & ctx = model.ctx;

    size_t ctx_size = 0;

    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;

        const int head_dim = n_embd / hparams.n_head;
        const int kv_heads = hparams.n_head; // 1 if MQA else hparams.n_head
        const int kv_dim   = kv_heads * head_dim;

        ctx_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_g
        ctx_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_b

        ctx_size += n_vocab*n_embd*ggml_type_sizef(wtype);         // wte
        ctx_size +=   n_ctx*n_embd*ggml_type_sizef(GGML_TYPE_F32); // wpe
        ctx_size += n_vocab*n_embd*ggml_type_sizef(wtype);         // lm_head

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_b

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_b

        ctx_size += n_layer*((n_embd + 2*kv_dim)*n_embd*ggml_type_sizef(wtype));         // c_attn_attn_w // TODO:
        ctx_size += n_layer*(       (n_embd + 2*kv_dim)*ggml_type_sizef(GGML_TYPE_F32)); // c_attn_attn_b

        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype));           // c_attn_proj_w
        ctx_size += n_layer*(       n_embd*ggml_type_sizef(GGML_TYPE_F32));   // c_attn_proj_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_fc_w
        ctx_size += n_layer*(       4*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_fc_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_proj_w
        ctx_size += n_layer*(         n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_proj_b

        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F32); // memory_k
        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F32); // memory_v

        ctx_size += (6 + 12*n_layer)*512; // object overhead

        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size/(1024.0*1024.0));
    }

    // create the ggml context
    {
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        model.ctx = ggml_init(params);
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // prepare memory for the weights
    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;

        const int head_dim = n_embd / hparams.n_head;
        const int kv_heads = hparams.n_head; // 1 if MQA else hparams.n_head
        const int kv_dim   = kv_heads * head_dim;

        model.layers.resize(n_layer);

        model.ln_f_g = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
        model.ln_f_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        model.wte     = ggml_new_tensor_2d(ctx, wtype,         n_embd, n_vocab);
        model.wpe     = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ctx);
        model.lm_head = ggml_new_tensor_2d(ctx, wtype,         n_embd, n_vocab);

        // map by name
        model.tensors["model/ln_f/g"] = model.ln_f_g;
        model.tensors["model/ln_f/b"] = model.ln_f_b;

        model.tensors["model/wte"]     = model.wte;
        model.tensors["model/wpe"]     = model.wpe;
        model.tensors["model/lm_head"] = model.lm_head;

        for (int i = 0; i < n_layer; ++i) {
            auto & layer = model.layers[i];

            layer.ln_1_g        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_1_b        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.ln_2_g        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_2_b        = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_attn_attn_w = ggml_new_tensor_2d(ctx, wtype,           n_embd, n_embd + 2*kv_dim);
            layer.c_attn_attn_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd + 2*kv_dim);

            layer.c_attn_proj_w = ggml_new_tensor_2d(ctx, wtype,           n_embd, n_embd);
            layer.c_attn_proj_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_mlp_fc_w    = ggml_new_tensor_2d(ctx, wtype,           n_embd, 4*n_embd); //TODO: 4*n_embd = config.n_inner
            layer.c_mlp_fc_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4*n_embd);

            layer.c_mlp_proj_w  = ggml_new_tensor_2d(ctx, wtype,         4*n_embd, n_embd);
            layer.c_mlp_proj_b  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            // map by name
            model.tensors["model/h" + std::to_string(i) + "/ln_1/g"]        = layer.ln_1_g;
            model.tensors["model/h" + std::to_string(i) + "/ln_1/b"]        = layer.ln_1_b;

            model.tensors["model/h" + std::to_string(i) + "/ln_2/g"]        = layer.ln_2_g;
            model.tensors["model/h" + std::to_string(i) + "/ln_2/b"]        = layer.ln_2_b;

            model.tensors["model/h" + std::to_string(i) + "/attn/c_attn/w"] = layer.c_attn_attn_w;
            model.tensors["model/h" + std::to_string(i) + "/attn/c_attn/b"] = layer.c_attn_attn_b;

            model.tensors["model/h" + std::to_string(i) + "/attn/c_proj/w"] = layer.c_attn_proj_w;
            model.tensors["model/h" + std::to_string(i) + "/attn/c_proj/b"] = layer.c_attn_proj_b;

            model.tensors["model/h" + std::to_string(i) + "/mlp/c_fc/w"]    = layer.c_mlp_fc_w;
            model.tensors["model/h" + std::to_string(i) + "/mlp/c_fc/b"]    = layer.c_mlp_fc_b;

            model.tensors["model/h" + std::to_string(i) + "/mlp/c_proj/w"]  = layer.c_mlp_proj_w;
            model.tensors["model/h" + std::to_string(i) + "/mlp/c_proj/b"]  = layer.c_mlp_proj_b;
        }
    }

    // key + value memory
    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        const int n_mem      = n_layer*n_ctx;
        const int n_elements = n_embd*n_mem;

        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);

        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);

        printf("%s: memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);
    }

    // load weights
    {
        size_t total_size = 0;

        bool has_lm_head = false;

        while (true) {
            int32_t n_dims;
            int32_t length;
            int32_t ttype;

            fin.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));
            fin.read(reinterpret_cast<char *>(&length), sizeof(length));
            fin.read(reinterpret_cast<char *>(&ttype),  sizeof(ttype));

            if (fin.eof()) {
                break;
            }

            int32_t nelements = 1;
            int32_t ne[2] = { 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne[i]), sizeof(ne[i]));
                nelements *= ne[i];
            }

            std::string name(length, 0);
            fin.read(&name[0], length);

            if (model.tensors.find(name) == model.tensors.end()) {
                fprintf(stderr, "%s: unknown tensor '%s' in model file\n", __func__, name.c_str());
                return false;
            }

            auto tensor = model.tensors[name];
            if (tensor->ne[0] != ne[0] || tensor->ne[1] != ne[1]) {
                fprintf(stderr, "%s: tensor '%s' has wrong shape in model file: got [%d, %d], expected [%d, %d]\n",
                        __func__, name.c_str(), (int) tensor->ne[0], (int) tensor->ne[1], ne[0], ne[1]);
                return false;
            }
            if (ggml_nelements(tensor) != nelements) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file. got %d, expected %d\n",
                        __func__, name.c_str(), (int) ggml_nelements(tensor), nelements);
                return false;
            }

            // for debugging
            if (0) {
                printf("%24s - [%5d, %5d], type = %6s, %6.2f MB, %9zu bytes\n", name.c_str(), ne[0], ne[1], ggml_type_name(ggml_type(ttype)), ggml_nbytes(tensor)/1024.0/1024.0, ggml_nbytes(tensor));
            }

            const size_t bpe = ggml_type_size(ggml_type(ttype));

            if ((nelements*bpe)/ggml_blck_size(tensor->type) != ggml_nbytes(tensor)) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file: got %zu, expected %zu\n",
                        __func__, name.c_str(), ggml_nbytes(tensor), nelements*bpe);
                return false;
            }

            fin.read(reinterpret_cast<char *>(tensor->data), ggml_nbytes(tensor));

            // GPT-2 models share the WTE tensor as the LM head
            if (name == "model/wte" && has_lm_head == false) {
                memcpy(model.lm_head->data, tensor->data, ggml_nbytes(tensor));
            }

            if (name == "model/lm_head") {
                has_lm_head = true;
            }

            total_size += ggml_nbytes(tensor);
        }

        printf("%s: model size  = %8.2f MB\n", __func__, total_size/1024.0/1024.0);
    }

    fin.close();

    return true;
}

```

This is a C++ function that uses thePT-Tronical model to perform natural language processing tasks on an input text. It is based on the GitHub repository of the same name, which includes models for classification, question-answering, and text generation tasks.

The function takes an input text represented as a binary ionic language (structured in `問名` and `answer`) and performs the following tasks:

1. Preprocessing:
	* Tokenization: The input text is tokenized into words, which are then mapped to integers using the Google Cloud Native Natural Language API.
	* Stopword removal: The input text is filtered out stopwords (such as "the", "and", etc.) that do not出现在 in the training data.
	* Run length normalization: The input text is padded to a fixed length to reduce the number of issues in downstream tasks.
2. Encoding:
	* converting inpL to a numerical format that can be sent as an HTTP request to the server.
3. Training:
	* Creating a new model and evaluating it on the input text using the trained model.
	* The new model is then used to predict the outputs for the rest of the input text using the softmax function.
4. Making predictions:
	* The output of the model for each token in the input text is calculated and combined to get the final prediction.
	* The prediction is returned for each token in the input text.
5. Making predictions for multiple clients:
	* The above model is used for making predictions for multiple clients.
	* Each client sends its own request with the input text, and the model returns the prediction for each token in the input text for all clients.

The function returns a boolean indicating whether the model was able to make predictions for the input text or not.

Note that the function uses several external libraries, including the GitHub packages for GitHubNLP and PyTorch. The function assumes that the model is pre-trained and pre-安裝了相关依赖项。


```cpp
// evaluate the transformer
//
//   - model:     the model
//   - n_threads: number of threads to use
//   - n_past:    the context size so far
//   - embd_inp:  the embeddings of the tokens in the context
//   - embd_w:    the predicted logits for the next token
//
bool starcoder_eval(
        const starcoder_model & model,
        const int n_threads,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp,
              std::vector<float>         & embd_w,
              size_t                     & mem_per_token) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_ctx   = hparams.n_ctx;
    const int n_head  = hparams.n_head;
    const int n_vocab = hparams.n_vocab;

    static size_t buf_size = 256u*1024*1024;
    static void * buf = malloc(buf_size);

    // use 2 scratch buffers
    // TODO: very hacky solution - reimplement in a more elegant way
    static size_t scr0_size = 256u*1024*1024;
    static void * scr0 = malloc(scr0_size);

    static size_t scr1_size = 256u*1024*1024;
    static void * scr1 = malloc(scr1_size);

    if (mem_per_token > 0 && mem_per_token*N > buf_size) {
        const size_t buf_size_new = 1.1*(mem_per_token*N); // add 10% to account for ggml object overhead
        //printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__, buf_size, buf_size_new);

        // reallocate
        buf_size = buf_size_new;
        buf = realloc(buf, buf_size);
        if (buf == nullptr) {
            fprintf(stderr, "%s: failed to allocate %zu bytes\n", __func__, buf_size);
            return false;
        }
    }

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    struct ggml_context * ctx0 = ggml_init(params);
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    for (int i = 0; i < N; ++i) {
        ((int32_t *) position->data)[i] = n_past + i;
    }

    // wte + wpe
    struct ggml_tensor * inpL =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.wte, embd),
                ggml_get_rows(ctx0, model.wpe, position));

    for (int il = 0; il < n_layer; ++il) {
        struct ggml_tensor * cur;

        ggml_set_scratch(ctx0, { 0, scr0_size, scr0, });

        // norm
        {
            // [ 768, N]
            cur = ggml_norm(ctx0, inpL, hparams.eps);

            // cur = ln_1_g*cur + ln_1_b
            // [ 768, N]
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0,
                        ggml_repeat(ctx0, model.layers[il].ln_1_g, cur),
                        cur),
                    ggml_repeat(ctx0, model.layers[il].ln_1_b, cur));
        }

        // attn
        // [2304, 768] - model.layers[il].c_attn_attn_w
        // [2304,   1] - model.layers[il].c_attn_attn_b
        // [ 768,   N] - cur (in)
        // [2304,   N] - cur (out)
        //
        // cur = attn_w*cur + attn_b
        // [2304, N]
        {
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_attn_attn_w,
                    cur);

            cur = ggml_add(ctx0,
                    ggml_repeat(ctx0, model.layers[il].c_attn_attn_b, cur),
                    cur);
        }

        // self-attention
        {
            struct ggml_tensor * Qcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 0*sizeof(float)*n_embd);
            struct ggml_tensor * Kcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 1*sizeof(float)*n_embd);
            struct ggml_tensor * Vcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 2*sizeof(float)*n_embd);

            // store key and value to memory
            if (N >= 1) {
                struct ggml_tensor * k = ggml_view_1d(ctx0, model.memory_k, N*n_embd, (ggml_element_size(model.memory_k)*n_embd)*(il*n_ctx + n_past));
                struct ggml_tensor * v = ggml_view_1d(ctx0, model.memory_v, N*n_embd, (ggml_element_size(model.memory_v)*n_embd)*(il*n_ctx + n_past));

                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));
            }

            // Q = Qcur.contiguous().view(n_embd/n_head, n_head, N).permute(0, 2, 1, 3)
            // [64, N, 12]
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Qcur,
                            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_embd/n_head, n_head, N)),
                        0, 2, 1, 3);

            // K = Kmem.view(n_embd/n_head, n_head, n_past + N).permute(0, 2, 1, 3)
            // [64, n_past + N, 12]
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_reshape_3d(ctx0,
                            ggml_view_1d(ctx0, model.memory_k, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(model.memory_k)*n_embd),
                            n_embd/n_head, n_head, n_past + N),
                        0, 2, 1, 3); //TODO: need to be tiled

            // GG: flash attention
            //struct ggml_tensor * V =
            //    ggml_cpy(ctx0,
            //            ggml_permute(ctx0,
            //                ggml_reshape_3d(ctx0,
            //                    ggml_view_1d(ctx0, model.memory_v, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(model.memory_v)*n_embd),
            //                    n_embd/n_head, n_head, n_past + N),
            //                1, 2, 0, 3),
            //            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_past + N, n_embd/n_head, n_head));

            //struct ggml_tensor * KQV = ggml_flash_attn(ctx0, Q, K, V, true);

            // K * Q
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q); //TODO: check if it broadcasts

            // KQ_scaled = KQ / sqrt(n_embd/n_head)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_scaled =
                ggml_scale_inplace(ctx0,
                        KQ,
                        ggml_new_f32(ctx0, 1.0f/sqrt(float(n_embd)/n_head))
                        );

            // KQ_masked = mask_past(KQ_scaled)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_masked = ggml_diag_mask_inf_inplace(ctx0, KQ_scaled, n_past);

            // KQ = soft_max(KQ_masked)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_soft_max = ggml_soft_max_inplace(ctx0, KQ_masked);

            // V_trans = Vmem.view(n_embd/n_head, n_head, n_past + N).permute(1, 2, 0, 3).contiguous()
            // [n_past + N, 64, 12]
            struct ggml_tensor * V_trans =
                ggml_cpy(ctx0,
                        ggml_permute(ctx0,
                            ggml_reshape_3d(ctx0,
                                ggml_view_1d(ctx0, model.memory_v, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(model.memory_v)*n_embd),
                                n_embd/n_head, n_head, n_past + N),
                            1, 2, 0, 3),
                        ggml_new_tensor_3d(ctx0, model.memory_v->type, n_past + N, n_embd/n_head, n_head));

            // KQV = transpose(V) * KQ_soft_max
            // [64, N, 12]
            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V_trans, KQ_soft_max);

            // KQV_merged = KQV.permute(0, 2, 1, 3)
            // [64, 12, N]
            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            // cur = KQV_merged.contiguous().view(n_embd, N)
            // [768, N]
            cur = ggml_cpy(ctx0,
                    KQV_merged,
                    ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_embd, N));
        }

        // projection
        // [ 768, 768] - model.layers[il].c_attn_proj_w
        // [ 768,   1] - model.layers[il].c_attn_proj_b
        // [ 768,   N] - cur (in)
        // [ 768,   N] - cur (out)
        //
        // cur = proj_w*cur + proj_b
        // [768, N]
        {
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_attn_proj_w,
                    cur);

            cur = ggml_add(ctx0,
                    ggml_repeat(ctx0, model.layers[il].c_attn_proj_b, cur),
                    cur);
        }

        // add the input
        cur = ggml_add(ctx0, cur, inpL);

        struct ggml_tensor * inpFF = cur;

        ggml_set_scratch(ctx0, { 0, scr1_size, scr1, });

        // feed-forward network
        {
            // norm
            {
                cur = ggml_norm(ctx0, inpFF, hparams.eps);

                // cur = ln_2_g*cur + ln_2_b
                // [ 768, N]
                cur = ggml_add(ctx0,
                        ggml_mul(ctx0,
                            ggml_repeat(ctx0, model.layers[il].ln_2_g, cur),
                            cur),
                        ggml_repeat(ctx0, model.layers[il].ln_2_b, cur));
            }

            // fully connected
            // [3072, 768] - model.layers[il].c_mlp_fc_w
            // [3072,   1] - model.layers[il].c_mlp_fc_b
            // [ 768,   N] - cur (in)
            // [3072,   N] - cur (out)
            //
            // cur = fc_w*cur + fc_b
            // [3072, N]
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_mlp_fc_w,
                    cur);

            cur = ggml_add(ctx0,
                    ggml_repeat(ctx0, model.layers[il].c_mlp_fc_b, cur),
                    cur);

            // GELU activation
            // [3072, N]
            cur = ggml_gelu(ctx0, cur);

            // projection
            // [ 768, 3072] - model.layers[il].c_mlp_proj_w
            // [ 768,    1] - model.layers[il].c_mlp_proj_b
            // [3072,    N] - cur (in)
            // [ 768,    N] - cur (out)
            //
            // cur = proj_w*cur + proj_b
            // [768, N]
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_mlp_proj_w,
                    cur);

            cur = ggml_add(ctx0,
                    ggml_repeat(ctx0, model.layers[il].c_mlp_proj_b, cur),
                    cur);
        }

        // input for next layer
        inpL = ggml_add(ctx0, cur, inpFF);
    }

    ggml_set_scratch(ctx0, { 0, scr0_size, scr0, });

    // norm
    {
        // [ 768, N]
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // inpL = ln_f_g*inpL + ln_f_b
        // [ 768, N]
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    ggml_set_scratch(ctx0, { 0, 0, nullptr, });

    // inpL = WTE * inpL
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // logits -> probs
    //inpL = ggml_soft_max_inplace(ctx0, inpL);

    // run the computation
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    //embd_w.resize(n_vocab*N);
    //memcpy(embd_w.data(), ggml_get_data(inpL), sizeof(float)*n_vocab*N);

    // return result just for the last token
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    //printf("used_mem = %zu MB\n", ggml_used_mem(ctx0)/(1024*1024));

    ggml_free(ctx0);

    return true;
}

```

This is a C++ implementation of the Santana pre-training script for StarChat. It is used to convert the StarChat training data into a format that can be used by the StarChat model.

The script takes in the input data (text data), pre-trained vocabulary, and the StarChat model parameters. It then performs the following operations:

* D登录： This is the input data that contains the text data and the pre-trained vocabulary.
* D模： This is the model that we want to use for prediction.
* Iterate through the input data:
	+ The script reads through the input data and applies it to the model.
	+ For each character in the input data, it performs a prediction:
		- First, it checks the character's ID in the pre-trained vocabulary.
		- If the character's ID is not found in the vocabulary, it uses the embeddings to represent the character.
		- Next, it applies the character's embeddings to the input data.
		- Then, it uses the model's forward and backward passes to update the input data.
		- Finally, it uses the embeddings again to perform the next forward or backward pass.
* The script has a variable called "starChat" that is of type ggml. This is because the script uses the Santana pre-training script for StarChat which is written in C++ and uses the ggml library for timing.
* The script has a variable called "model" which is of type SantanaModel.
* The script has a function called "run" which takes in the input data, the pre-trained vocabulary, the StarChat model parameters, and the embeddings.
* The function first initializes the SantanaModel with the input data and the pre-trained vocabulary.
* Then it reads the input data and applies it to the model.
* For each character in the input data, it performs a prediction and updates the input data.
* It then uses the embeddings again to perform the next forward or backward pass.
* The script also has a function called "checkModel" which takes in the StarChat model parameters and the embeddings.
* It checks if the StarChat model is valid and if the input data is a valid sequence.
* It also has a function called "printBoard" which takes in the StarChat model and the input data.
* It prints the StarChat model's board to the screen.

The script is then able to convert the text data into a format that can be used by the StarChat model.


```cpp
int main(int argc, char ** argv) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;

    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    printf("%s: seed = %d\n", __func__, params.seed);

    std::mt19937 rng(params.seed);
    if (params.prompt.empty()) {
        params.prompt = gpt_random_prompt(rng);
    }

    int64_t t_load_us = 0;

    gpt_vocab vocab;
    starcoder_model model;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!starcoder_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;

        test_gpt_tokenizer(vocab, params.token_test);
    }

    if (params.repeat_last_n == -1) {
        params.repeat_last_n = model.hparams.n_ctx;
    }
    printf("\n");
    printf("%s: temp           = %.3f\n", __func__, params.temp);
    printf("%s: top_k          = %d\n",   __func__, params.top_k);
    printf("%s: top_p          = %.3f\n", __func__, params.top_p);
    printf("%s: repeat_last_n  = %d\n",   __func__, params.repeat_last_n);
    printf("%s: repeat_penalty = %.3f\n", __func__, params.repeat_penalty);

    int n_past = 0;

    int64_t t_sample_us  = 0;
    int64_t t_predict_us = 0;

    std::vector<float> logits;

    std::vector<int32_t> last_n_tokens(model.hparams.n_ctx);
    std::fill(last_n_tokens.begin(), last_n_tokens.end(), 0);

    // tokenize the prompt
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());

    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());
    for (size_t i = 0; i < embd_inp.size(); i++) {
        printf("%s: token[%zu] = %6d, %s\n", __func__, i, embd_inp[i], vocab.id_to_token.at(embd_inp[i]).c_str());
    }
    printf("\n\n");

    // Handle StarChat "<|end|>" and OpenCoder "<|end_of_turn>" tokens.
    gpt_vocab::id starchat_end_token = -1;
    {
        const auto it = vocab.token_to_id.find("<|end|>");
        if (it != vocab.token_to_id.end()) {
            starchat_end_token = it->second;
        } else {
            const auto eot_token_id = vocab.token_to_id.find("<|end_of_turn|>");
            if (eot_token_id != vocab.token_to_id.end()) {
              starchat_end_token = eot_token_id->second;
            }
        }
    }

    // submit the input prompt token-by-token
    // this reduces the memory usage during inference, at the cost of a bit of speed at the beginning
    std::vector<gpt_vocab::id> embd;

    // determine the required inference memory per token:
    size_t mem_per_token = 0;
    starcoder_eval(model, params.n_threads, 0, { 0, 1, 2, 3 }, logits, mem_per_token);

    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // predict
        if (embd.size() > 0) {
            const int64_t t_start_us = ggml_time_us();

            if (!starcoder_eval(model, params.n_threads, n_past, embd, logits, mem_per_token)) {
                printf("Failed to predict\n");
                return 1;
            }

            t_predict_us += ggml_time_us() - t_start_us;
        }

        n_past += embd.size();
        embd.clear();

        if (i >= embd_inp.size()) {
            // sample next token
            const int   top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp  = params.temp;

            const int n_vocab = model.hparams.n_vocab;

            gpt_vocab::id id = 0;

            {
                const int64_t t_start_sample_us = ggml_time_us();

                id = gpt_sample_top_k_top_p_repeat(vocab, logits.data() + (logits.size() - n_vocab), last_n_tokens.data(), last_n_tokens.size(), top_k, top_p, temp, params.repeat_last_n, params.repeat_penalty, rng);
                t_sample_us += ggml_time_us() - t_start_sample_us;
            }

            // add it to the context
            embd.push_back(id);

            last_n_tokens.erase(last_n_tokens.begin());
            last_n_tokens.push_back(id);
        } else {
            // if here, it means we are still processing the input prompt
            for (size_t k = i; k < embd_inp.size(); k++) {
                embd.push_back(embd_inp[k]);

                last_n_tokens.erase(last_n_tokens.begin());
                last_n_tokens.push_back(embd_inp[k]);

                if (int32_t(embd.size()) >= params.n_batch) {
                    break;
                }
            }
            i += embd.size() - 1;
        }

        // display text
        for (auto id : embd) {
            printf("%s", vocab.id_to_token[id].c_str());
        }
        fflush(stdout);

        // check if model is santacoder
        if (model.hparams.n_layer <= 30 && embd.back() == 49152) {
            break;
        }
        // check if model is starcoder
        else if (embd.back() == 0) { //TODO: this is only for starcoder
            break;
        }
        // Handle StarChat "<|end|>" token.
        else if (embd.back() == starchat_end_token && i >= embd_inp.size()) {
            break;
        }
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n");
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/n_past);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    ggml_free(model.ctx);

    return 0;
}

```