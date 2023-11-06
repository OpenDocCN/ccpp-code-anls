# GGML源码解析 12

# `examples/mpt/quantize.cpp`

这段代码是一个通用的C++头文件，它包含了ggml和common-ggml两个头文件。ggml和common-ggml可能是其他源代码文件（可能是用于描述ggml库或者为其他库准备的）。然后，它又包含了一些通用的函数和变量，如cassert和cmath，这些对于使用ggml库或者编写C++程序非常有用。


```cpp
#include "ggml/ggml.h"

#include "common-ggml.h"
#include "common.h"

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <regex>
#include <string>
#include <vector>

```

This is a Python implementation of the function `__convert__`, which takes in two arguments, a float tensor and a Quantize object, and returns a tensor. The function uses the information in the Quantize object to convert the float tensor to a tensor that can be quantized. If the conversion fails, the function prints an error and returns `False`.

The function takes the following arguments:

* `finp`: a file-like input stream, usually `stdin`
* `fout`: a file-like output stream, usually `stdout`
* `htype`: the data type of the tensor. This should be either `float` or `torch.float16`
* `params`: a parameter object, which should contain any arguments that the user needs to pass to the function. In this function, it is assumed to be a list of the following types:
	+ `int32_t`
	+ `float32_t`
	+ `int64_t`
	+ `float64_t`
	+ `bool`
	+ `char`
	+ `double`
	+ `str`
	+ `funcname_inp`: the name of the tensor that needs to be converted. This should be a string.
* `n_heads`: an integer representing the number of head-level subword embeddings to use. This should be a parameter of the `Quantize` object.
* `n_layers`: an integer representing the number of sublayer blocks. This should be a parameter of the `Quantize` object.
* `n_vocab`: an integer representing the number of vocabulary elements. This should be a parameter of the `Quantize` object.
* `alibi_bias_max`: an optional integer representing the maximum value for the alibi token alias. This should be a parameter of the `Quantize` object.
* `clip_qkv`: an optional float or boolean parameter representing the�霸道权. This should be a parameter of the `Quantize` object.
* `ftype_dst`: an optional function type parameter representing the type of data to be stored in the destination tensor. This should be a parameter of the `Quantize` object.

The function firsts the inputs and outputs, then loads the model and initializes the input, and finally converts the input tensor to the Quantize format, if successful.


```cpp
struct mpt_hparams {
    int32_t d_model      = 0;
    int32_t max_seq_len  = 0;
    int32_t n_heads      = 0;
    int32_t n_layers     = 0;
    int32_t n_vocab      = 0;
    float alibi_bias_max = 0;
    float clip_qkv       = 0;
    int32_t ftype        = 0;
};

// quantize a model
bool mpt_model_quantize(const std::string & fname_inp,
                        const std::string & fname_out, ggml_ftype ftype) {

    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());

    auto finp = std::ifstream(fname_inp, std::ios::binary);
    if (!finp) {
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__,
                fname_inp.c_str());
        return false;
    }

    auto fout = std::ofstream(fname_out, std::ios::binary);
    if (!fout) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__,
                fname_out.c_str());
        return false;
    }

    // verify magic
    {
        uint32_t magic;
        finp.read((char *)&magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n",
                    __func__, fname_inp.c_str());
            return false;
        }

        fout.write((char *)&magic, sizeof(magic));
    }

    mpt_hparams hparams;

    // load hparams
    {
        finp.read((char *) &hparams.d_model,        sizeof(hparams.d_model));
        finp.read((char *) &hparams.max_seq_len,    sizeof(hparams.max_seq_len));
        finp.read((char *) &hparams.n_heads,        sizeof(hparams.n_heads));
        finp.read((char *) &hparams.n_layers,       sizeof(hparams.n_layers));
        finp.read((char *) &hparams.n_vocab,        sizeof(hparams.n_vocab));
        finp.read((char *) &hparams.alibi_bias_max, sizeof(hparams.alibi_bias_max));
        finp.read((char *) &hparams.clip_qkv,       sizeof(hparams.clip_qkv));
        finp.read((char *) &hparams.ftype,          sizeof(hparams.ftype));

        const int32_t qntvr_src =    hparams.ftype / GGML_QNT_VERSION_FACTOR;
        const int32_t ftype_dst = GGML_QNT_VERSION * GGML_QNT_VERSION_FACTOR + ftype;

        printf("%s: d_model        = %d\n", __func__, hparams.d_model);
        printf("%s: max_seq_len    = %d\n", __func__, hparams.max_seq_len);
        printf("%s: n_heads        = %d\n", __func__, hparams.n_heads);
        printf("%s: n_layers       = %d\n", __func__, hparams.n_layers);
        printf("%s: n_vocab        = %d\n", __func__, hparams.n_vocab);
        printf("%s: alibi_bias_max = %f\n", __func__, hparams.alibi_bias_max);
        printf("%s: clip_qkv       = %f\n", __func__, hparams.clip_qkv);
        printf("%s: ftype (src) = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr (src) = %d\n", __func__, qntvr_src);
        printf("%s: ftype (dst) = %d\n", __func__, ftype_dst);
        printf("%s: qntvr (dst) = %d\n", __func__, GGML_QNT_VERSION);

        fout.write((char *) &hparams.d_model,        sizeof(hparams.d_model));
        fout.write((char *) &hparams.max_seq_len,    sizeof(hparams.max_seq_len));
        fout.write((char *) &hparams.n_heads,        sizeof(hparams.n_heads));
        fout.write((char *) &hparams.n_layers,       sizeof(hparams.n_layers));
        fout.write((char *) &hparams.n_vocab,        sizeof(hparams.n_vocab));
        fout.write((char *) &hparams.alibi_bias_max, sizeof(hparams.alibi_bias_max));
        fout.write((char *) &hparams.clip_qkv,       sizeof(hparams.clip_qkv));
        fout.write((char *) &ftype_dst,              sizeof(ftype_dst));
    }

    // load vocab
    {
        const int32_t n_vocab = hparams.n_vocab;

        std::string word;
        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            finp.read((char *)&len, sizeof(len));
            fout.write((char *)&len, sizeof(len));

            word.resize(len);
            finp.read((char *)word.data(), len);
            fout.write((char *)word.data(), len);
        }
    }

    printf("%s: quantizing tensors\n", __func__);

    // regexes of tensor names to be quantized
    const std::vector<std::string> to_quant = {
        ".*weight",
    };

    if (!ggml_common_quantize_0(finp, fout, ftype, to_quant, {})) {
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__,
                fname_inp.c_str());
        return false;
    }

    finp.close();
    fout.close();

    return true;
}

```

This is a C++ program that seems to quantize a model using the f16 tables generated by the Model-F32 tool. The program takes two arguments:

* The first argument is a file name of a model in the f16 format that should be quantized.
* The second argument is a file name of a result file that will store the quantized model.

The program reads the model file and quantizes it using the Model-F32 tool. It then writes the quantized model to the result file.

The program uses the ggml library to perform the model quantization. The `ggml_init()` function initializes the ggml context and the `ggml_time_us()` function returns the amount of time elapsed since the start of the program.

The program also uses two helper functions:

* `__func__()` is a macro that generates the name of the function.
* `fprintf()` is a function that formats a text message and writes it to the standard error stream.

The program can be run using the following command:
```cpp
./model_quant <model_file> <result_file>
```
This will quantize the model in the specified file and write the result to the file specified.


```cpp
// usage:
//  ./mpt-quantize models/mpt/ggml-model.bin
//  models/mpt/ggml-model-quant.bin type
//
int main(int argc, char ** argv) {
    if (argc != 4) {
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n",
                argv[0]);
        ggml_print_ftypes(stderr);
        return 1;
    }

    // needed to initialize f16 tables
    {
        struct ggml_init_params params = {0, NULL, false};
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

        if (!mpt_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n",
                    __func__, fname_inp.c_str());
            return 1;
        }

        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // report timing
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__,
               t_quantize_us / 1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__,
               (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    return 0;
}

```

# MPT

Ref: https://github.com/mosaicml/llm-foundry#mpt

## Usage

```cppbash
# get the repo and build it
git clone https://github.com/ggerganov/ggml
cd ggml
mkdir build && cd build
cmake ..
make -j

# get the model from HuggingFace
# be sure to have git-lfs installed
git clone https://huggingface.co/mosaicml/mpt-30b

# convert model to FP16
python3 ../examples/mpt/convert-h5-to-ggml.py ./mpt-30b 1

# run inference using FP16 precision
./bin/mpt -m ./mpt-30b/ggml-model-f16.bin -p "I believe the meaning of life is" -t 8 -n 64

# quantize the model to 5-bits using Q5_0 quantization
./bin/mpt-quantize ./mpt-30b/ggml-model-f16.bin ./mpt-30b/ggml-model-q5_0.bin q5_0
```


# `examples/prompts/tokenize_huggingface.py`

这段代码的作用是设置一个名为“TOKENIZERS_PARALLELISM”的环境变量。这个环境变量传达了一个关于tokenizer（用于在模型中对输入文本进行编码的软件）并行的一个布尔值，其值设置为“false”。

具体来说，这个代码将“TOKENIZERS_PARALLELISM”环境变量设置为“false”，这意味着在运行这段代码后，任何与这段代码相同的tokenizer将不能并行处理输入数据。这个设置对于使用多个tokenizer（例如在训练多个模型时）非常重要，因为这有助于避免因并行处理输入数据而导致的不正确的训练结果。


```cpp
import os
from transformers import AutoTokenizer

os.environ['TOKENIZERS_PARALLELISM'] = "false"

list_repo_hf  = ["databricks/dolly-v2-3b",           # dolly-v2 (3b, 7b, 12b models share the same tokenizer)
                 "gpt2",                             # gpt-2 (gpt2-xl, gpt2-large share the same tokenizer)
                 "uer/gpt2-chinese-cluecorpussmall", # gpt-2-chinese
                 "EleutherAI/gpt-j-6b",              # gpt-j
                 "EleutherAI/gpt-neox-20b",          # gpt-neox
                 "EleutherAI/polyglot-ko-1.3b",      # gpt-neox (polyglot-ko 5.8b and 12.8b share the same tokenizer")
                 "rinna/japanese-gpt-neox-3.6b",     # gpt-neox
                 # mpt-7b (uses gpt-neox-20b tokenizer)
                 "replit/replit-code-v1-3b",         # replit
                 "bigcode/starcoder",                # starcoder (huggingface-cli login required)
                 "openai/whisper-tiny"               # whisper (base, large, large-v2 share the same tokenizer)
                 ]

```

这段代码是一个Python代码，它定义了两个字典repo2ggml和repo2language。这两个字典是用来存储不同语言的GPT模型的。

repo2ggml字典存储了以下数据：

* "databricks/dolly-v2-3b" : "dolly-v2"
* "gpt2"                             : "gpt-2"
* "uer/gpt2-chinese-cluecorpussmall" : "gpt-2-chinese"
* "EleutherAI/gpt-j-6b"              : "gpt-j"
* "EleutherAI/gpt-neox-20b"          : "gpt-neox"
* "EleutherAI/polyglot-ko-1.3b"      : "polyglot-ko"
* "rinna/japanese-gpt-neox-3.6b"     : "gpt-neox-japanese"
* "replit/replit-code-v1-3b"         : "replit"
* "bigcode/starcoder"                : "starcoder"
* "openai/whisper-tiny"              : "whisper"

repo2language字典存储了以下数据：

* "databricks/dolly-v2-3b" : "english"
* "gpt2"                             : "english"
* "uer/gpt2-chinese-cluecorpussmall" : "chinese"
* "EleutherAI/gpt-j-6b"              : "english"
* "EleutherAI/gpt-neox-20b"          : "english"
* "EleutherAI/polyglot-ko-1.3b"      : "korean"
* "rinna/japanese-gpt-neox-3.6b"     : "japanese"
* "replit/replit-code-v1-3b"         : "english"
* "bigcode/starcoder"                : "english"
* "openai/whisper-tiny"              : "english"


```cpp
repo2ggml     = {"databricks/dolly-v2-3b"           : "dolly-v2",
                 "gpt2"                             : "gpt-2",
                 "uer/gpt2-chinese-cluecorpussmall" : "gpt-2-chinese",
                 "EleutherAI/gpt-j-6b"              : "gpt-j",
                 "EleutherAI/gpt-neox-20b"          : "gpt-neox",
                 "EleutherAI/polyglot-ko-1.3b"      : "polyglot-ko",
                 "rinna/japanese-gpt-neox-3.6b"     : "gpt-neox-japanese",
                 "replit/replit-code-v1-3b"         : "replit",
                 "bigcode/starcoder"                : "starcoder",
                 "openai/whisper-tiny"              : "whisper"}

repo2language = {"databricks/dolly-v2-3b"           : "english",
                 "gpt2"                             : "english",
                 "uer/gpt2-chinese-cluecorpussmall" : "chinese",
                 "EleutherAI/gpt-j-6b"              : "english",
                 "EleutherAI/gpt-neox-20b"          : "english",
                 "EleutherAI/polyglot-ko-1.3b"      : "korean",
                 "rinna/japanese-gpt-neox-3.6b"     : "japanese",
                 "replit/replit-code-v1-3b"         : "english",
                 "bigcode/starcoder"                : "english",
                 "openai/whisper-tiny"              : "english"}

```

这段代码的作用是读取一个名为 "test-cases.txt" 的文件，里面包含了许多测试用例句子，每个句子由一个键-值对组成，键是目标语言，值是句子本身。通过这个文件，作者希望能够建立一个测试数据集，用于训练一个文本到图像的模型。

代码首先定义了一个常量 "delimiter"，其值为冒号 ":"。接着定义了一个空列表 "test_sentences"，用于存储所有的测试句子。

然后，使用 "with open" 语句打开名为 "test-cases.txt" 的文件，并读取其中的行 "lines"。接着，遍历每个行，如果行中包含 "delimiter"，则记录下目标语言和该行中的句子。将这些记录存储到 "test_sentences" 列表中。

接下来，定义了一个循环变量 "repo"，用于存储要处理的每个文本文件。然后，使用 "repo2language" 函数获取该文本文件的目标语言，并使用 "AutoTokenizer" 类从该文本文件中读取词汇表。接着，使用 "convert_tokens_to_ids" 方法将每个词汇转换为 ID，并将这些 ID 存储到一个列表 "tokens_hf" 中。

接着，使用循环变量 "repo" 和 "tokens_hf" 遍历每个测试句子，如果该句子正好是目标语言，则使用 "AutoTokenizer" 类读取该句子中的词汇 ID，并将这些词汇 ID 存储到一个字符串中。最后，将这些字符串打印出来，形成一个目标语作为关键字的列表。

最后，将生成的列表 "tokens_hf" 保存到一个名为 "repo2ggml_tokenized.txt" 的文件中。


```cpp
delimeter = ": "
test_sentences = []
with open("test-cases.txt", "r") as f:
    lines = [l.rstrip() for l in f.readlines()]
    for l in lines:
        if delimeter in l:
            language = l[:l.index(delimeter)]
            sentence = l[l.index(delimeter) + len(delimeter):]
            test_sentences.append((language.lower(), sentence))

for repo in list_repo_hf:

    target_language = repo2language[repo]

    tokenizer = AutoTokenizer.from_pretrained(repo, trust_remote_code=True)

    tokens_hf = []
    for language, sentence in test_sentences:
        if language == target_language:
            tokens = tokenizer.convert_tokens_to_ids(tokenizer.tokenize(sentence))
            tokens_hf.append((sentence, tokens))

    save_txt = repo2ggml[repo] + ".txt"
    with open(save_txt, "w") as f:
        f.writelines([sentence + " => " + ",".join(str(t) for t in tokens) + "\n" for sentence, tokens in tokens_hf])

```

# `examples/python/api.h`

这段代码是一个用于在LLVM中生成Metal模拟库的头文件列表。它包括了不同头文件，如"ggml.h"、"ggml-metal.h"、"ggml-opencl.h"、"k_quants.h"、"ggml-alloc.h"、"ggml-cuda.h"和"ggml-mpi.h"，这些头文件包含了LLVM为Metal库提供的函数和变量。

在运行以下代码后，将会生成一个名为"metal-lib.so"的库文件。这个库文件将允许开发者使用LLVM的Metal库来创建高效的Metal模拟。在运行代码之前，请确保您已经安装了LLVM，并且链接器设置为使用LLVM。

请注意，这些头文件中有些是仅在llama.cpp中存在的内容。要避免输出这些内容，可以手动注释掉这些部分。


```cpp
/*
  List here all the headers you want to expose in the Python bindings,
  then run `python regenerate.py` (see details in README.md)
*/

#include "ggml.h"
#include "ggml-metal.h"
#include "ggml-opencl.h"

// Headers below are currently only present in the llama.cpp repository, comment them out if you don't have them.
#include "k_quants.h"
#include "ggml-alloc.h"
#include "ggml-cuda.h"
#include "ggml-mpi.h"
```

# `examples/python/example_add_quant.py`

这段代码实现了将两个 numerical 数组相加并输出结果的功能。函数使用了 C++ GGML library，该库提供了对 Python 的数值库（如 NumPy）的包装，并支持C++编程。

首先，从ggml库中引入了lib.ggml_new_tensor_1d、lib.ggml_new_tensor_1d和lib.ggml_add函数，这些函数用于创建和初始化两个张量，以及将两个张量相加。

然后，定义了一个numpy数组a和一个只存储浮点数（double precision）的数组b。

接着，将a和b相加，并将结果保存回a中。

最后，创建一个ffi结构体gf，用于存储输入张量，然后使用ffi.new函数将gf存储在内存中。

输出结果的类型为'float32'，即存储为32位浮点数。


```cpp
from ggml import lib, ffi
from ggml.utils import init, copy, numpy
import numpy as np

ctx = init(mem_size=12*1024*1024) # automatically freed when pointer is GC'd
n = 256
n_threads = 4

a = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
b = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n) # can't both be quantized
sum = lib.ggml_add(ctx, a, b) # all zeroes for now. Will be quantized too!

# See cffi's doc on how to allocate native memory: it's very simple!
# https://cffi.readthedocs.io/en/latest/ref.html#ffi-interface
gf = ffi.new('struct ggml_cgraph*')
```

这段代码的主要作用是执行一个图形库（lib.ggml）的向前扩展操作，用于计算一个输入图（gf）的向前追根究底操作。在这个过程中，源代码（copy）和计算结果（numpy）被用来在代码中传递输入和结果。

具体来说，这段代码执行以下操作：

1. 首先，定义了一个输入图（gf）和一个求和操作（sum），将输入图和求和操作作为参数传递给lib.ggml_build_forward_expand函数。

2. 然后，从range(n)中复制n个输入值（i），并将复制的值存储到变量a中。

3. 接下来，从range(n)中复制n个输入值的100倍（i*100），并将复制的值存储到变量b中。

4. 接着，使用lib.ggml_graph_compute_with_ctx函数，将输入图（gf）和指定的计算上下文（ctx）传递给函数，并指定计算图的步长（n_threads）。这样，函数将在输入图和上下文中执行图形库的向前扩展操作。

5. 在执行完计算操作后，使用print函数输出计算结果（a、b、sum）。这里，我们通过设置allow_copy参数为True，来禁止输出变量ba复制的值，而仅输出变量a的值。


```cpp
lib.ggml_build_forward_expand(gf, sum)

copy(np.array([i for i in range(n)], np.float32), a)
copy(np.array([i*100 for i in range(n)], np.float32), b)

lib.ggml_graph_compute_with_ctx(ctx, gf, n_threads)

print(numpy(a, allow_copy=True))
print(numpy(b))
print(numpy(sum, allow_copy=True))
```

# `examples/python/example_test_all_quants.py`

这段代码的作用是计算二维网格中每个像素的雅克比矩阵（Jacobian Matrix）。

具体来说，该代码从ggml库中导入了一系列函数和数据结构，包括ffi函数、 numpy数组操作库、 copy数据复制库以及math库中的一些数学函数（如cos、sin、pi等）。同时，该代码还引入了matplotlib库用于绘制图像。

然后，根据输入的二维网格数据，该代码首先使用init函数设置了一个大小为100*1024*1024的内存分配缓冲区，用于存储网格数据。接着，该代码定义了一个numpy数组，用于存储原始的网格数据。

在接下来，该代码使用while循环，从网格的第一个像素开始，逐步计算出每个像素的雅克比矩阵。在计算过程中，该代码使用到了一些数学函数，如cos(j * 2 * pi / n)表示将一个角度转换为弧度，pi/n表示将弧度转换为角度，j和i表示当前像素的行和列号。另外，该代码还使用到了一些数据类型转换函数，如int(j * 2 * pi / n)将弧度转换为整数。

最后，该代码将计算得到的雅克比矩阵存储在numpy数组中，并使用math库中的pi函数计算出每个像素的值。由于numpy数组的每个元素都是float32类型，因此每个像素的值也是一个float32。


```cpp
from ggml import ffi, lib
from ggml.utils import init, numpy, copy
import numpy as np
from math import pi, cos, sin, ceil

import matplotlib.pyplot as plt

ctx = init(mem_size=100*1024*1024) # Will be auto-GC'd
n = 256

orig = np.array([
    [
        cos(j * 2 * pi / n) * (sin(i * 2 * pi / n))
        for j in range(n)
    ]
    for i in range(n)
], np.float32)
```

这段代码的作用是创建一个n维的double类型张量，并将其赋值为lib.GGML_new_tensor_2d函数返回的double类型张量。然后，将这个张量浅拷贝一份，以便于在需要时进行修改。

接下来，定义了一个包含lib.GGML_TYPE_COUNT类型中所有类型的列表。接着，遍历该列表，对于每个类型，判断其是否为量化类型，并且如果不是量化类型中的q8和q8k，则将其添加到quants列表中。最后，对quants列表进行排序，并将其输出，以便于输出其中一个元素。


```cpp
orig_tensor = lib.ggml_new_tensor_2d(ctx, lib.GGML_TYPE_F32, n, n)
copy(orig, orig_tensor)

quants = [
    type for type in range(lib.GGML_TYPE_COUNT)
    if lib.ggml_is_quantized(type) and
       type not in [lib.GGML_TYPE_Q8_1, lib.GGML_TYPE_Q8_K] # Apparently not supported
]
# quants = [lib.GGML_TYPE_Q2_K] # Test a single one

def get_name(type):
    name = lib.ggml_type_name(type)
    return ffi.string(name).decode('utf-8') if name else '?'

quants.sort(key=get_name)
```

这段代码的作用是创建一个包含一些图像数据的QuantumInformation对象（quants）列表。然后，创建一个包含4行和至少与quants列表中的元素数量相同的列的图形布局（layout）。

在图表的每一行中，我们为每个元素（即一个QuantumInformation对象）创建一个新的子图（subplot）。对于那些不等于“None”的元素，我们创建一个新的QuantumInformation对象，将原始图像复制并将其转换为浮点数。然后，我们计算与原始图像之间的差异（例如欧氏距离）。

最后，我们为每个元素计算压缩比率，并输出有关压缩率的统计信息。如果出现任何异常，我们将捕获并输出错误消息。


```cpp
quants.insert(0, None)
print(quants)

ncols=4
nrows = ceil(len(quants) / ncols)

plt.figure(figsize=(ncols * 5, nrows * 5), layout='tight')

for i, type in enumerate(quants):
    plt.subplot(nrows, ncols, i + 1)
    try:
        if type == None:
            plt.title('Original')
            plt.imshow(orig)
        else:
            quantized_tensor = lib.ggml_new_tensor_2d(ctx, type, n, n)
            copy(orig_tensor, quantized_tensor)
            quantized = numpy(quantized_tensor, allow_copy=True)
            d = quantized - orig
            results = {
                "l2": np.linalg.norm(d, 2),
                "linf": np.linalg.norm(d, np.inf),
                "compression":
                    round(lib.ggml_nbytes(orig_tensor) /
                          lib.ggml_nbytes(quantized_tensor), 1)
            }
            name = get_name(type)
            print(f'{name}: {results}')

            plt.title(f'{name} ({results["compression"]}x smaller)')
            plt.imshow(quantized, interpolation='nearest')
        
    except Exception as e:
        print(f'Error: {e}')

```

这段代码是使用Python的 Matplotlib 库中的 `plt.show()` 函数来显示图形。

`plt.show()` 函数的作用是显示图形并将其保存到屏幕上。这个函数会在屏幕上显示最后一个 `plt.show()` 函数的图形，并将其保存到剪贴板中，以便在其他程序中使用。

注意，`plt.show()` 函数中的参数 `图形` 必须是已经定义好的图形对象，比如使用 `plt.plot()` 函数绘制出来的图形。如果没有定义图形对象，`plt.show()` 函数会创建一个新的图形并将其保存到剪贴板中，然后将其删除。


```cpp
plt.show()
```

# Simple autogenerated Python bindings for ggml

This folder contains:

- Scripts to generate full Python bindings from ggml headers (+ stubs for autocompletion in IDEs)
- Some barebones utils (see [ggml/utils.py](./ggml/utils.py)):
  - `ggml.utils.init` builds a context that's freed automatically when the pointer gets GC'd
  - `ggml.utils.copy` **copies between same-shaped tensors (numpy or ggml), w/ automatic (de/re)quantization**
  - `ggml.utils.numpy` returns a numpy view over a ggml tensor; if it's quantized, it returns a copy (requires `allow_copy=True`)
- Very basic examples (anyone wants to port [llama2.c](https://github.com/karpathy/llama2.c)?)

Provided you set `GGML_LIBRARY=.../path/to/libggml_shared.so` (see instructions below), it's trivial to do some operations on quantized tensors:

```cpppython
# Make sure libllama.so is in your [DY]LD_LIBRARY_PATH, or set GGML_LIBRARY=.../libggml_shared.so

from ggml import lib, ffi
from ggml.utils import init, copy, numpy
import numpy as np

ctx = init(mem_size=12*1024*1024)
n = 256
n_threads = 4

a = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
b = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n) # Can't both be quantized
sum = lib.ggml_add(ctx, a, b) # all zeroes for now. Will be quantized too!

gf = ffi.new('struct ggml_cgraph*')
lib.ggml_build_forward_expand(gf, sum)

copy(np.array([i for i in range(n)], np.float32), a)
copy(np.array([i*100 for i in range(n)], np.float32), b)

lib.ggml_graph_compute_with_ctx(ctx, gf, n_threads)

print(numpy(a, allow_copy=True))
#  0.    1.0439453   2.0878906   3.131836    4.1757812   5.2197266. ...
print(numpy(b))
#  0.  100.        200.        300.        400.        500.         ...
print(numpy(sum, allow_copy=True))
#  0.  105.4375    210.875     316.3125    421.75      527.1875     ...
```

### Prerequisites

You'll need a shared library of ggml to use the bindings.

#### Build libggml_shared.so or libllama.so

As of this writing the best is to use [ggerganov/llama.cpp](https://github.com/ggerganov/llama.cpp)'s generated `libggml_shared.so` or `libllama.so`, which you can build as follows:

```cppbash
git clone https://github.com/ggerganov/llama.cpp
# On a CUDA-enabled system add -DLLAMA_CUBLAS=1
# On a Mac add -DLLAMA_METAL=1
cmake llama.cpp \
  -B llama_build \
  -DCMAKE_C_FLAGS=-Ofast \
  -DLLAMA_NATIVE=1 \
  -DLLAMA_LTO=1 \
  -DBUILD_SHARED_LIBS=1 \
  -DLLAMA_MPI=1 \
  -DLLAMA_BUILD_TESTS=0 \
  -DLLAMA_BUILD_EXAMPLES=0
( cd llama_build && make -j )

# On Mac, this will be libggml_shared.dylib instead
export GGML_LIBRARY=$PWD/llama_build/libggml_shared.so
# Alternatively, you can just copy it to your system's lib dir, e.g /usr/local/lib
```

#### (Optional) Regenerate the bindings and stubs

If you added or changed any signatures of the C API, you'll want to regenerate the bindings ([ggml/cffi.py](./ggml/cffi.py)) and stubs ([ggml/__init__.pyi](./ggml/__init__.pyi)).

Luckily it's a one-liner using [regenerate.py](./regenerate.py):

```cppbash
pip install -q cffi

python regenerate.py
```

By default it assumes `llama.cpp` was cloned in ../../../llama.cpp (alongside the ggml folder). You can override this with:

```cppbash
C_INCLUDE_DIR=$LLAMA_CPP_DIR python regenerate.py
```

You can also edit [api.h](./api.h) to control which files should be included in the generated bindings (defaults to `llama.cpp/ggml*.h`)

In fact, if you wanted to only generate bindings for the current version of the `ggml` repo itself (instead of `llama.cpp`; you'd loose support for k-quants), you could run:

```cppbash
API=../../include/ggml/ggml.h python regenerate.py
```

## Develop

Run tests:

```cppbash
pytest
```

### Alternatives

This example's goal is to showcase [cffi](https://cffi.readthedocs.io/)-generated bindings that are trivial to use and update, but there are already alternatives in the wild:

- https://github.com/abetlen/ggml-python: these bindings seem to be hand-written and use [ctypes](https://docs.python.org/3/library/ctypes.html). It has [high-quality API reference docs](https://ggml-python.readthedocs.io/en/latest/api-reference/#ggml.ggml) that can be used with these bindings too, but it doesn't expose Metal, CUDA, MPI or OpenCL calls, doesn't support transparent (de/re)quantization like this example does (see [ggml.utils](./ggml/utils.py) module), and won't pick up your local changes.
  
- https://github.com/abetlen/llama-cpp-python: these expose the C++ `llama.cpp` interface, which this example cannot easily be extended to support (`cffi` only generates bindings of C libraries)

- [pybind11](https://github.com/pybind/pybind11) and [nanobind](https://github.com/wjakob/nanobind) are two alternatives to cffi that support binding C++ libraries, but it doesn't seem either of them have an automatic generator (writing bindings is rather time-consuming).


# `examples/python/regenerate.py`

这段代码的作用是生成一个名为"./bindings.c"的文件，该文件包含了ggml库的绑定信息。

首先，它通过读取环境变量来获取LLVM的API名称。如果API无法从环境中获取，则使用默认的名称。

然后，它读取CC变量，如果没有指定CC，则使用“gcc”作为生成器。

接下来，它读取C_INCLUDE_DIR变量，如果没有指定C_INCLUDE_DIR，则将其设置为“../../../llama.cpp”。

最后，它使用cffi库中的函数generate_stubs将stubs.h文件转换为 stubs.c 文件。

除此之外，代码还做了以下事情：

1. 将 sizeof 表达式用它们的值替换。
2. 删除在Darwin头文件中发现的奇异语法。

这些改动使得生成的代码能够与ggml库进行交互，而不会因为奇异语法而出现问题。


```cpp
# Generates bindings for the ggml library.
#
# cffi requires prior C preprocessing of the headers, and it uses pycparser which chokes on a couple of things
# so we help it a bit (e.g. replace sizeof expressions with their value, remove exotic syntax found in Darwin headers).
import os, sys, re, subprocess
import cffi
from stubs import generate_stubs

API = os.environ.get('API', 'api.h')
CC = os.environ.get('CC') or 'gcc'
C_INCLUDE_DIR = os.environ.get('C_INCLUDE_DIR', '../../../llama.cpp')
CPPFLAGS = [
    "-I", C_INCLUDE_DIR,
    '-D__fp16=uint16_t',  # pycparser doesn't support __fp16
    '-D__attribute__(x)=',
    '-D_Static_assert(x, m)=',
] + [x for x in os.environ.get('CPPFLAGS', '').split(' ') if x != '']

```

这段代码是一个Python脚本，它的目的是定义一个名为"header"的变量，并输出一个包含多个文本行的字符串。现在，让我们逐步分析代码的作用。

1. `try:` 是一个try-except语句，用于处理可能出现的外部异常。
2. `header = subprocess.run([CC, "-E", *CPPFLAGS, API], capture_output=True, text=True, check=True)` 是尝试运行一个命令行工具cc -E *.c -o header. 它会首先传递所有的CPPFLAGS参数，然后执行该命令，并将输出和文件提交给try中的try-except语句。 `try-except` 语句会捕获subprocess.CalledProcessError，如果发生错误，则输出错误信息并跳转到最下面一行。
3. `except subprocess.CalledProcessError as e: print(f'{e.stderr}\n{e}', file=sys.stderr); raise` 是一个捕获子process.CalledProcessError的try语句。它将捕获当命令行工具执行失败时产生的任何错误信息，并将其打印到控制台。然后，它调用了一个raise语句，将捕获到的错误信息传递给下一个引起该异常的函数。
4. `header = '\n'.join([l for l in header.split('\n') if '__darwin_va_list' not in l])` 是用于将头文件中的所有行替换为不包含__darwin_va_list的行的函数。 `'\n'` 将分隔符设置为'\n'，`join` 函数将所有的行连接起来。 `


```cpp
try: header = subprocess.run([CC, "-E", *CPPFLAGS, API], capture_output=True, text=True, check=True).stdout
except subprocess.CalledProcessError as e: print(f'{e.stderr}\n{e}', file=sys.stderr); raise

header = '\n'.join([l for l in header.split('\n') if '__darwin_va_list' not in l]) # pycparser hates this

# Replace constant size expressions w/ their value (compile & run a mini exe for each, because why not).
# First, extract anyting *inside* square brackets and anything that looks like a sizeof call.
for expr in set(re.findall(f'(?<=\\[)[^\\]]+(?=])|sizeof\\s*\\([^()]+\\)', header)):
    if re.match(r'^(\d+|\s*)$', expr): continue # skip constants and empty bracket contents
    subprocess.run([CC, "-o", "eval_size_expr", *CPPFLAGS, "-x", "c", "-"], text=True, check=True,
                   input=f'''#include <stdio.h>
                             #include "{API}"
                             int main() {{ printf("%lu", (size_t)({expr})); }}''')
    size = subprocess.run(["./eval_size_expr"], capture_output=True, text=True, check=True).stdout
    print(f'Computed constexpr {expr} = {size}')
    header = header.replace(expr, size)

```

这段代码是一个名为 "ffibuilder" 的函数，它通过使用 CFFI（Common Forward Function Interface）创建一个 C++ 函数，并使用 CFFI 提供的语法对一个名为 "ggml.cffi" 的 C++ 文件进行构建。

具体来说，这段代码执行以下操作：

1. 创建一个名为 "ffibuilder" 的函数，并将其初始化为 cffi.FFI()。
2. 通过 ffi.FFI() 方法，将定义在 "ggml.cffi" 头文件中的函数声明复制到 "ffibuilder" 函数的签名中。
3. 通过设置 "f" 参数，指定要编译的 C++ 文件，如果没有指定文件，则默认情况下会使用 "ggml.cffi" 作为输入文件。
4. 通过调用 "compile" 函数，设置编译时参数为 "verbose=True"，这意味着在编译过程中会输出更多的信息。
5. 通过打开 "ggml/__init__.pyi" 文件，并将其写入 "generate_stubs(header)" 这个函数的输出。
6. 最后，根据输入文件 "ggml.cffi"，生成一些额外的 stub 函数。


```cpp
ffibuilder = cffi.FFI()
ffibuilder.cdef(header)
ffibuilder.set_source(f'ggml.cffi', None) # we're not compiling a native extension, as this quickly gets hairy
ffibuilder.compile(verbose=True)

with open("ggml/__init__.pyi", "wt") as f:
    f.write(generate_stubs(header))
```

# `examples/python/stubs.py`

这段代码的作用是生成.pyi文件中的C++绑定文件。它首先通过扩展`sys.path`以包含`regenerate.py`中生成的.pyi文件，然后使用`pycparser`库读取这些.pyi文件并解析其中的C++代码。

具体来说，这段代码实现以下操作：

1. 通过`sys.path.extend`增加`.和..目录，以便包含`regenerate.py`中的目录。
2. 从`pycparser`库中导入`c_ast`、`parse_file`和`CParser`类，以及`pycparser.plyparser`和`pycparser.c_ast`类。
3. 定义了一个`__c_type_to_python_type`的 mapping，将C++中的类型映射到Python中的类型。
4. 定义了`EllipsisParam`、`IdentifierType`、`FuncDecl`和`Typedef`类，以及`Struct`和`Enum`类。
5. 通过使用`typing.Tuple`类型，创建了一个元组类型，用于表示C++函数参数和返回值的类型。

总之，这段代码的作用是生成一些用于在.pyi文件中声明C++函数的文件，这些函数的参数和返回值类型已经被定义为Python中类型的一部分。


```cpp
"""
  This generates .pyi stubs for the cffi Python bindings generated by regenerate.py
"""
import sys, re, itertools
sys.path.extend(['.', '..']) # for pycparser

from pycparser import c_ast, parse_file, CParser
import pycparser.plyparser
from pycparser.c_ast import PtrDecl, TypeDecl, FuncDecl, EllipsisParam, IdentifierType, Struct, Enum, Typedef
from typing import Tuple

__c_type_to_python_type = {
    'void': 'None', '_Bool': 'bool',
    'char': 'int', 'short': 'int', 'int': 'int', 'long': 'int',
    'ptrdiff_t': 'int', 'size_t': 'int',
    'int8_t': 'int', 'uint8_t': 'int',
    'int16_t': 'int', 'uint16_t': 'int',
    'int32_t': 'int', 'uint32_t': 'int',
    'int64_t': 'int', 'uint64_t': 'int',
    'float': 'float', 'double': 'float',
    'ggml_fp16_t': 'np.float16',
}

```

This is a Python code snippet that generates signal processing code. It takes in a Python function node and generates the required code for it to be interpreted by the IPython engine.

The code snippet first checks if the input node is a pointer type. If it is, it extracts the type and name of the pointer. If it's not a pointer, it then checks if the function has a defined name. If the name is missing, it generates a new name for the function.

The code then gathers the function arguments, which are extracted from the input node's arguments. If the input node has an EllipsisParam, the code generates a new argument name for it.

The code then generates the required code for the function. It starts with the function definition and continues until the end of the code block.

The generated code includes comments to explain what each section is doing. Additionally, it includes code for handling comments, such as extracting the function name from an EllipsisParam.


```cpp
def format_type(t: TypeDecl):
    if isinstance(t, PtrDecl) or isinstance(t, Struct):
        return 'ffi.CData'
    if isinstance(t, Enum):
        return 'int'
    if isinstance(t, TypeDecl):
        return format_type(t.type)
    if isinstance(t, IdentifierType):
        assert len(t.names) == 1, f'Expected a single name, got {t.names}'
        return __c_type_to_python_type.get(t.names[0]) or 'ffi.CData'
    return t.name

class PythonStubFuncDeclVisitor(c_ast.NodeVisitor):
    def __init__(self):
        self.sigs = {}
        self.sources = {}

    def get_source_snippet_lines(self, coord: pycparser.plyparser.Coord) -> Tuple[list[str], list[str]]:
        if coord.file not in self.sources:
            with open(coord.file, 'rt') as f:
                self.sources[coord.file] = f.readlines()
        source_lines = self.sources[coord.file]
        ncomment_lines = len(list(itertools.takewhile(lambda i: re.search(r'^\s*(//|/\*)', source_lines[i]), range(coord.line - 2, -1, -1))))
        comment_lines = [l.strip() for l in source_lines[coord.line - 1 - ncomment_lines:coord.line - 1]]
        decl_lines = []
        for line in source_lines[coord.line - 1:]:
            decl_lines.append(line.rstrip())
            if (';' in line) or ('{' in line): break
        return (comment_lines, decl_lines)

    def visit_Enum(self, node: Enum):
        if node.values is not None:
          for e in node.values.enumerators:
              self.sigs[e.name] = f'  @property\n  def {e.name}(self) -> int: ...'

    def visit_Typedef(self, node: Typedef):
        pass

    def visit_FuncDecl(self, node: FuncDecl):
        ret_type = node.type
        is_ptr = False
        while isinstance(ret_type, PtrDecl):
            ret_type = ret_type.type
            is_ptr = True

        fun_name = ret_type.declname
        if fun_name.startswith('__'):
            return

        args = []
        argnames = []
        def gen_name(stem):
            i = 1
            while True:
                new_name = stem if i == 1 else f'{stem}{i}'
                if new_name not in argnames: return new_name
                i += 1

        for a in node.args.params:
            if isinstance(a, EllipsisParam):
                arg_name = gen_name('args')
                argnames.append(arg_name)
                args.append('*' + gen_name('args'))
            elif format_type(a.type) == 'None':
                continue
            else:
                arg_name = a.name or gen_name('arg')
                argnames.append(arg_name)
                args.append(f'{arg_name}: {format_type(a.type)}')

        ret = format_type(ret_type if not is_ptr else node.type)

        comment_lines, decl_lines = self.get_source_snippet_lines(node.coord)

        lines = [f'  def {fun_name}({", ".join(args)}) -> {ret}:']
        if len(comment_lines) == 0 and len(decl_lines) == 1:
            lines += [f'    """{decl_lines[0]}"""']
        else:
            lines += ['    """']
            lines += [f'    {c.lstrip("/* ")}' for c in comment_lines]
            if len(comment_lines) > 0:
                lines += ['']
            lines += [f'    {d}' for d in decl_lines]
            lines += ['    """']
        lines += ['    ...']
        self.sigs[fun_name] = '\n'.join(lines)

```

这段代码定义了一个名为 `generate_stubs` 的函数，它接受一个头文件（通常是.ggml文件）作为参数。函数的作用是生成一个.pyi格式的 Python stub 文件，用于描述 GGML API 的接口。

具体来说，函数内部使用 `CParser().parse` 方法将头文件解析为 C 语言源代码，然后使用 `PythonStubFuncDeclVisitor` 类遍历 C 语言源文件中的头文件，并提取出定义的函数、变量等信息。

接下来，函数将这些信息组合成一段代码，并将其输出为.pyi格式的文件。在代码中，首先通过 `# auto-generated file` 声明该文件是由程序自动生成的。然后引入了 `ggml.ffi` 和 `numpy` 库，并定义了一个名为 `lib` 的类。

接着，函数通过循环遍历头文件中定义的函数列表，为每个函数签定了名称、参数等信息，最终输出了生成的 stub 文件。


```cpp
def generate_stubs(header: str):
    """
      Generates a .pyi Python stub file for the GGML API using C header files.
    """

    v = PythonStubFuncDeclVisitor()
    v.visit(CParser().parse(header, "<input>"))

    keys = list(v.sigs.keys())
    keys.sort()

    return '\n'.join([
        '# auto-generated file',
        'import ggml.ffi as ffi',
        'import numpy as np',
        'class lib:',
        *[v.sigs[k] for k in keys]
    ])

```

# `examples/python/test_tensor.py`

This is a test case for a function `f32_to_f16()` which converts a 32-bit floating-point number to a 16-bit floating-point number. The function is part of a library that is using the Git请立即 Mail library.

The function takes in a single argument of type `f32_to_f16()` and returns None. The library is using the `lib.ggml_new_tensor_1d()` function to create a new tensor of type `lib.GGML_TYPE_F16` with a value of 1. The `lib.ggml_get_f32_1d()` function is then used to get the value of the tensor and copy it to the input argument.

The function has three test methods:

1. `test_f32_to_f16()`: This test case verifies that the function correctly converts a 32-bit floating-point number to a 16-bit floating-point number.
2. `test_f16_to_f16()`: This test case verifies that the function correctly converts a 16-bit floating-point number to a 16-bit floating-point number.
3. `test_f16_to_f32()`: This test case verifies that the function correctly converts a 16-bit floating-point number to a 32-bit floating-point number.

All of the test methods use the `raises()` method to raise an `NotImplementedError` if the function is not implemented correctly.


```cpp
import pytest
from pytest import raises

from ggml import lib, ffi
from ggml.utils import init, copy, numpy
import numpy as np
import numpy.testing as npt

@pytest.fixture()
def ctx():
    print("setup")
    yield init(mem_size=10*1024*1024)
    print("teardown")

class TestNumPy:
    
    # Single element

    def test_set_get_single_i32(self, ctx):
        i = lib.ggml_new_i32(ctx, 42)
        assert lib.ggml_get_i32_1d(i, 0) == 42
        assert numpy(i) == np.array([42], dtype=np.int32)

    def test_set_get_single_f32(self, ctx):
        i = lib.ggml_new_f32(ctx, 4.2)
        
        epsilon = 0.000001 # Not sure why so large a difference??
        pytest.approx(lib.ggml_get_f32_1d(i, 0), 4.2, epsilon)
        pytest.approx(numpy(i), np.array([4.2], dtype=np.float32), epsilon)

    def _test_copy_np_to_ggml(self, a: np.ndarray, t: ffi.CData):
        a2 = a.copy() # Clone original
        copy(a, t)
        npt.assert_array_equal(numpy(t), a2)

    # I32

    def test_copy_np_to_ggml_1d_i32(self, ctx):
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_I32, 10)
        a = np.arange(10, dtype=np.int32)
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_2d_i32(self, ctx):
        t = lib.ggml_new_tensor_2d(ctx, lib.GGML_TYPE_I32, 2, 3)
        a = np.arange(2 * 3, dtype=np.int32).reshape((2, 3))
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_3d_i32(self, ctx):
        t = lib.ggml_new_tensor_3d(ctx, lib.GGML_TYPE_I32, 2, 3, 4)
        a = np.arange(2 * 3 * 4, dtype=np.int32).reshape((2, 3, 4))
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_4d_i32(self, ctx):
        t = lib.ggml_new_tensor_4d(ctx, lib.GGML_TYPE_I32, 2, 3, 4, 5)
        a = np.arange(2 * 3 * 4 * 5, dtype=np.int32).reshape((2, 3, 4, 5))
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_4d_n_i32(self, ctx):
        dims = [2, 3, 4, 5] # GGML_MAX_DIMS is 4, going beyond would crash
        pdims = ffi.new('int64_t[]', len(dims))
        for i, d in enumerate(dims): pdims[i] = d
        t = lib.ggml_new_tensor(ctx, lib.GGML_TYPE_I32, len(dims), pdims)
        a = np.arange(np.prod(dims), dtype=np.int32).reshape(tuple(pdims))
        self._test_copy_np_to_ggml(a, t)

    # F32

    def test_copy_np_to_ggml_1d_f32(self, ctx):
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 10)
        a = np.arange(10, dtype=np.float32)
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_2d_f32(self, ctx):
        t = lib.ggml_new_tensor_2d(ctx, lib.GGML_TYPE_F32, 2, 3)
        a = np.arange(2 * 3, dtype=np.float32).reshape((2, 3))
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_3d_f32(self, ctx):
        t = lib.ggml_new_tensor_3d(ctx, lib.GGML_TYPE_F32, 2, 3, 4)
        a = np.arange(2 * 3 * 4, dtype=np.float32).reshape((2, 3, 4))
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_4d_f32(self, ctx):
        t = lib.ggml_new_tensor_4d(ctx, lib.GGML_TYPE_F32, 2, 3, 4, 5)
        a = np.arange(2 * 3 * 4 * 5, dtype=np.float32).reshape((2, 3, 4, 5))
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_4d_n_f32(self, ctx):
        dims = [2, 3, 4, 5] # GGML_MAX_DIMS is 4, going beyond would crash
        pdims = ffi.new('int64_t[]', len(dims))
        for i, d in enumerate(dims): pdims[i] = d
        t = lib.ggml_new_tensor(ctx, lib.GGML_TYPE_F32, len(dims), pdims)
        a = np.arange(np.prod(dims), dtype=np.float32).reshape(tuple(pdims))
        self._test_copy_np_to_ggml(a, t)

    # F16

    def test_copy_np_to_ggml_1d_f16(self, ctx):
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F16, 10)
        a = np.arange(10, dtype=np.float16)
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_2d_f16(self, ctx):
        t = lib.ggml_new_tensor_2d(ctx, lib.GGML_TYPE_F16, 2, 3)
        a = np.arange(2 * 3, dtype=np.float16).reshape((2, 3))
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_3d_f16(self, ctx):
        t = lib.ggml_new_tensor_3d(ctx, lib.GGML_TYPE_F16, 2, 3, 4)
        a = np.arange(2 * 3 * 4, dtype=np.float16).reshape((2, 3, 4))
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_4d_f16(self, ctx):
        t = lib.ggml_new_tensor_4d(ctx, lib.GGML_TYPE_F16, 2, 3, 4, 5)
        a = np.arange(2 * 3 * 4 * 5, dtype=np.float16).reshape((2, 3, 4, 5))
        self._test_copy_np_to_ggml(a, t)

    def test_copy_np_to_ggml_4d_n_f16(self, ctx):
        dims = [2, 3, 4, 5] # GGML_MAX_DIMS is 4, going beyond would crash
        pdims = ffi.new('int64_t[]', len(dims))
        for i, d in enumerate(dims): pdims[i] = d
        t = lib.ggml_new_tensor(ctx, lib.GGML_TYPE_F16, len(dims), pdims)
        a = np.arange(np.prod(dims), dtype=np.float16).reshape(tuple(pdims))
        self._test_copy_np_to_ggml(a, t)

    # Mismatching shapes

    def test_copy_mismatching_shapes_1d(self, ctx):
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 10)
        a = np.arange(10, dtype=np.float32)
        copy(a, t) # OK
        
        a = a.reshape((5, 2))
        with raises(AssertionError): copy(a, t)
        with raises(AssertionError): copy(t, a)
            
    def test_copy_mismatching_shapes_2d(self, ctx):
        t = lib.ggml_new_tensor_2d(ctx, lib.GGML_TYPE_F32, 2, 3)
        a = np.arange(6, dtype=np.float32)
        copy(a.reshape((2, 3)), t) # OK
        
        a = a.reshape((3, 2))
        with raises(AssertionError): copy(a, t)
        with raises(AssertionError): copy(t, a)

    def test_copy_mismatching_shapes_3d(self, ctx):
        t = lib.ggml_new_tensor_3d(ctx, lib.GGML_TYPE_F32, 2, 3, 4)
        a = np.arange(24, dtype=np.float32)
        copy(a.reshape((2, 3, 4)), t) # OK
        
        a = a.reshape((2, 4, 3))
        with raises(AssertionError): copy(a, t)
        with raises(AssertionError): copy(t, a)

    def test_copy_mismatching_shapes_4d(self, ctx):
        t = lib.ggml_new_tensor_4d(ctx, lib.GGML_TYPE_F32, 2, 3, 4, 5)
        a = np.arange(24*5, dtype=np.float32)
        copy(a.reshape((2, 3, 4, 5)), t) # OK
        
        a = a.reshape((2, 3, 5, 4))
        with raises(AssertionError): copy(a, t)
        with raises(AssertionError): copy(t, a)

    def test_copy_f16_to_f32(self, ctx):
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 1)
        a = np.array([123.45], dtype=np.float16)
        copy(a, t)
        np.testing.assert_allclose(lib.ggml_get_f32_1d(t, 0), 123.45, rtol=1e-3)

    def test_copy_f32_to_f16(self, ctx):
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F16, 1)
        a = np.array([123.45], dtype=np.float32)
        copy(a, t)
        np.testing.assert_allclose(lib.ggml_get_f32_1d(t, 0), 123.45, rtol=1e-3)

    def test_copy_f16_to_Q5_K(self, ctx):
        n = 256
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
        a = np.arange(n, dtype=np.float16)
        copy(a, t)
        np.testing.assert_allclose(a, numpy(t, allow_copy=True), rtol=0.05)

    def test_copy_Q5_K_to_f16(self, ctx):
        n = 256
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
        copy(np.arange(n, dtype=np.float32), t)
        a = np.arange(n, dtype=np.float16)
        copy(t, a)
        np.testing.assert_allclose(a, numpy(t, allow_copy=True), rtol=0.05)

    def test_copy_i16_f32_mismatching_types(self, ctx):
        t = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 1)
        a = np.arange(1, dtype=np.int16)
        with raises(NotImplementedError): copy(a, t)
        with raises(NotImplementedError): copy(t, a)

```

这段代码定义了一个名为 TestTensorCopy 的类，其中包含两个测试方法：

1. `test_copy_self`：这个方法测试的是在同一个 Tensor 上进行复制操作，它创建了一个浮点数类型的 Tensor，并将其复制到另一个相同的 Tensor 上。然后，通过 `lib.ggml_get_i32_1d` 方法来测试 Tensor 是否正确复制。如果 Tensor 正确复制，则输出 "Success"。

2. `test_copy_1d`：这个方法测试的是在同一个 Tensor 上进行不同长度的复制操作，它创建了一个 1D 浮点数类型的 Tensor，并将其复制到另一个相同的 1D 浮点数类型的 Tensor 上。然后，通过 `lib.ggml_new_tensor_1d` 方法来创建两个 Tensor，一个 Tensor 包含从 0 到 9 的整数，另一个 Tensor 包含从 0 到 9 的浮点数。接下来，通过 `np.arange` 函数来创建一个包含从 1 到 10 的整数的 NumPy 数组。然后，使用 `copy` 函数将 NumPy 数组复制到 Tensor1 上，使用 `copy` 函数将 Tensor1 复制到 Tensor2 上，最后使用 `numpy` 函数和 `allclose` 函数来测试两个 Tensor 是否相互靠近。如果 Tensor1 和 Tensor2 正确复制，则输出 "Success"。


```cpp
class TestTensorCopy:

    def test_copy_self(self, ctx):
        t = lib.ggml_new_i32(ctx, 42)
        copy(t, t)
        assert lib.ggml_get_i32_1d(t, 0) == 42

    def test_copy_1d(self, ctx):
        t1 = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 10)
        t2 = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, 10)
        a = np.arange(10, dtype=np.float32)
        copy(a, t1)
        copy(t1, t2)
        assert np.allclose(a, numpy(t2))
        assert np.allclose(numpy(t1), numpy(t2))

```

这段代码定义了一个名为 TestGraph 的类，其中的 `test_add` 方法对一个输入图（每行是一个节点）和一个二进制数组进行操作，以计算图中的连通性。

具体来说，这段代码实现以下步骤：

1. 定义了一个输入图 `ctx`，它是一个含有 `n` 个节点的输入图。
2. 创建了两个输入图 `ta` 和 `tb`，它们的类型都是 `lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)`，即每个输入图都是从节点 0 到节点 `n` 的一个单精度实数向量。
3. 使用 `lib.ggml_add(ctx, ta, tb)` 函数计算两个输入图的并集，结果保存到输出向量 `tsum` 中。
4. 检查 `tsum` 的类型是否为 `lib.ggml_type_f32`，如果是，则执行后续操作。
5. 创建了一个输出图 `gf`，其类型为 `ffi.new('struct ggml_cgraph*')`，即一个指向图的指针。
6. 使用 `lib.ggml_build_forward_expand(gf, tsum)` 函数，根据 `tsum` 的类型构建输入图的链表，并将其输出到 `gf`。
7. 使用 `lib.ggml_graph_compute_with_ctx(ctx, gf, 1)` 函数，在 `ctx` 和 `gf` 之间的边界计算图中所有节点的度数。
8. 最后，检查计算得到的度数是否与输入图中的节点连通性一致。


```cpp
class TestGraph:

    def test_add(self, ctx):
        n = 256
        ta = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)
        tb = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)
        tsum = lib.ggml_add(ctx, ta, tb)
        assert tsum.type == lib.GGML_TYPE_F32

        gf = ffi.new('struct ggml_cgraph*')
        lib.ggml_build_forward_expand(gf, tsum)

        a = np.arange(0, n, dtype=np.float32)
        b = np.arange(n, 0, -1, dtype=np.float32)
        copy(a, ta)
        copy(b, tb)

        lib.ggml_graph_compute_with_ctx(ctx, gf, 1)

        assert np.allclose(numpy(tsum, allow_copy=True), a + b)

```

这段代码是一个名为 `TestQuantization` 的测试类，用于测试量化算法的性能。它包含一个测试函数 `test_quantized_add`，该函数使用lib.ggml_new_tensor_1d函数创建了两个张量，一个来自 `lib.GGML_TYPE_Q5_K` 类型的一个节点，一个来自 `lib.GGML_TYPE_F32` 类型的一个节点。然后，它使用lib.ggml_add函数对这些张量进行加法操作，并使用assert语句来验证张量的类型是否为`lib.GGML_TYPE_Q5_K`。

在代码的下面部分，它创建了一个名为 `test_example` 的函数，该函数创建了一个包含n+1个节点的张量，并将两个张量从n+1个节点的张量中相加。然后，它使用lib.ggml_graph_compute_with_ctx函数对张量进行计算，并使用numpy库的`sum`函数将结果存储为numpy数组。

最后，它包含一个测试函数，该函数使用示例函数计算一个未量化和张量的差值的平方根，并使用assert语句来验证结果是否符合预期。


```cpp
class TestQuantization:

    def test_quantized_add(self, ctx):
        n = 256
        ta = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_Q5_K, n)
        tb = lib.ggml_new_tensor_1d(ctx, lib.GGML_TYPE_F32, n)
        tsum = lib.ggml_add(ctx, ta, tb)
        assert tsum.type == lib.GGML_TYPE_Q5_K

        gf = ffi.new('struct ggml_cgraph*')
        lib.ggml_build_forward_expand(gf, tsum)

        a = np.arange(0, n, dtype=np.float32)
        b = np.arange(n, 0, -1, dtype=np.float32)
        copy(a, ta)
        copy(b, tb)

        lib.ggml_graph_compute_with_ctx(ctx, gf, 1)

        unquantized_sum = a + b
        sum = numpy(tsum, allow_copy=True)

        diff = np.linalg.norm(unquantized_sum - sum, np.inf)
        assert diff > 4
        assert diff < 5

```

# `examples/python/ggml/cffi.py`

It looks like you have provided a C++ header file with several uber的可移植的功能ensions，拓展，定义，用，b:".\x00\x00\x03\xBF"...\x00\x00\x04\x4A"...\x00\x00\x03\xBA"...\x00\x00\x00"...\x00\x00\x6C"...\x00\x00\x03\xBF"...\x00\x00\x04\x4A"...\x00\x00\x03\xBA"...".\x00\x00\x03\xBA"...\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".\x00\x00\x03\xBA"...".\x00\x00\x04\x4A"...".



```cpp
# auto-generated file
import _cffi_backend

ffi = _cffi_backend.FFI('ggml.cffi',
    _version = 0x2601,
    _types = b'\x00\x00\xB6\x0D\x00\x00\x09\x0B\x00\x00\x00\x0F\x00\x00\xB6\x0D\x00\x04\x2F\x03\x00\x00\x00\x0F\x00\x00\xB6\x0D\x00\x04\x31\x03\x00\x04\x3D\x03\x00\x00\x00\x0F\x00\x00\xB6\x0D\x00\x04\x32\x03\x00\x00\x00\x0F\x00\x00\xB6\x0D\x00\x04\x34\x03\x00\x03\xFE\x03\x00\x04\x53\x03\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x00\xB6\x0D\x00\x04\x3D\x03\x00\x00\x00\x0F\x00\x00\xB6\x0D\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x00\xB6\x0D\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x00\xB6\x0D\x00\x04\x3E\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\xB6\x0D\x00\x00\x10\x11\x00\x00\x00\x0F\x00\x00\xB6\x0D\x00\x00\x00\x0F\x00\x02\xD0\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x0F\x0D\x00\x00\x04\x0B\x00\x00\x00\x0F\x00\x00\x0F\x0D\x00\x00\x01\x11\x00\x00\x00\x0F\x00\x00\x0F\x0D\x00\x00\x0B\x0B\x00\x00\x00\x0F\x00\x00\x0F\x0D\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x00\x0F\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x0F\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x16\x0D\x00\x00\x0B\x11\x00\x04\x38\x03\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x04\x16\x0D\x00\x00\x0B\x11\x00\x00\x44\x11\x00\x00\x08\x11\x00\x04\x30\x03\x00\x00\x4B\x11\x00\x00\x00\x0F\x00\x04\x16\x0D\x00\x00\x0B\x11\x00\x00\x20\x09\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x00\x01\x0D\x00\x00\x01\x0B\x00\x00\x00\x0F\x00\x01\x14\x0D\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x00\x34\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x02\x7E\x0D\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x00\xF4\x0D\x00\x00\x01\x11\x00\x00\x00\x0F\x00\x00\xF4\x0D\x00\x00\x15\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\xF4\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\xF4\x0D\x00\x00\x06\x01\x00\x00\x00\x0F\x00\x04\x18\x0D\x00\x00\x01\x11\x00\x00\x00\x0F\x00\x02\xE9\x0D\x00\x00\x0E\x11\x00\x00\x00\x0F\x00\x00\x22\x0D\x00\x00\x01\x11\x00\x00\x00\x0F\x00\x00\x22\x0D\x00\x00\x4B\x11\x00\x04\x33\x03\x00\x00\x00\x0F\x00\x00\x22\x0D\x00\x00\x0E\x11\x00\x00\x00\x0F\x00\x00\x22\x0D\x00\x04\x35\x03\x00\x00\x00\x0F\x00\x00\x22\x0D\x00\x00\x15\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x22\x0D\x00\x00\x21\x11\x00\x00\x00\x0F\x00\x00\x22\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x00\x0F\x00\x00\x22\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x22\x0D\x00\x00\x00\x0F\x00\x00\xDB\x0D\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x00\xDB\x0D\x00\x00\x00\x0F\x00\x03\xB0\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x03\xB5\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x04\x0D\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x00\x04\x0D\x00\x00\x10\x11\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x00\x4B\x0D\x00\x00\x0B\x11\x00\x00\x00\x0F\x00\x00\x4B\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x04\x30\x0D\x00\x00\x0F\x11\x00\x00\x0B\x03\x00\x00\xB0\x11\x00\x00\x00\x0F\x00\x04\x30\x0D\x00\x00\x0B\x11\x00\x00\x4B\x11\x00\x00\x01\x01\x00\x00\x00\x0F\x00\x04\x30\x0D\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x00\x0B\x0D\x00\x00\x1B\x09\x00\x00\x00\x0F\x00\x04\x33\x0D\x00\x00\x4B\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x0E\x0D\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x7F\x0D\x00\x00\x00\x0F\x00\x00\x50\x0D\x00\x00\x07\x0B\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x4B\x11\x00\x00\x0F\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x0F\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x01\x11\x00\x00\x07\x01\x00\x00\xDB\x03\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x01\x11\x00\x00\x0B\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x01\x11\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x01\x11\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x01\x11\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x0D\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x05\x0B\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x01\x01\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0A\x0B\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0D\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0D\x01\x00\x00\x0D\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x0D\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x0D\x01\x00\x00\x0D\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0B\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0B\x01\x00\x00\x0B\x01\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x0B\x01\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x01\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x01\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x03\x5C\x03\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x03\x62\x03\x00\x00\x07\x01\x00\x00\x10\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x02\xD8\x03\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x03\x4F\x03\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x08\x11\x00\x03\x54\x03\x00\x00\x07\x01\x00\x00\x10\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x02\xD3\x03\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x03\x44\x03\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x03\x48\x03\x00\x00\x07\x01\x00\x00\x10\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x0B\x11\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x08\x11\x00\x00\x0F\x11\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x08\x11\x00\x00\x0F\x11\x00\x00\x01\x0F\x00\x00\x08\x0D\x00\x00\x08\x11\x00\x00\x0D\x01\x00\x00\x00\x0F\x00\x00\x08\x0D\x00\x00\x08\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x21\x0D\x00\x00\x0F\x11\x00\x00\x24\x09\x00\x00\x00\x0F\x00\x00\x21\x0D\x00\x00\x00\x0F\x00\x03\xBA\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x03\xBF\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x00\x01\x11\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x00\x01\x11\x00\x00\xF4\x03\x00\x00\x10\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\xDB\x03\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x02\x35\x11\x00\x00\x10\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x02\x39\x11\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x00\x04\x11\x00\x00\x4B\x11\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x00\x0B\x11\x00\x00\x21\x09\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x04\x32\x03\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x00\x15\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x00\x21\x11\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x11\x0D\x00\x00\x00\x0F\x00\x00\x6C\x0D\x00\x00\x0D\x01\x00\x00\x00\x0F\x00\x00\x6C\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x00\x10\x0D\x00\x02\x4B\x11\x00\x00\x00\x0F\x00\x00\x10\x0D\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x00\x10\x0D\x00\x00\x21\x11\x00\x00\x00\x0F\x00\x00\x10\x0D\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x02\xE1\x0D\x00\x00\x21\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x01\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x03\xF8\x03\x00\x00\xF4\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x03\xF9\x03\x00\x02\x7E\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x03\xFA\x03\x00\x02\x7E\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x03\xFB\x03\x00\x02\x7E\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x03\xFC\x03\x00\x02\x7E\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x03\xFD\x03\x00\x02\x7E\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0F\x11\x00\x00\x0F\x11\x00\x00\x07\x01\x00\x00\x0F\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\x35\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\x35\x11\x00\x03\xF8\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\x35\x11\x00\x03\xF9\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\x35\x11\x00\x03\xFA\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\x35\x11\x00\x03\xFB\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\x35\x11\x00\x03\xFC\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\x35\x11\x00\x03\xFD\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\x35\x11\x00\x00\x6C\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\x35\x11\x00\x00\x10\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x07\x01\x00\x03\xFE\x03\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x07\x01\x00\x02\x7E\x11\x00\x02\x35\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x07\x01\x00\x02\x7E\x11\x00\x02\x35\x11\x00\x02\x35\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x07\x01\x00\x02\x7E\x11\x00\x04\x53\x03\x00\x02\xE1\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x04\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x04\x11\x00\x00\x22\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x04\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x4B\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x4B\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x04\x30\x03\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\xF8\x11\x00\x00\x0F\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\xF8\x11\x00\x02\xF8\x11\x00\x00\x0F\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0B\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0B\x11\x00\x00\x01\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0B\x11\x00\x00\x4B\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0B\x11\x00\x00\x44\x11\x00\x00\x50\x11\x00\x00\x0B\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0B\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\x4B\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0E\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0E\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0E\x11\x00\x00\x4B\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0E\x11\x00\x00\x4B\x11\x00\x00\x01\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0E\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x7F\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x7F\x11\x00\x02\xE9\x11\x00\x02\xE9\x11\x00\x02\xE9\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x7F\x11\x00\x00\x4B\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x04\x37\x03\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x08\x11\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x08\x11\x00\x00\x15\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x10\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x08\x11\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x08\x11\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x10\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x08\x11\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x08\x11\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x10\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x15\x11\x00\x00\x07\x01\x00\x00\x0D\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x15\x11\x00\x00\x07\x01\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x15\x11\x00\x00\x15\x11\x00\x00\x08\x11\x00\x00\x10\x11\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x01\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x0F\x03\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x0F\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x01\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x34\x11\x00\x02\xE1\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x0D\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x05\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x03\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x04\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x08\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x00\x06\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x0F\x11\x00\x02\xE1\x11\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x15\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x21\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x21\x11\x00\x00\x10\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x0A\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x6C\x03\x00\x02\x7E\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x10\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x10\x11\x00\x00\x08\x11\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x02\xE1\x11\x00\x02\x7E\x11\x00\x00\x07\x01\x00\x00\x00\x0F\x00\x04\x53\x0D\x00\x00\x00\x0F\x00\x00\x24\x03\x00\x00\x0D\x09\x00\x00\x0E\x09\x00\x00\x0F\x09\x00\x00\x10\x09\x00\x00\x11\x09\x00\x00\x12\x09\x00\x00\x13\x09\x00\x00\x14\x09\x00\x00\x04\x09\x00\x00\x05\x09\x00\x00\x06\x09\x00\x00\x07\x09\x00\x00\x08\x09\x00\x00\x09\x09\x00\x00\x0A\x09\x00\x00\x02\x01\x00\x03\xFE\x05\x00\x00\x00\x80\x00\x03\xFE\x05\x00\x00\x00\x10\x00\x03\xFE\x05\x00\x00\x00\xC0\x00\x03\xFE\x05\x00\x00\x00\x25\x00\x03\xFE\x05\x00\x00\x00\x28\x00\x03\xFE\x05\x00\x00\x00\x04\x00\x03\xFE\x05\x00\x00\x00\x38\x00\x03\xFE\x05\x00\x00\x00\x40\x00\x03\xFE\x05\x00\x00\x1F\xF0\x00\x03\xFE\x05\x00\x00\x00\x08\x00\x00\x00\x0B\x00\x00\x02\x0B\x00\x00\x03\x0B\x00\x00\x06\x0B\x00\x00\x08\x0B\x00\x00\x0B\x09\x00\x00\x22\x05\x00\x00\x10\x00\x00\x00\x22\x05\x00\x00\x00\x08\x00\x00\x0F\x01\x00\x00\xDB\x05\x00\x00\x00\x04\x00\x00\x09\x01\x00\x03\xB0\x05\x00\x00\x00\x10\x00\x03\xB5\x05\x00\x00\x00\x10\x00\x03\xB5\x05\x00\x00\x01\x00\x00\x00\x00\x09\x00\x00\x01\x09\x00\x00\x02\x09\x00\x00\x03\x09\x00\x04\x2C\x03\x00\x00\x0C\x09\x00\x04\x2E\x03\x00\x00\x15\x09\x00\x00\x16\x09\x00\x00\x17\x09\x00\x00\x18\x09\x00\x00\x19\x09\x00\x00\x1A\x09\x00\x00\x1C\x09\x00\x00\x1D\x09\x00\x04\x37\x03\x00\x00\x1E\x09\x00\x00\x1F\x09\x00\x00\x08\x05\x00\x00\x10\x00\x00\x00\x08\x05\x00\x00\x00\x06\x00\x00\x22\x09\x00\x00\x23\x09\x00\x03\xBA\x03\x00\x03\xBA\x05\x00\x00\x00\x80\x00\x03\xBA\x05\x00\x00\x00\x0C\x00\x03\xBA\x05\x00\x00\x00\x10\x00\x03\xBA\x05\x00\x00\x00\x20\x00\x03\xBA\x05\x00\x00\x00\x40\x00\x00\x0C\x01\x00\x00\x11\x05\x00\x00\x00\x04\x00\x00\x10\x05\x00\x00\x20\x51\x00\x02\xC6\x03\x00\x02\xDE\x03\x00\x03\xE0\x03\x00\x03\xE7\x03\x00\x00\x00\x01',
    _globals = (b'\xFF\xFF\xFF\x0BGGML_BACKEND_CPU',0,b'\xFF\xFF\xFF\x0BGGML_BACKEND_GPU',10,b'\xFF\xFF\xFF\x0BGGML_BACKEND_GPU_SPLIT',20,b'\xFF\xFF\xFF\x0BGGML_FTYPE_ALL_F32',0,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_F16',1,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q2_K',10,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q3_K',11,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q4_0',2,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q4_1',3,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q4_1_SOME_F16',4,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q4_K',12,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q5_0',8,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q5_1',9,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q5_K',13,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q6_K',14,b'\xFF\xFF\xFF\x0BGGML_FTYPE_MOSTLY_Q8_0',7,b'\xFF\xFF\xFF\x0BGGML_FTYPE_UNKNOWN',-1,b'\xFF\xFF\xFF\x1FGGML_GRAPH_SIZE',164520,b'\xFF\xFF\xFF\x0BGGML_LINESEARCH_BACKTRACKING_ARMIJO',0,b'\xFF\xFF\xFF\x0BGGML_LINESEARCH_BACKTRACKING_STRONG_WOLFE',2,b'\xFF\xFF\xFF\x0BGGML_LINESEARCH_BACKTRACKING_WOLFE',1,b'\xFF\xFF\xFF\x0BGGML_LINESEARCH_DEFAULT',1,b'\xFF\xFF\xFF\x0BGGML_LINESEARCH_FAIL',-128,b'\xFF\xFF\xFF\x0BGGML_LINESEARCH_INVALID_PARAMETERS',-124,b'\xFF\xFF\xFF\x0BGGML_LINESEARCH_MAXIMUM_ITERATIONS',-125,b'\xFF\xFF\xFF\x0BGGML_LINESEARCH_MAXIMUM_STEP',-126,b'\xFF\xFF\xFF\x0BGGML_LINESEARCH_MINIMUM_STEP',-127,b'\xFF\xFF\xFF\x0BGGML_OBJECT_GRAPH',1,b'\xFF\xFF\xFF\x1FGGML_OBJECT_SIZE',32,b'\xFF\xFF\xFF\x0BGGML_OBJECT_TENSOR',0,b'\xFF\xFF\xFF\x0BGGML_OBJECT_WORK_BUFFER',2,b'\xFF\xFF\xFF\x0BGGML_OPT_ADAM',0,b'\xFF\xFF\xFF\x0BGGML_OPT_DID_NOT_CONVERGE',1,b'\xFF\xFF\xFF\x0BGGML_OPT_FAIL',4,b'\xFF\xFF\xFF\x0BGGML_OPT_INVALID_WOLFE',3,b'\xFF\xFF\xFF\x0BGGML_OPT_LBFGS',1,b'\xFF\xFF\xFF\x0BGGML_OPT_NO_CONTEXT',2,b'\xFF\xFF\xFF\x0BGGML_OPT_OK',0,b'\xFF\xFF\xFF\x0BGGML_OP_ACC',4,b'\xFF\xFF\xFF\x0BGGML_OP_ADD',2,b'\xFF\xFF\xFF\x0BGGML_OP_ADD1',3,b'\xFF\xFF\xFF\x0BGGML_OP_ALIBI',40,b'\xFF\xFF\xFF\x0BGGML_OP_ARGMAX',14,b'\xFF\xFF\xFF\x0BGGML_OP_CLAMP',41,b'\xFF\xFF\xFF\x0BGGML_OP_CONT',26,b'\xFF\xFF\xFF\x0BGGML_OP_CONV_1D',42,b'\xFF\xFF\xFF\x0BGGML_OP_CONV_2D',43,b'\xFF\xFF\xFF\x0BGGML_OP_COUNT',62,b'\xFF\xFF\xFF\x0BGGML_OP_CPY',25,b'\xFF\xFF\xFF\x0BGGML_OP_CROSS_ENTROPY_LOSS',60,b'\xFF\xFF\xFF\x0BGGML_OP_CROSS_ENTROPY_LOSS_BACK',61,b'\xFF\xFF\xFF\x0BGGML_OP_DIAG',33,b'\xFF\xFF\xFF\x0BGGML_OP_DIAG_MASK_INF',34,b'\xFF\xFF\xFF\x0BGGML_OP_DIAG_MASK_ZERO',35,b'\xFF\xFF\xFF\x0BGGML_OP_DIV',7,b'\xFF\xFF\xFF\x0BGGML_OP_DUP',1,b'\xFF\xFF\xFF\x0BGGML_OP_FLASH_ATTN',46,b'\xFF\xFF\xFF\x0BGGML_OP_FLASH_ATTN_BACK',48,b'\xFF\xFF\xFF\x0BGGML_OP_FLASH_FF',47,b'\xFF\xFF\xFF\x0BGGML_OP_GET_ROWS',31,b'\xFF\xFF\xFF\x0BGGML_OP_GET_ROWS_BACK',32,b'\xFF\xFF\xFF\x0BGGML_OP_LOG',10,b'\xFF\xFF\xFF\x0BGGML_OP_MAP_BINARY',53,b'\xFF\xFF\xFF\x0BGGML_OP_MAP_CUSTOM1',57,b'\xFF\xFF\xFF\x0BGGML_OP_MAP_CUSTOM1_F32',54,b'\xFF\xFF\xFF\x0BGGML_OP_MAP_CUSTOM2',58,b'\xFF\xFF\xFF\x0BGGML_OP_MAP_CUSTOM2_F32',55,b'\xFF\xFF\xFF\x0BGGML_OP_MAP_CUSTOM3',59,b'\xFF\xFF\xFF\x0BGGML_OP_MAP_CUSTOM3_F32',56,b'\xFF\xFF\xFF\x0BGGML_OP_MAP_UNARY',52,b'\xFF\xFF\xFF\x0BGGML_OP_MEAN',13,b'\xFF\xFF\xFF\x0BGGML_OP_MUL',6,b'\xFF\xFF\xFF\x0BGGML_OP_MUL_MAT',21,b'\xFF\xFF\xFF\x0BGGML_OP_NONE',0,b'\xFF\xFF\xFF\x0BGGML_OP_NORM',18,b'\xFF\xFF\xFF\x0BGGML_OP_OUT_PROD',22,b'\xFF\xFF\xFF\x0BGGML_OP_PERMUTE',29,b'\xFF\xFF\xFF\x0BGGML_OP_POOL_1D',44,b'\xFF\xFF\xFF\x0BGGML_OP_POOL_2D',45,b'\xFF\xFF\xFF\x0BGGML_OP_POOL_AVG',1,b'\xFF\xFF\xFF\x0BGGML_OP_POOL_COUNT',2,b'\xFF\xFF\xFF\x0BGGML_OP_POOL_MAX',0,b'\xFF\xFF\xFF\x0BGGML_OP_REPEAT',15,b'\xFF\xFF\xFF\x0BGGML_OP_REPEAT_BACK',16,b'\xFF\xFF\xFF\x0BGGML_OP_RESHAPE',27,b'\xFF\xFF\xFF\x0BGGML_OP_RMS_NORM',19,b'\xFF\xFF\xFF\x0BGGML_OP_RMS_NORM_BACK',20,b'\xFF\xFF\xFF\x0BGGML_OP_ROPE',38,b'\xFF\xFF\xFF\x0BGGML_OP_ROPE_BACK',39,b'\xFF\xFF\xFF\x0BGGML_OP_SCALE',23,b'\xFF\xFF\xFF\x0BGGML_OP_SET',24,b'\xFF\xFF\xFF\x0BGGML_OP_SILU_BACK',17,b'\xFF\xFF\xFF\x0BGGML_OP_SOFT_MAX',36,b'\xFF\xFF\xFF\x0BGGML_OP_SOFT_MAX_BACK',37,b'\xFF\xFF\xFF\x0BGGML_OP_SQR',8,b'\xFF\xFF\xFF\x0BGGML_OP_SQRT',9,b'\xFF\xFF\xFF\x0BGGML_OP_SUB',5,b'\xFF\xFF\xFF\x0BGGML_OP_SUM',11,b'\xFF\xFF\xFF\x0BGGML_OP_SUM_ROWS',12,b'\xFF\xFF\xFF\x0BGGML_OP_TRANSPOSE',30,b'\xFF\xFF\xFF\x0BGGML_OP_UNARY',51,b'\xFF\xFF\xFF\x0BGGML_OP_VIEW',28,b'\xFF\xFF\xFF\x0BGGML_OP_WIN_PART',49,b'\xFF\xFF\xFF\x0BGGML_OP_WIN_UNPART',50,b'\xFF\xFF\xFF\x0BGGML_TASK_COMPUTE',1,b'\xFF\xFF\xFF\x0BGGML_TASK_FINALIZE',2,b'\xFF\xFF\xFF\x0BGGML_TASK_INIT',0,b'\xFF\xFF\xFF\x1FGGML_TENSOR_SIZE',288,b'\xFF\xFF\xFF\x0BGGML_TYPE_COUNT',19,b'\xFF\xFF\xFF\x0BGGML_TYPE_F16',1,b'\xFF\xFF\xFF\x0BGGML_TYPE_F32',0,b'\xFF\xFF\xFF\x0BGGML_TYPE_I16',17,b'\xFF\xFF\xFF\x0BGGML_TYPE_I32',18,b'\xFF\xFF\xFF\x0BGGML_TYPE_I8',16,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q2_K',10,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q3_K',11,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q4_0',2,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q4_1',3,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q4_K',12,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q5_0',6,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q5_1',7,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q5_K',13,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q6_K',14,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q8_0',8,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q8_1',9,b'\xFF\xFF\xFF\x0BGGML_TYPE_Q8_K',15,b'\xFF\xFF\xFF\x0BGGML_UNARY_OP_ABS',0,b'\xFF\xFF\xFF\x0BGGML_UNARY_OP_ELU',5,b'\xFF\xFF\xFF\x0BGGML_UNARY_OP_GELU',7,b'\xFF\xFF\xFF\x0BGGML_UNARY_OP_GELU_QUICK',8,b'\xFF\xFF\xFF\x0BGGML_UNARY_OP_NEG',2,b'\xFF\xFF\xFF\x0BGGML_UNARY_OP_RELU',6,b'\xFF\xFF\xFF\x0BGGML_UNARY_OP_SGN',1,b'\xFF\xFF\xFF\x0BGGML_UNARY_OP_SILU',9,b'\xFF\xFF\xFF\x0BGGML_UNARY_OP_STEP',3,b'\xFF\xFF\xFF\x0BGGML_UNARY_OP_TANH',4,b'\xFF\xFF\xFF\x0BGGUF_TYPE_ARRAY',9,b'\xFF\xFF\xFF\x0BGGUF_TYPE_BOOL',7,b'\xFF\xFF\xFF\x0BGGUF_TYPE_COUNT',10,b'\xFF\xFF\xFF\x0BGGUF_TYPE_FLOAT32',6,b'\xFF\xFF\xFF\x0BGGUF_TYPE_INT16',3,b'\xFF\xFF\xFF\x0BGGUF_TYPE_INT32',5,b'\xFF\xFF\xFF\x0BGGUF_TYPE_INT8',1,b'\xFF\xFF\xFF\x0BGGUF_TYPE_STRING',8,b'\xFF\xFF\xFF\x0BGGUF_TYPE_UINT16',2,b'\xFF\xFF\xFF\x0BGGUF_TYPE_UINT32',4,b'\xFF\xFF\xFF\x0BGGUF_TYPE_UINT8',0,b'\x00\x02\x9A\x23__assert_rtn',0,b'\x00\x02\x7C\x23dequantize_row_q2_K',0,b'\x00\x02\x81\x23dequantize_row_q3_K',0,b'\x00\x02\x86\x23dequantize_row_q4_K',0,b'\x00\x02\x8B\x23dequantize_row_q5_K',0,b'\x00\x02\x90\x23dequantize_row_q6_K',0,b'\x00\x02\x95\x23dequantize_row_q8_K',0,b'\x00\x00\xFA\x23ggml_abs',0,b'\x00\x00\xFA\x23ggml_abs_inplace',0,b'\x00\x01\xDD\x23ggml_acc',0,b'\x00\x01\xDD\x23ggml_acc_inplace',0,b'\x00\x01\x84\x23ggml_add',0,b'\x00\x01\x84\x23ggml_add1',0,b'\x00\x01\x84\x23ggml_add1_inplace',0,b'\x00\x01\x84\x23ggml_add_inplace',0,b'\x00\x01\x26\x23ggml_alibi',0,b'\x00\x02\xEC\x23ggml_allocr_alloc',0,b'\x00\x02\x42\x23ggml_allocr_alloc_graph',0,b'\x00\x02\xE4\x23ggml_allocr_free',0,b'\x00\x00\x03\x23ggml_allocr_is_measure',0,b'\x00\x00\xA2\x23ggml_allocr_new',0,b'\x00\x00\x9F\x23ggml_allocr_new_measure',0,b'\x00\x02\xE4\x23ggml_allocr_reset',0,b'\x00\x02\xE7\x23ggml_allocr_set_parse_seq',0,b'\x00\x00\x17\x23ggml_are_same_shape',0,b'\x00\x00\xFA\x23ggml_argmax',0,b'\x00\x00\x74\x23ggml_blck_size',0,b'\x00\x00\xB3\x23ggml_build_backward',0,b'\x00\x00\xB8\x23ggml_build_forward',0,b'\x00\x00\xAA\x23ggml_build_forward_ctx',0,b'\x00\x02\xF3\x23ggml_build_forward_expand',0,b'\x00\x00\x1B\x23ggml_cl_can_mul_mat',0,b'\x00\x03\x6B\x23ggml_cl_free_data',0,b'\x00\x03\xE0\x23ggml_cl_host_free',0,b'\x00\x02\x72\x23ggml_cl_host_malloc',0,b'\x00\x03\xEC\x23ggml_cl_init',0,b'\x00\x03\x78\x23ggml_cl_mul',0,b'\x00\x03\x7D\x23ggml_cl_mul_mat',0,b'\x00\x02\x54\x23ggml_cl_mul_mat_get_wsize',0,b'\x00\x03\xE3\x23ggml_cl_transform_tensor',0,b'\x00\x01\x1B\x23ggml_clamp',0,b'\x00\x00\xFA\x23ggml_cont',0,b'\x00\x00\xFA\x23ggml_cont_inplace',0,b'\x00\x01\x90\x23ggml_conv_1d',0,b'\x00\x01\x89\x23ggml_conv_1d_ph',0,b'\x00\x01\x98\x23ggml_conv_2d',0,b'\x00\x00\x90\x23ggml_cpu_has_arm_fma',0,b'\x00\x00\x90\x23ggml_cpu_has_avx',0,b'\x00\x00\x90\x23ggml_cpu_has_avx2',0,b'\x00\x00\x90\x23ggml_cpu_has_avx512',0,b'\x00\x00\x90\x23ggml_cpu_has_avx512_vbmi',0,b'\x00\x00\x90\x23ggml_cpu_has_avx512_vnni',0,b'\x00\x00\x90\x23ggml_cpu_has_blas',0,b'\x00\x00\x90\x23ggml_cpu_has_clblast',0,b'\x00\x00\x90\x23ggml_cpu_has_cublas',0,b'\x00\x00\x90\x23ggml_cpu_has_f16c',0,b'\x00\x00\x90\x23ggml_cpu_has_fma',0,b'\x00\x00\x90\x23ggml_cpu_has_fp16_va',0,b'\x00\x00\x90\x23ggml_cpu_has_gpublas',0,b'\x00\x00\x90\x23ggml_cpu_has_neon',0,b'\x00\x00\x90\x23ggml_cpu_has_sse3',0,b'\x00\x00\x90\x23ggml_cpu_has_vsx',0,b'\x00\x00\x90\x23ggml_cpu_has_wasm_simd',0,b'\x00\x01\x84\x23ggml_cpy',0,b'\x00\x01\x84\x23ggml_cpy_inplace',0,b'\x00\x01\x84\x23ggml_cross_entropy_loss',0,b'\x00\x01\xA3\x23ggml_cross_entropy_loss_back',0,b'\x00\x03\x41\x23ggml_cuda_assign_buffers',0,b'\x00\x03\x41\x23ggml_cuda_assign_buffers_force_inplace',0,b'\x00\x03\x41\x23ggml_cuda_assign_buffers_no_scratch',0,b'\x00\x00\x1B\x23ggml_cuda_can_mul_mat',0,b'\x00\x00\x06\x23ggml_cuda_compute_forward',0,b'\x00\x03\x41\x23ggml_cuda_free_data',0,b'\x00\x03\xEC\x23ggml_cuda_free_scratch',0,b'\x00\x00\x90\x23ggml_cuda_get_device_count',0,b'\x00\x02\xCE\x23ggml_cuda_get_device_description',0,b'\x00\x03\xE0\x23ggml_cuda_host_free',0,b'\x00\x02\x72\x23ggml_cuda_host_malloc',0,b'\x00\x02\xCB\x23ggml_cuda_set_main_device',0,b'\x00\x02\x79\x23ggml_cuda_set_mul_mat_q',0,b'\x00\x03\xD8\x23ggml_cuda_set_scratch_size',0,b'\x00\x02\xA0\x23ggml_cuda_set_tensor_split',0,b'\x00\x03\xE3\x23ggml_cuda_transform_tensor',0,b'\x00\x00\x95\x23ggml_cycles',0,b'\x00\x00\x95\x23ggml_cycles_per_ms',0,b'\x00\x00\xFA\x23ggml_diag',0,b'\x00\x01\x21\x23ggml_diag_mask_inf',0,b'\x00\x01\x21\x23ggml_diag_mask_inf_inplace',0,b'\x00\x01\x21\x23ggml_diag_mask_zero',0,b'\x00\x01\x21\x23ggml_diag_mask_zero_inplace',0,b'\x00\x01\x84\x23ggml_div',0,b'\x00\x01\x84\x23ggml_div_inplace',0,b'\x00\x00\xFA\x23ggml_dup',0,b'\x00\x00\xFA\x23ggml_dup_inplace',0,b'\x00\x02\x0B\x23ggml_dup_tensor',0,b'\x00\x02\x4D\x23ggml_element_size',0,b'\x00\x00\xFA\x23ggml_elu',0,b'\x00\x00\xFA\x23ggml_elu_inplace',0,b'\x00\x01\xA9\x23ggml_flash_attn',0,b'\x00\x01\xB0\x23ggml_flash_attn_back',0,b'\x00\x01\xB8\x23ggml_flash_ff',0,b'\x00\x02\x16\x23ggml_format_name',0,b'\x00\x00\x6B\x23ggml_fp16_to_fp32',0,b'\x00\x03\xDB\x23ggml_fp16_to_fp32_row',0,b'\x00\x02\x62\x23ggml_fp32_to_fp16',0,b'\x00\x02\xC1\x23ggml_fp32_to_fp16_row',0,b'\x00\x03\x03\x23ggml_free',0,b'\x00\x00\x53\x23ggml_ftype_to_ggml_type',0,b'\x00\x00\xFA\x23ggml_gelu',0,b'\x00\x00\xFA\x23ggml_gelu_inplace',0,b'\x00\x00\xFA\x23ggml_gelu_quick',0,b'\x00\x00\xFA\x23ggml_gelu_quick_inplace',0,b'\x00\x02\x6C\x23ggml_get_data',0,b'\x00\x00\x5D\x23ggml_get_data_f32',0,b'\x00\x00\x63\x23ggml_get_f32_1d',0,b'\x00\x00\x81\x23ggml_get_i32_1d',0,b'\x00\x02\x4A\x23ggml_get_max_tensor_size',0,b'\x00\x02\x69\x23ggml_get_mem_buffer',0,b'\x00\x02\x4A\x23ggml_get_mem_size',0,b'\x00\x00\x36\x23ggml_get_name',0,b'\x00\x00\x0A\x23ggml_get_no_alloc',0,b'\x00\x01\x84\x23ggml_get_rows',0,b'\x00\x01\xA3\x23ggml_get_rows_back',0,b'\x00\x00\xCE\x23ggml_get_tensor',0,b'\x00\x00\x56\x23ggml_get_unary_op',0,b'\x00\x00\x77\x23ggml_graph_compute',0,b'\x00\x03\x0A\x23ggml_graph_compute_with_ctx',0,b'\x00\x02\xFE\x23ggml_graph_dump_dot',0,b'\x00\x02\xFA\x23ggml_graph_export',0,b'\x00\x00\xCA\x23ggml_graph_get_tensor',0,b'\x00\x00\xAE\x23ggml_graph_import',0,b'\x00\x02\x60\x23ggml_graph_overhead',0,b'\x00\x00\xBE\x23ggml_graph_plan',0,b'\x00\x02\xF7\x23ggml_graph_print',0,b'\x00\x02\xF0\x23ggml_graph_reset',0,b'\x00\x00\xBB\x23ggml_init',0,b'\x00\x03\xEC\x23ggml_init_cublas',0,b'\x00\x00\x6E\x23ggml_internal_get_type_traits',0,b'\x00\x00\x14\x23ggml_is_contiguous',0,b'\x00\x00\x27\x23ggml_is_numa',0,b'\x00\x00\x14\x23ggml_is_permuted',0,b'\x00\x00\x00\x23ggml_is_quantized',0,b'\x00\x00\x14\x23ggml_is_transposed',0,b'\x00\x00\xFA\x23ggml_log',0,b'\x00\x00\xFA\x23ggml_log_inplace',0,b'\x00\x01\xE6\x23ggml_map_binary_f32',0,b'\x00\x01\xE6\x23ggml_map_binary_inplace_f32',0,b'\x00\x02\x04\x23ggml_map_custom1',0,b'\x00\x01\xFF\x23ggml_map_custom1_f32',0,b'\x00\x02\x04\x23ggml_map_custom1_inplace',0,b'\x00\x01\xFF\x23ggml_map_custom1_inplace_f32',0,b'\x00\x01\xF2\x23ggml_map_custom2',0,b'\x00\x01\xEC\x23ggml_map_custom2_f32',0,b'\x00\x01\xF2\x23ggml_map_custom2_inplace',0,b'\x00\x01\xEC\x23ggml_map_custom2_inplace_f32',0,b'\x00\x01\xC7\x23ggml_map_custom3',0,b'\x00\x01\xC0\x23ggml_map_custom3_f32',0,b'\x00\x01\xC7\x23ggml_map_custom3_inplace',0,b'\x00\x01\xC0\x23ggml_map_custom3_inplace_f32',0,b'\x00\x01\xFA\x23ggml_map_unary_f32',0,b'\x00\x01\xFA\x23ggml_map_unary_inplace_f32',0,b'\x00\x00\xFA\x23ggml_mean',0,b'\x00\x00\x0D\x23ggml_metal_add_buffer',0,b'\x00\x03\x1C\x23ggml_metal_free',0,b'\x00\x00\x71\x23ggml_metal_get_concur_list',0,b'\x00\x03\x2C\x23ggml_metal_get_tensor',0,b'\x00\x03\x23\x23ggml_metal_graph_compute',0,b'\x00\x03\x27\x23ggml_metal_graph_find_concurrency',0,b'\x00\x03\xE0\x23ggml_metal_host_free',0,b'\x00\x02\x72\x23ggml_metal_host_malloc',0,b'\x00\x00\x7B\x23ggml_metal_if_optimized',0,b'\x00\x00\xC2\x23ggml_metal_init',0,b'\x00\x03\x1F\x23ggml_metal_set_n_cb',0,b'\x00\x03\x2C\x23ggml_metal_set_tensor',0,b'\x00\x03\xEC\x23ggml_mpi_backend_free',0,b'\x00\x03\xEC\x23ggml_mpi_backend_init',0,b'\x00\x03\x33\x23ggml_mpi_eval_init',0,b'\x00\x03\x30\x23ggml_mpi_free',0,b'\x00\x03\x39\x23ggml_mpi_graph_compute_post',0,b'\x00\x03\x39\x23ggml_mpi_graph_compute_pre',0,b'\x00\x00\xC5\x23ggml_mpi_init',0,b'\x00\x00\x7E\x23ggml_mpi_rank',0,b'\x00\x01\x84\x23ggml_mul',0,b'\x00\x01\x84\x23ggml_mul_inplace',0,b'\x00\x01\x84\x23ggml_mul_mat',0,b'\x00\x02\x4D\x23ggml_nbytes',0,b'\x00\x02\x4D\x23ggml_nbytes_pad',0,b'\x00\x02\x50\x23ggml_nbytes_split',0,b'\x00\x00\xFA\x23ggml_neg',0,b'\x00\x00\xFA\x23ggml_neg_inplace',0,b'\x00\x00\x92\x23ggml_nelements',0,b'\x00\x00\xF2\x23ggml_new_f32',0,b'\x00\x00\xA7\x23ggml_new_graph',0,b'\x00\x00\xF6\x23ggml_new_i32',0,b'\x00\x00\xD2\x23ggml_new_tensor',0,b'\x00\x00\xD8\x23ggml_new_tensor_1d',0,b'\x00\x00\xDD\x23ggml_new_tensor_2d',0,b'\x00\x00\xE3\x23ggml_new_tensor_3d',0,b'\x00\x00\xEA\x23ggml_new_tensor_4d',0,b'\x00\x00\xFA\x23ggml_norm',0,b'\x00\x00\xFA\x23ggml_norm_inplace',0,b'\x00\x00\x92\x23ggml_nrows',0,b'\x00\x03\xEC\x23ggml_numa_init',0,b'\x00\x00\x2D\x23ggml_op_name',0,b'\x00\x00\x2D\x23ggml_op_symbol',0,b'\x00\x00\x4E\x23ggml_opt',0,b'\x00\x00\xC7\x23ggml_opt_default_params',0,b'\x00\x03\x0F\x23ggml_opt_init',0,b'\x00\x00\x42\x23ggml_opt_resume',0,b'\x00\x00\x47\x23ggml_opt_resume_g',0,b'\x00\x01\x84\x23ggml_out_prod',0,b'\x00\x01\x34\x23ggml_permute',0,b'\x00\x00\xFE\x23ggml_pool_1d',0,b'\x00\x01\x06\x23ggml_pool_2d',0,b'\x00\x03\x3E\x23ggml_print_object',0,b'\x00\x03\x19\x23ggml_print_objects',0,b'\x00\x02\x33\x23ggml_quantize_chunk',0,b'\x00\x02\x3B\x23ggml_quantize_q2_K',0,b'\x00\x02\x3B\x23ggml_quantize_q3_K',0,b'\x00\x02\x3B\x23ggml_quantize_q4_0',0,b'\x00\x02\x3B\x23ggml_quantize_q4_1',0,b'\x00\x02\x3B\x23ggml_quantize_q4_K',0,b'\x00\x02\x3B\x23ggml_quantize_q5_0',0,b'\x00\x02\x3B\x23ggml_quantize_q5_1',0,b'\x00\x02\x3B\x23ggml_quantize_q5_K',0,b'\x00\x02\x3B\x23ggml_quantize_q6_K',0,b'\x00\x02\x3B\x23ggml_quantize_q8_0',0,b'\x00\x00\xFA\x23ggml_relu',0,b'\x00\x00\xFA\x23ggml_relu_inplace',0,b'\x00\x01\x84\x23ggml_repeat',0,b'\x00\x01\x84\x23ggml_repeat_back',0,b'\x00\x01\x84\x23ggml_reshape',0,b'\x00\x01\x46\x23ggml_reshape_1d',0,b'\x00\x01\x4B\x23ggml_reshape_2d',0,b'\x00\x01\x51\x23ggml_reshape_3d',0,b'\x00\x01\x58\x23ggml_reshape_4d',0,b'\x00\x01\x16\x23ggml_rms_norm',0,b'\x00\x01\x84\x23ggml_rms_norm_back',0,b'\x00\x01\x16\x23ggml_rms_norm_inplace',0,b'\x00\x01\x34\x23ggml_rope',0,b'\x00\x01\x34\x23ggml_rope_back',0,b'\x00\x01\x3C\x23ggml_rope_custom',0,b'\x00\x01\x3C\x23ggml_rope_custom_inplace',0,b'\x00\x01\x34\x23ggml_rope_inplace',0,b'\x00\x01\x84\x23ggml_scale',0,b'\x00\x01\x84\x23ggml_scale_inplace',0,b'\x00\x01\xDD\x23ggml_set',0,b'\x00\x01\xD0\x23ggml_set_1d',0,b'\x00\x01\xD0\x23ggml_set_1d_inplace',0,b'\x00\x01\xD6\x23ggml_set_2d',0,b'\x00\x01\xD6\x23ggml_set_2d_inplace',0,b'\x00\x02\x1A\x23ggml_set_f32',0,b'\x00\x03\x6E\x23ggml_set_f32_1d',0,b'\x00\x02\x1E\x23ggml_set_i32',0,b'\x00\x03\x73\x23ggml_set_i32_1d',0,b'\x00\x01\xDD\x23ggml_set_inplace',0,b'\x00\x02\x12\x23ggml_set_name',0,b'\x00\x03\x06\x23ggml_set_no_alloc',0,b'\x00\x03\x15\x23ggml_set_param',0,b'\x00\x02\x46\x23ggml_set_scratch',0,b'\x00\x02\x0F\x23ggml_set_zero',0,b'\x00\x00\xFA\x23ggml_sgn',0,b'\x00\x00\xFA\x23ggml_sgn_inplace',0,b'\x00\x00\xFA\x23ggml_silu',0,b'\x00\x01\x84\x23ggml_silu_back',0,b'\x00\x00\xFA\x23ggml_silu_inplace',0,b'\x00\x00\xFA\x23ggml_soft_max',0,b'\x00\x01\x84\x23ggml_soft_max_back',0,b'\x00\x01\x84\x23ggml_soft_max_back_inplace',0,b'\x00\x00\xFA\x23ggml_soft_max_inplace',0,b'\x00\x00\xFA\x23ggml_sqr',0,b'\x00\x00\xFA\x23ggml_sqr_inplace',0,b'\x00\x00\xFA\x23ggml_sqrt',0,b'\x00\x00\xFA\x23ggml_sqrt_inplace',0,b'\x00\x00\xFA\x23ggml_step',0,b'\x00\x00\xFA\x23ggml_step_inplace',0,b'\x00\x01\x84\x23ggml_sub',0,b'\x00\x01\x84\x23ggml_sub_inplace',0,b'\x00\x00\xFA\x23ggml_sum',0,b'\x00\x00\xFA\x23ggml_sum_rows',0,b'\x00\x00\xFA\x23ggml_tanh',0,b'\x00\x00\xFA\x23ggml_tanh_inplace',0,b'\x00\x02\x60\x23ggml_tensor_overhead',0,b'\x00\x03\xEC\x23ggml_time_init',0,b'\x00\x00\x95\x23ggml_time_ms',0,b'\x00\x00\x95\x23ggml_time_us',0,b'\x00\x00\xFA\x23ggml_transpose',0,b'\x00\x00\x30\x23ggml_type_name',0,b'\x00\x02\x30\x23ggml_type_size',0,b'\x00\x00\x60\x23ggml_type_sizef',0,b'\x00\x01\x11\x23ggml_unary',0,b'\x00\x01\x11\x23ggml_unary_inplace',0,b'\x00\x02\x4A\x23ggml_used_mem',0,b'\x00\x02\xDE\x23ggml_vec_dot_q2_K_q8_K',0,b'\x00\x02\xDE\x23ggml_vec_dot_q3_K_q8_K',0,b'\x00\x02\xDE\x23ggml_vec_dot_q4_K_q8_K',0,b'\x00\x02\xDE\x23ggml_vec_dot_q5_K_q8_K',0,b'\x00\x02\xDE\x23ggml_vec_dot_q6_K_q8_K',0,b'\x00\x01\x7E\x23ggml_view_1d',0,b'\x00\x01\x76\x23ggml_view_2d',0,b'\x00\x01\x6C\x23ggml_view_3d',0,b'\x00\x01\x60\x23ggml_view_4d',0,b'\x00\x02\x0B\x23ggml_view_tensor',0,b'\x00\x01\x21\x23ggml_win_part',0,b'\x00\x01\x2D\x23ggml_win_unpart',0,b'\x00\x03\xCC\x23gguf_add_tensor',0,b'\x00\x00\x88\x23gguf_find_key',0,b'\x00\x00\x88\x23gguf_find_tensor',0,b'\x00\x03\x84\x23gguf_free',0,b'\x00\x02\x59\x23gguf_get_alignment',0,b'\x00\x02\x75\x23gguf_get_arr_data',0,b'\x00\x00\x8C\x23gguf_get_arr_n',0,b'\x00\x00\x3D\x23gguf_get_arr_str',0,b'\x00\x00\x59\x23gguf_get_arr_type',0,b'\x00\x02\x6F\x23gguf_get_data',0,b'\x00\x02\x59\x23gguf_get_data_offset',0,b'\x00\x00\x39\x23gguf_get_key',0,b'\x00\x00\x59\x23gguf_get_kv_type',0,b'\x00\x03\xD4\x23gguf_get_meta_data',0,b'\x00\x02\x59\x23gguf_get_meta_size',0,b'\x00\x00\x85\x23gguf_get_n_kv',0,b'\x00\x00\x85\x23gguf_get_n_tensors',0,b'\x00\x00\x29\x23gguf_get_tensor_name',0,b'\x00\x02\x5C\x23gguf_get_tensor_offset',0,b'\x00\x00\x20\x23gguf_get_val_bool',0,b'\x00\x00\x67\x23gguf_get_val_f32',0,b'\x00\x00\x97\x23gguf_get_val_i16',0,b'\x00\x00\x8C\x23gguf_get_val_i32',0,b'\x00\x00\x9B\x23gguf_get_val_i8',0,b'\x00\x00\x39\x23gguf_get_val_str',0,b'\x00\x02\x65\x23gguf_get_val_u16',0,b'\x00\x02\x2C\x23gguf_get_val_u32',0,b'\x00\x02\x28\x23gguf_get_val_u8',0,b'\x00\x00\x85\x23gguf_get_version',0,b'\x00\x02\x26\x23gguf_init_empty',0,b'\x00\x02\x22\x23gguf_init_from_file',0,b'\x00\x03\x9C\x23gguf_set_arr_data',0,b'\x00\x03\x8C\x23gguf_set_arr_str',0,b'\x00\x03\xD0\x23gguf_set_kv',0,b'\x00\x03\xC6\x23gguf_set_tensor_data',0,b'\x00\x03\x97\x23gguf_set_tensor_type',0,b'\x00\x03\x87\x23gguf_set_val_bool',0,b'\x00\x03\xA3\x23gguf_set_val_f32',0,b'\x00\x03\xAD\x23gguf_set_val_i16',0,b'\x00\x03\xA8\x23gguf_set_val_i32',0,b'\x00\x03\xB2\x23gguf_set_val_i8',0,b'\x00\x03\x92\x23gguf_set_val_str',0,b'\x00\x03\xC1\x23gguf_set_val_u16',0,b'\x00\x03\xBC\x23gguf_set_val_u32',0,b'\x00\x03\xB7\x23gguf_set_val_u8',0,b'\x00\x00\x33\x23gguf_type_name',0,b'\x00\x03\x87\x23gguf_write_to_file',0,b'\x00\x02\xC6\x23quantize_row_q2_K',0,b'\x00\x02\xA3\x23quantize_row_q2_K_reference',0,b'\x00\x02\xC6\x23quantize_row_q3_K',0,b'\x00\x02\xA8\x23quantize_row_q3_K_reference',0,b'\x00\x02\xC6\x23quantize_row_q4_K',0,b'\x00\x02\xAD\x23quantize_row_q4_K_reference',0,b'\x00\x02\xC6\x23quantize_row_q5_K',0,b'\x00\x02\xB2\x23quantize_row_q5_K_reference',0,b'\x00\x02\xC6\x23quantize_row_q6_K',0,b'\x00\x02\xB7\x23quantize_row_q6_K_reference',0,b'\x00\x02\xC6\x23quantize_row_q8_K',0,b'\x00\x02\xBC\x23quantize_row_q8_K_reference',0),
    _struct_unions = ((b'\x00\x00\x04\x27\x00\x00\x00\x02$1',b'\x00\x00\x22\x11n_iter',b'\x00\x00\xF4\x11sched',b'\x00\x00\xF4\x11decay',b'\x00\x00\xF4\x11alpha',b'\x00\x00\xF4\x11beta1',b'\x00\x00\xF4\x11beta2',b'\x00\x00\xF4\x11eps',b'\x00\x00\xF4\x11eps_f',b'\x00\x00\xF4\x11eps_g'),(b'\x00\x00\x04\x28\x00\x00\x00\x02$2',b'\x00\x00\x22\x11m',b'\x00\x00\x22\x11n_iter',b'\x00\x00\x22\x11max_linesearch',b'\x00\x00\xF4\x11eps',b'\x00\x00\xF4\x11ftol',b'\x00\x00\xF4\x11wolfe',b'\x00\x00\xF4\x11min_step',b'\x00\x00\xF4\x11max_step',b'\x00\x04\x14\x11linesearch'),(b'\x00\x00\x04\x29\x00\x00\x00\x02$3',b'\x00\x00\x08\x11x',b'\x00\x00\x08\x11g1',b'\x00\x00\x08\x11g2',b'\x00\x00\x08\x11m',b'\x00\x00\x08\x11v',b'\x00\x00\x08\x11mh',b'\x00\x00\x08\x11vh',b'\x00\x00\x08\x11pf',b'\x00\x00\xF4\x11fx_best',b'\x00\x00\xF4\x11fx_prev',b'\x00\x00\x22\x11n_no_improvement'),(b'\x00\x00\x04\x2A\x00\x00\x00\x02$4',b'\x00\x00\x08\x11x',b'\x00\x00\x08\x11xp',b'\x00\x00\x08\x11g',b'\x00\x00\x08\x11gp',b'\x00\x00\x08\x11d',b'\x00\x00\x08\x11pf',b'\x00\x00\x08\x11lmal',b'\x00\x00\x08\x11lmys',b'\x00\x00\x08\x11lms',b'\x00\x00\x08\x11lmy',b'\x00\x00\xF4\x11fx_best',b'\x00\x00\xF4\x11step',b'\x00\x00\x22\x11j',b'\x00\x00\x22\x11k',b'\x00\x00\x22\x11end',b'\x00\x00\x22\x11n_no_improvement'),(b'\x00\x00\x03\xF7\x00\x00\x00\x03$__mbstate_t',b'\x00\x03\xFF\x11__mbstate8',b'\x00\x00\xDB\x11_mbstateL'),(b'\x00\x00\x03\xF8\x00\x00\x00\x02$block_q2_K',b'\x00\x04\x44\x11scales',b'\x00\x04\x48\x11qs',b'\x00\x00\x6C\x11d',b'\x00\x00\x6C\x11dmin'),(b'\x00\x00\x03\xF9\x00\x00\x00\x02$block_q3_K',b'\x00\x04\x46\x11hmask',b'\x00\x04\x48\x11qs',b'\x00\x04\x42\x11scales',b'\x00\x00\x6C\x11d'),(b'\x00\x00\x03\xFA\x00\x00\x00\x02$block_q4_K',b'\x00\x00\x6C\x11d',b'\x00\x00\x6C\x11dmin',b'\x00\x04\x42\x11scales',b'\x00\x04\x40\x11qs'),(b'\x00\x00\x03\xFB\x00\x00\x00\x02$block_q5_K',b'\x00\x00\x6C\x11d',b'\x00\x00\x6C\x11dmin',b'\x00\x04\x42\x11scales',b'\x00\x04\x46\x11qh',b'\x00\x04\x40\x11qs'),(b'\x00\x00\x03\xFC\x00\x00\x00\x02$block_q6_K',b'\x00\x04\x40\x11ql',b'\x00\x04\x48\x11qh',b'\x00\x04\x23\x11scales',b'\x00\x00\x6C\x11d'),(b'\x00\x00\x03\xFD\x00\x00\x00\x02$block_q8_K',b'\x00\x00\xF4\x11d',b'\x00\x04\x25\x11qs',b'\x00\x04\x21\x11bsums'),(b'\x00\x00\x04\x18\x00\x00\x00\x02$ggml_type_traits_t',b'\x00\x00\x0F\x11type_name',b'\x00\x00\x22\x11blck_size',b'\x00\x00\x11\x11type_size',b'\x00\x00\xB6\x11is_quantized',b'\x00\x04\x52\x11to_float',b'\x00\x04\x4F\x11from_float',b'\x00\x04\x4F\x11from_float_reference',b'\x00\x04\x50\x11vec_dot',b'\x00\x00\x01\x11vec_dot_type'),(b'\x00\x00\x04\x2C\x00\x00\x00\x02__darwin_pthread_handler_rec',b'\x00\x04\x51\x11__routine',b'\x00\x00\x10\x11__arg',b'\x00\x04\x2B\x11__next'),(b'\x00\x00\x03\xEF\x00\x00\x00\x02_opaque_pthread_attr_t',b'\x00\x04\x20\x11__sig',b'\x00\x04\x0B\x11__opaque'),(b'\x00\x00\x03\xF0\x00\x00\x00\x02_opaque_pthread_cond_t',b'\x00\x04\x20\x11__sig',b'\x00\x04\x07\x11__opaque'),(b'\x00\x00\x03\xF1\x00\x00\x00\x02_opaque_pthread_condattr_t',b'\x00\x04\x20\x11__sig',b'\x00\x04\x11\x11__opaque'),(b'\x00\x00\x03\xF2\x00\x00\x00\x02_opaque_pthread_mutex_t',b'\x00\x04\x20\x11__sig',b'\x00\x04\x0B\x11__opaque'),(b'\x00\x00\x03\xF3\x00\x00\x00\x02_opaque_pthread_mutexattr_t',b'\x00\x04\x20\x11__sig',b'\x00\x04\x11\x11__opaque'),(b'\x00\x00\x03\xF4\x00\x00\x00\x02_opaque_pthread_once_t',b'\x00\x04\x20\x11__sig',b'\x00\x04\x11\x11__opaque'),(b'\x00\x00\x03\xF5\x00\x00\x00\x02_opaque_pthread_rwlock_t',b'\x00\x04\x20\x11__sig',b'\x00\x04\x03\x11__opaque'),(b'\x00\x00\x03\xF6\x00\x00\x00\x02_opaque_pthread_rwlockattr_t',b'\x00\x04\x20\x11__sig',b'\x00\x04\x01\x11__opaque'),(b'\x00\x00\x04\x2E\x00\x00\x00\x02_opaque_pthread_t',b'\x00\x04\x20\x11__sig',b'\x00\x04\x2B\x11__cleanup_stack',b'\x00\x04\x0F\x11__opaque'),(b'\x00\x00\x04\x2F\x00\x00\x00\x10ggml_allocr',),(b'\x00\x00\x04\x30\x00\x00\x00\x02ggml_cgraph',b'\x00\x00\x22\x11n_nodes',b'\x00\x00\x22\x11n_leafs',b'\x00\x04\x39\x11nodes',b'\x00\x04\x39\x11grads',b'\x00\x04\x39\x11leafs',b'\x00\x04\x4D\x11visited_hash_table',b'\x00\x00\x22\x11perf_runs',b'\x00\x00\xDB\x11perf_cycles',b'\x00\x00\xDB\x11perf_time_us'),(b'\x00\x00\x04\x31\x00\x00\x00\x02ggml_compute_params',b'\x00\x04\x17\x11type',b'\x00\x00\x22\x11ith',b'\x00\x00\x22\x11nth',b'\x00\x00\x11\x11wsize',b'\x00\x00\x10\x11wdata'),(b'\x00\x00\x04\x32\x00\x00\x00\x10ggml_context',),(b'\x00\x00\x04\x33\x00\x00\x00\x02ggml_cplan',b'\x00\x00\x11\x11work_size',b'\x00\x04\x3F\x11work_data',b'\x00\x00\x22\x11n_threads',b'\x00\x04\x19\x11n_tasks',b'\x00\x03\xEE\x11abort_callback',b'\x00\x00\x10\x11abort_callback_data'),(b'\x00\x00\x00\xBC\x00\x00\x00\x02ggml_init_params',b'\x00\x00\x11\x11mem_size',b'\x00\x00\x10\x11mem_buffer',b'\x00\x00\xB6\x11no_alloc'),(b'\x00\x00\x04\x34\x00\x00\x00\x10ggml_metal_context',),(b'\x00\x00\x04\x35\x00\x00\x00\x10ggml_mpi_context',),(b'\x00\x00\x04\x37\x00\x00\x00\x02ggml_object',b'\x00\x00\x11\x11offs',b'\x00\x00\x11\x11size',b'\x00\x04\x36\x11next',b'\x00\x04\x15\x11type',b'\x00\x04\x09\x11padding'),(b'\x00\x00\x04\x38\x00\x00\x00\x02ggml_opt_context',b'\x00\x00\x0B\x11ctx',b'\x00\x00\x50\x11params',b'\x00\x00\x22\x11iter',b'\x00\x00\xDB\x11nx',b'\x00\x00\xB6\x11just_initialized',b'\x00\x04\x29\x11adam',b'\x00\x04\x2A\x11lbfgs'),(b'\x00\x00\x00\x50\x00\x00\x00\x02ggml_opt_params',b'\x00\x00\xC8\x11type',b'\x00\x00\x22\x11n_threads',b'\x00\x00\x22\x11past',b'\x00\x00\xF4\x11delta',b'\x00\x00\x22\x11max_no_improvement',b'\x00\x00\xB6\x11print_forward_graph',b'\x00\x00\xB6\x11print_backward_graph',b'\x00\x04\x27\x11adam',b'\x00\x04\x28\x11lbfgs'),(b'\x00\x00\x02\x48\x00\x00\x00\x02ggml_scratch',b'\x00\x00\x11\x11offs',b'\x00\x00\x11\x11size',b'\x00\x00\x10\x11data'),(b'\x00\x00\x04\x3D\x00\x00\x00\x02ggml_tensor',b'\x00\x00\x01\x11type',b'\x00\x04\x13\x11backend',b'\x00\x00\x22\x11n_dims',b'\x00\x04\x1E\x11ne',b'\x00\x04\x4B\x11nb',b'\x00\x00\x2E\x11op',b'\x00\x04\x1B\x11op_params',b'\x00\x00\xB6\x11is_param',b'\x00\x00\x08\x11grad',b'\x00\x04\x3B\x11src',b'\x00\x00\x22\x11perf_runs',b'\x00\x00\xDB\x11perf_cycles',b'\x00\x00\xDB\x11perf_time_us',b'\x00\x00\x10\x11data',b'\x00\x04\x0D\x11name',b'\x00\x00\x10\x11extra',b'\x00\x04\x09\x11padding'),(b'\x00\x00\x04\x3E\x00\x00\x00\x10gguf_context',),(b'\x00\x00\x02\x24\x00\x00\x00\x02gguf_init_params',b'\x00\x00\xB6\x11no_alloc',b'\x00\x00\xB0\x11ctx')),
    _enums = (b'\x00\x00\x04\x13\x00\x00\x00\x16ggml_backend\x00GGML_BACKEND_CPU,GGML_BACKEND_GPU,GGML_BACKEND_GPU_SPLIT',b'\x00\x00\x00\x54\x00\x00\x00\x15ggml_ftype\x00GGML_FTYPE_UNKNOWN,GGML_FTYPE_ALL_F32,GGML_FTYPE_MOSTLY_F16,GGML_FTYPE_MOSTLY_Q4_0,GGML_FTYPE_MOSTLY_Q4_1,GGML_FTYPE_MOSTLY_Q4_1_SOME_F16,GGML_FTYPE_MOSTLY_Q8_0,GGML_FTYPE_MOSTLY_Q5_0,GGML_FTYPE_MOSTLY_Q5_1,GGML_FTYPE_MOSTLY_Q2_K,GGML_FTYPE_MOSTLY_Q3_K,GGML_FTYPE_MOSTLY_Q4_K,GGML_FTYPE_MOSTLY_Q5_K,GGML_FTYPE_MOSTLY_Q6_K',b'\x00\x00\x04\x14\x00\x00\x00\x16ggml_linesearch\x00GGML_LINESEARCH_DEFAULT,GGML_LINESEARCH_BACKTRACKING_ARMIJO,GGML_LINESEARCH_BACKTRACKING_WOLFE,GGML_LINESEARCH_BACKTRACKING_STRONG_WOLFE',b'\x00\x00\x04\x15\x00\x00\x00\x16ggml_object_type\x00GGML_OBJECT_TENSOR,GGML_OBJECT_GRAPH,GGML_OBJECT_WORK_BUFFER',b'\x00\x00\x00\x2E\x00\x00\x00\x16ggml_op\x00GGML_OP_NONE,GGML_OP_DUP,GGML_OP_ADD,GGML_OP_ADD1,GGML_OP_ACC,GGML_OP_SUB,GGML_OP_MUL,GGML_OP_DIV,GGML_OP_SQR,GGML_OP_SQRT,GGML_OP_LOG,GGML_OP_SUM,GGML_OP_SUM_ROWS,GGML_OP_MEAN,GGML_OP_ARGMAX,GGML_OP_REPEAT,GGML_OP_REPEAT_BACK,GGML_OP_SILU_BACK,GGML_OP_NORM,GGML_OP_RMS_NORM,GGML_OP_RMS_NORM_BACK,GGML_OP_MUL_MAT,GGML_OP_OUT_PROD,GGML_OP_SCALE,GGML_OP_SET,GGML_OP_CPY,GGML_OP_CONT,GGML_OP_RESHAPE,GGML_OP_VIEW,GGML_OP_PERMUTE,GGML_OP_TRANSPOSE,GGML_OP_GET_ROWS,GGML_OP_GET_ROWS_BACK,GGML_OP_DIAG,GGML_OP_DIAG_MASK_INF,GGML_OP_DIAG_MASK_ZERO,GGML_OP_SOFT_MAX,GGML_OP_SOFT_MAX_BACK,GGML_OP_ROPE,GGML_OP_ROPE_BACK,GGML_OP_ALIBI,GGML_OP_CLAMP,GGML_OP_CONV_1D,GGML_OP_CONV_2D,GGML_OP_POOL_1D,GGML_OP_POOL_2D,GGML_OP_FLASH_ATTN,GGML_OP_FLASH_FF,GGML_OP_FLASH_ATTN_BACK,GGML_OP_WIN_PART,GGML_OP_WIN_UNPART,GGML_OP_UNARY,GGML_OP_MAP_UNARY,GGML_OP_MAP_BINARY,GGML_OP_MAP_CUSTOM1_F32,GGML_OP_MAP_CUSTOM2_F32,GGML_OP_MAP_CUSTOM3_F32,GGML_OP_MAP_CUSTOM1,GGML_OP_MAP_CUSTOM2,GGML_OP_MAP_CUSTOM3,GGML_OP_CROSS_ENTROPY_LOSS,GGML_OP_CROSS_ENTROPY_LOSS_BACK,GGML_OP_COUNT',b'\x00\x00\x01\x01\x00\x00\x00\x16ggml_op_pool\x00GGML_OP_POOL_MAX,GGML_OP_POOL_AVG,GGML_OP_POOL_COUNT',b'\x00\x00\x04\x16\x00\x00\x00\x15ggml_opt_result\x00GGML_OPT_OK,GGML_OPT_DID_NOT_CONVERGE,GGML_OPT_NO_CONTEXT,GGML_OPT_INVALID_WOLFE,GGML_OPT_FAIL,GGML_LINESEARCH_FAIL,GGML_LINESEARCH_MINIMUM_STEP,GGML_LINESEARCH_MAXIMUM_STEP,GGML_LINESEARCH_MAXIMUM_ITERATIONS,GGML_LINESEARCH_INVALID_PARAMETERS',b'\x00\x00\x00\xC8\x00\x00\x00\x16ggml_opt_type\x00GGML_OPT_ADAM,GGML_OPT_LBFGS',b'\x00\x00\x04\x17\x00\x00\x00\x16ggml_task_type\x00GGML_TASK_INIT,GGML_TASK_COMPUTE,GGML_TASK_FINALIZE',b'\x00\x00\x00\x01\x00\x00\x00\x16ggml_type\x00GGML_TYPE_F32,GGML_TYPE_F16,GGML_TYPE_Q4_0,GGML_TYPE_Q4_1,GGML_TYPE_Q5_0,GGML_TYPE_Q5_1,GGML_TYPE_Q8_0,GGML_TYPE_Q8_1,GGML_TYPE_Q2_K,GGML_TYPE_Q3_K,GGML_TYPE_Q4_K,GGML_TYPE_Q5_K,GGML_TYPE_Q6_K,GGML_TYPE_Q8_K,GGML_TYPE_I8,GGML_TYPE_I16,GGML_TYPE_I32,GGML_TYPE_COUNT',b'\x00\x00\x01\x14\x00\x00\x00\x16ggml_unary_op\x00GGML_UNARY_OP_ABS,GGML_UNARY_OP_SGN,GGML_UNARY_OP_NEG,GGML_UNARY_OP_STEP,GGML_UNARY_OP_TANH,GGML_UNARY_OP_ELU,GGML_UNARY_OP_RELU,GGML_UNARY_OP_GELU,GGML_UNARY_OP_GELU_QUICK,GGML_UNARY_OP_SILU',b'\x00\x00\x00\x34\x00\x00\x00\x16gguf_type\x00GGUF_TYPE_UINT8,GGUF_TYPE_INT8,GGUF_TYPE_UINT16,GGUF_TYPE_INT16,GGUF_TYPE_UINT32,GGUF_TYPE_INT32,GGUF_TYPE_FLOAT32,GGUF_TYPE_BOOL,GGUF_TYPE_STRING,GGUF_TYPE_ARRAY,GGUF_TYPE_COUNT'),
    _typenames = (b'\x00\x00\x00\xDB__darwin_blkcnt_t',b'\x00\x00\x00\x22__darwin_blksize_t',b'\x00\x00\x00\x11__darwin_clock_t',b'\x00\x00\x00\x22__darwin_ct_rune_t',b'\x00\x00\x00\x22__darwin_dev_t',b'\x00\x00\x03\xBF__darwin_fsblkcnt_t',b'\x00\x00\x03\xBF__darwin_fsfilcnt_t',b'\x00\x00\x03\xBF__darwin_gid_t',b'\x00\x00\x03\xBF__darwin_id_t',b'\x00\x00\x04\x4A__darwin_ino64_t',b'\x00\x00\x04\x4A__darwin_ino_t',b'\x00\x00\x04\x20__darwin_intptr_t',b'\x00\x00\x03\xBF__darwin_mach_port_name_t',b'\x00\x00\x03\xBF__darwin_mach_port_t',b'\x00\x00\x03\xF7__darwin_mbstate_t',b'\x00\x00\x00\x6C__darwin_mode_t',b'\x00\x00\x03\xBF__darwin_natural_t',b'\x00\x00\x00\xDB__darwin_off_t',b'\x00\x00\x00\x22__darwin_pid_t',b'\x00\x00\x03\xEF__darwin_pthread_attr_t',b'\x00\x00\x03\xF0__darwin_pthread_cond_t',b'\x00\x00\x03\xF1__darwin_pthread_condattr_t',b'\x00\x00\x00\x11__darwin_pthread_key_t',b'\x00\x00\x03\xF2__darwin_pthread_mutex_t',b'\x00\x00\x03\xF3__darwin_pthread_mutexattr_t',b'\x00\x00\x03\xF4__darwin_pthread_once_t',b'\x00\x00\x03\xF5__darwin_pthread_rwlock_t',b'\x00\x00\x03\xF6__darwin_pthread_rwlockattr_t',b'\x00\x00\x04\x2D__darwin_pthread_t',b'\x00\x00\x04\x20__darwin_ptrdiff_t',b'\x00\x00\x00\x22__darwin_rune_t',b'\x00\x00\x03\xBF__darwin_sigset_t',b'\x00\x00\x00\x11__darwin_size_t',b'\x00\x00\x03\xBF__darwin_socklen_t',b'\x00\x00\x04\x20__darwin_ssize_t',b'\x00\x00\x00\x22__darwin_suseconds_t',b'\x00\x00\x04\x20__darwin_time_t',b'\x00\x00\x03\xBF__darwin_uid_t',b'\x00\x00\x03\xBF__darwin_useconds_t',b'\x00\x00\x04\x05__darwin_uuid_string_t',b'\x00\x00\x04\x44__darwin_uuid_t',b'\x00\x00\x00\x22__darwin_wchar_t',b'\x00\x00\x00\x22__darwin_wint_t',b'\x00\x00\x03\xB0__int16_t',b'\x00\x00\x00\x22__int32_t',b'\x00\x00\x00\xDB__int64_t',b'\x00\x00\x03\xB5__int8_t',b'\x00\x00\x03\xF7__mbstate_t',b'\x00\x00\x00\x6C__uint16_t',b'\x00\x00\x03\xBF__uint32_t',b'\x00\x00\x04\x4A__uint64_t',b'\x00\x00\x03\xBA__uint8_t',b'\x00\x00\x03\xF8block_q2_K',b'\x00\x00\x03\xF9block_q3_K',b'\x00\x00\x03\xFAblock_q4_K',b'\x00\x00\x03\xFBblock_q5_K',b'\x00\x00\x03\xFCblock_q6_K',b'\x00\x00\x03\xFDblock_q8_K',b'\x00\x00\x01\xEAggml_binary_op_f32_t',b'\x00\x00\x02\x02ggml_custom1_op_f32_t',b'\x00\x00\x02\x07ggml_custom1_op_t',b'\x00\x00\x01\xF0ggml_custom2_op_f32_t',b'\x00\x00\x01\xF6ggml_custom2_op_t',b'\x00\x00\x01\xC5ggml_custom3_op_f32_t',b'\x00\x00\x01\xCCggml_custom3_op_t',b'\x00\x00\x00\x6Cggml_fp16_t',b'\x00\x00\x04\x4Fggml_from_float_t',b'\x00\x00\x04\x52ggml_to_float_t',b'\x00\x00\x04\x18ggml_type_traits_t',b'\x00\x00\x01\xFDggml_unary_op_f32_t',b'\x00\x00\x04\x50ggml_vec_dot_t',b'\x00\x00\x03\xB0int16_t',b'\x00\x00\x00\x22int32_t',b'\x00\x00\x00\xDBint64_t',b'\x00\x00\x03\xB5int8_t',b'\x00\x00\x03\xB0int_fast16_t',b'\x00\x00\x00\x22int_fast32_t',b'\x00\x00\x00\xDBint_fast64_t',b'\x00\x00\x03\xB5int_fast8_t',b'\x00\x00\x03\xB0int_least16_t',b'\x00\x00\x00\x22int_least32_t',b'\x00\x00\x00\xDBint_least64_t',b'\x00\x00\x03\xB5int_least8_t',b'\x00\x00\x04\x20intmax_t',b'\x00\x00\x04\x20intptr_t',b'\x00\x00\x04\x1Dmax_align_t',b'\x00\x00\x04\x20ptrdiff_t',b'\x00\x00\x00\xDBregister_t',b'\x00\x00\x00\x11rsize_t',b'\x00\x00\x00\x11size_t',b'\x00\x00\x04\x4Asyscall_arg_t',b'\x00\x00\x00\x6Cu_int16_t',b'\x00\x00\x03\xBFu_int32_t',b'\x00\x00\x04\x4Au_int64_t',b'\x00\x00\x03\xBAu_int8_t',b'\x00\x00\x00\x6Cuint16_t',b'\x00\x00\x03\xBFuint32_t',b'\x00\x00\x04\x4Auint64_t',b'\x00\x00\x03\xBAuint8_t',b'\x00\x00\x00\x6Cuint_fast16_t',b'\x00\x00\x03\xBFuint_fast32_t',b'\x00\x00\x04\x4Auint_fast64_t',b'\x00\x00\x03\xBAuint_fast8_t',b'\x00\x00\x00\x6Cuint_least16_t',b'\x00\x00\x03\xBFuint_least32_t',b'\x00\x00\x04\x4Auint_least64_t',b'\x00\x00\x03\xBAuint_least8_t',b'\x00\x00\x00\x11uintmax_t',b'\x00\x00\x00\x11uintptr_t',b'\x00\x00\x04\x4Auser_addr_t',b'\x00\x00\x00\xDBuser_long_t',b'\x00\x00\x00\xDBuser_off_t',b'\x00\x00\x04\x4Auser_size_t',b'\x00\x00\x00\xDBuser_ssize_t',b'\x00\x00\x00\xDBuser_time_t',b'\x00\x00\x04\x4Auser_ulong_t',b'\x00\x00\x00\x22wchar_t'),
)

```

# `examples/python/ggml/utils.py`



这段代码定义了一个名为“init”的函数，用于初始化一个ggml(一个流行的GNU库，用于处理GeotIFF格式的数据)的上下文。

该函数接受四个参数：

- “mem_size”是一个整数，用于指定要分配的内存大小。这个参数告诉ggml要分配多少内存，以便能够存储数据。
- “mem_buffer”是一个ffi.CData类型，用于指定要分配的内存缓冲区。这个参数告诉ggml如何将数据存储到内存中。如果这个参数为NULL，那么ggml将自动分配内存。
- “no_alloc”是一个布尔值，表示当内存分配失败时，ggml将不会抛出任何错误。这个参数告诉ggml不要在初始化失败时崩溃。

函数的实现基本上是：

1. 创建一个“struct ggml_init_params*”类型的变量params，并将其初始化为mem_size、mem_buffer和no_alloc的值。
2. 使用lib.ggml_init函数初始化ggml上下文，并将lib.ggml_free函数作为回调函数，以便在分配内存失败时回滚到之前的状态。
3. 返回ggml_init函数的返回值，以便ggml上下文的初始化完成。

该函数的作用是初始化ggml上下文并返回，这样就可以在后续的数据处理过程中使用它。


```cpp
"""
  Common helpers for working with ggml + numpy
"""
from ggml import ffi, lib
from typing import Union, Optional
import numpy as np

def init(mem_size: int, mem_buffer: ffi.CData = ffi.NULL, no_alloc: bool = False) -> ffi.CData:
    """
      Initialize a ggml context, which will be freed automatically when the pointer is garbage collected.
    """
    params = ffi.new('struct ggml_init_params*')
    params.mem_size = mem_size
    params.mem_buffer = mem_buffer
    params.no_alloc = no_alloc
    return ffi.gc(lib.ggml_init(params[0]), lib.ggml_free)
 
```

这段代码定义了一个名为TensorLike的元组类型，其元素可以是任意numpy.ndarray或ffi.CData类型。

定义了一个名为copy的函数，用于从源张量（一个numpy.ndarray或ffi.CData）中复制到目标张量（另一个numpy.ndarray或ffi.CData）中。

该函数使用了两个参数，from_tensor和to_tensor，它们必须是同一种类型（如两个numpy.ndarray或两个ffi.CData），并且必须具有相同的形状。允许使用allow_requantize参数，如果两个张量都被量化，则不会抛出异常。

在该函数内部，首先检查两个输入张量的形状是否相同，然后检查两个输入张量的类型是否相同。如果两个张量具有相同的形状和类型，则从张量中复制数据，并确保对齐。如果两个张量具有不同的形状和类型，则在使用allow_requantize参数时会抛出异常。否则，函数会将两个张量的数据复制到目标张量中，并允许对两个张量进行量化。


```cpp
TensorLike = Union[ffi.CData, np.ndarray]

def copy(from_tensor: TensorLike, to_tensor: TensorLike, allow_requantize: bool = True):
    """
      Copy the contents of one tensor to another, doing any necessary (de/re)quantization transparently.
      Works across numpy & ggml tensors, but they must have the same shape (and be contiguous).

      Parameters
      ----------
      from_tensor : TensorLike
          The tensor to copy from (a numpy array or possibly-quantized ggml tensor)
      to_tensor : TensorLike
          The tensor to copy to (a numpy array or possibly-quantized ggml tensor)
      allow_requantize : bool
          If False, will throw an error if requantization is required (i.e. both from_tensor
          and to_tensor are quantized with different quantization types)
    """
    if id(from_tensor) == id(to_tensor):
        return
 
    __expect_same_layout("source", from_tensor, "destination", to_tensor)
    __check_shape_consistent_with_type(from_tensor)
    __check_shape_consistent_with_type(to_tensor)

    from_type = __get_type(from_tensor)
    to_type = __get_type(to_tensor)

    if from_type == to_type:
        ffi.memmove(__get_data(to_tensor), __get_data(from_tensor), __get_nbytes(from_tensor))
    else:
        assert allow_requantize or not lib.ggml_is_quantized(from_type) or not lib.ggml_is_quantized(to_type), \
            f"Requantizing from {__type_name(from_type)} to {__type_name(to_type)} is disabled. Force with allow_requantize=True"
 
        __set_floats(to_tensor, __get_floats(from_tensor))

```

This function is a higher-level version of converting a tensor object to a numpy array. It has several options to control the conversion process, including the allow_copy and allow_requantize options.

The tensor object should be in the form of a binary file dataf (such as an HDF5 or Goo file) or a regular tensor object. The function reads the data from the tensor object, checks the quantization status of the tensor, and converts the tensor object to a numpy array if the tensor is quantized. If the tensor is multi-dimensional, the function will reshape the tensor data to fit the accepted shape of the numpy array.

The allow_copy option specifies whether the tensor should be converted to a copy or not. If it is set to True, the function will make a copy of the tensor data. If it is set to False, and the tensor is multi-dimensional, the function will return the data in a new numpy array, allowing the user to modify the data without affecting the original tensor.

The allow_requantize option specifies whether the tensor should be quantized or not. If it is set to True, the function will attempt to convert the tensor to a numpy array. If it is not set to True, or the tensor is multi-dimensional, the function will raise an error.

Note that this function uses the `ffi` and `numpy` modules to interact with the OSRM file system and perform the necessary transformations to the data.


```cpp
def numpy(tensor: ffi.CData, allow_copy: Union[bool, np.ndarray] = False, allow_requantize=False) -> np.ndarray:
    """
      Convert a ggml tensor to a numpy array.
      If the tensor isn't quantized, the returned numpy array will be a view over its data.
 
      If it is quantized (and allow_copy is True), the copy will involve dequantization and the returned array will
      be a copy of the original tensor (any changes to the numpy array won't then be reflected back to the tensor).

      Parameters
      ----------
      tensor : ffi.CData
          The tensor to convert to a numpy array
      allow_copy : bool or np.ndarray
          If False, will throw an error if the tensor is quantized (since dequantization requires extra memory).
          If True, will dequantize the tensor and return a copy of the data in a new float32 numpy array.
          If an np.ndarray, will copy the data into the given array (which must be the same shape as the tensor) when dequantization is needed
      allow_requantize : bool
          If allow_copy is a tensor with a different quantization type than the source tensor, will throw an error unless allow_requantize is True.
    """
    shape = __get_shape(tensor)

    if lib.ggml_is_quantized(tensor.type):
        if allow_copy == False:
            raise ValueError(f"{__describe(tensor)} is quantized, conversion to numpy requires a copy (pass allow_copy=True; changes to the numpy array won't affect the original).")
        elif isinstance(allow_copy, np.ndarray):
            __expect_same_layout("source tensor", tensor, "dequantization output tensor", allow_copy)
            destination = allow_copy
        else:
            destination = np.empty(shape, dtype=np.float32)

        copy(tensor, destination, allow_requantize=allow_requantize)
        return destination
    else:
        dtype = __type_to_dtype(tensor.type)
        if not dtype:
            raise NotImplementedError(f'Cannot convert {__describe(tensor)} to numpy')

        assert __is_contiguous(tensor), f"Cannot convert {__describe(tensor)} to numpy (support contiguous tensors only)"
        nbytes = lib.ggml_nelements(tensor) * lib.ggml_type_size(tensor.type)
        array = np.frombuffer(ffi.buffer(lib.ggml_get_data(tensor), nbytes), dtype=dtype)
        array.shape = shape
        return array

```

这段Python代码定义了一个函数`__type_name(int) -> str`，该函数接收一个整数类型的参数`type`，并返回一个字符类型的函数值。该函数的作用是将`type`类型转换为字符串类型，并返回该类型的名称。

该函数的实现主要依赖于两个已定义的函数`lib.ggml_type_name(int)`和`ffi.string(str)`，以及一个未定义的函数类型参数。

`lib.ggml_type_name(int)`函数的作用是将`int`类型转换为相应的类型名称，并返回该名称。这个函数的实现依赖于`lib.ggml_type_name()`函数，这个函数应该是一个通用的类型名称函数，接受一个`int`类型的参数，并返回该类型的名称。不过，由于该函数没有定义具体的实现，因此无法确定它的类型名称。

`ffi.string(str)`函数的作用是将一个`str`类型的参数转换为相应的字符串类型，并返回该类型的函数值。这个函数的实现依赖于`ffi.string()`函数，这个函数应该是一个通用的字符串类型函数，接受一个`str`类型的参数，并返回该类型的函数值。

该段代码的主要目的是定义一个类型名称函数`__type_name(int)`，该函数将接收一个整数类型的参数，并返回该类型的名称。该函数的实现依赖于`lib.ggml_type_name(int)`和`ffi.string(str)`函数，以及一个未定义的函数类型参数。


```cpp
def __type_name(type: int) -> str:
    name = lib.ggml_type_name(type)
    return ffi.string(name).decode('utf-8') if name else None

__k_quant_types = set([
  lib.GGML_TYPE_Q2_K,
  lib.GGML_TYPE_Q3_K,
  lib.GGML_TYPE_Q4_K,
  lib.GGML_TYPE_Q5_K,
  lib.GGML_TYPE_Q6_K,
  lib.GGML_TYPE_Q8_K,
])

__type_to_dtype_dict = {
  lib.GGML_TYPE_I8: np.int8,
  lib.GGML_TYPE_I16: np.int16,
  lib.GGML_TYPE_I32: np.int32,
  lib.GGML_TYPE_F16: np.float16,
  lib.GGML_TYPE_F32: np.float32,
}

```

这段代码定义了几个函数，用于将不同的数据类型转换为对应的numpy数据类型。

第一个函数是一个类型函数，接收一个整数类型的参数，返回一个numpy数据类型对象。如果输入的参数是一个ndarray, 函数将根据该ndarray的类型返回相应的numpy数据类型。

第二个函数是一个dtype函数，接收一个numpy数据类型参数，返回相应的数据类型名称。如果输入的参数是一个ndarray, 函数将返回ndpy类型对应的名称。

第三个函数是一个描述函数，接收一个numpy数据类型参数，返回一个字符串，表示该输入的numpy数据类型的描述。

第四个函数是一个获取函数，接收一个numpy数据类型参数，返回一个numpy数据类型对象。如果输入的参数是一个ndarray, 函数将根据该ndarray的类型返回相应的numpy数据类型。

第五个函数是一个获取函数，接收一个numpy数据类型参数和一个strides参数，返回一个numpy数据类型对象。如果输入的参数是一个ndarray, 函数将根据该ndarray的strides参数返回相应的numpy数据类型。

第六个函数是一个获取函数，接收一个numpy数据类型参数，返回一个numpy数据类型对象。如果输入的参数是一个ndarray, 函数将根据该ndarray的strides参数返回相应的numpy数据类型。

第七个函数是一个获取函数，接收一个numpy数据类型参数，返回一个numpy数据类型对象。如果输入的参数是一个ndarray, 函数将根据该ndarray的strides参数返回相应的numpy数据类型。

第八个函数是一个获取函数，接收一个numpy数据类型参数，返回一个numpy数据类型对象。如果输入的参数是一个ndarray, 函数将根据该ndarray的strides参数返回相应的numpy数据类型。


```cpp
def __type_to_dtype(type: int) -> Optional[np.dtype]: return __type_to_dtype_dict.get(type)
def __dtype_to_type(dtype: np.dtype):
    if dtype == np.float32: return lib.GGML_TYPE_F32
    elif dtype == np.float16: return lib.GGML_TYPE_F16
    elif dtype == np.int32: return lib.GGML_TYPE_I32
    elif dtype == np.int16: return lib.GGML_TYPE_I16
    elif dtype == np.int8: return lib.GGML_TYPE_I8
    else: raise ValueError(f"Unsupported dtype: {dtype}")

def __describe(tensor: ffi.CType): return f'Tensor[{__type_name(__get_type(tensor))}, {__get_shape(tensor)}]'
def __get_type(tensor: TensorLike): return __dtype_to_type(tensor.dtype) if isinstance(tensor, np.ndarray) else tensor.type
def __get_shape(x: TensorLike): return x.shape if isinstance(x, np.ndarray) else tuple([x.ne[i] for i in range(x.n_dims)])
def __get_strides(x: TensorLike): return x.strides if isinstance(x, np.ndarray) else tuple([x.nb[i] for i in range(x.n_dims)])
def __get_data(x: TensorLike) -> ffi.CData: return ffi.from_buffer(x) if isinstance(x, np.ndarray) else lib.ggml_get_data(x)
def __get_nbytes(tensor: TensorLike): return tensor.nbytes if isinstance(tensor, np.ndarray) else lib.ggml_nbytes(tensor)
```

这段代码定义了两个函数，分别用于获取一个张量（tensor）中的元素类型和是否为连续张量。

第一个函数 `__get_nelements` 接收一个任意张量（tensor）并返回其元素数量（size）。如果输入张量是 NumPy 数组，函数将直接返回其元素数量。否则，它将调用一个名为 `lib.ggml_nelements` 的函数来获取元素数量。注意，这个函数是在 lib.ggml_ne elements 的基础上进行封装的，因此需要安装 lib.ggml 库。

第二个函数 `__is_contiguous` 接收一个任意张量（tensor）并返回一个布尔值，表示张量是否是连续的。如果输入张量是 NumPy 数组，函数将直接返回 False。否则，它将调用一个名为 `lib.ggml_is_contiguous` 的函数来获取张量是否是连续的。同样，这个函数也需要安装 lib.ggml 库。

`__get_floats` 函数接收一个任意张量（tensor），并返回一个 ffi.CData 类型的数据。它的实现方式是遍历输入张量中的数据，并获取其类型（如果已经是 lib.GGML_TYPE_F32，则返回浮点数类型；否则，根据输入类型类型选择相应的函数进行获取数据）。

总的来说，这段代码定义了两个函数，用于获取张量中的元素类型和连续性，以及从张量中获取浮点数数据。


```cpp
def __get_nelements(tensor: TensorLike): return tensor.size if isinstance(tensor, np.ndarray) else lib.ggml_nelements(tensor)
def __is_contiguous(tensor: TensorLike): return tensor.flags['C_CONTIGUOUS'] if isinstance(tensor, np.ndarray) else lib.ggml_is_contiguous(tensor)

def __get_floats(tensor: TensorLike) -> ffi.CData:
    data, type = __get_data(tensor), __get_type(tensor)
    if type == lib.GGML_TYPE_F32:
        return ffi.cast('float*', data)
    else:
      nelements = __get_nelements(tensor)
      floats = ffi.new('float[]', nelements)
      if type == lib.GGML_TYPE_F16:
          lib.ggml_fp16_to_fp32_row(ffi.cast('uint16_t*', data), floats, nelements)
      elif lib.ggml_is_quantized(type):
          qtype = lib.ggml_internal_get_type_traits(type)
          assert qtype.to_float, f"Type {__type_name(type)} is not supported by ggml"
          qtype.to_float(data, floats, nelements)
      else:
          raise NotImplementedError(f'Cannot read floats from {__describe(tensor)}')
      return floats

```

这段代码定义了一个名为 `__set_floats` 的函数，它接受一个名为 `tensor` 的Tensor-like对象和一个名为 `ffi.CData` 的ffi数据类型指针作为参数。

该函数首先通过调用另一个函数 `__get_data` 和 `__get_type` 来获取Tensor对象的相关信息，比如数据类型、字节数以及元素数量等。然后，它根据数据类型来检查是否需要将数据存储为float类型的数据。如果是，函数会使用ffi库中的 `ffi.memmove` 函数来复制`ffi.CData` 提供的数据到Tensor对象中。否则，如果数据类型不是`lib.GGML_TYPE_F32`，函数会尝试将数据转换为`lib.GGML_TYPE_F16` 类型，并使用`lib.ggml_fp32_to_fp16_row`函数将其转换为16位整数类型的数据。如果数据类型不支持写入float数组，函数会引发 `NotImplementedError` 异常。

总结起来，该函数的主要作用是读取一个Tensor对象的数据，并根据数据类型将其存储为float类型的数据，或者将其转换为另一种数据类型。


```cpp
def __set_floats(tensor: TensorLike, f32_data: ffi.CData) -> None:
    data, type, nbytes = __get_data(tensor), __get_type(tensor), __get_nbytes(tensor)
    if type == lib.GGML_TYPE_F32:
        ffi.memmove(data, f32_data, nbytes)
    else:
      nelements = __get_nelements(tensor)
      if type == lib.GGML_TYPE_F16:
          lib.ggml_fp32_to_fp16_row(f32_data, ffi.cast('uint16_t*', data), nelements)
      elif lib.ggml_is_quantized(type):
          qtype = lib.ggml_internal_get_type_traits(type)
          assert qtype.from_float, f"Type {__type_name(type)} is not supported by ggml"
          qtype.from_float(f32_data, data, nelements)
      else:
          raise NotImplementedError(f'Cannot write floats to {__describe(tensor)}')

```

这段代码定义了一个名为 `__expect_same_layout` 的函数，它的输入参数 `name1` 和 `name2` 分别表示两个输入张量的名称，而输入张量 `tensor1` 和 `tensor2` 则是对应的输入数据。

函数的作用是检查两个输入张量 `tensor1` 和 `tensor2` 的形状是否相同，如果形状相同，则返回 `True`，否则返回 `False`。判断依据是检查两个张量是否为 contiguous，即张量可以连续分配。同时，函数还判断两个张量是否可以被量化，如果张量可以被量化，则函数会判断本地库是否支持使用 KEG-毫米法进行量化。

具体来说，函数首先获取输入张量 `tensor1` 和 `tensor2` 的形状，然后判断两个张量的形状是否相同，如果形状相同，则执行后续步骤，否则输出错误信息。如果形状相同，但本地库不支持使用 KEG-毫米法进行量化，则函数返回 `False`。如果形状不同，则判断两个张量是否为 contiguous，即可以连续分配。然后，函数会遍历两个张量，判断每个张量的最后一个维度是否可以被 16 整除，如果不是，则输出错误信息。如果两个张量都可以被量化，则函数会尝试使用 KEG-毫米法进行量化，并判断本地库是否支持使用 KEG-毫米法进行量化。如果本地库不支持使用 KEG-毫米法进行量化，则函数返回 `False`。


```cpp
def __expect_same_layout(name1: str, tensor1: TensorLike, name2: str, tensor2: TensorLike):
    shape1, shape2 = __get_shape(tensor1), __get_shape(tensor2)
    assert shape1 == shape2, f"Shape mismatch: {name1} has {shape1} but {name2} has {shape2}"
    assert __is_contiguous(tensor1) and __is_contiguous(tensor2), f"Only contiguous tensors are supported (got {name1} with strides {__get_strides(tensor1)} and {name2} with strides {__get_strides(tensor2)})"

def __check_shape_consistent_with_type(tensor: TensorLike):
    type = __get_type(tensor)
    if not lib.ggml_is_quantized(type):
        return
    shape = __get_shape(tensor)

    block_size = lib.ggml_blck_size(type)
    assert not (block_size == 0 and type in __k_quant_types), f"Can't quantize, native library was not compiled with USE_K_QUANTS!"
    assert block_size > 0, f"Invalid block size {block_size} for type {__type_name(type)}"
    for i, d in enumerate(shape):
        assert d % block_size == 0, f"Dimension {i} of {__describe(tensor)} is not divisible by {block_size}, required for quantization."

```