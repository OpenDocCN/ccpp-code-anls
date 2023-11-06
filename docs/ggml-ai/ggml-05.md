# GGML源码解析 5

# Dolly-V2

Transformer architecture: GPT-NeoX

Modeled from examples/stablelm

Ref: https://github.com/databrickslabs/dolly

Ref: https://github.com/stability-AI/stableLM/#stablelm-alpha

## Usage

```cppbash
# get the repo and build it
git clone https://github.com/ggerganov/ggml
cd ggml
mkdir build && cd build
cmake ..
make -j

# get the Dolly-V2 3B model
git clone https://huggingface.co/databricks/dolly-v2-3b

# install Python dependencies
python3 -m pip install -r ../requirements.txt

# convert model to FP16
python3 ../examples/dolly-v2/convert-h5-to-ggml.py ./dolly-v2-3b/ 1

# run inference using FP16 precision
./bin/dollyv2 -m ./dolly-v2-3b/ggml-model-f16.bin -p "State the meaning of life." -t 6 -n 64

main: seed = 1683218142
dollyv2_model_load: loading model from './dolly-v2-3b/ggml-model-f16.bin' - please wait ...
dollyv2_model_load: n_vocab = 50280
dollyv2_model_load: n_ctx   = 2048
dollyv2_model_load: n_embd  = 2560
dollyv2_model_load: n_head  = 32
dollyv2_model_load: n_layer = 32
dollyv2_model_load: n_rot   = 20
dollyv2_model_load: ftype   = 1
dollyv2_model_load: ggml ctx size = 7374.91 MB
dollyv2_model_load: memory_size =   640.00 MB, n_mem = 65536
dollyv2_model_load: ................................................ done
dollyv2_model_load: model size =  5295.10 MB / num tensors = 388
main: number of tokens in prompt = 32
main: token[0] =  30003, Below
main: token[1] =    310,  is
main: token[2] =    271,  an
main: token[3] =   9775,  instruction
main: token[4] =    326,  that
main: token[5] =   8631,  describes
main: token[6] =    247,  a
main: token[7] =   4836,  task
main: token[8] =    964, .
main: token[9] =  19566,  Write
main: token[10] =    247,  a
main: token[11] =   2380,  response
main: token[12] =    326,  that
main: token[13] =  20420,  appropriately
main: token[14] =  29141,  completes
main: token[15] =    253,  the
main: token[16] =   2748,  request
main: token[17] =    964, .
main: token[18] =    187, 

main: token[19] =    187, 

main: token[20] =  50278, ### Instruction:
main: token[21] =    187, 

main: token[22] =   5443, State
main: token[23] =    253,  the
main: token[24] =   4495,  meaning
main: token[25] =    273,  of
main: token[26] =   1495,  life
main: token[27] =    964, .
main: token[28] =    187, 

main: token[29] =    187, 

main: token[30] =  50279, ### Response:
main: token[31] =    187, 


Below is an instruction that describes a task. Write a response that appropriately completes the request.

### Instruction:
State the meaning of life.

### Response:
The meaning of life is to love and be loved.

### End

main: mem per token = 16136720 bytes
main:     load time =  2202.58 ms
main:   sample time =     2.57 ms
main:  predict time =  1497.14 ms / 33.27 ms per token
main:    total time =  6187.27 ms
```

## 5-bit integer quantization mode

```cppbash
# quantize the model to 5-bits using Q5_0 quantization
./bin/dollyv2-quantize ./dolly-v2-3b/ggml-model-f16.bin ./dolly-v2-3b/ggml-model-q5_0.bin q5_0

# run the quantized model
./bin/dollyv2 -m ./dolly-v2-3b/ggml-model-q5_0.bin -p "State the meaning of life." -t 6 -n 64

main: seed = 1683218518
dollyv2_model_load: loading model from './dolly-v2-3b/ggml-model-q5_0.bin' - please wait ...
dollyv2_model_load: n_vocab = 50280
dollyv2_model_load: n_ctx   = 2048
dollyv2_model_load: n_embd  = 2560
dollyv2_model_load: n_head  = 32
dollyv2_model_load: n_layer = 32
dollyv2_model_load: n_rot   = 20
dollyv2_model_load: ftype   = 8
dollyv2_model_load: ggml ctx size = 3902.68 MB
dollyv2_model_load: memory_size =   640.00 MB, n_mem = 65536
dollyv2_model_load: ................................................ done
dollyv2_model_load: model size =  1822.87 MB / num tensors = 388
main: number of tokens in prompt = 32
main: token[0] =  30003, Below
main: token[1] =    310,  is
main: token[2] =    271,  an
main: token[3] =   9775,  instruction
main: token[4] =    326,  that
main: token[5] =   8631,  describes
main: token[6] =    247,  a
main: token[7] =   4836,  task
main: token[8] =    964, .
main: token[9] =  19566,  Write
main: token[10] =    247,  a
main: token[11] =   2380,  response
main: token[12] =    326,  that
main: token[13] =  20420,  appropriately
main: token[14] =  29141,  completes
main: token[15] =    253,  the
main: token[16] =   2748,  request
main: token[17] =    964, .
main: token[18] =    187, 

main: token[19] =    187, 

main: token[20] =  50278, ### Instruction:
main: token[21] =    187, 

main: token[22] =   5443, State
main: token[23] =    253,  the
main: token[24] =   4495,  meaning
main: token[25] =    273,  of
main: token[26] =   1495,  life
main: token[27] =    964, .
main: token[28] =    187, 

main: token[29] =    187, 

main: token[30] =  50279, ### Response:
main: token[31] =    187, 


Below is an instruction that describes a task. Write a response that appropriately completes the request.

### Instruction:
State the meaning of life.

### Response:
The meaning of life is the discovery of the true self.

### End

main: mem per token = 16127760 bytes
main:     load time =  1011.09 ms
main:   sample time =     2.79 ms
main:  predict time =  1271.62 ms / 27.64 ms per token
main:    total time =  2802.51 ms
```

## Notes

- No guarantees for correctness
- The tokenizer is currently hacked - probably works only for English
- Non-parallel residual is not supported
- Contributions and improvements are welcome


# `examples/gpt-2/convert-cerebras-to-ggml.py`

这段代码的作用是将Cerebras模型的结构转换为GGML格式，以便在不同的环境中使用。它主要涉及以下步骤：

1. 从Cerebras模型的结构中提取一些信息。
2. 将这些信息转换为JSON格式。
3. 将JSON格式数据转换为Python结构。
4. 将Python结构转换为GGML格式。

这段代码需要以下依赖：

- Cerebras模型的结构定义（作为输入）
- Python 3.6 或更高版本
- torch

它假设您已经有了一个Cerebras模型，可以将其结构定义作为参数传入函数中，也可以通过从命令行或其他输入来源获取它。


```cpp
# Convert Cerebras models to ggml format
#
# ref: https://www.cerebras.net/blog/cerebras-gpt-a-family-of-open-compute-efficient-large-language-models/
#

import sys
import struct
import json
import torch
import numpy as np
import re

from transformers import AutoModelForCausalLM

# ref: https://github.com/openai/gpt-2/blob/master/src/encoder.py
```

这段代码定义了一个名为 `bytes_to_unicode` 的函数，它返回一个二进制字符列表（bytes）和一个对应的字符串列表（unicode strings）。

该函数使用了双字节编码（bpe）技术，这种编码方式适用于Unicode字符集中的字符。由于该函数需要覆盖约10%的32K个二进制字符，因此它需要一个相对较大的Unicode字符集。

函数首先生成一个包含ASCII字符（包括标点符号和一些特殊符号）和Unicode字符的字符列表。接着，它将这个列表中的所有二进制字符（包括Unicode字符）存储在一个二进制字符列表（bs）中，并存储到一个控制字符（char）中。

函数还使用一个类似的二进制字符列表（cs）存储Unicode字符映射（即，一个二进制字符对应一个Unicode字符）。然后，它遍历存储在cs中的二进制字符，检查它们是否在bs中。如果是，则将bs中的对应二进制字符添加到cs中，并将bs中对应位置的字符类型更改为Unicode类型。最后，函数将cs中的所有二进制字符转换为相应的Unicode字符，并返回一个字典，将二进制字符列表和Unicode字符列表存储为键。


```cpp
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

这段代码是一个 Python 脚本，它的主要目的是帮助用户创建一个固有地（Cerebras）模型为 Gephi（GGML）格式的文件。

这段代码的功能是读取用户提供的模型文件夹名称，然后根据用户提供的参数（使用 F32 或 F16）创建一个 16 位或 32 位的 Gephi 模型文件。对于 16 位，文件名为 `ggml-model-f16.bin`，而对于 32 位，文件名为 `ggml-model-f32.bin`。


```cpp
if len(sys.argv) < 2:
    print("Usage: convert-cerebras-to-ggml.py dir-model [use-f32]\n")
    sys.exit(1)

# output in the same directory as the model
dir_model = sys.argv[1]
fname_out = sys.argv[1] + "/ggml-model-f16.bin"

with open(dir_model + "/vocab.json", "r", encoding="utf-8") as f:
    encoder = json.load(f)

with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

# use 16-bit or 32-bit floats
```

这段代码的作用是：

1. 如果从命令行中运行的参数数量大于2，则设置 `use_f16` 为 `False`，否则设置 `use_f16` 为 `True`。
2. 创建一个名为 `fname_out` 的文件，并将其写入到 `/ggml-model-f32.bin` 的路径。
3. 从预训练模型中加载一个名为 `dir_model` 的模型，并将其转换为使用 F16 精度的权重。
4. 将模型的状态量列表 `list_vars` 打印出来。
5. 创建一个名为 `fout` 的文件，并将其写入到 `fname_out` 的路径。


```cpp
use_f16 = True
if len(sys.argv) > 2:
    use_f16 = False
    fname_out = sys.argv[1] + "/ggml-model-f32.bin"

model = AutoModelForCausalLM.from_pretrained(dir_model, low_cpu_mem_usage=True)
#print (model)

list_vars = model.state_dict()
#print (list_vars)

print(hparams)

fout = open(fname_out, "wb")

```

这段代码的主要作用是输出一个二进制字符数组中的数据，并将其转换为相应的 Python 数据类型。

具体来说，代码中定义了一个名为 "fout" 的文件输出流和一个名为 "struct" 的结构体，其中包含了一些变量和它们的取值。

首先，将一个名为 "0x67676d6c" 的字节数组用 "i" 格式进行打包，并将其写入 "fout" 文件中，这样 "fout" 文件中的第一个输出就是 0x67676d6c。

然后，将一个名为 "vocab_size" 的整数变量 hparams["vocab_size"] 用 "i" 格式进行打包，并将其写入 "fout" 文件中，这样 "fout" 文件中的第二个输出就是 "vocab_size"。

接着，将一个名为 "n_positions" 的整数变量 hparams["n_positions"] 用 "i" 格式进行打包，并将其写入 "fout" 文件中，这样 "fout" 文件中的第三个输出就是 "n_positions"。

然后，将一个名为 "n_embd" 的整数变量 hparams["n_embd"] 用 "i" 格式进行打包，并将其写入 "fout" 文件中，这样 "fout" 文件中的第四个输出就是 "n_embd"。

接下来，将一个名为 "n_head" 的整数变量 hparams["n_head"] 用 "i" 格式进行打包，并将其写入 "fout" 文件中，这样 "fout" 文件中的第五个输出就是 "n_head"。

然后，将一个名为 "n_layer" 的整数变量 hparams["n_layer"] 用 "i" 格式进行打包，并将其写入 "fout" 文件中，这样 "fout" 文件中的第六个输出就是 "n_layer"。

接着，通过一个名为 "use_f16" 的布尔变量来控制是否使用 f16 数据类型输出，如果为真，则使用 f16 数据类型输出，否则使用整数数据类型输出。

然后，定义了一个名为 "byte_encoder" 的函数，该函数将一个名为 "byte_to_unicode" 的函数包装起来，并将从 "byte_encoder" 变量中获取的 byte 数组包装成一个字符串，使用 "i" 格式格式化字符串中的 "i" 表示从 0x48开始，这样 "byte_encoder" 函数返回的字符串就是一个整数类型，用 "i" 格式化后就是字节数组。

接着，定义了一个名为 "fout.write" 的函数，该函数将 "byte_encoder" 函数包装起来，用于向 "fout" 文件中写入数据，每次写入的数据是一个字节数组，用 "i" 格式表示，其中第一位是 0x48（字节数组中的第一个字节），后面跟着一系列 "i" 格式表示字节数组中的第几个元素。

然后，通过一个循环来遍历 "encoder" 变量中的所有键，并将每个键的字符串打包成一个字节数组，用 "i" 格式表示，其中第一位是 0x48（字节数组中的第一个字节），后面跟着一系列 "i" 格式表示字节数


```cpp
fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("i", hparams["n_positions"]))
fout.write(struct.pack("i", hparams["n_embd"]))
fout.write(struct.pack("i", hparams["n_head"]))
fout.write(struct.pack("i", hparams["n_layer"]))
fout.write(struct.pack("i", use_f16))

byte_encoder = bytes_to_unicode()
byte_decoder = {v:k for k, v in byte_encoder.items()}

fout.write(struct.pack("i", len(encoder)))

for key in encoder:
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

```

This is a Python code snippet that performs some preprocessing steps on a text file `text.txt`.

The code has the following functions:

1. `parse_header()`: This function takes a single argument, `header`, which is the header text in the format of `<attn/c_attn/w> <attn/c_proj/w> <mlp/c_fc/w> <mlp/c_proj/w>`. This function converts the header to a list of integers, removes the last 4 columns (which are assumed to be the dimensions of the text), and formats the header by transposing it.
2. `create_header()`: This function takes a single argument, `header_format`, which is the format string of the header. This function takes the `header_format` and adds a `<attn/c_attn/w>` header line.
3. `parse_data()`: This function takes two arguments: `data` and `fout`. It is the main function that reads the input file and formats it according to the rules specified in the `parse_header()` function. It creates header file in `fout` by adding the header and writes the data to `fout`.
4. `main()`: This is the main function that calls the `parse_data()` function.

The code also imports some standard libraries: `


```cpp
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

    # for efficiency - transpose the projection matrices
    # "model/h.*/attn/c_attn/w"
    # "model/h.*/attn/c_proj/w"
    # "model/h.*/mlp/c_fc/w"
    # "model/h.*/mlp/c_proj/w"
    if name[-14:] == "/attn/c_attn/w" or \
       name[-14:] == "/attn/c_proj/w" or \
       name[-11:] == "/mlp/c_fc/w" or \
       name[-13:] == "/mlp/c_proj/w":
        print("  Transposing")
        data = data.transpose()

    # header
    str = name.encode('utf-8')
    fout.write(struct.pack("iii", n_dims, len(str), ftype))
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    fout.write(str);

    # data
    data.tofile(fout)

```

这段代码的作用是关闭名为 fout 的输出文件，并输出一个消息表明操作已经完成。具体解释如下：

1. `fout.close()` 表示关闭名为 fout 的输出文件。在程序中，通常使用 `with` 语句来打开和关闭文件。这个语句确保文件在操作完成后自动关闭，避免了手动关闭文件的操作失误。

2. `print("Done. Output file: " + fname_out)` 表示输出一条消息，其中 `fname_out` 是一个变量，表示输出文件的文件名。这条消息说明已经完成关闭文件的操作，并指出了输出的文件名。

3. `print("")` 表示输出一个空行，用于在消息和文件名之间添加一些空白行，使输出更易于阅读。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/gpt-2/convert-ckpt-to-ggml.py`

这段代码是一个用于将TensorFlow中模型的 checkpoint文件转换为ggml兼容文件的工具。它主要实现了以下几个功能：

1.加载模型：通过调用TensorFlow中的`tf.keras.backend`模块，加载用户指定的模型 checkpoint 文件并将其转换为模型的函数集合。

2.写入模型变量：遍历模型的所有变量，将其转换为二进制文件。具体来说，这段代码将模型的参数（如维度、名称长度、维度大小等）存储为一个字节序列，然后将其写入到一个二进制文件中。

3.支持16位浮点数：通过在代码中设置`use_f32=True`，可以使得函数在转换大矩阵（如Batchnorm，Reshape等）时，将其转换为16位浮点数。

4.允许用户设置转换为16位浮点数：通过添加命令行参数`--use-f32`，用户可以自行设置转换为16位浮点数。


```cpp
# Convert a model checkpoint to a ggml compatible file
#
# Load the model using TensorFlow.
# Iterate over all variables and write them to a binary file.
#
# For each variable, write the following:
#   - Number of dimensions (int)
#   - Name length (int)
#   - Dimensions (int[n_dims])
#   - Name (char[name_length])
#   - Data (float[n_dims])
#
# By default, the bigger matrices are converted to 16-bit floats.
# This can be disabled by adding the "use-f32" CLI argument.
#
```

这段代码定义了一个函数`bytes_to_unicode()`，该函数将一个字节序列转换为相应的Unicode字符串。

函数的作用是帮助用户在开始编写ggml文件时，将模型参数和词汇表统一成Unicode类型。函数首先从给定的byte序列中构建一个包含Unicode字符的列表，然后从该列表中基于BPE编码器的输出，生成一个包含相应Unicode字符的列表。这样，用户就可以在ggml文件中直接使用Unicode字符，而不是依赖于BPE编码器从头生成Unicode词汇表。

函数的实现基于以下两个辅助函数：

* `bytes_to_unicode()`函数：将一个字节序列转换为相应的Unicode字符串。这个函数将递归地遍历给定的byte序列，并将遇到的字节转换为相应的Unicode字符。
* `ord_to_bytes()`函数：将一个Unicode字符转换为相应的bytes序列。这个函数将给定的Unicode字符转换为相应的bytes序列，然后返回这个bytes序列。

这两个函数的实现基于以下Python库：`


```cpp
# At the start of the ggml file we write the model parameters
# and vocabulary.
#

import sys
import json
import struct
import numpy as np
import tensorflow as tf

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

这段代码是一个用于将 numpy 数组转换为不同浮点数类型的辅助方法。它接受两个参数：数据（numpy 数组）和浮点数类型（可以是 0 或 1）。如果输入的浮点数类型不正确，则会抛出一个错误。

当程序运行时，如果传入的参数数量少于 3，则会输出 Usage 消息并退出。正确的使用方法是：convert-ckpt-to-ggml.py <dir-model> <ftype>，其中 <dir-model> 是模型目录，<ftype> 是浮点数类型，可以是 0（默认值）或 1（表示 fp16 类型）。实际运行时，还会将模型名称（与输入参数不同）输出到控制台。


```cpp
# helper method to convert a numpy array to different float types
def convert_to_ftype(data, ftype):
    # fp16
    if ftype == 1:
        return data.astype(np.float16)

    assert False, "Invalid ftype: " + str(ftype)

if len(sys.argv) < 3:
    print("Usage: convert-ckpt-to-ggml.py dir-model ftype\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)

# output in the same directory as the model
```

这段代码的作用是读取一个名为"dir_model"的文件夹，并读取其中的"encoder.json"文件和"hparams.json"文件，然后对文件中的数据类型进行转换，并输出转换后的数据类型对应的字符串。

具体来说，代码首先读取dir_model中的"encoder.json"文件，并使用json.load()函数将文件中的数据类型转换成json数据类型，然后读取第二个参数（即"/ggml-model.bin"）并将其作为文件名，输出文件夹名称和文件名。

接着，代码又读取dir_model中的"hparams.json"文件，并使用json.load()函数将文件中的数据类型转换成json数据类型，然后定义了一个ftype_str变量，用于存储数据类型对应的字符串，最后通过map()函数将数据类型转换成字符串。


```cpp
dir_model = sys.argv[1]
fname_out = sys.argv[1] + "/ggml-model.bin"

with open(dir_model + "/encoder.json", "r", encoding="utf-8") as f:
    encoder = json.load(f)

with open(dir_model + "/hparams.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

# possible data types
#   ftype == 0 -> float32
#   ftype == 1 -> float16
#
# map from ftype to string
ftype_str = ["f32", "f16"]

```

这段代码的作用是训练一个名为"magic: ggml"模型的子词列表，并将其保存为名为"model-<ftype>.bin"的文件。

具体来说，代码首先定义了一个名为"ftype"的整数变量，其值为1。然后，代码检查命令行参数的数量是否大于2，如果是，就从第二个参数中读取一个整数，将其作为"ftype"的值，并检查该值是否小于0或大于1，如果是，就输出一条错误消息并退出程序。否则，代码将读取第一个参数中指定的文件名和模型的类型，创建一个写入器并将其写入文件中，文件类型为模型的类型，并使用结构体类型来存储模型的参数。

另外，代码还定义了一个名为"fout"的文件输出者，用于将模型的参数写入到文件中。


```cpp
ftype = 1
if len(sys.argv) > 2:
    ftype = int(sys.argv[2])
    if ftype < 0 or ftype > 1:
        print("Invalid ftype: " + str(ftype))
        sys.exit(1)
    fname_out = sys.argv[1] + "/ggml-model-" + ftype_str[ftype] + ".bin"

list_vars = tf.train.list_variables(dir_model)

fout = open(fname_out, "wb")

fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["n_vocab"]))
fout.write(struct.pack("i", hparams["n_ctx"]))
```

这段代码的主要作用是输出一个二进制序列 `text`，该序列代表了文本数据中的每个字符。它使用了输入参数 `text` 和 `encoder`，其中 `text` 是通过 `bytes_to_unicode` 函数生成的字节序列，`encoder` 是一个编码字典，它将文本数据中的每个字符映射到了对应的编码。

具体来说，这段代码首先通过 `fout.write(struct.pack("i", hparams["n_embd"]))` 将 `params["n_embd"]` 这个文本数据中的 embedding 长度编码成了一个整数类型，并输出了这个整数。接着，它又通过 `fout.write(struct.pack("i", hparams["n_head"]))` 将 `params["n_head"]` 这个文本数据中的 head 长度编码成了一个整数类型，并输出了这个整数。然后，它又通过 `fout.write(struct.pack("i", hparams["n_layer"]))` 将 `params["n_layer"]` 这个文本数据中的 layer 长度编码成了一个整数类型，并输出了这个整数。

接下来的代码中，`byte_encoder` 是一个将文本数据转换为字节序列的函数，它的实现类似于 `str.maketrans` 函数，将字符和它自己之间的所有 trans 时，返回的第一个非空bytes序列。`fout.write(struct.pack("i", ftype))` 将 ftype 变量编码成了一个整数类型，并输出了这个整数。

接着，代码通过 `for key in encoder:` 遍历 encoder 字典中的每一个键，并在循环体内执行以下操作：通过 `byte_decoder` 中的映射关系，将每个文本字符转换成对应的编码，然后通过 `fout.write(struct.pack("i", len(text)))` 将编码长度编码成了一个整数类型，并输出了这个整数。接着，代码通过 `fout.write(text)` 将编码后的文本字符输出到了 fout 文件中。


```cpp
fout.write(struct.pack("i", hparams["n_embd"]))
fout.write(struct.pack("i", hparams["n_head"]))
fout.write(struct.pack("i", hparams["n_layer"]))
fout.write(struct.pack("i", ftype))

byte_encoder = bytes_to_unicode()
byte_decoder = {v:k for k, v in byte_encoder.items()}

fout.write(struct.pack("i", len(encoder)))

for key in encoder:
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

```

This is a C++ program that does the following:

1. If the name ends with "/attn/c\_attn/w" or "/attn/c_proj/w" or "/mlp/c_fc/w" or "/mlp/c_proj/w", it will transpose the data.
2. It loads the header information from the first line.
3. It converts the data type of the data based on the type specified in the header (either float32 or model/h.*/attn/c\_attn/w)
4. It writes the header information to the file.
5. It writes the dimensions of the data to the file.
6. It writes the header information to the file.
7. It writes the data to the file.


```cpp
for name, shape in list_vars:
    print("Processing variable: " + name + " with shape: ", shape)

    data = tf.train.load_variable(dir_model, name).squeeze()
    n_dims = len(data.shape);

    # for efficiency - transpose the projection matrices
    # "model/h.*/attn/c_attn/w"
    # "model/h.*/attn/c_proj/w"
    # "model/h.*/mlp/c_fc/w"
    # "model/h.*/mlp/c_proj/w"
    if name[-14:] == "/attn/c_attn/w" or \
       name[-14:] == "/attn/c_proj/w" or \
       name[-11:] == "/mlp/c_fc/w" or \
       name[-13:] == "/mlp/c_proj/w":
        print("  Transposing")
        data = data.transpose()

    dshape = data.shape

    ftype_cur = 0
    if ftype != 0:
        # match name:
        #  "model/wte"
        #  "model/h.*/attn/c_attn/w"
        #  "model/h.*/attn/c_proj/w"
        #  "model/h.*/mlp/c_fc/w"
        #  "model/h.*/mlp/c_proj/w"
        if name == "model/wte" or name[-2:] == "/w":
            print("  Converting to " + ftype_str[ftype])
            data = convert_to_ftype(data, ftype)
            ftype_cur = ftype
        else:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0

    # header
    str = name.encode('utf-8')
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
    for i in range(n_dims):
        fout.write(struct.pack("i", dshape[n_dims - 1 - i]))
    fout.write(str);

    # data
    data.tofile(fout)

```

这段代码的作用是关闭名为 fout 的文件，并输出一个消息表明操作已经完成。

具体来说，第一个 `fout.close()` 函数用于关闭已经打开的文件，如果没有打开文件，则会报错。第二个 `print()` 函数用于输出一条消息，其中包含一个字符串 "Done"，然后跟着一个空格，最后是文件名 fname_out。这个字符串用于指示文件名。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/gpt-2/convert-h5-to-ggml.py`

这段代码的作用是将一个名为 GPT-2 h5 transformer模型转换为 ggml 格式。它主要实现了以下几个功能：

1. 读取 GPT-2 Model 并将其保存为 binary 文件。
2. 为每个变量写入一个二进制数据块，其中包括：
  - 数据块包含模型的维度（int）
  - 变量名称长度（int）
  - 维度（int[n_dims])
  - 变量名称（char[name_length])
  - 数据（float[n_dims])
3. 默认情况下，将大矩阵转换为 16 位浮点数。
4. 通过添加 "use-f32" CLI 参数，可以禁用这个转换。


```cpp
# Convert GPT-2 h5 transformer model to ggml format
#
# Load the model using GPT2Model.
# Iterate over all variables and write them to a binary file.
#
# For each variable, write the following:
#   - Number of dimensions (int)
#   - Name length (int)
#   - Dimensions (int[n_dims])
#   - Name (char[name_length])
#   - Data (float[n_dims])
#
# By default, the bigger matrices are converted to 16-bit floats.
# This can be disabled by adding the "use-f32" CLI argument.
#
```

这段代码的作用是读取一个ggml文件中的模型参数和词汇表，并将其转换为python可读的格式。

具体来说，它实现了以下几个步骤：

1. 读取输入文件中的模型参数和词汇表。
2. 将每个模型的参数转换为unicode字符，并在代码中使用一个由utf-8字节组成的列表和一个由utf-8字节和unicode字符组成的列表来存储这些参数。
3. 将每个unicode字符转换为一个相应的GPT2 Model。
4. 将两个列表（存储模型参数和词汇表）合并为一个dictionary，键为bytes，值为两个列表。
5. 将dictionary中的键（存储模型参数）转换为字符串，以便在代码中进行访问。


```cpp
# At the start of the ggml file we write the model parameters
# and vocabulary.
#

import sys
import struct
import json
import numpy as np
import re

from transformers import GPT2Model

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

这段代码是一个Python脚本，它的作用是执行一个转换H5格式到GGML格式的操作。以下是代码的功能解释：

1. 检查命令行参数的数量，如果少于2个，则输出使用说明，并退出脚本。
2. 获取第一个命令行参数，并将其存储在变量dir_model中。
3. 构建用于输出GGML模型的文件名，该文件名为dir_model + "/ggml-model.bin"。
4. 使用文件句柄打开dir_model中的vocab.json文件，并读取其中的编码器。
5. 使用文件句柄打开dir_model中的added_tokens.json文件，并读取其中的编码器。
6. 使用文件句柄打开dir_model中的config.json文件，并读取其中的配置信息。
7. 将读取到的使用说明和其他信息存储在变量中，并按照适当的格式输出。
8. 脚本使用sys.exit()函数来强制退出，如果输出数组包含1个参数，则退出脚本。


```cpp
if len(sys.argv) < 2:
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")
    sys.exit(1)

# output in the same directory as the model
dir_model = sys.argv[1]
fname_out = sys.argv[1] + "/ggml-model.bin"

with open(dir_model + "/vocab.json", "r", encoding="utf-8") as f:
    encoder = json.load(f)

with open(dir_model + "/added_tokens.json", "r", encoding="utf-8") as f:
    encoder_added = json.load(f)

with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

```

这段代码是一个Python脚本，主要作用是使用GPU（图形处理器，GPU为NVIDIA默认的GPU）训练一个预训练的GPT2模型，并将训练好的模型导出为F32架构的模型文件。以下是该代码的详细解释：

1. `# use 16-bit or 32-bit floats`：声明该脚本使用16位或32位浮点数。
2. `use_f16 = True`：如果使用16位浮点数，则设`use_f16`为True；如果使用32位浮点数，则设`use_f16`为False。
3. `if len(sys.argv) > 2:`：当运行该脚本时，`sys.argv`（从命令行输入）大于2时执行该段代码。
4. `use_f16 = False`：如果`len(sys.argv) > 2`，则执行以下操作，否则设置为`use_f16 = True`。
5. `fname_out = sys.argv[1] + "/ggml-model-f32.bin"`：将用户输入的文件名（包括参数）与`/ggml-model-f32.bin`拼接，得到导出的模型文件名。
6. `model = GPT2Model.from_pretrained(dir_model, low_cpu_mem_usage=True)`：从指定的目录（`dir_model`）中使用预训练的GPT2模型。
7. `list_vars = model.state_dict()`：从预训练模型中获取状态（state）变量（state）。
8. `fout = open(fname_out, "wb")`：打开文件输出流（`fout`），将导出的模型文件存储到文件中。
9. `fout.write(struct.pack("i", 0x67676d6c))`：向文件中写入数据，数据是一个整数（`i`类型），值为`0x67676d6c`（GPU中表示文件格式的魔数）。
10. `# print (model)`：打印预训练模型。
11. `print(list_vars)`：打印状态变量列表。

该脚本的作用是使用GPU训练一个预训练的GPT2模型，并将训练好的模型导出为F32架构的模型文件。


```cpp
# use 16-bit or 32-bit floats
use_f16 = True
if len(sys.argv) > 2:
    use_f16 = False
    fname_out = sys.argv[1] + "/ggml-model-f32.bin"

model = GPT2Model.from_pretrained(dir_model, low_cpu_mem_usage=True)
#print (model)

list_vars = model.state_dict()
#print (list_vars)

fout = open(fname_out, "wb")

fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
```

这段代码的主要作用是输出一个名为 "params" 的结构体的内容，该结构体包含以下字段：

* "vocab_size"：一个整数类型的成员变量，表示词汇表的大小。
* "n_positions"：一个整数类型的成员变量，表示句子的位置，包括起始位置和终止位置。
* "n_embd"：一个整数类型的成员变量，表示嵌入的词汇数量。
* "n_head"：一个整数类型的成员变量，表示句子的头数量。
* "n_layer"：一个整数类型的成员变量，表示句子的层数。
* "rotary_dim"：一个整数类型的成员变量，表示旋转维度。
* "use_f16"：一个布尔类型的成员变量，表示是否使用 one-六年级的 F16 数据类型。

函数首先导入了两个名为 "bytes_to_unicode" 和 "utils" 的函数，然后定义了一个名为 "byte_encoder" 的函数，它将一个字节数组转换为 Unicode 字符串。

接着，定义了一个名为 "fout" 的文件输出流对象，并使用 "write" 函数将参数 "params" 中的每个字段转换为 Unicode 字符串并输出到文件中。

然后，定义了一个名为 "byte_decoder" 的函数，它将一个字节数组中的每个 Unicode 字符映射为相应的 ASCII 编码。

接下来，定义了一个名为 "text" 的变量，用于存储文本数据，然后使用 "write" 函数将其中的每个元素转换为 Unicode 字符串并输出到文件中。

接着，使用 for 循环遍历 "params" 中的每个键，并将每个键的字符串转换为一个字节数组，并使用 "write" 函数将其中的每个元素转换为 Unicode 字符串并输出到文件中。

最后，通过调用 "write" 函数将一个布尔类型的参数 "use_f16" 输出到文件中。


```cpp
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("i", hparams["n_positions"]))
fout.write(struct.pack("i", hparams["n_embd"]))
fout.write(struct.pack("i", hparams["n_head"]))
fout.write(struct.pack("i", hparams["n_layer"]))
#fout.write(struct.pack("i", hparams["rotary_dim"]))
fout.write(struct.pack("i", use_f16))

byte_encoder = bytes_to_unicode()
byte_decoder = {v:k for k, v in byte_encoder.items()}

fout.write(struct.pack("i", len(encoder) + len(encoder_added)))

for key in encoder:
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

```

It looks like this is a Python script that reads in a header file with variable names and outputs the data in a binary format. The header file seems to be defining a linear regression model with a softmax output layer, and the variable names correspond to the different parameters in the model.

The script first reads in the header file and then loops through the variables to determine their data type and store it in a variable. Next, it outputs the data in the binary format using the `struct.pack` function.

The script then loops through the dimensions of the data and outputs the shape of each dimension. Finally, it outputs the data type and the binary data.


```cpp
for key in encoder_added:
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

for name in list_vars.keys():
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " + name + " with shape: ", data.shape)

    # we don't need these
    if name.endswith("attn.masked_bias") or name.endswith(".attn.bias"):
        print("  Skipping variable: " + name)
        continue

    n_dims = len(data.shape);

    # ftype == 0 -> float32, ftype == 1 -> float16
    ftype = 0;
    if use_f16:
        if name[-7:] == ".weight" and n_dims == 2:
            print("  Converting to float16")
            data = data.astype(np.float16)
            ftype = 1
        else:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype = 0

    # for efficiency - transpose these matrices:
    #  "transformer.h.*.mlp.c_proj.weight
    if name.endswith(".mlp.c_proj.weight"):
        print("  Transposing")
        data = data.transpose()

    # rename headers to keep compatibility
    if name == "ln_f.weight":
        name = "model/ln_f/g"
    elif name == "ln_f.bias":
        name = "model/ln_f/b"
    elif name == "wte.weight":
        name = "model/wte"
    elif name == "wpe.weight":
        name = "model/wpe"
    elif re.match(r"h\.\d+\.ln_1\.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_1/g"
    elif re.match(r"h\.\d+\.ln_1\.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_1/b"
    elif re.match(r"h\.\d+\.attn\.c_attn\.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_attn/w"
    elif re.match(r"h\.\d+\.attn\.c_attn\.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_attn/b"
    elif re.match(r"h\.\d+\.attn\.c_proj\.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_proj/w"
    elif re.match(r"h.\d+.attn.c_proj.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/attn/c_proj/b"
    elif re.match(r"h.\d+.ln_2.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_2/g"
    elif re.match(r"h.\d+.ln_2.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/ln_2/b"
    elif re.match(r"h.\d+.mlp.c_fc.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_fc/w"
    elif re.match(r"h.\d+.mlp.c_fc.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_fc/b"
    elif re.match(r"h.\d+.mlp.c_proj.weight", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_proj/w"
    elif re.match(r"h.\d+.mlp.c_proj.bias", name):
        i = re.findall("\d+", name)[0]
        name = f"model/h{i}/mlp/c_proj/b"
    else:
        print("Unrecognized variable name. %s", name)

    str = name.encode('utf-8')

    fout.write(struct.pack("iii", n_dims, len(str), ftype))
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    fout.write(str);

    # data
    data.tofile(fout)

```

这段代码的作用是关闭名为 "fout" 的文件，并输出一个消息表明操作已经完成。具体来说，代码首先打开名为 "fout" 的文件，然后使用 `close()` 方法关闭它。接着，代码使用 `print()` 函数输出了一个消息，其中包含一个文件名 "fout"，和一个字符串 "Done"。此外，代码还使用了一个空行来分隔输出的多个部分。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/gpt-2/main-alloc.cpp`

这段代码是一个机器学习数据预处理工具的C语言 implementation，主要用于读取和转换数据文件，为训练神经网络做准备。主要作用是读取数据文件，对数据进行清洗和预处理，包括：

1. 读取数据文件，将所有行存储在一个二维数组中；
2. 对每一行数据进行处理，包括：
  a. 删除常务虚列（即对数据进行去重处理）；
  b. 根据需要对数据进行归一化处理，使得每个特征在同一区间内；
  c. 对数据进行标准化处理，包括将数据映射到最小值和最大值之间，以及将数据映射到具有特定比例的范围内；
  d. 根据需要对数据进行特征选择，包括选择在前n个最具决定性的特征上进行得分，以及选择具有最高方差和最小方差的特征；
  e. 对数据进行归一化处理，使得每个特征在同一区间内；
  f. 将每一行的特征存储到一个包含多个特征的 vector 中。

代码中包含多个头文件，对数据预处理的不同阶段提供了不同功能的函数，包括从 common.h 和 common-ggml.h 中导入一些通用的函数，以及从 ggml-alloc.h 中导入的内存管理函数。


```cpp
#include "ggml/ggml.h"
#include "ggml/ggml-alloc.h"

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

```

这段代码是用来定义和初始化GPT-2模型的参数hparams的。它包含了一系列的整型和浮点型变量，用于表示模型的不同部分，如词汇量(n_vocab)、上下文长度(n_ctx)、嵌入维度(n_embd)、头数(n_head)、层数(n_layer)和类型(ftype)，同时还有一个浮点型变量eps，表示迭代的精度。

除了这些变量外，还有一句通过_MSC_VER来检查定义时是否使用了预定义的C/C++标准。如果定义时使用了该标准，那么会去除下面两行注释，即
```cpp
#pragma warning(disable: 4244 4267) // possible loss of data
```
如果没有使用该标准，那么这一行注释就会被忽略。


```cpp
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// default hparams (GPT-2 117M)
struct gpt2_hparams {
    int32_t n_vocab = 50257;
    int32_t n_ctx   = 1024;
    int32_t n_embd  = 768;
    int32_t n_head  = 12;
    int32_t n_layer = 12;
    int32_t ftype   = 1;
    float   eps     = 1e-5f;
};

```

这是一个结构体定义，表示一个名为“gpt2_layer”的GPT2模型层。这个结构体包含多个成员变量，用于表示不同部分的输入数据和注意力权重。

// normalization
```cpp
// ln_1_g 和 ln_1_b 是变量，分别表示输入数据经过线性变换后的第一层数据。
```
// ln_2_g 和 ln_2_b 是变量，分别表示输入数据经过另一种线性变换后的第一层数据。
```cpp
// attention
```
// c_attn_attn_w 和 c_attn_attn_b 是变量，表示注意力权重向量。
```cpp
// c_attn_proj_w 和 c_attn_proj_b 是变量，表示处理注意力权重的线性变换。
```
// mlp
```cpp
// c_mlp_fc_w 和 c_mlp_fc_b 是变量，表示经过多层全连接层后的第一层数据。
```
// c_mlp_proj_w 和 c_mlp_proj_b 是变量，表示对多层全连接层的线性变换。
```cpp
这个结构体定义了GPT2模型层中的各个部分，用于表示输入数据和计算注意力权重。注意力机制是GPT2模型的核心部分，它允许模型在理解输入数据的同时，能够捕捉长距离依赖关系。


```
struct gpt2_layer {
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

```cpp

这是一段定义了一个名为 `gpt2_model` 的结构体的代码。这个结构体包含了一些 GPT2 的参数，比如隐藏层参数 `hparams`，以及一些标量，比如 `ln_f_g` 和 `ln_f_b`，它们是 GPT2 的两个内部层的一个激活值。

这个结构体还包含一个名为 `layers` 的容器，它里面存放着一些 `ggml_layer` 的结构体，这些结构体可能是从 GPT2 模型结构中获取出来的层。

此外，这个结构体还包括一个 `memory_k` 和一个 `memory_v` 成员变量，它们都是指向内存的指针，这个指针可能是 `ggml_tensor` 类型的，用于存储模型中的内存。

此外，这个结构体还包括一个 `ctx` 成员和一个 `tensors` 成员，它们分别是 GPT2 上下文和一些标量的映射，这些标量可能是 `ggml_tensor` 类型的，用于在训练和推理过程中使用。


```
struct gpt2_model {
    gpt2_hparams hparams;

    // normalization
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte;     // position embedding
    struct ggml_tensor * wpe;     //    token embedding
    struct ggml_tensor * lm_head; // language model head

    std::vector<gpt2_layer> layers;

    // key + value memory
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    //
    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

```cpp

This is a function that reads a model file (e.g., a graph machine learning model), and calculates the model size. It takes a binary tensor object as an input and outputs the total size in MB.

The function first checks if the given tensor has the same data type as the input model. If they do, the function reads the tensor's data and finishes reading the file. If not, it tries to read the data using different data types (e.g., from a file, a string, etc.) and skips the read if it fails.

Then, the function calculates the size of the tensor and updates the `total_size` variable accordingly. Finally, it prints the total size in MB.


```
// load the model's weights from a file
bool gpt2_model_load(const std::string & fname, gpt2_model & model, gpt_vocab & vocab) {
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

        ctx_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_g
        ctx_size += n_embd*ggml_type_sizef(GGML_TYPE_F32); // ln_f_b

        ctx_size += n_vocab*n_embd*ggml_type_sizef(wtype);         // wte
        ctx_size +=   n_ctx*n_embd*ggml_type_sizef(GGML_TYPE_F32); // wpe
        ctx_size += n_vocab*n_embd*ggml_type_sizef(wtype);         // lm_head

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_b

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_b

        ctx_size += n_layer*(3*n_embd*n_embd*ggml_type_sizef(wtype));         // c_attn_attn_w
        ctx_size += n_layer*(       3*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_attn_attn_b

        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype));           // c_attn_proj_w
        ctx_size += n_layer*(       n_embd*ggml_type_sizef(GGML_TYPE_F32));   // c_attn_proj_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_fc_w
        ctx_size += n_layer*(       4*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_fc_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_proj_w
        ctx_size += n_layer*(         n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_proj_b

        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F32); // memory_k
        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F32); // memory_v

        ctx_size += (6 + 12*n_layer)*512; // object overhead

        printf("%s: ggml tensor size = %d bytes\n", __func__, (int) sizeof(ggml_tensor));
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

            layer.c_attn_attn_w = ggml_new_tensor_2d(ctx, wtype,           n_embd, 3*n_embd);
            layer.c_attn_attn_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 3*n_embd);

            layer.c_attn_proj_w = ggml_new_tensor_2d(ctx, wtype,           n_embd, n_embd);
            layer.c_attn_proj_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_mlp_fc_w    = ggml_new_tensor_2d(ctx, wtype,           n_embd, 4*n_embd);
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
            if (ggml_nelements(tensor) != nelements) {
                fprintf(stderr, "%s: tensor '%s' has wrong size in model file\n", __func__, name.c_str());
                return false;
            }

            if (tensor->ne[0] != ne[0] || tensor->ne[1] != ne[1]) {
                fprintf(stderr, "%s: tensor '%s' has wrong shape in model file: got [%d, %d], expected [%d, %d]\n",
                        __func__, name.c_str(), (int) tensor->ne[0], (int) tensor->ne[1], ne[0], ne[1]);
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

```cpp

This is a function definition for a neural network model, which includes the input layer, the model parameters, and the forward function for the network. The input layer takes in a single feature tensor of size N, which is passed through a function `ggml_mul_mat` to perform matrix multiplication with a learn rate tensor `model.layers[il].c_mlp_proj_w` and a learn rate tensor `model.layers[il].c_mlp_proj_b`. The output of the input layer is then passed through a function `ggml_add` to add it to a learn rate tensor `inpFF`, which is then passed through a function `ggml_norm` to calculate the EPS-based normalization of the input. The function also includes a function `ggml_mul` which performs matrix multiplication with a learn rate tensor, this function is used in the forward function.


```
// build the computation graph
struct ggml_cgraph * gpt2_graph(
        const gpt2_model & model,
        struct ggml_allocr * allocr,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_ctx   = hparams.n_ctx;
    const int n_head  = hparams.n_head;

    // since we are using ggml-alloc, this buffer only needs enough space to hold the ggml_tensor and ggml_cgraph structs, but not the tensor data
    static size_t buf_size = ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead();
    static std::vector<uint8_t> buf(buf_size);

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf.data(),
        /*.no_alloc   =*/ true, // the tensors will be allocated later by ggml_allocr_alloc_graph()
    };

    struct ggml_context * ctx0 = ggml_init(params);

    struct ggml_cgraph  * gf = ggml_new_graph(ctx0);

    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    ggml_allocr_alloc(allocr, embd);

    // avoid writing to tensors if we are only measuring the memory usage
    if (!ggml_allocr_is_measure(allocr)) {
        memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));
    }

    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    ggml_allocr_alloc(allocr, position);
    if (!ggml_allocr_is_measure(allocr)) {
        for (int i = 0; i < N; ++i) {
            ((int32_t *) position->data)[i] = n_past + i;
        }
    }

    struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
    ggml_allocr_alloc(allocr, KQ_scale);
    if (!ggml_allocr_is_measure(allocr)) {
        ggml_set_f32(KQ_scale, 1.0f/sqrtf(float(n_embd)/n_head));
    }

    // wte + wpe
    struct ggml_tensor * inpL =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.wte, embd),
                ggml_get_rows(ctx0, model.wpe, position));

    for (int il = 0; il < n_layer; ++il) {
        struct ggml_tensor * cur;

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
                        0, 2, 1, 3);

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
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            // KQ_scaled = KQ / sqrt(n_embd/n_head)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_scaled =
                ggml_scale(ctx0,
                        KQ,
                        KQ_scale);

            // KQ_masked = mask_past(KQ_scaled)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_masked = ggml_diag_mask_inf(ctx0, KQ_scaled, n_past);

            // KQ = soft_max(KQ_masked)
            // [n_past + N, N, 12]
            struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_masked);

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

    // inpL = WTE * inpL
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // logits -> probs
    //inpL = ggml_soft_max(ctx0, inpL);

    ggml_build_forward_expand(gf, inpL);

    ggml_free(ctx0);

    return gf;
}

```cpp

This is a function that performs inference on a question and answer pair using the pre-trained BERT model. It takes as input a question and answer pair represented as a vector of integers, and a vector of floating point numbers. The function returns a boolean indicating whether the inference was successful.

The function first initializes some variables, such as the model and the vector of floating point numbers representing the input question and answer pair. It then allocates memory for the input tensor and the output tensor, and runs the inference using the `ggml_graph_compute()` function.

If the inference was successful, the function returns `true`. Otherwise, it returns `false`.


```
// evaluate the transformer
//
//   - model:     the model
//   - allocr:    ggml_allocr to use to allocate the compute buffer
//   - n_threads: number of threads to use
//   - n_past:    the context size so far
//   - embd_inp:  the embeddings of the tokens in the context
//   - embd_w:    the predicted logits for the next token
//
bool gpt2_eval(
        const gpt2_model & model,
        struct ggml_allocr * allocr,
        const int n_threads,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp,
              std::vector<float>         & embd_w) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_vocab = hparams.n_vocab;

    // reset the allocator to free all the memory allocated during the previous inference
    ggml_allocr_reset(allocr);

    struct ggml_cgraph * gf = gpt2_graph(model, allocr, n_past, embd_inp);

    // allocate tensors
    ggml_allocr_alloc_graph(allocr, gf);

    // run the computation
    struct ggml_cplan plan = ggml_graph_plan(gf, n_threads);
    static std::vector<uint8_t> work_buffer;
    work_buffer.resize(plan.work_size);
    plan.work_data = work_buffer.data();
    ggml_graph_compute(gf, &plan);

    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    // in this case, the output tensor is the last one in the graph
    struct ggml_tensor * inpL = gf->nodes[gf->n_nodes - 1];

    //embd_w.resize(n_vocab*N);
    //memcpy(embd_w.data(), ggml_get_data(inpL), sizeof(float)*n_vocab*N);

    // return result just for the last token
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    return true;
}

```cpp

This is a C++ implementation of the Text-to-Token (T2T) model training script using MXNet. The script performs the following tasks:

1. Loads the pre-trained MXNet model from a file using the function `__func__`.
2. Processes the input data, which is a list of 500,000 prompts and their corresponding text IDs.
3. Creates a list of embedded IDs, which are passed to the function `embd.push_back(id)`.
4. Samples each embedded ID, which is done using the function `t_sample_us`.
5. Creates a list of embedded prompts, which are done using the function `embd.push_back(50256)`.
6. Analyzes the timing information and reports it.
7. Finally, it returns the load time, sample time, and total time in milliseconds.

Note: This script assumes that the input data is already available and stored in memory, and that the MXNet model has already been loaded and is ready to use.


```
int main(int argc, char ** argv) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;
    params.model = "models/gpt-2-117M/ggml-model.bin";

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
    gpt2_model model;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!gpt2_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;

        test_gpt_tokenizer(vocab, params.token_test);
    }

    // keep this buffer alive while evaluating the model
    std::vector<uint8_t> compute_buffer;

    struct ggml_allocr * allocr = NULL;
    // allocate the compute buffer
    {
        allocr = ggml_allocr_new_measure(GGML_MEM_ALIGN);

        // create the worst case graph for memory usage estimation
        int n_tokens = std::min(model.hparams.n_ctx, params.n_batch);
        int n_past = model.hparams.n_ctx - n_tokens;
        struct ggml_cgraph * gf = gpt2_graph(model, allocr, n_past, std::vector<gpt_vocab::id>(n_tokens, 0));

        // compute the required memory
        size_t mem_size = ggml_allocr_alloc_graph(allocr, gf) + GGML_MEM_ALIGN;

        // recreate the allocator with the required memory
        ggml_allocr_free(allocr);
        compute_buffer.resize(mem_size);
        allocr = ggml_allocr_new(compute_buffer.data(), mem_size, GGML_MEM_ALIGN);

        fprintf(stderr, "%s: compute buffer size: %.2f MB\n", __func__, mem_size/1024.0/1024.0);
    }

    int n_past = 0;

    int64_t t_sample_us  = 0;
    int64_t t_predict_us = 0;

    std::vector<float> logits;

    // tokenize the prompt
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());

    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
    printf("%s: number of tokens in prompt = %zu, first 8 tokens: ", __func__, embd_inp.size());
    for (int i = 0; i < std::min(8, (int) embd_inp.size()); i++) {
        printf("%d ", embd_inp[i]);
    }
    printf("\n\n");

    // submit the input prompt token-by-token
    // this reduces the memory usage during inference, at the cost of a bit of speed at the beginning
    std::vector<gpt_vocab::id> embd;

    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // predict
        if (embd.size() > 0) {
            const int64_t t_start_us = ggml_time_us();

            if (!gpt2_eval(model, allocr, params.n_threads, n_past, embd, logits)) {
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

                id = gpt_sample_top_k_top_p(vocab, logits.data() + (logits.size() - n_vocab), top_k, top_p, temp, rng);

                t_sample_us += ggml_time_us() - t_start_sample_us;
            }

            // add it to the context
            embd.push_back(id);
        } else {
            // if here, it means we are still processing the input prompt
            for (size_t k = i; k < embd_inp.size(); k++) {
                embd.push_back(embd_inp[k]);
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

        // end of text token
        if (embd.back() == 50256) {
            break;
        }
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n");
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/n_past);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    ggml_free(model.ctx);

    return 0;
}

```