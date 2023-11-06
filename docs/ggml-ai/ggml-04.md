# GGML源码解析 4

# `examples/dolly-v2/convert-h5-to-ggml.py`

这段代码的作用是将HDF5格式的训练数据转换为GGML格式的文件，并输出到同一目录中。它主要分为以下几个步骤：

1. 导入所需的库：sys、struct、json、numpy、transformers。
2. 定义了一个变量dir_model，表示训练数据的目录。
3. 如果命令行参数数量小于3个，则输出 usage信息并退出。
4. 设置ftype参数为0时为float32，为1时为float16。
5. 导入AutoModelForCausalLM和AutoTokenizer，来自transformers库。
6. 如果dir_model参数不含有模型文件，则询问用户是否要指定模型文件。
7. 设置dir_model参数为模型文件的目录后，开始转换HDF5文件到GGML格式。


```cpp
import sys
import struct
import json
import numpy as np

from transformers import AutoModelForCausalLM, AutoTokenizer

if len(sys.argv) < 3:
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)

# output in the same directory as the model
dir_model = sys.argv[1]
```

这段代码的作用是读取两个 JSON 文件并输出一个名为 "fname_out" 的文件名，同时读取第二个 JSON 文件中的配置信息。

具体来说，代码首先读取一个名为 "dir_model" 的目录中的第一个命令行参数（即第一个参数）和一个名为 "tokenizer.json" 的文件，并将其中的 JSON 数据读取到 encoder 变量中。接着，代码又读取一个名为 "dir_model" 的目录中的第二个命令行参数（即第二个参数）和一个名为 "config.json" 的文件，并将其中的 JSON 数据读取到 hparams 变量中。

接着，代码定义了一个 ftype_str 类型，用于存储可能的数据类型及其对应的文件名。该类型包含两个元素，分别对应 ftype 为 0 和 ftype 为 1 的情况。

最后，根据 ftype 类型，代码读取 ftype_str 中的对应文件名，并将这些文件名添加到 fname_out 中，并将 fname_out 写入到当前工作目录下的 "dir_model" 目录中。


```cpp
fname_out = sys.argv[1] + "/ggml-model.bin"

with open(dir_model + "/tokenizer.json", "r", encoding="utf-8") as f:
    encoder = json.load(f)

with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

# possible data types
#   ftype == 0 -> float32
#   ftype == 1 -> float16
#
# map from ftype to string
ftype_str = ["f32", "f16"]

```

这段代码的作用是定义一个 ftype 变量并检查用户输入的参数，如果参数个数大于 2，则从用户输入的第二个参数中提取 ftype 类型并检查它是否为负数或大于 1，如果是，则输出一条错误信息并退出程序。否则，尝试加载从第一个参数中提取的文件名，并使用 AutoModelForCausalLM 和 AutoTokenizer 对文件进行预处理，最后输出模型的文件名。

具体来说，代码首先定义了一个 ftype 变量，然后使用 if len(sys.argv) > 2 作为判断是否需要从第二个参数中提取 ftype 的条件，如果条件成立，则从第二个参数中提取 ftype 类型并检查它是否为负数或大于 1，如果是，则输出一条错误信息并退出程序，否则继续执行后面的操作。接着，定义了一个 fname_out 变量，用于输出模型文件名，然后使用 AutoModelForCausalLM 和 AutoTokenizer 对文件进行预处理，最后输出模型的文件名。


```cpp
ftype = 1
if len(sys.argv) > 2:
    ftype = int(sys.argv[2])
    if ftype < 0 or ftype > 1:
        print("Invalid ftype: " + str(ftype))
        sys.exit(1)
    fname_out = sys.argv[1] + "/ggml-model-" + ftype_str[ftype] + ".bin"


tokenizer = AutoTokenizer.from_pretrained(dir_model)
model = AutoModelForCausalLM.from_pretrained(dir_model, low_cpu_mem_usage=True)
#print (model)

#print(tokenizer.encode('I believe the meaning of life is'))

```

这段代码的主要作用是输出一个文本文件，文件名为指定的输出文件（fname_out）。其内容为：

1. 定义了一个名为list_vars的变量，该变量是一个PyTorch中的state_dict，表示模型在预训练过程中的状态信息。
2. 通过循环遍历list_vars中的所有键（即模型的state_dict中所有的key），输出了每个key的名称、该key的形状（即形状）和该key的数据类型。
3. 在循环的下一个行中，通过`struct.pack`函数输出了一个包含6个字节（即两个int类型的值）的固定格式，该格式根据key的类型进行包装。
4. 通过文件操作，创建并写入了指定文件名（fname_out）的文件中。
5. 在写入文件内容的代码中，使用了PyTorch中的`struct.pack`函数，将不同类型的数据进行包装，包括一个int类型的值，一个两个int类型的值的包装，以及一个float类型的值。


```cpp
list_vars = model.state_dict()
for name in list_vars.keys():
    print(name, list_vars[name].shape, list_vars[name].dtype)

fout = open(fname_out, "wb")

print(hparams)

fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("i", hparams["max_position_embeddings"]))
fout.write(struct.pack("i", hparams["hidden_size"]))
fout.write(struct.pack("i", hparams["num_attention_heads"]))
fout.write(struct.pack("i", hparams["num_hidden_layers"]))
fout.write(struct.pack("i", int(hparams["rotary_pct"]*(hparams["hidden_size"]//hparams["num_attention_heads"]))))
```

This code looks at a file called `data.txt`, which is assumed to contain a list of variables with variable names and numerical data.

The code reads the data and writes it to a file called `output.txt`. The file is formatted with an integer header containing the shape of the data, followed by a block of the data.

The code also formats special variable names that end in ".attention.masked_bias" or ".attention.bias" or ".attention.rotary_emb.inv_freq". These variables are skipped in the processing.

The code uses the `numpy` function to convert the data to a NumPy array, and then uses the `astype` function to convert the data to the correct data type (float16 or float32). The `ftype` variable is set based on the data type, and the `Converting to float16` or `Converting to float32` method is used depending on the data type.


```cpp
fout.write(struct.pack("i", hparams["use_parallel_residual"]))
fout.write(struct.pack("i", ftype))

# TODO: temporary hack to not deal with implementing the tokenizer
dot_token = tokenizer.encode('.')[0]
for i in range(hparams["vocab_size"]):
    text = tokenizer.decode([dot_token, i]).encode('utf-8')
    # remove the first byte (it's always '.')
    text = text[1:]
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

for name in list_vars.keys():
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " + name + " with shape: ", data.shape)

    # we don't need these
    if name.endswith(".attention.masked_bias") or     \
       name.endswith(".attention.bias") or \
       name.endswith(".attention.rotary_emb.inv_freq"):
        print("  Skipping variable: " + name)
        continue

    n_dims = len(data.shape);

    # ftype == 0 -> float32, ftype == 1 -> float16
    ftype_cur = 0;
    if ftype != 0:
        if name[-7:] == ".weight" and n_dims == 2:
            print("  Converting to float16")
            data = data.astype(np.float16)
            ftype_cur = 1
        else:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0
    else:
        if data.dtype != np.float32:
            print("  Converting to float32")
            data = data.astype(np.float32)
            ftype_cur = 0

    # header
    str = name.encode('utf-8')
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    fout.write(str);

    # data
    data.tofile(fout)

```

这段代码的作用是关闭名为 "fout" 的文件，并输出一个消息，指出文件已经完成。然后关闭屏幕。

代码中使用了两个函数：`fout.close()` 和 `print()`。`fout.close()` 函数用于关闭文件输出流(即 "fout")，这是在写文件时使用的。 `print()` 函数则用于在屏幕上输出一些信息。第一个 `print()` 函数输出了一行消息，指出文件已经完成。第二个 `print()` 函数用于关闭屏幕，这样程序就不会继续输出更多的信息。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/dolly-v2/main.cpp`

这段代码是一个通用的C++头文件，它包含了多个自定义的函数和变量，以下是对代码中各部分的解释：

1. `#include "ggml/ggml.h"`：引入了ggml库，ggml是一个C++的机器学习库，它可以轻松地进行数据预处理、训练和评估机器学习模型。

2. `#include "common.h"`：引入了common库，这是一个通用的C++库，提供了许多通用的函数和变量，方便用户进行开发和调试。

3. `#include "common-ggml.h"`：引入了ggml库中的common-ggml库，这个库是ggml的子库，提供了更多的便捷函数，方便用户进行机器学习模型的训练和评估。

4. `#include <cassert>`：引入了cassert库，这是一个用于检查C++代码中是否有错误的库，可以确保代码的健壮性。

5. `#include <cmath>`：引入了cmath库，这是一个用于数学计算的库，提供了许多数学函数，如sin、cos、sqrt等。

6. `#include <cstdio>`：引入了cstdio库，这是一个通用的C++输入输出库，提供了多种输入输出方式，如scanf、printf等。

7. `#include <cstring>`：引入了cstring库，这是一个用于字符串操作的库，提供了许多字符串的函数，如strlen、strcpy等。

8. `#include <cinttypes>`：引入了cinttypes库，这是一个用于输入输出整型数据的库，提供了各种整型数据类型的函数，如int、double等。

9. `#include <fstream>`：引入了fstream库，这是一个通用的输入输出库，提供了文件读写函数，如is_open、close等。

10. `#include <iostream>`：引入了iostream库，这是一个通用的输入输出库，提供了各种输入输出方式，如cout、cin等。

11. `#include <map>`：引入了map库，这是一个通用的容器库，提供了各种容器函数，如map、set、begin、end等。

12. `#include <string>`：引入了string库，这是一个用于字符串操作的库，提供了各种字符串的函数，如strlen、strcpy等。

13. `#include <vector>`：引入了vector库，这是一个用于向量操作的库，提供了各种向量函数，如push_back、begin、end等。

14. `<fstream>`：这里的fstream库包括is_open和close两个函数，用于文件读写。

15. `<iostream>`：这里的iostream库包括is_open，is_open是用来判断is_open函数的返回状态，is_open函数返回true，说明当前标准输入（通常是键盘）处于打开状态，可以进行文件读写。


```cpp
#include "ggml/ggml.h"

#include "common.h"
#include "common-ggml.h"

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <cinttypes>
#include <fstream>
#include <iostream>
#include <map>
#include <string>
#include <vector>

```

这段代码是一个C/C++的预处理指令，用于检查操作系统是否支持名为“Dolly”的交互式本地客户端。如果没有定义名为“_WIN32”的定义，那么它将包含DOLLY_INTERACTIVE_PORT定义，这意味着我们可能需要为这个接口提供一些定义。

如果定义了DOLLY_INTERACTIVE_PORT，那么它将包含一些头文件和函数声明，这些函数可能与网络协议或套接字操作有关。

此外，通过#if defined(DOLLY_INTERACTIVE_PORT)检查当前操作系统是否支持交互式本地客户端。如果是，它将包含一些库头文件和函数声明，这些函数将允许我们在应用程序中使用DOLLY_INTERACTIVE_PORT接口定义中的函数。

最后，通过#if defined(_MSC_VER)检查当前操作系统是否支持管道子进程。如果是，它将包含一些预处理指令，用于通知编译器不要输出4244和4267错误。


```cpp
#if !defined(_WIN32)
#define DOLLY_INTERACTIVE_PORT
#endif

#if defined(DOLLY_INTERACTIVE_PORT)
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>
#endif

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

```

这段代码定义了一个名为 `dollyv2_hparams` 的结构体，其中包含了 Dolly 模型的参数设置。这个结构体在代码中没有被直接使用，但是它定义了一些与模型相关的参数。

具体来说，这个结构体定义了以下参数：

- `n_vocab`:Dolly 模型的词汇表大小，这个参数用于指定每个词汇在模型中的映射。
- `n_ctx`:Dolly 模型中的编码器层的最大位置嵌入数量。
- `n_embd`:Dolly 模型中的编码器层的隐藏层大小。
- `n_head`:Dolly 模型中的注意力头数。
- `n_layer`:Dolly 模型的隐藏层数。
- `n_rot`:Dolly 模型中的旋转比，它用于控制词汇表中的词汇在编码器层和注意力头中的分布。
- `par_res`:Dolly 模型的预处理年代入(PARAM_RESIDUAR划分为16个部分)。
- `ftype`:Dolly 模型的类型，它指定了使用哪种浮点数类型来保存模型参数。
- `eps`:Dolly 模型的优化技巧，用于防止梯度消失和爆炸。

这个结构体在定义之后，并没有被赋值，也没有在代码中使用。


```cpp
// default hparams (Dolly-V2 3B)
struct dollyv2_hparams {
    int32_t n_vocab = 50254; // tokenizer.vocab_size
    int32_t n_ctx   = 2048;  // model.config.max_position_embeddings
    int32_t n_embd  = 2560;  // model.config.hidden_size
    int32_t n_head  = 32;    // model.config.num_attention_heads
    int32_t n_layer = 32;    // model.config.num_hidden_layers
    int32_t n_rot   = 20;    // rotary_pct[25%] * (n_embd / n_head)
    int32_t par_res = 1; // 1 = true, 0 = false
    int32_t ftype   = GGML_FTYPE_MOSTLY_F16;
    float   eps     = 1e-5f;
};

const std::string INSTRUCTION_KEY = "### Instruction:";
const std::string RESPONSE_KEY    = "### Response:";
```

这段代码定义了一个名为DollyV2的结构体，该结构体包含了生成DollyV2提示格式文本所需的所有组件。

该代码中定义了一个名为`INTRO_BLURB`的常量，它指定了在DollyV2提示格式文本中的开头部分。

该代码中定义了一个名为`END_KEY`的常量，它指定了在DollyV2提示格式文本中的结尾部分。

该代码中定义了一个名为`prompt_for_generation`的函数，该函数接收一个用于描述DollyV2提示格式文本的指令作为参数，并返回一个完整的DollyV2提示格式文本。该函数将`INTRO_BLURB`、`END_KEY`和`INSTRUCTION_KEY`与传入的指令合并，然后将结果字符串中的`\n`替换为`<br />`，并将`<br />`之后的任何字符替换为`<br /> + </br />`，从而得到完整的DollyV2提示格式文本。

该代码中定义了一个名为`dollyv2_layer`的结构体，该结构体包含了DollyV2层中所有需要用到的组件。该结构体中包含一个名为`ln_1_g`的`ggml_tensor`，它用于进行预处理；一个名为`ln_1_b`的`ggml_tensor`，它用于进行预处理；一个名为`c_attn_attn_w`的`ggml_tensor`，它用于计算注意力权重；一个名为`c_attn_attn_b`的`ggml_tensor`，它用于计算注意力偏置；一个名为`ln_2_g`的`ggml_tensor`，它用于进行后处理；一个名为`ln_2_b`的`ggml_tensor`，它用于进行后处理；一个名为`c_mlp_fc_w`的`ggml_tensor`，它用于计算多层感知器FC层的权重；一个名为`c_mlp_fc_b`的`ggml_tensor`，它用于计算多层感知器FC层的偏置；一个名为`c_mlp_proj_w`的`ggml_tensor`，它用于计算多层感知器FC层的投影；一个名为`c_mlp_proj_b`的`ggml_tensor`，它用于计算多层感知器FC层的投影。


```cpp
const std::string END_KEY         = "### End";
const std::string INTRO_BLURB     = "Below is an instruction that describes a task. Write a response that appropriately completes the request.";

// dollyv2 prompt format
std::string prompt_for_generation(const std::string& instruction) {
    return INTRO_BLURB + "\n\n" + INSTRUCTION_KEY + "\n" + instruction + "\n\n" + RESPONSE_KEY + "\n";
}

struct dollyv2_layer {
    // pre normalization
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    // attention
    struct ggml_tensor * c_attn_attn_w;
    struct ggml_tensor * c_attn_attn_b;

    struct ggml_tensor * c_attn_proj_w;
    struct ggml_tensor * c_attn_proj_b;

    // post normalization
    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;

    // ff
    struct ggml_tensor * c_mlp_fc_w;
    struct ggml_tensor * c_mlp_fc_b;

    struct ggml_tensor * c_mlp_proj_w;
    struct ggml_tensor * c_mlp_proj_b;
};

```

这是一个DollyV2模型结构体，它定义了模型在训练或推理过程中的参数和成员变量。以下是这个结构体的主要成员及其作用：

1. `dollyv2_hparams`：这是一个高层参数结构体，定义了模型架构的参数，包括层数、输入尺寸、激活函数等。这个参数结构体用于指定模型的高度，也就是模型的层数。
2. `ln_f_g` 和 `ln_f_b`：这两个成员变量都是全局线性函数（不是注意力机制）的输出张量。这个函数对于每一层来说，都是对输入的`ln_f`函数，然后对于每一层还加上了`ln_f_b`。
3. `wte`：这是一个文本到位置的编码张量，用于对文本序列进行编码。
4. `lmh_g` 和 `lmh_b`：这两个成员变量都是语言模型（注意，这个结构体使用的是DollyV2模型，而不是Transformer或其变种）的头部。这个头部包含两个成员变量，一个是位置嵌入，另一个是语言模型参数。
5. `layers`：这是一个由多个`DollyV2Layer`组成的层列表。每个`DollyV2Layer`都负责在输入数据和模型参数之间进行计算。
6. `memory_k` 和 `memory_v`：这两个成员变量都是`ggml_tensor`类型的，用于存储模型在训练或推理过程中的内存数据。这些数据可以是训练过程中的参数、梯度、或模型存储。
7. `ctx`：这是一个`ggml_context`类型的成员变量，用于管理整个计算过程的上下文。这个上下文可以包括当前层、张量、以及输出数据的布局。
8. `tensors`：这是一个`std::map<std::string, struct ggml_tensor *>`类型的成员变量，用于存储模型在训练过程中的参数。这个参数映射允许开发者使用标准库中定义好的名称来引用参数。

这个结构体定义了DollyV2模型的主要参数和成员变量。这些参数和成员变量然后在DollyV2层的实现中进行具体的计算和设置。


```cpp
struct dollyv2_model {
    dollyv2_hparams hparams;

    // normalization
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte; // position embedding

    struct ggml_tensor * lmh_g; // language model head
    //struct ggml_tensor * lmh_b; // language model bias

    std::vector<dollyv2_layer> layers;

    // key + value memory
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    //
    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

```

This is a C language function that performs a simple tensor type check on a machine learning model file. The function takes a single argument of type tensor, which is read from a file specified by the "filename" parameter. The tensor is then checked against the expected shape and size, and if any


```cpp
// load the model's weights from a file
bool dollyv2_model_load(const std::string & fname, dollyv2_model & model, gpt_vocab & vocab) {
    printf("%s: loading model from '%s' - please wait ...\n", __func__, fname.c_str());

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
        fin.read((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
        fin.read((char *) &hparams.par_res, sizeof(hparams.par_res));
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: n_rot   = %d\n", __func__, hparams.n_rot);
        printf("%s: par_res = %d\n", __func__, hparams.par_res);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

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

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }

        vocab.add_special_token("### End");
        vocab.add_special_token("### Instruction:");
        vocab.add_special_token("### Response:");
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

        ctx_size += n_embd*n_vocab*ggml_type_sizef(wtype); // wte

        ctx_size += n_embd*n_vocab*ggml_type_sizef(wtype);           // lmh_g
        //ctx_size +=        n_vocab*ggml_type_sizef(GGML_TYPE_F32); // lmh_b

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_b

        ctx_size += n_layer*(3*n_embd*n_embd*ggml_type_sizef(wtype));         // c_attn_attn_w
        ctx_size += n_layer*(       3*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_attn_attn_b

        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype));         // c_attn_proj_w
        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_attn_proj_b

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_2_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_fc_w
        ctx_size += n_layer*(       4*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_fc_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_proj_w
        ctx_size += n_layer*(         n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_proj_b

        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F32); // memory_k
        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F32); // memory_v

        ctx_size += (6 + 16*n_layer)*512; // object overhead

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
        const int n_vocab = hparams.n_vocab;

        model.layers.resize(n_layer);

        model.wte    = ggml_new_tensor_2d(ctx, wtype,         n_embd, n_vocab);

        model.ln_f_g = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
        model.ln_f_b = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        model.lmh_g  = ggml_new_tensor_2d(ctx, wtype,         n_embd, n_vocab);
        //model.lmh_b  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_vocab);

        // map by name
        model.tensors["gpt_neox.embed_in.weight"] = model.wte;

        model.tensors["gpt_neox.final_layer_norm.weight"] = model.ln_f_g;
        model.tensors["gpt_neox.final_layer_norm.bias"]   = model.ln_f_b;

        model.tensors["embed_out.weight"] = model.lmh_g;
        //model.tensors["lm_head.bias"]   = model.lmh_b;

        for (int i = 0; i < n_layer; ++i) {
            auto & layer = model.layers[i];

            layer.ln_1_g          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_1_b          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_attn_attn_w   = ggml_new_tensor_2d(ctx, wtype,           n_embd, 3*n_embd);
            layer.c_attn_attn_b   = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 3*n_embd);

            layer.c_attn_proj_w   = ggml_new_tensor_2d(ctx, wtype,           n_embd,   n_embd);
            layer.c_attn_proj_b   = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.ln_2_g          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_2_b          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_mlp_fc_w      = ggml_new_tensor_2d(ctx, wtype,           n_embd, 4*n_embd);
            layer.c_mlp_fc_b      = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4*n_embd);

            layer.c_mlp_proj_w    = ggml_new_tensor_2d(ctx, wtype,         4*n_embd,   n_embd);
            layer.c_mlp_proj_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            // map by name

            // unmapped: attention.rotary_emb, mlp.act

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".input_layernorm.weight"] = layer.ln_1_g;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".input_layernorm.bias"]   = layer.ln_1_b;

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".attention.query_key_value.weight"] = layer.c_attn_attn_w;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".attention.query_key_value.bias"]   = layer.c_attn_attn_b;

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".attention.dense.weight"] = layer.c_attn_proj_w;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".attention.dense.bias"]   = layer.c_attn_proj_b;

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".post_attention_layernorm.weight"] = layer.ln_2_g;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".post_attention_layernorm.bias"]   = layer.ln_2_b;

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".mlp.dense_h_to_4h.weight"] = layer.c_mlp_fc_w;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".mlp.dense_h_to_4h.bias"]   = layer.c_mlp_fc_b;

            model.tensors["gpt_neox.layers." + std::to_string(i) + ".mlp.dense_4h_to_h.weight"] = layer.c_mlp_proj_w;
            model.tensors["gpt_neox.layers." + std::to_string(i) + ".mlp.dense_4h_to_h.bias"]   = layer.c_mlp_proj_b;
        }
    }

    // key + value memory
    {
        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        const int64_t n_mem      = n_layer*n_ctx;
        const int64_t n_elements = n_embd*n_mem;

        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);

        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);

        printf("%s: memory_size = %8.2f MB, n_mem = %" PRId64 "\n", __func__, memory_size/1024.0/1024.0, n_mem);
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
                fprintf(stderr, "%s: tensor '%s' has wrong shape in model file: got [%5d, %5d], expected [%5d, %5d]\n",
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

            total_size += ggml_nbytes(tensor);
            if (++n_tensors % 8 == 0) {
                printf(".");
                fflush(stdout);
            }
        }

        printf(" done\n");

        printf("%s: model size = %8.2f MB / num tensors = %d\n", __func__, total_size/1024.0/1024.0, n_tensors);
    }

    fin.close();

    return true;
}

```

这段代码定义了一个 feed-forward neural network 中的一个后向传播层的 forward 函数。

后向传播层接收一个输入向量 `inp`，并对其进行标量 eps 的平凡化。然后，它将这个标量向量与一系列的输入向量（由 `layer` 参数中的循环层定义）相乘，并使用激活函数 `gelu` 对结果进行非线性化。接着，它将结果通过投影层（由 `layer.c_mlp_proj_w` 和 `layer.c_mlp_proj_b` 参数定义）投影到指定的目标空间。最后，它返回经过投影后的结果向量。

这段代码的作用是实现了一个具有两个隐藏层的 feed-forward neural network 的 forward 函数。它将输入向量 `inp` 传递给网络，然后使用激活函数对结果进行非线性化，并将结果通过投影层投射到指定的目标空间。这个函数在训练神经网络时被重复多次，从而对网络的参数进行更新。


```cpp
// feed-forward network
ggml_tensor * gpt_neox_ff(
        const dollyv2_layer & layer,
        ggml_context        * ctx0,
        ggml_tensor         * inp,
        float                 eps) {
    ggml_tensor * cur = ggml_norm(ctx0, inp, eps);

    cur = ggml_add(ctx0,
        ggml_mul(ctx0,
            ggml_repeat(ctx0, layer.ln_2_g, cur),
            cur),
        ggml_repeat(ctx0, layer.ln_2_b, cur));

    cur = ggml_mul_mat(ctx0,
            layer.c_mlp_fc_w,
            cur);

    cur = ggml_add(ctx0,
            ggml_repeat(ctx0, layer.c_mlp_fc_b, cur),
            cur);

    // GELU activation
    cur = ggml_gelu(ctx0, cur);

    // projection
    // cur = proj_w*cur + proj_b
    cur = ggml_mul_mat(ctx0,
            layer.c_mlp_proj_w,
            cur);

    cur = ggml_add(ctx0,
            ggml_repeat(ctx0, layer.c_mlp_proj_b, cur),
            cur);
    return cur;
}

```

This is a function that performs a forward pass through a given language model. The input to the function is a list of integers, representing the indices of the words in a sentence, and a list of float values, representing the raw logits generated by the model. The function performs the following operations:

1. It initializes the output tensor `embd_w` with the same data type as the input logits, and a size of `n_vocab` (vocabulary size)
2. It loops through the input logits, applying the forward pass to each logit, using the context of the first logit to compute the forward pass
3. It applies the memory allocation macro (ggml_mul_mat, ggml_add, ggml_soft_max_inplace) to the input logits, so that the memory layout of the input is consistent across all elements.
4. It runs the computation using the specified number of threads.
5. It prints the number of used memory after the computation.
6. It returns the result of the computation.

It should be noted that this function is specific to the model you've provided and it may not be same as the other models' implementation.


```cpp
// evaluate the transformer
//
//   - model:     the model
//   - n_threads: number of threads to use
//   - n_past:    the context size so far
//   - embd_inp:  the embeddings of the tokens in the context
//   - embd_w:    the predicted logits for the next token
//
bool dollyv2_eval(
        const dollyv2_model & model,
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
    const int n_rot   = hparams.n_rot;

    static size_t buf_size = 256u*1024*1024;
    static void * buf = malloc(buf_size);

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

    // KQ_pos - contains the positions
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    int * data = (int *) KQ_pos->data;
    for (int i = 0; i < N; ++i) {
        data[i] = n_past + i;
    }

    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    // wte
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model.wte, embd);

    for (int il = 0; il < n_layer; ++il) {
        struct ggml_tensor * cur;

        // self-attention
        {
            {
                cur = ggml_norm(ctx0, inpL, hparams.eps);

                cur = ggml_add(ctx0,
                        ggml_mul(ctx0,
                            ggml_repeat(ctx0, model.layers[il].ln_1_g, cur),
                            cur),
                        ggml_repeat(ctx0, model.layers[il].ln_1_b, cur));
            }

            // compute QKV
            {
                cur = ggml_mul_mat(ctx0,
                        model.layers[il].c_attn_attn_w,
                        cur);

                cur = ggml_add(ctx0,
                        ggml_repeat(ctx0, model.layers[il].c_attn_attn_b, cur),
                        cur);
            }

            struct ggml_tensor * Qcur = ggml_cont(ctx0, ggml_view_3d(ctx0, cur, n_embd/n_head, n_head, N, cur->nb[1]/n_head, cur->nb[1], 0*sizeof(float)*n_embd/n_head));
            struct ggml_tensor * Kcur = ggml_cont(ctx0, ggml_view_3d(ctx0, cur, n_embd/n_head, n_head, N, cur->nb[1]/n_head, cur->nb[1], 1*sizeof(float)*n_embd/n_head));
            struct ggml_tensor * Vcur = ggml_cont(ctx0, ggml_view_3d(ctx0, cur, n_embd/n_head, n_head, N, cur->nb[1]/n_head, cur->nb[1], 2*sizeof(float)*n_embd/n_head));

            // using mode = 2 for GPT-NeoX mode
            Qcur = ggml_rope_inplace(ctx0, Qcur, KQ_pos, n_rot, 2, 0);
            Kcur = ggml_rope_inplace(ctx0, Kcur, KQ_pos, n_rot, 2, 0);

            // store key and value to memory
            {
                Vcur = ggml_transpose(ctx0, ggml_reshape_2d(ctx0, Vcur, n_embd, N));

                struct ggml_tensor * k = ggml_view_1d(ctx0, model.memory_k, N*n_embd, (ggml_element_size(model.memory_k)*n_embd)*(il*n_ctx + n_past));
                struct ggml_tensor * v = ggml_view_2d(ctx0, model.memory_v, N, n_embd,
                        (   n_ctx)*ggml_element_size(model.memory_v),
                        (il*n_ctx)*ggml_element_size(model.memory_v)*n_embd + n_past*ggml_element_size(model.memory_v));

                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
                ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));
            }

            // Q = Qcur.contiguous().view(n_embd/n_head, n_head, N).permute(0, 2, 1, 3)
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        Qcur,
                        0, 2, 1, 3);

            // K = Kmem.view(n_embd/n_head, n_head, n_past + N).permute(0, 2, 1, 3)
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_reshape_3d(ctx0,
                            ggml_view_1d(ctx0, model.memory_k, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(model.memory_k)*n_embd),
                            n_embd/n_head, n_head, n_past + N),
                        0, 2, 1, 3);

            // K * Q
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            // KQ_scaled = KQ / sqrt(n_embd/n_head)
            struct ggml_tensor * KQ_scaled =
                ggml_scale_inplace(ctx0,
                        KQ,
                        ggml_new_f32(ctx0, 1.0f/sqrt(float(n_embd)/n_head))
                        );

            // KQ_masked = mask_past(KQ_scaled)
            struct ggml_tensor * KQ_masked = ggml_diag_mask_inf_inplace(ctx0, KQ_scaled, n_past);

            // KQ = soft_max(KQ_masked)
            struct ggml_tensor * KQ_soft_max = ggml_soft_max_inplace(ctx0, KQ_masked);

            // V_trans = Vmem.view(n_embd/n_head, n_head, n_past + N).permute(1, 2, 0, 3).contiguous()
            struct ggml_tensor * V =
                ggml_view_3d(ctx0, model.memory_v,
                        n_past + N, n_embd/n_head, n_head,
                        n_ctx*ggml_element_size(model.memory_v),
                        n_ctx*ggml_element_size(model.memory_v)*n_embd/n_head,
                        il*n_ctx*ggml_element_size(model.memory_v)*n_embd);

            // KQV = transpose(V) * KQ_soft_max
            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);

            // KQV_merged = KQV.permute(0, 2, 1, 3)
            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            // cur = KQV_merged.contiguous().view(n_embd, N)
            cur = ggml_cpy(ctx0,
                    KQV_merged,
                    ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_embd, N));

            // projection
            {
                cur = ggml_mul_mat(ctx0,
                        model.layers[il].c_attn_proj_w,
                        cur);

                cur = ggml_add(ctx0, ggml_repeat(ctx0, model.layers[il].c_attn_proj_b, cur), cur);
            }
        }

        if (hparams.par_res == 0) {
            struct ggml_tensor * inpFF = ggml_add(ctx0, cur, inpL);

            cur = gpt_neox_ff(model.layers[il], ctx0, inpFF, hparams.eps);

            // input for next layer
            inpL = ggml_add(ctx0, cur, inpFF);
        } else {
            struct ggml_tensor * inpFF = cur;

            // this is independent of the self-attention result, so it could be done in parallel to the self-attention
            // note here we pass inpL instead of cur
            cur = gpt_neox_ff(model.layers[il], ctx0, inpL, hparams.eps);

            // layer input + FF
            cur  = ggml_add(ctx0, cur, inpFF);

            // input for next layer
            inpL = ggml_add(ctx0, cur, inpL);
        }

    }

    // norm
    {
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // inpL = ln_f_g*inpL + ln_f_b
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    // lm_head
    {
        inpL = ggml_mul_mat(ctx0, model.lmh_g, inpL);

        //inpL = ggml_add(ctx0,
        //        ggml_repeat(ctx0, model.lmh_b, inpL),
        //        inpL);
    }

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

    // return result for just the last token
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    //printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    ggml_free(ctx0);

    return true;
}

```

This is a Python implementation of the text generation model that uses a GPT language model and a decoder model to generate text.

It utilizes a batching mechanism, where a fixed number of embeddings are processed at a time, and uses a past燕麦曲线(PCH) based attention mechanism to select which parts of the input text to use as the input for the decoder model.

The `gpt_sample_top_k_top_p` function is used to sample the top `top_k` words from the vocabulary of the model, considering the top `top_p` scores. The `ggml_time_us` function is used to convert the time-based metric to microseconds.

The input text is first processed through the decoder model, and then passed through the PCH to select the embeddings to use for the attention mechanism. The selected embeddings are then added to the attention mechanism, and the decoder model is then run on them to generate the final output.

This model is trained on a不朽的情怀， Occlusion角落，Samtop。You can change the things that you want to train by replacing the lines in `params.top_k` and `params.top_p` with the values that you want to use.


```cpp
std::string execute_prompt(
        const dollyv2_model &model,
        gpt_vocab &vocab,
        const std::string &prompt,
        gpt_params &params,
        std::mt19937 &rng,
        int64_t t_load_us,
        int64_t t_sample_us,
        int64_t t_predict_us,
        size_t mem_per_token,
        int n_past,
        bool stream_response_to_cout = false) {
    std::string output = "";
    std::vector<float> logits;

    // tokenize the prompt
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, prompt);

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int)embd_inp.size());

    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());
    for (size_t i = 0; i < embd_inp.size(); i++) {
        printf("%s: token[%zu] = %6d, %s\n", __func__, i, embd_inp[i], vocab.id_to_token.at(embd_inp[i]).c_str());
    }
    printf("\n");

    std::vector<gpt_vocab::id> embd;

    dollyv2_eval(model, params.n_threads, 0, {0, 1, 2, 3}, logits, mem_per_token);

    const int32_t end_token = vocab.token_to_id["### End"];

    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // predict
        if (embd.size() > 0) {
            const int64_t t_start_us = ggml_time_us();

            if (!dollyv2_eval(model, params.n_threads, n_past, embd, logits, mem_per_token)) {
                printf("Failed to predict\n");
                return output;
            }

            t_predict_us += ggml_time_us() - t_start_us;
        }

        n_past += embd.size();
        embd.clear();

        if (i >= embd_inp.size()) {
            // sample next token
            const int top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp = params.temp;

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
                if (int32_t(embd.size()) > params.n_batch) {
                    break;
                }
            }
            i += embd.size() - 1;
        }

        // display text
        for (auto id : embd) {
            output += vocab.id_to_token[id];
            if (stream_response_to_cout) {
                printf("%s", vocab.id_to_token[id].c_str());
            }
        }
        if (stream_response_to_cout) {
            fflush(stdout);
        }

        // end of text token
        if (embd.back() == 0 || (end_token > 0 && embd.back() == end_token)) {
            return output;
        }
    }
    return output;
}

```

这段代码是一个用于创建套接字的函数，名为setup_port。它接受一个整数参数port，用于设置服务器监听的端口号。函数实现的主要步骤如下：

1. 创建一个套接字sockfd，并初始化为非负整数。这通常是从系统套接字数组中获得的。
2. 创建一个指向套接字地址的int类型的指针servaddr，并初始化为0。
3. 将INADDR_ANY存储的IP地址和端口（port）存储到servaddr中。
4. 使用bind函数将套接字绑定到端口上。如果绑定失败，会打印错误信息并返回-1。
5. 使用listen函数开始监听套接字。如果监听失败，会打印错误信息并返回-1。
6. 通过调用socket和bind函数，创建一个套接字连接，并在连接上发送一个请求（如连接请求）。
7. 返回新创建的套接字（sockfd）。


```cpp
#if defined(DOLLY_INTERACTIVE_PORT)
int setup_port(const int port) {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        fprintf(stderr, "%s: Failed to create new socket\n", __func__);
        return -1;
    }

    sockaddr_in servaddr;
    std::memset(&servaddr, 0, sizeof(servaddr));

    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(port);

    if (bind(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr)) < 0) {
        fprintf(stderr, "%s: Failed to bind to port %i\n", __func__, port);
        return -1;
    }

    if (listen(sockfd, 10) < 0) {
        fprintf(stderr, "%s: Failed to listen to socket on port %i\n", __func__, port);
        return -1;
    }
    return sockfd;
}

```

这段代码定义了一个名为 `read_from_port` 的函数，用于从服务器接收数据并返回给客户端。

函数接收两个整数参数 `sockfd` 和 `clientfd`，用于客户端的套接字和服务器的主机地址和端口号。

函数首先检查 `clientfd` 是否为负数，如果是，就打印错误消息并返回一个空字符串。

然后，函数创建一个大小为 4096 的字符缓冲区 `buffer`，并将其初始化为空字符串。

接下来，函数使用 `read` 函数从客户端套接字 `clientfd` 读取数据，并存储在 `buffer` 中。如果读取失败，就打印错误消息并返回一个空字符串。

最后，函数将 `buffer` 中的内容输出给客户端，并返回给客户端的套接字对应的字符串。如果所有数据读取完成后仍然没有结束，函数也会返回一个空字符串。


```cpp
std::string read_from_port(int sockfd, int clientfd) {
    if (clientfd < 0) {
        fprintf(stderr, "%s: Failed to accept new connection\n", __func__);
        return "";
    }

    char buffer[4096];
    std::memset(buffer, 0, sizeof(buffer));

    if (read(clientfd, buffer, sizeof(buffer)) < 0) {
        fprintf(stderr, "%s: Failed to read from client\n", __func__);
    } else {
        std::cout << "Received: " << buffer;
        return std::string(buffer);
    }
    return std::string("");
}
```

这段代码是一个 C 语言程序，主要作用是使用 GPT-3 模型进行自然语言处理。以下是程序的主要步骤：

1. 初始化 GPT-3 模型的时间和随机数种子。
2. 加载 GPT-3 模型的二进制文件。
3. 设置模型的参数，包括种子点。
4. 加载用于模型的元数据。
5. 准备处理输入数据的环境。
6. 循环进行模型预测。
7. 对于每个预测的文本，使用模型进行预测并计算出预测结果的时间复杂度。

该程序的作用是使用 GPT-3 模型对输入文本进行自然语言处理，并输出预测结果的时间复杂度。


```cpp
#endif

int main(int argc, char ** argv) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;
    params.model = "models/dolly-v2-3b/ggml-model-f16.bin";

    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    printf("%s: seed = %d\n", __func__, params.seed);

    std::mt19937 rng(params.seed);

    int64_t t_load_us = 0;
    int64_t t_sample_us = 0;
    int64_t t_predict_us = 0;

    // determine the required inference memory per token:
    size_t mem_per_token = 0;

    int n_past = 0;

    gpt_vocab vocab;
    dollyv2_model model;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!dollyv2_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;

        test_gpt_tokenizer(vocab, params.token_test);
    }

```

这段代码的作用是判断用户是否选择了一个交互式端口（DOLLY_INTERACTIVE_PORT），如果用户选择了这个端口，那么它会尝试使用该端口连接到远程服务器，并输出一条消息表明模型已经准备好接收数据。

具体来说，代码首先检查DOLLY_INTERACTIVE_PORT是否已经被定义。如果是，代码就会执行下面的判断：

```cpp
if (defined(DOLLY_INTERACTIVE_PORT)) {
   int sockfd = -1;
   if (params.interactive_port != -1) {
       sockfd = setup_port(params.interactive_port);
       if (sockfd == -1) {
           return 1;
       }
       fprintf(stdout, "Model is ready on port %i\n", params.interactive_port);
       fflush(stdout);
   }
}
```

如果DOLLY_INTERACTIVE_PORT没有被定义，那么代码就会输出一个错误。如果用户选择了一个交互式端口，那么代码就会尝试使用该端口连接到远程服务器，并输出一条消息表明模型已经准备好接收数据。

具体来说，代码接下来会执行下面的判断：

```cpp
if (params.interactive && params.interactive_port != -1) {
   while (true) {
       std::string prompt_input;
       int timeout;

       // 从用户那里获取输入
       std::getline(std::cin, prompt_input);

       // 检查用户输入是否正确
       if (strcmp(prompt_input.c_str(), "exit") == 0) {
           break;
       }
       else if (strcmp(prompt_input.c_str(), "quit") == 0) {
           break;
       }
       else {
           // 时间限制
           timeout = 1000;
       }

       // 发送请求并等待结果
       int ret = send(sockfd, prompt_input.c_str(), strlen(prompt_input.c_str()), timeout);

       // 检查出错则退出
       if (ret == -1) {
           return 1;
       }
       else if (ret == 0) {
           return 2;
       }
   }
}
```

总的来说，这段代码的主要作用是判断用户是否选择了一个交互式端口，并使用该端口连接到远程服务器。如果用户选择了该端口，那么代码就会输出一条消息并等待用户的输入。如果用户输入了“exit”或“quit”，那么代码就会退出等待。如果用户发送了一个有效的请求并等待了足够长的时间，那么代码就会返回一个非负的值。如果代码在尝试连接过程中遇到了错误，那么代码就会返回一个负的值。


```cpp
#if defined(DOLLY_INTERACTIVE_PORT)
    int sockfd = -1;
    if (params.interactive_port != -1) {
        sockfd = setup_port(params.interactive_port);
        if (sockfd == -1) {
            return 1;
        }
        fprintf(stdout, "Model is ready on port %i\n", params.interactive_port);
        fflush(stdout);
    }
#endif

    if (params.interactive || params.interactive_port != -1) {
        while (true) {
            std::string prompt_input;
```

这段代码是一个用于处理用户输入的Python程序。它使用了C++的第三方库DOLLY_INTERACTIVE_PORT来实现网络编程。

程序的主要功能是接收用户输入并输出结果。首先，它检查定义了一个DOLLY_INTERACTIVE_PORT变量，如果这个变量被定义了，那么就会进入交互式模式，否则提示用户输入问题并进入非交互式模式。

如果用户输入“exit”，程序就会退出交互式模式并继续接收用户输入。否则，程序会获取用户输入的问题，并尝试使用所给的模型进行预测。模型的预测结果将被存储在std::string类型的变量中。

注意，这段代码缺少一些必要的注释，以便更好地理解它的作用。


```cpp
#if defined(DOLLY_INTERACTIVE_PORT)
            int clientfd = -1;
            if (params.interactive_port != -1) {
                sockaddr_in clientaddr;
                socklen_t clientaddrlen = sizeof(clientaddr);
                clientfd = accept(sockfd, (struct sockaddr *)&clientaddr, &clientaddrlen);
                prompt_input = read_from_port(sockfd, clientfd);
            } else
#endif
            {
                printf("Please enter your quesiton:\n>");
                fflush(stdout);

                std::getline(std::cin, prompt_input);
            }

            if (strcmp(prompt_input.c_str(), "exit") == 0) {
                break;
            }

            const std::string prompt = prompt_for_generation(prompt_input);
            // call the model
            const std::string response = execute_prompt(model, vocab, prompt, params, rng, t_load_us, t_sample_us, t_predict_us, mem_per_token, n_past, true);

```

This is a C++ program that appears to use Google's Params object to interact with the language model. It first sets up a connection with a client and begins listening for messages from the client. When a message is received, it processes the message and sends a response back to the client. It also reads and executes a prompt from the client and sends the response to the client.

The program uses the `std::string` class to store the prompt and response strings. It uses the `ggml_time_us()` function to measure the timing of its functions. It uses the `mem_per_token` and `n_past` constants to keep track of the number of times it has seen the prompt and the number of times it has executed the prompt so far.

The program also uses the `ggml_free()` function to release the resources associated with the Params object.

Overall, this program appears to be a simple tool for interacting with a language model using the `params` object from Google's Cloud Speech API.


```cpp
#if defined(DOLLY_INTERACTIVE_PORT)
            if (params.interactive_port != -1) {
                if (write(clientfd, response.c_str(), response.size()) < 0) {
                    fprintf(stderr, "%s: Failed to write answer '%s' to client\n", __func__, response.c_str());
                }

                if (close(clientfd) < 0) {
                    fprintf(stderr, "%s: Failed to close client socket\n", __func__);
                }
            } else
#endif
            {
                printf("%s\n\n", response.c_str());
            }
            fflush(stdout);
        }
    } else {
        if (params.prompt.empty()) {
            params.prompt = gpt_random_prompt(rng);
        }

        const std::string prompt = prompt_for_generation(params.prompt);
        execute_prompt(model, vocab, prompt, params, rng, t_load_us, t_sample_us, t_predict_us, mem_per_token, n_past, true);
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n");
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us / 1000.0f);
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us / 1000.0f);
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us / 1000.0f, t_predict_us / 1000.0f / n_past);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    ggml_free(model.ctx);

```

这段代码是一个C语言函数，它对一个名为DOLLY_INTERACTIVE_PORT的定义进行判断，如果定义存在并且是字符串"/path/to/dolly/interactive/port"，那么函数将尝试关闭服务器套接字，并输出一条错误消息。

具体来说，代码首先检查DOLLY_INTERACTIVE_PORT是否已经被定义为非负整数。如果是，那么将尝试使用close函数关闭服务器套接字。如果close函数返回负值，那么将输出一条错误消息，并包含函数的名称。

这段代码的作用是检查服务器套接字是否可以被正常关闭，如果无法关闭，则输出错误消息并返回0。


```cpp
#if defined(DOLLY_INTERACTIVE_PORT)
    if (params.interactive_port != -1 && close(sockfd) < 0) {
        fprintf(stderr, "%s: Failed to close server socket\n", __func__);
    }
#endif

    return 0;
}

```

# `examples/dolly-v2/quantize.cpp`

这段代码是一个通用的C++头文件，其中包含了一些通用的函数和变量，以及一些与数学相关的常量。

具体来说，这段代码包含以下内容：

1. ggml header文件包含头文件ggml.h，该文件是ggml库的定义文件，该库可以用于绘制图形、创建文本、添加元素等任务。

2. common header文件包含头文件common.h和common-ggml.h，这两个文件可能是其他库或头文件，需要根据具体的使用场景来包含。

3. 标准库头文件包含cassert、cmath、cstdio和cstring，这些头文件定义了一些通用的类型和函数，包括用于数学计算的标准库函数。

4. fstream文件输入流头文件包含fileinput，该文件输入流允许从文件中读取输入数据。

5. regex数学函数头文件包含regex，该头文件提供了一些通用的数学函数，包括正则表达式匹配、替换和搜索等。

6. 整数类型头文件包含int和 long，这些头文件定义了整数类型以及整数的一些常见的操作。

7. 浮点数类型头文件包含float和 double，这些头文件定义了浮点数类型以及浮点数的一些常见的操作。

8. 函数指针头文件包含function，该文件定义了如何将函数作为指针进行传递。

9. 引用头文件包含引用，该头文件定义了如何引用一个函数或类。

10. 点数头文件包含point，该头文件定义了如何表示一个点，该头文件还包含point的size函数，用于计算点的大小。

11. 坐标系统头文件包含coord，该头文件定义了如何表示一个坐标系统，该头文件还包含coord的别名函数，用于计算坐标。

12. 文本操作头文件包含text，该头文件定义了如何处理文本数据，包括文本的插入、删除、替换和搜索等操作。

13. 图形头文件包含graph，该头文件定义了如何表示一个图形，该头文件还包含一些常见的图形操作。

14. 颜色头文件包含color，该头文件定义了如何表示一个颜色，该头文件还包含一些常见的颜色转换函数。

15. 图像操作头文件包含image，该头文件定义了如何处理图像数据，包括图像的插入、删除、替换和搜索等操作。

16. 数学函数头文件包含math，该头文件定义了如何使用一些数学函数，包括数学的加法、减法、乘法和除法函数。


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
#include <regex>

```

This is a C++ program that reads the input of a neural network model and Quantizes the model using the Gmail Graphite library.

It takes a neural network model file, a file output stream, and a type of quantization to use.

The program reads the neural network model file and the input file.

Then it reads the input and creates a vector of all the tensor names to be quantized.

It then uses the Gmail Graphite library to quantize the model according to the specified type.

The quantized model is then written to the output stream.

This program looks like it has been functional for aangpr18, but I do not have any information about the people who have tested it.

It is always recommended to use this program only as a simple test and not as a production-ready product.

Please note that this program is for educational purposes only and should be used with caution.


```cpp
// default hparams (dollyv2 3B)
struct dollyv2_hparams {
    int32_t n_vocab = 50254; // tokenizer.vocab_size
    int32_t n_ctx   = 2048;  // model.config.max_position_embeddings
    int32_t n_embd  = 2560;  // model.config.hidden_size
    int32_t n_head  = 32;    // model.config.num_attention_heads
    int32_t n_layer = 32;    // model.config.num_hidden_layers
    int32_t n_rot   = 20;    // rotary_pct[25%] * (n_embd / n_head)
    int32_t par_res = 1; // 1 = true, 0 = false
    int32_t ftype   = GGML_FTYPE_MOSTLY_F16;
};

// quantize a model
bool dollyv2_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
    gpt_vocab vocab;

    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());

    auto finp = std::ifstream(fname_inp, std::ios::binary);
    if (!finp) {
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__, fname_inp.c_str());
        return false;
    }

    auto fout = std::ofstream(fname_out, std::ios::binary);
    if (!fout) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname_out.c_str());
        return false;
    }

    // verify magic
    {
        uint32_t magic;
        finp.read((char *) &magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname_inp.c_str());
            return false;
        }

        fout.write((char *) &magic, sizeof(magic));
    }

    dollyv2_hparams hparams;

    // load hparams
    {
        finp.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        finp.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        finp.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        finp.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        finp.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        finp.read((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
        finp.read((char *) &hparams.par_res, sizeof(hparams.par_res));
        finp.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr_src =    hparams.ftype / GGML_QNT_VERSION_FACTOR;
        const int32_t ftype_dst = GGML_QNT_VERSION * GGML_QNT_VERSION_FACTOR + ftype;

        printf("%s: n_vocab     = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx       = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd      = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head      = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer     = %d\n", __func__, hparams.n_layer);
        printf("%s: par_res     = %d\n", __func__, hparams.par_res);
        printf("%s: ftype (src) = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr (src) = %d\n", __func__, qntvr_src);
        printf("%s: ftype (dst) = %d\n", __func__, ftype_dst);
        printf("%s: qntvr (dst) = %d\n", __func__, GGML_QNT_VERSION);

        fout.write((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fout.write((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fout.write((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fout.write((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fout.write((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fout.write((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
        fout.write((char *) &hparams.par_res, sizeof(hparams.par_res));
        fout.write((char *) &ftype_dst,       sizeof(ftype_dst));
    }

    // load vocab
    {
        const int32_t n_vocab = hparams.n_vocab;

        std::string word;
        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            finp.read ((char *) &len, sizeof(len));
            fout.write((char *) &len, sizeof(len));

            word.resize(len);
            finp.read ((char *) word.data(), len);
            fout.write((char *) word.data(), len);

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // regexes of tensor names to be quantized
    const std::vector<std::string> to_quant = {
        ".*weight",
    };

    if (!ggml_common_quantize_0(finp, fout, ftype, to_quant, {})) {
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__, fname_inp.c_str());
        return false;
    }

    finp.close();
    fout.close();

    return true;
}

```

This is a C++ program that uses the DollyV2 library to perform model quantization on a machine learning model specified by the input file "model-f32.bin" and the type of the model specified by the input file "model-quant.bin".

The program first checks the input files and ensures that they are valid before proceeding. If the input files are valid, the program performs model quantization by running the DollyV2 library's "dollyv2\_model\_quantize" function on the specified model.

The program then reports the timing information, which includes the time taken to perform the quantization operation and the total time taken to run the program.

Note that the program assumes that the input files "model-f32.bin" and "model-quant.bin" are valid files that contain the input and output data for the specified model, respectively. Additionally, the program assumes that the user has installed the DollyV2 library and that it has been set up correctly on the system.


```cpp
// usage:
//  ./dollyv2-quantize models/dolly-v2-3B/ggml-model.bin models/dolly-v2-3B/ggml-model-quant.bin type
//
int main(int argc, char ** argv) {
    if (argc != 4) {
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n", argv[0]);
        ggml_print_ftypes(stderr);
        return 1;
    }

    // needed to initialize f16 tables
    {
        struct ggml_init_params params = { 0, NULL, false };
        struct ggml_context * ctx = ggml_init(params);
        ggml_free(ctx);
    }

    const std::string fname_inp = argv[1];
    const std::string fname_out = argv[2];

    const ggml_ftype ftype = ggml_parse_ftype(argv[3]);

    const int64_t t_main_start_us = ggml_time_us();

    int64_t t_quantize_us = 0;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!dollyv2_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n", __func__, fname_inp.c_str());
            return 1;
        }

        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__, t_quantize_us/1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    return 0;
}

```