# GGML源码解析 11

# MNIST Examples for GGML

These are simple examples of how to use GGML for inferencing.
The first example uses convolutional neural network (CNN), the second one uses fully connected neural network.

## Building the examples

```cppbash
git clone https://github.com/ggerganov/ggml
cd ggml
mkdir build && cd build
cmake ..
make -j4 mnist-cnn mnist
```

## MNIST with CNN

This implementation achieves ~99% accuracy on the MNIST test set.

### Training the model

Use the `mnist-cnn.py` script to train the model and convert it to GGUF format:

```cpp
$ python3 ../examples/mnist/mnist-cnn.py train mnist-cnn-model
...
Keras model saved to 'mnist-cnn-model'
```

Convert the model to GGUF format:

```cpp
$ python3 ../examples/mnist/mnist-cnn.py convert mnist-cnn-model
...
Model converted and saved to 'mnist-cnn-model.gguf'
```

### Running the example

```cppbash
$ ./bin/mnist-cnn mnist-cnn-model.gguf ../examples/mnist/models/mnist/t10k-images.idx3-ubyte
main: loaded model in     5.17 ms
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ * * * * * _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ * * * * * * * * _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ * * * * * _ _ _ * * _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ * * _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ * * * _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ * * * _ _ _ _ _ _ _ _ * _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ * * * _ _ _ _ _ _ _ _ * * _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ * * * _ _ _ _ _ _ _ _ * * _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ * * * _ _ _ _ * * * * * _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ * * * * * * * * * _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ * * * * * * * * * * _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ * * * * * * _ _ * * * _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ * * * _ _ _ _ _ _ _ * * * _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ * * _ _ _ _ _ _ _ _ _ * * _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ * * _ _ _ _ _ _ _ _ _ * * * _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ * * _ _ _ _ _ _ _ _ _ * * * _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ * * * _ _ _ _ _ _ _ _ * * * _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ * * * _ _ _ _ _ _ * * * _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ * * * * * * * * * * _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ * * * * * * _ _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 

ggml_graph_dump_dot: dot -Tpng mnist-cnn.dot -o mnist-cnn.dot.png && open mnist-cnn.dot.png
main: predicted digit is 8
```

Computation graph:

![mnist dot](https://user-images.githubusercontent.com/1991296/263763842-3b679b45-7ca1-4ee9-b19a-82e34396624f.png)

## MNIST with fully connected network

A fully connected layer + relu, followed by a fully connected layer + softmax.

### Training the Model

A Google Colab notebook for training a simple two-layer network to recognize digits is located here. You can
use this to save a pytorch model to be converted to ggml format.

[Colab](https://colab.research.google.com/drive/12n_8VNJnolBnX5dVS0HNWubnOjyEaFSb?usp=sharing)

GGML "format" is whatever you choose for efficient loading. In our case, we just save the hyperparameters used
plus the model weights and biases. Run convert-h5-to-ggml.py to convert your pytorch model. The output format is:

- magic constant (int32)
- repeated list of tensors
- number of dimensions of tensor (int32)
- tensor dimension (int32 repeated)
- values of tensor (int32)

Run ```cppconvert-h5-to-ggml.py mnist_model.state_dict``` where `mnist_model.state_dict` is the saved pytorch model from the Google Colab. For
quickstart, it is included in the mnist/models directory.

```cppbash
mkdir -p models/mnist
python3 ../examples/mnist/convert-h5-to-ggml.py ../examples/mnist/models/mnist/mnist_model.state_dict
```

### Running the example

```cppbash
./bin/mnist ./models/mnist/ggml-model-f32.bin ../examples/mnist/models/mnist/t10k-images.idx3-ubyte
```

Computation graph:

![mnist dot](https://user-images.githubusercontent.com/1991296/231882071-84e29d53-b226-4d73-bdc2-5bd6dcb7efd1.png)


## Web demo

The example can be compiled with Emscripten like this:

```cppbash
cd examples/mnist
emcc -I../../include -I../../include/ggml -I../../examples ../../src/ggml.c main.cpp -o web/mnist.js -s EXPORTED_FUNCTIONS='["_wasm_eval","_wasm_random_digit","_malloc","_free"]' -s EXPORTED_RUNTIME_METHODS='["ccall"]' -s ALLOW_MEMORY_GROWTH=1 --preload-file models/mnist
```

Online demo: https://mnist.ggerganov.com


# `examples/mpt/convert-h5-to-ggml.py`

这段代码的作用是定义了一个名为 `bytes_to_unicode` 的函数，它将一个字节序列转换为相应的 Unicode 字符串。

具体来说，这个函数接收一个字节序列（即一个字符串），然后将其转换为一个含有 Unicode 字符的列表。为了实现这个转换，它使用了一个称为 `bytes_to_unicode` 的函数，这个函数接收一个字节序列并返回一个含有 Unicode 字符的列表。它还使用了一个称为 `list_utf8_to_unicode` 的函数，这个函数接收一个字节序列并返回一个含有 Unicode 字符的列表。

此外，这个函数还定义了一个名为 `bytes_to_utf8` 的函数，这个函数将一个含有 Unicode 字符的列表转换为字节序列。


```cpp
import os
import struct
import sys

import torch
from transformers import AutoConfig, AutoTokenizer


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
    bs = (
        list(range(ord("!"), ord("~") + 1))
        + list(range(ord("¡"), ord("¬") + 1))
        + list(range(ord("®"), ord("ÿ") + 1))
    )
    cs = bs[:]
    n = 0
    for b in range(2**8):
        if b not in bs:
            bs.append(b)
            cs.append(2**8 + n)
            n += 1

    cs = [chr(n) for n in cs]

    return dict(zip(bs, cs))


```



这段代码是一个Python函数，名为 `count_model_parts`，用于计算模型目录中的模型部分数量。函数输入参数 `dir_model` 是指模型目录的路径。

函数首先通过 `os.listdir` 函数获取模型目录中的所有文件名，然后遍历这些文件名，使用Python的 `if` 语句判断文件名是否以 "pytorch_model-" 开头。如果是，则说明这是一个模型文件，函数将计数器 `num_parts` 加一。

最后，函数会输出模型目录中的模型部分数量，并在需要时使用 `print` 函数进行输出。

这段代码的作用是计算模型目录中的模型部分数量，并输出该数量。


```cpp
def count_model_parts(dir_model: str) -> int:
    """Returns the number of model parts in the model directory."""
    num_parts = 0
    for filename in os.listdir(dir_model):
        if filename.startswith("pytorch_model-"):
            num_parts += 1

    if num_parts > 0:
        print(f"Found {num_parts} model parts in {dir_model}")
    return num_parts


if len(sys.argv) < 3:
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)


```

这段代码的作用是计算指定目录下模型的数量，并输出一个名为"ggml-model-<ftype>.bin"的文件，其中ftype是数据类型，可以是float32或float16。它通过`count_model_parts`函数来计算模型数量，然后通过`sys.argv`列表获取用户输入的第二个参数作为数据类型参数。如果用户输入的参数不是数字，则程序会输出一个错误消息并退出。如果数据类型不正确，程序也会输出错误消息并退出。最后，程序会根据用户输入的数据类型和模型数量，输出一个名为"ggml-model-<ftype>.bin"的文件，其中ftype是数据类型参数。


```cpp
# output in the same directory as the model
dir_model = sys.argv[1]
# get number of model parts
num_parts = count_model_parts(dir_model)

# possible data types
#   ftype == 0 -> float32
#   ftype == 1 -> float16
#
# map from ftype to string
ftype_str = ["f32", "f16"]

ftype = 1
if len(sys.argv) > 2:
    ftype = int(sys.argv[2])
    if ftype < 0 or ftype > 1:
        print("Invalid ftype: " + str(ftype))
        sys.exit(1)
    fname_out = dir_model + "/ggml-model-" + ftype_str[ftype] + ".bin"


```

这段代码的作用是创建一个预训练模型（model）和一些设置，然后将它们存储到一个文件中。

具体来说，首先从Hugging Face库中加载预训练模型（dir_model），然后从预训练模型中加载配置（config）。接着，将这些配置转换为字典形式，并将其存储到一个名为config.txt的文件中。

接下来，打开一个名为fname_out的文件，并使用struct库中的write函数将预训练模型的几个参数（包括magic: ggml）写入文件。这些参数分别是：

* "i"：表示输入为整数类型
* "i"：表示输入为整数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节
* "i"：表示输入为整数类型，占用4个字节
* "f"：表示输入为浮点数类型，占用4个字节


```cpp
tokenizer = AutoTokenizer.from_pretrained(dir_model, trust_remote_code=True)
config = AutoConfig.from_pretrained(dir_model, trust_remote_code=True)
hparams = config.to_dict()

fout = open(fname_out, "wb")

fout.write(struct.pack("i", 0x67676D6C))  # magic: ggml in hex
fout.write(struct.pack("i", hparams["d_model"]))
fout.write(struct.pack("i", hparams["max_seq_len"]))
fout.write(struct.pack("i", hparams["n_heads"]))
fout.write(struct.pack("i", hparams["n_layers"]))
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("f", hparams["attn_config"]["alibi_bias_max"]))
fout.write(struct.pack("f", hparams["attn_config"]["clip_qkv"] or 0.0))
fout.write(struct.pack("i", ftype))

```

这段代码的作用是定义了一个名为 "vocab_size" 的参数，并将其存储在名为 "hparams" 的字典中。接着，它使用 `tokenizer.vocab` 方法获取了预定义词汇表中的所有词汇，并将这些词汇添加到了编码器中。然后，它将预定义的词汇添加到了编码器的词汇表中。

接下来，它将所有给定的文本转换为字节序列，并将这些字节流编碼为 utf-8 编码。接着，它将所有文本的字节流存储到一个名为 "fout" 的文件中。在循环中，它记录了文件中已读取的字节数和文本的长度，并将这两个值作为 "i" 型结构体发送到文件中。最后，它还记录了已读取的文本长度，以便在循环结束后计算输出。


```cpp
vocab_size = hparams["vocab_size"]

encoder = tokenizer.vocab
# Add added_tokens (special tokens) to the encoder
encoder.update(tokenizer.get_added_vocab())

byte_encoder = bytes_to_unicode()
byte_decoder = {v: k for k, v in byte_encoder.items()}

counter = 0
# sort by value
for key in sorted(encoder, key=encoder.get):
    # workaround for key error when c not found
    text = ""
    for c in key:
        if c not in byte_decoder:
            text += c
        else:
            text += chr(byte_decoder[c])
    text = bytearray(text, encoding="utf-8")
    fout.write(struct.pack("i", len(text)))
    fout.write(text)
    counter += 1

```

It looks like this code is processing a binary file that has a PyTorch model in it. The code reads the contents of the file, splits it into smaller parts, and then processes each part in order to perform some operation on the data. The parts of the file are identified by a "part" string, which starts with the number followed by a ".", for example "pytorch_model-1.bin" or "pytorch_model-2.bin".

The code reads the header information from the first part of the file, which has three fields: the number of dimensions, the number of elements in each dimension, and the data type of the variables. The header information is used to determine the shape of the data and to create an array of the correct data type.

The code then reads the data from the rest of the file, one chunk at a time. Each chunk is processed by writing the header information to a file-like object and then processing the data using the code. The data is processed by converting it to a numpy array and then to a PyTorch tensor. The processed data is then written back to the file.

It's important to note that the code assumes that the input file is valid and that the PyTorch version and model architecture specified in the code are correctly specified. Additionally, the actual implementation of the code may vary depending on the specific use case.


```cpp
# Repeat last token until vocab_size
while counter < vocab_size:
    fout.write(struct.pack("i", len(text)))
    fout.write(text)
    counter += 1

if num_parts == 0:
    part_names = ("pytorch_model.bin",)
else:
    part_names = (
        f"pytorch_model-{n:05}-of-{num_parts:05}.bin" for n in range(1, num_parts + 1)
    )

for part_name in part_names:
    print(f"\n* Loading part: {part_name}")
    model_part = torch.load(f"{dir_model}/{part_name}", map_location="cpu")

    for name in model_part.keys():
        data = model_part[name].squeeze()
        n_dims = len(data.shape)

        # ftype == 0 -> float32, ftype == 1 -> float16
        # default type is fp32
        ftype_cur = 0
        if ftype == 1 and name[-7:] == ".weight" and n_dims > 1:
            ftype_cur = 1
        data = data.to(dtype=torch.float16 if ftype_cur == 1 else torch.float32).numpy()

        print(
            "Processing variable: " + name + " with shape: ",
            data.shape,
            "->",
            data.dtype,
        )

        # header
        str = name.encode("utf-8")
        fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
        for i in range(n_dims):
            fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
        fout.write(str)

        # data
        data.tofile(fout)

    # release memory
    del model_part

```

这段代码的作用是关闭名为 fout 的输出文件，并输出一个消息表明操作已经完成。

具体来说，首先使用 fout.close() 方法关闭输出文件，这是Python中关闭文件的一种方式，它将文件句柄中的内容发送到操作系统资源中，确保文件关闭并释放所有资源。

然后使用 print() 函数输出一条消息，其中 fname_out 是输出文件的名称。这条消息的作用是告诉程序员操作已经完成，可以尝试输出文件的内容。

最后使用 print() 函数输出一个空格，通常是用于在输出中增加一些额外的空间或控制字符数。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/mpt/main.cpp`

这段代码是一个通用的C++头文件，其中包含了一些通用的数学和数据结构函数，以及一些标准库头文件。

具体来说，这段代码包含以下内容：

1. ggml和common-ggml头文件：这两个文件都是ggml库的一部分，其中ggml是一个C++图形渲染引擎，common-ggml是一个通用的C++头文件，用于定义一些通用的函数和数据结构。

2. stdio、stdlib和std头文件：这些文件是C++标准库的一部分，包含了一些通用的输入/输出、字符串处理和数学函数。

3. cmath和cstddef头文件：这些文件也是c++标准库的一部分，包含了一些数学函数，包括一些常见的数学常量，如sin、cos、sqrt等。

4. cstring和cinttypes头文件：这两个文件同样是c++标准库的一部分，包含了一些字符串处理函数和整型数据类型。

5. fstream和std::vector头文件：这两个文件是std库的一部分，用于输入/输出文件，其中fstream用于文件输入/输出，std::vector用于动态数组。

6. common.h：这个头文件可能是某个通用的C++库或框架的一部分，定义了一些通用的函数和数据结构。

7. utf8h头文件：这个头文件可能是UTF-8编码的Unicode编码支持库的一部分，用于定义了一些UTF-8编码的函数和数据结构。

8. ggml::ggml头文件：这个头文件是ggml库的一部分，定义了一些ggml特有的函数和数据结构。


```cpp
#include "ggml/ggml.h"

#include "common-ggml.h"
#include "common.h"

#include <cmath>
#include <cstddef>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <cinttypes>
#include <map>
#include <string>
#include <utility>
#include <vector>

```

这段代码是一个C++代码片段，用于设置一个名为"mpt_hparams"的结构体的参数。这个结构体用于在训练大语言模型（如THULAC，GPT等）时设置超参数。以下是代码的作用：

1. 如果定义了_MSC_VER，那么会预先设置以下42个 warning 指令：
  ```cpp
  #pragma warning(disable: 4244 4267)
  ```
  这些指令通常用于编译器优化，可以帮助我们避免可能的泄漏数据、缓冲区溢出等问题。

2. 如果未定义_MSC_VER，则设置mpt_hparams结构体的默认参数。

3. 定义mpt_hparams结构体，其中包含了许多可以训练的参数，包括：
  - d_model：模型在纸带上可读取的最大尺寸。
  - max_seq_len：能处理的最大序列长度。
  - n_heads：头数。
  - n_layers：能堆叠的层数。
  - n_vocab：已经训练好的词汇表大小。
  - alibi_bias_max：当前Albi谦虚估计器的偏置。
  - clip_qkv：使用的量化键值。
  - ftype：是否使用FP16数据类型。
  - n_ctx：上下文数量。

4. 使用`#if defined(_MSC_VER)`和`#endif`来检查定义的_MSC_VER。如果它存在，那么执行以下操作：
  ```cpp
  #pragma warning(disable: 4244 4267)
  ```
  - 如果定义了，就不输出上述42个 warning 指令。

5. 设置mpt_hparams结构体的参数。

6. 在训练过程中，可以通过调用`struct mpt_hparams`类型的变量来访问这些参数，以调整模型训练过程中的超参数。


```cpp
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// no defaults for now
struct mpt_hparams {
    int32_t d_model      = 0;
    int32_t max_seq_len  = 0;
    int32_t n_heads      = 0;
    int32_t n_layers     = 0;
    int32_t n_vocab      = 0;
    float alibi_bias_max = 0;
    float clip_qkv       = 0;
    int32_t ftype        = 0;
    int32_t n_ctx        = 0;

};

```

这是一个结构体定义，表示一个名为 mpt_layer 的数据结构。这个结构体包含了一些未初始化的成员变量，包括：

1. 一个名为 norm_1_weight 的成员变量，这是一个未初始化的结构体指针，指向一个未定义的名为 ggml_tensor 的类型，可能是表示权重。
2. 一个名为 c_attn_wqkv_weight 的成员变量，这是一个未初始化的结构体指针，指向一个未定义的名为 ggml_tensor 的类型，可能是表示注意力权重。
3. 一个名为 c_attn_out_proj_weight 的成员变量，这是一个未初始化的结构体指针，指向一个未定义的名为 ggml_tensor 的类型，可能是表示注意力后输出projection的权重。
4. 一个名为 norm_2_weight 的成员变量，这是一个未初始化的结构体指针，指向一个未定义的名为 ggml_tensor 的类型，可能是表示第二个归一化的权重。
5. 一个名为 ffn_up_proj 的成员变量，这是一个未初始化的结构体指针，指向一个未定义的名为 ggml_tensor 的类型，可能是表示特征上采样projection的权重。
6. 一个名为 ffn_down_proj 的成员变量，这是一个未初始化的结构体指针，指向一个未定义的名为 ggml_tensor 的类型，可能是表示特征下采样projection的权重。


```cpp
struct mpt_layer {
    // pre normalization
    struct ggml_tensor * norm_1_weight;

    // attention
    struct ggml_tensor * c_attn_wqkv_weight;
    struct ggml_tensor * c_attn_out_proj_weight;

    // post normalization
    struct ggml_tensor * norm_2_weight;

    // ff
    struct ggml_tensor * ffn_up_proj;
    struct ggml_tensor * ffn_down_proj;
};

```

这是一个结构体模型，其中包含了以下成员变量：

1. `mpt_hparams`：包含了模型设置，例如隐藏层数量、词向量大小等参数。
2. `wte_weight`：是一个指向位置嵌入向量的指针，用于表示文本中的位置信息。
3. `norm_f_weight`：是一个指向语言模型头的向量的指针，用于表示预处理阶段的信息。
4. `layers`：是一个容器，存储了模型中的所有层。
5. `memory_k`：是一个指向内存变量的指针，用于存储模型参数的 key。
6. `memory_v`：是一个指向内存变量的指针，用于存储模型参数的 value。
7. `ctx`：是一个指向上下文对象的指针，用于管理模型在计算过程中的计算上下文。
8. `tensors`：是一个哈希表，用于存储模型中的张量。

这个结构体模型是一个轻量级的不带标量的语言模型，它可以在需要时动态地加载和配置参数。它可以帮助我们构建一个灵活、可扩展的语言模型，以适应不同的应用场景。


```cpp
struct mpt_model {
    mpt_hparams hparams;

    struct ggml_tensor * wte_weight;    // position embedding
    struct ggml_tensor * norm_f_weight; // language model head

    std::vector<mpt_layer> layers;

    // key + value memory
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

```

这是一个结构体定义，表示一个用于自然语言处理(NLP)的工具函数mpt_params。

该结构体包含以下成员变量：

- n_threads：整数，表示预测任务将使用多少个线程。根据系统硬件线程限制，该值将最小化4，并使用std::min()函数获取。

- seed：整数，表示随机数种子。如果没有指定，将使用默认的随机数种子。

- n_predict：整数，表示预测的文本数量。

- n_batch：整数，表示批处理大小。

- n_ctx：整数，表示上下文大小。

- model：字符串，表示使用的神经网络模型。

- prompt：字符串，表示用于启动预测对话的文本。

- token_test：字符串，表示用于测试模型的文本。

- perplexity：布尔值，表示模型的效用指标是否使用衡量模型的性能。

- top_k：整数，表示提取的词或短语的个数。

- top_p：浮点数，表示提取的词或短语的概率。

- temp：浮点数，表示临时变量。

- repeat_last_n：整数，表示重复的文本中，最后一个可以重复的文本的个数。

- repeat_penalty：浮点数，表示重复文本的惩罚因子。

这些成员变量将用于在模型启动时进行设置，并在预测过程中提供所需的参数。


```cpp
struct mpt_params {
    int32_t n_threads = std::min(4, (int32_t) std::thread::hardware_concurrency());

    int32_t seed           = -1; // RNG seed
    int32_t n_predict      = 200; // new tokens to predict
    int32_t n_batch        = 8; // batch size for prompt processing
    int32_t n_ctx          = 512;

    std::string model      = ""; // model path
    std::string prompt     = "";
    std::string token_test = "";

    bool    perplexity     = false;

    // sampling parameters
    int32_t top_k          = 0;
    float   top_p          = 1.0f;
    float   temp           = 0.8f;
    int32_t repeat_last_n  = 64;
    float   repeat_penalty = 1.02f;

};

```

This is a command-line tool called `cli` that has several options for training and testing a language model. The available options are:

* `-f FNAME, --file FNAME`: specifies the file that contains the prompt (usually the start of a prompt)
* `-tt TOKEN_TEST, --token_test TOKEN_TEST`: specifies the token test to use
* `-n N, --n_predict N`: specifies the number of tokens to predict (default: `params.n_vocab`)
* `--top_k N             top-k sampling (default: `params.top_k`)
* `--top_p N             top-p sampling (default: `params.top_p`)
* `--temp N              temperature (default: `params.temp`)
* `--repeat-last-n N     last n tokens to consider for penalize (default: `params.repeat_last_n`)
* `--repeat-penalty N    penalize repeat sequence of tokens (default: `params.repeat_penalty`)
* `--perplexity          compute perplexity over the prompt
* `-c N, --ctx-size N    size of the prompt context (default: `params.n_ctx`)
* `-b N, --batch-size N  batch size for prompt processing (default: `params.n_batch`)
* `-m FNAME, --model FNAME`: specifies the path to the model
* `-p N        show model predictions (default: `params.top_p`)

Each option is optional, and if an option is not specified at the command-line, the default value is specified.

For example, to train the language model with the specified options, you would use:
```cpp
cli train --file <filename> --model <model_path>
```
This would train the language model using the specified model and the specified file. If the `-t` or `--token_test` option is specified, the model would also be tested with the specified token test.


```cpp
void mpt_print_usage(int /*argc*/, char ** argv, const mpt_params & params) {
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help            show this help message and exit\n");
    fprintf(stderr, "  -s SEED, --seed SEED  RNG seed (default: -1)\n");
    fprintf(stderr, "  -t N, --threads N     number of threads to use during computation (default: %d)\n", params.n_threads);
    fprintf(stderr, "  -p PROMPT, --prompt PROMPT\n");
    fprintf(stderr, "                        prompt to start generation with (default: random)\n");
    fprintf(stderr, "  -f FNAME, --file FNAME\n");
    fprintf(stderr, "                        load prompt from a file\n");
    fprintf(stderr, "  -tt TOKEN_TEST, --token_test TOKEN_TEST\n");
    fprintf(stderr, "                        test tokenization\n");
    fprintf(stderr, "  -n N, --n_predict N   number of tokens to predict (default: %d)\n", params.n_predict);
    fprintf(stderr, "  --top_k N             top-k sampling (default: %d, 0 = n_vocab)\n", params.top_k);
    fprintf(stderr, "  --top_p N             top-p sampling (default: %.2f)\n", params.top_p);
    fprintf(stderr, "  --temp N              temperature (default: %.2f)\n", params.temp);
    fprintf(stderr, "  --repeat-last-n N     last n tokens to consider for penalize (default: %d, 0 = disabled, -1 = ctx_size)\n", params.repeat_last_n);
    fprintf(stderr, "  --repeat-penalty N    penalize repeat sequence of tokens (default: %.2f, 1.0 = disabled)\n", (double)params.repeat_penalty);
    fprintf(stderr, "  --perplexity          compute perplexity over the prompt\n");
    fprintf(stderr, "  -c N, --ctx-size N    size of the prompt context (default: %d)\n", params.n_ctx);
    fprintf(stderr, "  -b N, --batch_size N  batch size for prompt processing (default: %d)\n", params.n_batch);
    fprintf(stderr, "  -m FNAME, --model FNAME\n");
    fprintf(stderr, "                        model path (default: %s)\n", params.model.c_str());
    fprintf(stderr, "\n");
}

```

This is a command-line tool called "Tokenizer" that takes in a list of parameters, including a filename to read the input from. The input can be either a text file or a string of parameters.

The tool has several parameters that can be used to control the behavior of the tool. The most used parameters are:

---repeat-last-n, the number of repeating words to keep track of, default 0
---repeat-penalty, the penalty for repeating a word, default 0
---perplexity, whether to use perplexity instead of true, default false
---ctx-size, the number of context words to use, default 0
---model, the name of the language model to use, default the default model
-h, the help message
-f, the filename to read the input from

The tool also has two positional parameters that can be used to read the input from the user:

---no-业， whether to use the default word instead of the industry-specific word
---test, whether to run the tool on a test or not

The tool takes command-line arguments and reads them using the `strcpy` function. The input is stored in the `params` vector, which has several elements indicating the different parameters. The elements in the vector are stored in the corresponding position of the `params` struct.


```cpp
bool mpt_params_parse(int argc, char ** argv, mpt_params & params) {
    for (int i = 1; i < argc; i++) {
        std::string arg = argv[i];

        if (arg == "-s" || arg == "--seed") {
            params.seed = std::stoi(argv[++i]);
        } else if (arg == "-t" || arg == "--threads") {
            params.n_threads = std::stoi(argv[++i]);
        } else if (arg == "-p" || arg == "--prompt") {
            params.prompt = argv[++i];
        } else if (arg == "-n" || arg == "--n_predict") {
            params.n_predict = std::stoi(argv[++i]);
        } else if (arg == "--top_k") {
            params.top_k = std::max(1, std::stoi(argv[++i]));
        } else if (arg == "--top_p") {
            params.top_p = std::stof(argv[++i]);
        } else if (arg == "--temp") {
            params.temp = std::stof(argv[++i]);
        } else if (arg == "--repeat-last-n") {
            params.repeat_last_n = std::stof(argv[++i]);
        } else if (arg == "--repeat-penalty") {
            params.repeat_penalty = std::stof(argv[++i]);
        } else if (arg == "--perplexity") {
            params.perplexity = true;
        } else if (arg == "-c" || arg == "--ctx-size") {
            params.n_ctx = std::stoi(argv[++i]);
        } else if (arg == "-b" || arg == "--batch_size") {
            params.n_batch = std::stoi(argv[++i]);
        } else if (arg == "-m" || arg == "--model") {
            params.model = argv[++i];
        } else if (arg == "-h" || arg == "--help") {
            mpt_print_usage(argc, argv, params);
            exit(0);
        } else if (arg == "-f" || arg == "--file") {
            if (++i > argc) {
                fprintf(stderr, "Invalid file param");
                break;
            }
            std::ifstream file(argv[i]);
            if (!file) {
                fprintf(stderr, "error: failed to open file '%s'\n", argv[i]);
                break;
            }
            params.prompt.clear();
            std::copy(std::istreambuf_iterator<char>(file), std::istreambuf_iterator<char>(), back_inserter(params.prompt));
            if (params.prompt.back() == '\n') {
                params.prompt.pop_back();
            }
        } else if (arg == "-tt" || arg == "--token_test") {
            params.token_test = argv[++i];
        } else {
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            mpt_print_usage(argc, argv, params);
            exit(0);
        }
    }

    return true;
}

```

This is a C function that parses a model file (written in the GGML format) and performs some checks on the input tensor. It prints some information about the tensor, the model file, and the total number of tensors.

Here's a brief summary of what this function does:

1. It parses the model file and extracts some information about the input tensor.
2. It checks whether the tensor's data matches the shape specified in the model file.
3. It checks whether the tensor has the expected number of elements.
4. It reads the tensor's data from the model file.
5. It calculates the total number of elements in the tensor.
6. It prints some information about the tensor, the model file, and the total number of tensors.

The function takes two arguments: a model file name and a tensor type. It returns a boolean indicating whether the tensor is valid. If the function fails, it prints an error message and returns `false`. If the tensor is valid, it returns `true`.


```cpp
// load the model's weights from a file
bool mpt_model_load(const std::string & fname, mpt_model & model, gpt_vocab & vocab) {
    printf("%s: loading model from '%s' - please wait ...\n", __func__, fname.c_str());

    auto fin = std::ifstream(fname, std::ios::binary);
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // verify magic
    {
        uint32_t magic;
        fin.read((char *)&magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // load hparams
    {
        auto & hparams = model.hparams;

        fin.read((char *) &hparams.d_model,        sizeof(hparams.d_model));
        fin.read((char *) &hparams.max_seq_len,    sizeof(hparams.max_seq_len));
        fin.read((char *) &hparams.n_heads,        sizeof(hparams.n_heads));
        fin.read((char *) &hparams.n_layers,       sizeof(hparams.n_layers));
        fin.read((char *) &hparams.n_vocab,        sizeof(hparams.n_vocab));
        fin.read((char *) &hparams.alibi_bias_max, sizeof(hparams.alibi_bias_max));
        fin.read((char *) &hparams.clip_qkv,       sizeof(hparams.clip_qkv));
        fin.read((char *) &hparams.ftype,          sizeof(hparams.ftype));

        hparams.n_ctx = std::min(hparams.max_seq_len, hparams.n_ctx);

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        printf("%s: d_model        = %d\n", __func__, hparams.d_model);
        printf("%s: max_seq_len    = %d\n", __func__, hparams.max_seq_len);
        printf("%s: n_ctx          = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_heads        = %d\n", __func__, hparams.n_heads);
        printf("%s: n_layers       = %d\n", __func__, hparams.n_layers);
        printf("%s: n_vocab        = %d\n", __func__, hparams.n_vocab);
        printf("%s: alibi_bias_max = %f\n", __func__, hparams.alibi_bias_max);
        printf("%s: clip_qkv       = %f\n", __func__, hparams.clip_qkv);
        printf("%s: ftype          = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr          = %d\n", __func__, qntvr);

        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // load vocab
    {
        const int32_t n_vocab = model.hparams.n_vocab;

        std::string word;
        std::vector<char> buf(128);

        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            // Convert token from utf-8
            std::wstring word_multibytes = convert_to_wstring(word);
            word.resize(word_multibytes.size());
            for (size_t w = 0; w < word_multibytes.size(); w++) {
                word[w] = uint8_t(word_multibytes[w]);
            }

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // for the big tensors, we have the option to store the data in 16-bit
    // floats or quantized in order to save memory and also to speed up the
    // computation
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype)(model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n", __func__, fname.c_str(),
                model.hparams.ftype);
        return false;
    }

    auto & ctx = model.ctx;

    size_t ctx_size = 0;

    const auto & hparams = model.hparams;
    const size_t n_ctx = hparams.n_ctx;

    {
        const size_t n_embd = hparams.d_model;
        const size_t n_layer = hparams.n_layers;
        const size_t n_vocab = hparams.n_vocab;

        ctx_size += n_embd * n_vocab * ggml_type_sizef(wtype); // wte_weight
        ctx_size += n_embd * ggml_type_sizef(GGML_TYPE_F32);   // norm_f_weight

        ctx_size += n_layer * (n_embd * ggml_type_sizef(GGML_TYPE_F32));      // ln_1_weight
        ctx_size += n_layer * (3 * n_embd * n_embd * ggml_type_sizef(wtype)); // attn_Wqkv_weight
        ctx_size += n_layer * (n_embd * n_embd * ggml_type_sizef(wtype));     // attn_out_proj_weight
        ctx_size += n_layer * (n_embd * ggml_type_sizef(GGML_TYPE_F32));      // ln_2_weight
        ctx_size += n_layer * (4 * n_embd * n_embd * ggml_type_sizef(wtype)); // mlp_mlp_up_weight
        ctx_size += n_layer * (n_embd * n_embd * 4 * ggml_type_sizef(wtype)); // mlp_mlp_down_weight

        ctx_size += n_ctx * n_layer * n_embd * ggml_type_sizef(GGML_TYPE_F16); // memory_k
        ctx_size += n_ctx * n_layer * n_embd * ggml_type_sizef(GGML_TYPE_F16); // memory_v

        ctx_size += (1 + 6 * n_layer) * 512; // object overhead

        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size / (1024.0 * 1024.0));
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

        const size_t n_embd = hparams.d_model;
        const size_t n_layer = hparams.n_layers;
        const size_t n_vocab = hparams.n_vocab;

        model.layers.resize(n_layer);

        model.wte_weight    = ggml_new_tensor_2d(ctx, wtype, n_embd, n_vocab);
        model.norm_f_weight = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        // map by name
        model.tensors["transformer.wte.weight"]    = model.wte_weight;
        model.tensors["transformer.norm_f.weight"] = model.norm_f_weight;

        for (int i = 0; i < (int) n_layer; ++i) {
            auto & layer = model.layers[i];

            layer.norm_1_weight          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,     n_embd);
            layer.c_attn_wqkv_weight     = ggml_new_tensor_2d(ctx, wtype,             n_embd, 3 * n_embd);
            layer.c_attn_out_proj_weight = ggml_new_tensor_2d(ctx, wtype,             n_embd,     n_embd);
            layer.norm_2_weight          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,     n_embd);
            layer.ffn_up_proj            = ggml_new_tensor_2d(ctx, wtype,             n_embd, 4 * n_embd);
            layer.ffn_down_proj          = ggml_new_tensor_2d(ctx, wtype,         4 * n_embd,     n_embd);

            // map by name
            model.tensors["transformer.blocks." + std::to_string(i) + ".norm_1.weight"]        = layer.norm_1_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".attn.Wqkv.weight"]     = layer.c_attn_wqkv_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".attn.out_proj.weight"] = layer.c_attn_out_proj_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".norm_2.weight"]        = layer.norm_2_weight;
            model.tensors["transformer.blocks." + std::to_string(i) + ".ffn.up_proj.weight"]   = layer.ffn_up_proj;
            model.tensors["transformer.blocks." + std::to_string(i) + ".ffn.down_proj.weight"] = layer.ffn_down_proj;
        }
    }

    // key + value memory
    {
        const auto & hparams = model.hparams;

        const size_t n_embd  = hparams.d_model;
        const size_t n_layer = hparams.n_layers;

        const int64_t n_mem      = n_layer * n_ctx;
        const int64_t n_elements = n_embd  * n_mem;

        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);

        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);

        printf("%s: memory_size = %8.2f MB, n_mem = %" PRId64 "\n", __func__, memory_size / 1024.0 / 1024.0, n_mem);
    }

    // load weights
    {
        int n_tensors = 0;
        size_t total_size = 0;

        printf("%s: ", __func__);

        while (true) {
            int32_t n_dims;
            int32_t length;
            int32_t ttype;

            fin.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));
            fin.read(reinterpret_cast<char *>(&length), sizeof(length));
            fin.read(reinterpret_cast<char *>(&ttype), sizeof(ttype));

            if (fin.eof()) {
                break;
            }

            int32_t nelements = 1;
            int32_t ne[2] = {1, 1};
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
                fprintf(stderr,
                        "%s: tensor '%s' has wrong shape in model file: got [%5d, "
                        "%5d], expected [%5d, %5d]\n",
                        __func__, name.c_str(), (int)tensor->ne[0], (int)tensor->ne[1], ne[0], ne[1]);
                return false;
            }

            // for debugging
            if (0) {
                printf("%24s - [%5d, %5d], type = %6s, %6.2f MB, %9zu bytes\n", name.c_str(), ne[0], ne[1],
                       ggml_type_name(ggml_type(ttype)), ggml_nbytes(tensor) / 1024.0 / 1024.0, ggml_nbytes(tensor));
            }

            const size_t bpe = ggml_type_size(ggml_type(ttype));

            if ((nelements * bpe) / ggml_blck_size(tensor->type) != ggml_nbytes(tensor)) {
                fprintf(stderr,
                        "%s: tensor '%s' has wrong size in model file: got %zu, "
                        "expected %zu\n",
                        __func__, name.c_str(), ggml_nbytes(tensor), nelements * bpe);
                return false;
            }

            fin.read(reinterpret_cast<char *>(tensor->data), ggml_nbytes(tensor));

            total_size += ggml_nbytes(tensor);
            if (++n_tensors % 8 == 0) {
                printf(".");
                fflush(stdout);
            }
        }

        printf(" done\n");

        printf("%s: model size = %8.2f MB / num tensors = %d\n", __func__, total_size / 1024.0 / 1024.0, n_tensors);
    }

    fin.close();

    return true;
}

```

This function appears to implement a function called `ggml_mul_token_scaled` which performs multi-token multiplication of a pre-trained model with a weighted token embedding. The input to the function is a two-dimensional tensor `Qcur`, representing the output embeddings for a fixed vocabulary word, and a one-dimensional tensor `inpL`, representing the input embeddings for a fixed vocabulary word.

The function returns an optional integer `Qcur_dim` which represents the dimensionality of the output embeddings. If the dimensionality is not specified, the default value is `4096`, which is the same as the default dimensionality of the pre-trained model.

The function performs the following operations:

1. It creates a temporary variable `temp` of type `float` and sets its value to `0`.
2. It creates a new variable `Qcur_dim` of the same type as `Qcur` and initializes its value to `4096`.
3. It multiplies `Qcur_dim` by the input tensor `inpL` using a weighted dot product operation.
4. It returns the result of the multiplication.

Note that the input to the function must have the same shape as the input tensor `inpL`, and that the input tensor must have a non-empty `Qcur` dimension.


```cpp
// evaluate the transformer
//
//   - model:     the model
//   - n_threads: number of threads to use
//   - n_past:    the context size so far
//   - embd_inp:  the embeddings of the tokens in the context
//   - embd_w:    the predicted logits for the next token
//
bool mpt_eval(const mpt_model & model, const int n_threads, const int n_past,
              const std::vector<gpt_vocab::id> & embd_inp, std::vector<float> & embd_w, bool logits_all, size_t & mem_per_token) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_embd  = hparams.d_model;
    const int n_layer = hparams.n_layers;
    const int n_head  = hparams.n_heads;
    const int n_vocab = hparams.n_vocab;
    const int n_ctx   = hparams.n_ctx;
    const float eps   = 1e-5f;

    static size_t buf_size = 256u * 1024 * 1024;
    static void * buf = malloc(buf_size);

    // use 2 scratch buffers
    // TODO: very hacky solution - reimplement in a more elegant way
    static size_t scr0_size = 256u*1024*1024;
    static void * scr0 = malloc(scr0_size);

    static size_t scr1_size = 256u*1024*1024;
    static void * scr1 = malloc(scr1_size);

    if (mem_per_token > 0 && mem_per_token * N > buf_size) {
        const size_t buf_size_new = 1.1 * (mem_per_token * N); // add 10% to account for ggml object overhead
        // printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__,
        // buf_size, buf_size_new);

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
    memcpy(embd->data, embd_inp.data(), N * ggml_element_size(embd));

    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model.wte_weight, embd);

    for (int il = 0; il < n_layer; ++il) {

        struct ggml_tensor * cur;

        ggml_set_scratch(ctx0, { 0, scr0_size, scr0, });

        // a = self.ln_1(x)
        {
            cur = ggml_norm(ctx0, inpL, eps);

            cur = ggml_mul(ctx0, ggml_repeat(ctx0, model.layers[il].norm_1_weight, cur), cur);
        }

        // self-attention
        //  b, _, past_key_value = self.attn(a, past_key_value=past_key_value,
        //  attn_bias=attn_bias, attention_mask=attention_mask,
        //  is_causal=is_causal)
        {
            // compute QKV
            cur = ggml_mul_mat(ctx0, model.layers[il].c_attn_wqkv_weight, cur);

            if (model.hparams.clip_qkv > 0.0f) {
                cur = ggml_clamp(ctx0, cur, -model.hparams.clip_qkv, model.hparams.clip_qkv);
            }

            struct ggml_tensor * Qcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 0 * sizeof(float) * n_embd);
            struct ggml_tensor * Kcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 1 * sizeof(float) * n_embd);
            struct ggml_tensor * Vcur = ggml_view_2d(ctx0, cur, n_embd, N, cur->nb[1], 2 * sizeof(float) * n_embd);

            // store key and value to memory
            {
                struct ggml_tensor * k =
                    ggml_view_1d(ctx0, model.memory_k, N * n_embd,
                                 (ggml_element_size(model.memory_k) * n_embd) * (il * n_ctx + n_past));
                struct ggml_tensor * v =
                    ggml_view_1d(ctx0, model.memory_v, N * n_embd,
                                 (ggml_element_size(model.memory_v) * n_embd) * (il * n_ctx + n_past));

                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));
            }

            // Q = Qcur.contiguous().view(n_embd/n_head, n_head, N).permute(0,
            // 2, 1, 3) [64, N, 12]
            struct ggml_tensor * Q = ggml_permute(
                ctx0, ggml_cpy(ctx0, Qcur, ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_embd / n_head, n_head, N)), 0, 2,
                1, 3);

            // K = Kmem.view(n_embd/n_head, n_head, n_past + N).permute(0, 2, 1,
            // 3) [64, n_past + N, 12]
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                             ggml_reshape_3d(ctx0,
                                             ggml_view_1d(ctx0, model.memory_k, (n_past + N) * n_embd,
                                                          il * n_ctx * ggml_element_size(model.memory_k) * n_embd),
                                             n_embd / n_head, n_head, n_past + N),
                             0, 2, 1, 3);
            // K * Q
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            // KQ_scaled = KQ / sqrt(n_embd/n_head)
            struct ggml_tensor * KQ_scaled =
                ggml_scale(ctx0, KQ, ggml_new_f32(ctx0, 1.0f / sqrt(float(n_embd) / n_head)));

            struct ggml_tensor * KQ_scaled_alibi =
                ggml_alibi(ctx0, KQ_scaled, n_past, n_head, model.hparams.alibi_bias_max);

            // KQ_masked = mask_past(KQ_scaled)
            struct ggml_tensor * KQ_masked = ggml_diag_mask_inf(ctx0, KQ_scaled_alibi, n_past);

            // KQ = soft_max(KQ_masked)
            struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_masked);

            // V_trans = Vmem.view(n_embd/n_head, n_head, n_past + N).permute(1,
            // 2, 0, 3).contiguous() [n_past + N, 64, 12]
            struct ggml_tensor * V_trans = ggml_cpy(
                ctx0,
                ggml_permute(ctx0,
                             ggml_reshape_3d(ctx0,
                                             ggml_view_1d(ctx0, model.memory_v, (n_past + N) * n_embd,
                                                          il * n_ctx * ggml_element_size(model.memory_v) * n_embd),
                                             n_embd / n_head, n_head, n_past + N),
                             1, 2, 0, 3),
                ggml_new_tensor_3d(ctx0, model.memory_v->type, n_past + N, n_embd / n_head, n_head));

            // KQV = transpose(V) * KQ_soft_max
            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V_trans, KQ_soft_max);

            // KQV_merged = KQV.permute(0, 2, 1, 3)
            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            // cur = KQV_merged.contiguous().view(n_embd, N)
            cur = ggml_cpy(ctx0, KQV_merged, ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_embd, N));

            // projection
            { cur = ggml_mul_mat(ctx0, model.layers[il].c_attn_out_proj_weight, cur); }
        }

        inpL = ggml_add(ctx0, inpL, cur);

        ggml_set_scratch(ctx0, { 0, scr1_size, scr1, });

        // m = self.ln_2(x)
        {
            cur = ggml_norm(ctx0, inpL, eps);

            cur = ggml_mul(ctx0, ggml_repeat(ctx0, model.layers[il].norm_2_weight, cur), cur);
        }

        // n = self.mlp(m)
        {

            cur = ggml_mul_mat(ctx0, model.layers[il].ffn_up_proj, cur);

            // GELU activation
            cur = ggml_gelu(ctx0, cur);

            // projection
            // cur = proj_w*cur + proj_b
            cur = ggml_mul_mat(ctx0, model.layers[il].ffn_down_proj, cur);
        }

        // x = x + n
        inpL = ggml_add(ctx0, inpL, cur);
    }

    ggml_set_scratch(ctx0, { 0, scr0_size, scr0, });

    // norm
    {
        inpL = ggml_norm(ctx0, inpL, eps);
        // inpL = ln_f_g*inpL
        inpL = ggml_mul(ctx0, ggml_repeat(ctx0, model.norm_f_weight, inpL), inpL);
    }

    ggml_set_scratch(ctx0, { 0, 0, nullptr, });

    // output embedding weight tied to input embedding
    inpL = ggml_mul_mat(ctx0, model.wte_weight, inpL);

    // logits -> probs
    // inpL = ggml_soft_max(ctx0, inpL);

    // run the computation
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // std::cout << "Qcur" << std::endl;
    // print_tensor(Qcur);

    // if (n_past%100 == 0) {
    // ggml_graph_print(&gf);
    // ggml_graph_dump_dot(&gf, NULL, "mpt-model.dot");
    // }

    if (logits_all) {
        // return result for all tokens
        embd_w.resize(n_vocab *N);
        memcpy(embd_w.data(), (float *)ggml_get_data(inpL) , sizeof(float) * n_vocab * N);
    } else {
        // return result for just the last token
        embd_w.resize(n_vocab);
        memcpy(embd_w.data(), (float *)ggml_get_data(inpL) + (n_vocab * (N - 1)), sizeof(float) * n_vocab);
    }

    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0) / N;
    }
    // printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    ggml_free(ctx0);

    return true;
}

```

这段代码定义了一个名为 `softmax` 的函数，它接受一个 `std::vector<float>` 类型的输入参数 `logits`。

函数首先创建了一个名为 `probs` 的 `std::vector<float>` 类型的变量，该变量具有与输入参数 `logits` 相同的大小。然后，函数初始化 `probs` 中的所有元素为输入参数 `logits` 中的最大 logit 值。

接下来，函数遍历输入参数 `logits`。对于每个输入值 `v`，函数首先计算该输入的 logit 值，然后将其与之前计算的最大 logit 值相比较。如果当前输入的 logit 值大于最大 logit 值，则将其与最大 logit 值相等。

接着，函数计算输入参数 `logits` 中每个元素的归一化概率。具体地，函数将输入元素的 logit 值除以它们的累积归一化因子，使得它们的和为 1。这样，函数可以计算出每个元素在概率分布中的权重。

最后，函数返回生成的 `probs` 向量。


```cpp
std::vector<float> softmax(const std::vector<float> & logits) {
    std::vector<float> probs(logits.size());
    float max_logit = logits[0];
    for (float v : logits) max_logit = std::max(max_logit, v);
    double sum_exp = 0.0;
    for (size_t i = 0; i < logits.size(); i++) {
        // Subtract the maximum logit value from the current logit value for numerical stability
        const float logit = logits[i] - max_logit;
        const float exp_logit = expf(logit);
        sum_exp += exp_logit;
        probs[i] = exp_logit;
    }
    for (size_t i = 0; i < probs.size(); i++) probs[i] /= sum_exp;
    return probs;
}

```

This is a C++ implementation of the ELMo model training and evaluation function. It performs three operations: 

1. Forward pass: It calculates the probabilities of all possible next tokens given the previous ones.
2. Backward pass: It calculates the negative log-likelihood (NLL) and the total log-likelihood (NLLCHUNK) for each chunk of tokens.
3. Average negative log-likelihood per token: It calculates the average negative log-likelihood per token.

The function takes in two parameters:

* `params.n_vocab`: The number of vocabulary tokens.
* `params.n_ctx`: The number of context tokens.

It returns two outputs:

* The first one is a tuple containing the average negative log-likelihood and the total log-likelihood for all chunks of tokens.
* The second one is the average negative log-likelihood for all tokens.

The function uses the followingfflush(stdout):


```cpp
int perplexity(const mpt_params & params) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    printf("%s: n_threads = %d\n", __func__, params.n_threads);
    printf("%s: n_batch   = %d\n", __func__, params.n_batch);
    printf("%s: n_ctx     = %d\n", __func__, params.n_ctx);
    printf("\n");

    int64_t t_load_us = 0;

    gpt_vocab vocab;
    mpt_model model;

    model.hparams.n_ctx = params.n_ctx;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!mpt_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;
    }

    int64_t t_predict_us = 0;

    std::vector<float> logits;

    // tokenize the prompt
    std::vector<int> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());

    // determine the required inference memory per token:
    size_t mem_per_token = 0;
    mpt_eval(model, params.n_threads, 0, {0, 1, 2, 3}, logits, false, mem_per_token);

    int count   = 0;

    const int n_chunk = embd_inp.size() / params.n_ctx;

    const int n_vocab = model.hparams.n_vocab;
    const int n_batch = params.n_batch;

    double nll = 0.0;
    fprintf(stderr, "%s: calculating perplexity over %d chunks, batch_size=%d\n", __func__, n_chunk, n_batch);

    for (int i = 0; i < n_chunk; ++i) {

        const int start =     i * params.n_ctx;
        const int end   = start + params.n_ctx;

        const int num_batches = (params.n_ctx + n_batch - 1) / n_batch;

        std::vector<float> logits;

        const auto t_start = std::chrono::high_resolution_clock::now();

        for (int j = 0; j < num_batches; ++j) {

            const int batch_start = start + j * n_batch;
            const int batch_size  = std::min(end - batch_start, n_batch);

            std::vector<gpt_vocab::id> embd;

            for(int p=0;p<batch_size;p++) {
                embd.push_back( embd_inp[batch_start+p]  );
            }

            std::vector<float> batch_logits;// = llama_get_logits(ctx);

            const int64_t t_start_us = ggml_time_us();

            if (!mpt_eval(model, params.n_threads, j * batch_size, embd, batch_logits, true, mem_per_token)) {
                printf("%s: failed to evaluate model\n", __func__);
                return 1;
            }

            t_predict_us += ggml_time_us() - t_start_us;

            logits.insert(logits.end(), batch_logits.data(), batch_logits.data() + batch_size * n_vocab);

        }

        const auto t_end = std::chrono::high_resolution_clock::now();

        if (i == 0) {
            const float t_total = std::chrono::duration<float>(t_end - t_start).count();
            fprintf(stderr, "%s: %.2f seconds per pass - ETA ", __func__, t_total);
            int total_seconds = (int)(t_total * n_chunk);
            if (total_seconds >= 60*60) {
                fprintf(stderr, "%d hours ", total_seconds / (60*60));
                total_seconds = total_seconds % (60*60);
            }
            fprintf(stderr, "%d minutes\n", total_seconds / 60);

            printf("\nChunk\tPPL cumulative\tPPL chunk\n");
        }

        // We get the logits for all the tokens in the context window (params.n_ctx)
        // from llama_eval above.  Now, based on https://huggingface.co/docs/transformers/perplexity,
        // calculate the perplexity over the last half of the window (so the model always has
        // some context to predict the token).
        //
        // We rely on the fact that attention in the forward pass only looks at previous
        // tokens here, so the logits returned for each token are an accurate representation
        // of what the model would have predicted at that point.
        //
        // Example, we have a context window of 512, we will compute perplexity for each of the
        // last 256 tokens.  Then, we split the input up into context window size chunks to
        // process the entire prompt.

        double nllchunk = 0.0;
        int countchunk = 0;

        for (int j = std::min(512, params.n_ctx / 2); j < params.n_ctx - 1; ++j) {
            // Calculate probability of next token, given the previous ones.
            const std::vector<float> tok_logits(
                logits.begin() + (j + 0) * n_vocab,
                logits.begin() + (j + 1) * n_vocab);

            const float prob = softmax(tok_logits)[embd_inp[ start+ j + 1]];

            nllchunk += -std::log(prob);
            ++countchunk;
        }

		nll += nllchunk;
		count += countchunk;

        // perplexity is e^(average negative log-likelihood)
        printf("%d\t%.8lf\t%.8lf\n", i + 1, std::exp(nll / count), std::exp(nllchunk/countchunk) );
        fflush(stdout);
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n");
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        printf("%s:     load time = %8.2f ms\n",   __func__, t_load_us / 1000.0f);
        printf("%s:     eval time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us / 1000.0f, t_predict_us / 1000.0f / (n_chunk * params.n_ctx));
        printf("%s:    total time = %8.2f ms\n",   __func__, (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    ggml_free(model.ctx);

    return 0;
}

```

This is a C++ program that reads in pre-trained word embeddings for a language model and uses them to generate text. It reads in the pre-trained embeddings for a fixed number of tokens in a sentence and then uses the embedded words to predict the next word in the text.

The program takes in an input string of pre-trained embeddings and outputs the predicted text. It uses a vocabulary of pre-trained words and a maximum batch size to read in the pre-trained embeddings.

The program reads in the pre-trained embeddings for a fixed number of tokens in a sentence and then uses the embedded words to predict the next word in the text. It uses a learning rate of 0.01 and a number of training iterations of 10.

The program outputs the time taken to process the input and the total time taken to process the input. It also outputs the memory usage and the load time of the pre-trained embeddings.

This program can be useful for understanding how a language model uses pre-trained embeddings to generate text.


```cpp
int main(int argc, char ** argv) {
    mpt_params params;

    if (mpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    if (params.perplexity) {
        return perplexity(params);
    }

    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    if (params.n_predict < 0) {
        params.n_predict = 0;
    }

    printf("%s: seed      = %d\n",   __func__, params.seed);
    printf("%s: n_threads = %d\n",   __func__, params.n_threads);
    printf("%s: n_batch   = %d\n",   __func__, params.n_batch);
    printf("%s: n_ctx     = %d\n",   __func__, params.n_ctx);
    printf("%s: n_predict = %d\n\n", __func__, params.n_predict);

    std::mt19937 rng(params.seed);
    if (params.prompt.empty()) {
        params.prompt = gpt_random_prompt(rng);
    }

    int64_t t_load_us = 0;

    gpt_vocab vocab;
    mpt_model model;

    model.hparams.n_ctx = params.n_ctx;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!mpt_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;

        test_gpt_tokenizer(vocab, params.token_test);
    }

    if (params.top_k == 0) {
        params.top_k = model.hparams.n_vocab;
    }

    if (params.repeat_last_n == -1) {
        params.repeat_last_n = params.n_ctx;
    }

    printf("\n");
    printf("%s: temp           = %.3f\n", __func__, params.temp);
    printf("%s: top_k          = %d\n",   __func__, params.top_k);
    printf("%s: top_p          = %.3f\n", __func__, params.top_p);
    printf("%s: repeat_last_n  = %d\n",   __func__, params.repeat_last_n);
    printf("%s: repeat_penalty = %.3f\n", __func__, params.repeat_penalty);

    int64_t t_sample_us = 0;
    int64_t t_predict_us = 0;

    std::vector<int32_t> last_n_tokens(params.n_ctx);
    std::fill(last_n_tokens.begin(), last_n_tokens.end(), 0);

    // tokenize the prompt
    std::vector<int> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    printf("\n");
    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());

    for (size_t i = 0; i < embd_inp.size(); i++) {
        printf("%s: token[%zu] = %6d\n", __func__, i, embd_inp[i]);
    }
    printf("\n");

    std::vector<gpt_vocab::id> embd;
    std::vector<float> logits;

    // determine the required inference memory per token:
    size_t mem_per_token = 0;
    mpt_eval(model, params.n_threads, 0, {0, 1, 2, 3}, logits, false, mem_per_token);

    int n_past     = 0;
    int n_consumed = 0;
    int n_sampled  = 0;

    while (n_sampled < params.n_predict) {
        // predict
        if (embd.size() > 0) {
            const int64_t t_start_us = ggml_time_us();

            if (!mpt_eval(model, params.n_threads, n_past, embd, logits, false, mem_per_token)) {
                printf("%s: failed to predict\n", __func__);
                return 1;
            }

            t_predict_us += ggml_time_us() - t_start_us;

            n_past += embd.size();
            embd.clear();
        }

        if ((int)embd_inp.size() <= n_consumed) {
            // sample next token

            const int top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp = params.temp;
            const int repeat_last_n = params.repeat_last_n;
            const float repeat_penalty = params.repeat_penalty;

            gpt_vocab::id id = 0;

            {
                const int64_t t_start_sample_us = ggml_time_us();

                id = gpt_sample_top_k_top_p_repeat(vocab, logits.data() + (logits.size() - model.hparams.n_vocab), last_n_tokens.data(), last_n_tokens.size(), top_k, top_p, temp, repeat_last_n, repeat_penalty, rng);

                last_n_tokens.erase(last_n_tokens.begin());
                last_n_tokens.push_back(id);

                t_sample_us += ggml_time_us() - t_start_sample_us;
            }

            // add it to the context
            embd.push_back(id);
            ++n_sampled;

        } else {
            // if here, it means we are still processing the input prompt
            while ((int) embd_inp.size() > n_consumed) {
                embd.push_back(embd_inp[n_consumed]);

                last_n_tokens.erase(last_n_tokens.begin());
                last_n_tokens.push_back(embd_inp[n_consumed]);

                ++n_consumed;
                if ((int) embd.size() >= params.n_batch) {
                    break;
                }
            }
        }

        // display text
        for (auto id : embd) {
           printf("%s", vocab.id_to_token[id].c_str());
        }
        fflush(stdout);

        // end of text token
        if (embd.back() == 0) {
            break;
        }
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n\n");
        printf("%s: sampled tokens = %8d\n", __func__, n_sampled);
        printf("%s:  mem per token = %8zu bytes\n", __func__, mem_per_token);
        printf("%s:      load time = %8.2f ms\n", __func__, t_load_us / 1000.0f);
        printf("%s:    sample time = %8.2f ms / %.2f ms per token\n", __func__, t_sample_us / 1000.0f, t_sample_us / 1000.0f / n_sampled);
        printf("%s:      eval time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us / 1000.0f, t_predict_us / 1000.0f / n_past);
        printf("%s:     total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    ggml_free(model.ctx);

    return 0;
}

```