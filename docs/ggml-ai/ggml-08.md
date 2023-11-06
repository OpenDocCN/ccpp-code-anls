# GGML源码解析 8

# `examples/gpt-2/quantize.cpp`

这段代码是一个C++程序，它包括了多个头文件和一些全局变量。现在我来逐步解释一下它的作用。

首先，它包含了两个头文件：common.h和common-ggml.h。其中，common.h可能包含了程序中一些通用的函数和数据结构，common-ggml.h可能包含了某些特定的函数和数据结构，这两个文件与整个程序的运行是密切相关的。

然后，它引入了一些标准库头文件：cassert、cmath、cstdio、cstring。这些头文件中包含了一些常用的工具函数，如断言、数学函数、输入输出、字符串处理等等，对程序的运行应该有很大的帮助。

接着，它引入了一个文件：stdin。这个文件可能是程序的输入文件，也可能是其他程序的外部输入。

最后，它引入了一个名为ggml的第三方库。根据这个库的名称，我们可以猜测它可能是一个数据结构或算法库，对程序的运行可能有很大的帮助。

综上所述，这段代码的作用是定义了一个程序，它可能是一个数据结构或算法库的开发工具，对程序的运行可能有很大的帮助。


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

This function reads a pre-trained neural network model file and quantize the model to a file. The input parameters include the file name of the model, the data type of the input, and the options for quantization.

The function reads the model file and generates a mapping of token IDs to scales, and also generates a mapping of scales to token IDs. The generated maps are stored in the `vocab.token_to_id` and `vocab.id_to_token` maps.

The function then reads the input data and generates a vector of to-be-quantized tensor names based on the options provided. If the quantization process fails, an error message is printed.

The function returns true if the model was successfully quantized and written to the output file.


```cpp
// default hparams (GPT-2 117M)
struct gpt2_hparams {
    int32_t n_vocab = 50257;
    int32_t n_ctx   = 1024;
    int32_t n_embd  = 768;
    int32_t n_head  = 12;
    int32_t n_layer = 12;
    int32_t ftype   = 1;
};

// quantize a model
bool gpt2_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
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

    gpt2_hparams hparams;

    // load hparams
    {
        finp.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        finp.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        finp.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        finp.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        finp.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        finp.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr_src =    hparams.ftype / GGML_QNT_VERSION_FACTOR;
        const int32_t ftype_dst = GGML_QNT_VERSION * GGML_QNT_VERSION_FACTOR + ftype;

        printf("%s: n_vocab     = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx       = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd      = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head      = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer     = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype (src) = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr (src) = %d\n", __func__, qntvr_src);
        printf("%s: ftype (dst) = %d\n", __func__, ftype_dst);
        printf("%s: qntvr (dst) = %d\n", __func__, GGML_QNT_VERSION);

        fout.write((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fout.write((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fout.write((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fout.write((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fout.write((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fout.write((char *) &ftype_dst,       sizeof(ftype_dst));
    }

    // load vocab
    {
        int32_t n_vocab = 0;
        finp.read ((char *) &n_vocab, sizeof(n_vocab));
        fout.write((char *) &n_vocab, sizeof(n_vocab));

        if (n_vocab != hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname_inp.c_str(), n_vocab, hparams.n_vocab);
            return false;
        }

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
        "model/wte",
        "model/lm_head",
        "model/h.*/attn/c_attn/w",
        "model/h.*/attn/c_proj/w",
        "model/h.*/mlp/c_fc/w",
        "model/h.*/mlp/c_proj/w",
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

This is a C++ program that appears to quantize a neural network model represented by the F16 float32 data type. The program takes two arguments: the input and output model files, and the type of the quantization to perform (either "model-f32" or "model-quant").

The program first checks that the input file is valid and then initializes the ggml library to configure the input model file. It then reads the quantization commands from the second argument and performs the quantization by running the "gpt2\_model\_quantize" function on the input model file.

Finally, the program reports the timing information for the quantization process and prints the total time taken to complete the task.


```cpp
// usage:
//  ./gpt-2-quantize models/gpt-2-117M/ggml-model.bin models/gpt-2-117M/ggml-model-quant.bin type
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

        if (!gpt2_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
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

# gpt-2

This is a C++ example running GPT-2 inference using the [ggml](https://github.com/ggerganov/ggml) library.

The program runs on the CPU - no video card is required.

The [Cerebras-GPT](https://huggingface.co/cerebras) models are also supported.

The example supports the following GPT-2 models:

| Model | Description  | Disk Size |
| ---   | ---          | ---       |
| 117M  | Small model  | 240 MB    |
| 345M  | Medium model | 680 MB    |
| 774M  | Large model  | 1.5 GB    |
| 1558M | XL model     | 3.0 GB    |

Sample performance on MacBook M1 Pro:

| Model | Size  | Time / Token |
| ---   | ---   | ---    |
| GPT-2 |  117M |   5 ms |
| GPT-2 |  345M |  12 ms |
| GPT-2 |  774M |  23 ms |
| GPT-2 | 1558M |  42 ms |

*TODO: add tables for Cerebras-GPT models*

Sample output:

```cpp
$ ./bin/gpt-2 -h
usage: ./bin/gpt-2 [options]

options:
  -h, --help            show this help message and exit
  -s SEED, --seed SEED  RNG seed (default: -1)
  -t N, --threads N     number of threads to use during computation (default: 8)
  -p PROMPT, --prompt PROMPT
                        prompt to start generation with (default: random)
  -n N, --n_predict N   number of tokens to predict (default: 200)
  --top_k N             top-k sampling (default: 40)
  --top_p N             top-p sampling (default: 0.9)
  --temp N              temperature (default: 1.0)
  -b N, --batch_size N  batch size for prompt processing (default: 8)
  -m FNAME, --model FNAME
                        model path (default: models/gpt-2-117M/ggml-model.bin)

$ ./bin/gpt-2
gpt2_model_load: loading model from 'models/gpt-2-117M/ggml-model.bin'
gpt2_model_load: n_vocab = 50257
gpt2_model_load: n_ctx   = 1024
gpt2_model_load: n_embd  = 768
gpt2_model_load: n_head  = 12
gpt2_model_load: n_layer = 12
gpt2_model_load: f16     = 1
gpt2_model_load: ggml ctx size = 311.12 MB
gpt2_model_load: memory size =    72.00 MB, n_mem = 12288
gpt2_model_load: model size  =   239.08 MB
main: number of tokens in prompt = 1

So this is going to be the end of the line for us.

If the Dolphins continue to do their business, it's possible that the team could make a bid to bring in new defensive coordinator Scott Linehan.

Linehan's job is a little daunting, but he's a great coach and an excellent coach. I don't believe we're going to make the playoffs.

We're going to have to work hard to keep our heads down and get ready to go.<|endoftext|>

main: mem per token =  2048612 bytes
main:     load time =   106.32 ms
main:   sample time =     7.10 ms
main:  predict time =   506.40 ms / 5.06 ms per token
main:    total time =   629.84 ms
```

## Downloading and converting the original models (GPT-2)

You can download the original model files using the [download-model.sh](download-model.sh) Bash script. The models are
in Tensorflow format, so in order to use them with ggml, you need to convert them to appropriate format. This is done
via the [convert-ckpt-to-ggml.py](convert-ckpt-to-ggml.py) python script.

Here is the entire process for the GPT-2 117M model (download from official site + conversion):

```cpp
cd ggml/build
../examples/gpt-2/download-model.sh 117M

Downloading model 117M ...
models/gpt-2-117M/checkpoint                      100%[=============================>]      77  --.-KB/s    in 0s
models/gpt-2-117M/encoder.json                    100%[=============================>]   1018K  1.20MB/s    in 0.8s
models/gpt-2-117M/hparams.json                    100%[=============================>]      90  --.-KB/s    in 0s
models/gpt-2-117M/model.ckpt.data-00000-of-00001  100%[=============================>] 474.70M  1.21MB/s    in 8m 39s
models/gpt-2-117M/model.ckpt.index                100%[=============================>]   5.09K  --.-KB/s    in 0s
models/gpt-2-117M/model.ckpt.meta                 100%[=============================>] 460.11K   806KB/s    in 0.6s
models/gpt-2-117M/vocab.bpe                       100%[=============================>] 445.62K   799KB/s    in 0.6s
Done! Model '117M' saved in 'models/gpt-2-117M/'

Run the convert-ckpt-to-ggml.py script to convert the model to ggml format.

  python /Users/john/ggml/examples/gpt-2/convert-ckpt-to-ggml.py models/gpt-2-117M/ 1

```

This conversion requires that you have python and Tensorflow installed on your computer. Still, if you want to avoid
this, you can download the already converted ggml models as described below.

## Downloading and converting the original models (Cerebras-GPT)

Clone the respective repository from here: https://huggingface.co/cerebras

Use the [convert-cerebras-to-ggml.py](convert-cerebras-to-ggml.py) script to convert the model to `ggml` format:

```cpp
cd ggml/build
git clone https://huggingface.co/cerebras/Cerebras-GPT-111M models/
python ../examples/gpt-2/convert-cerebras-to-ggml.py models/Cerebras-GPT-111M/

```

## Downloading the ggml model directly (GPT-2)

For convenience, I will be hosting the converted ggml model files in order to make it easier to run the examples. This
way, you can directly download a single binary file and start using it. No python or Tensorflow is required.

Here is how to get the 117M ggml model:

```cpp
cd ggml/build
../examples/gpt-2/download-ggml-model.sh 117M

Downloading ggml model 117M ...
models/gpt-2-117M/ggml-model.bin         100%[===============================>] 239.58M  8.52MB/s    in 28s
Done! Model '117M' saved in 'models/gpt-2-117M/ggml-model.bin'
You can now use it like this:

  $ ./bin/gpt-2 -m models/gpt-2-117M/ggml-model.bin -p "This is an example"

```

At some point, I might decide to stop hosting these models. So in that case, simply revert to the manual process above.

## Quantizing the models

You can also try to quantize the `ggml` models via 4-bit integer quantization.
Keep in mind that for smaller models, this will render them completely useless.
You generally want to quantize larger models.

```cpp
# quantize GPT-2 F16 to Q4_0 (faster but less precise)
./bin/gpt-2-quantize models/gpt-2-1558M/ggml-model-f16.bin models/gpt-2-1558M/ggml-model-q4_0.bin 2
./bin/gpt-2 -m models/gpt-2-1558M/ggml-model-q4_0.bin -p "This is an example"

# quantize Cerebras F16 to Q4_1 (slower but more precise)
./bin/gpt-2-quantize models/Cerebras-GPT-6.7B/ggml-model-f16.bin models/Cerebras-GPT-6.7B/ggml-model-q4_1.bin 3
./bin/gpt-2 -m models/Cerebras-GPT-6.7B/ggml-model-q4_1.bin -p "This is an example"

```

## Batched generation example

You can try the batched generation from a given prompt using the gpt-2-batched binary.

Sample output:

```cpp
$ gpt-2-batched -np 5 -m models/gpt-2-117M/ggml-model.bin -p "Hello my name is" -n 50

main: seed = 1697037431
gpt2_model_load: loading model from 'models/gpt-2-117M/ggml-model.bin'
gpt2_model_load: n_vocab = 50257
gpt2_model_load: n_ctx   = 1024
gpt2_model_load: n_embd  = 768
gpt2_model_load: n_head  = 12
gpt2_model_load: n_layer = 12
gpt2_model_load: ftype   = 1
gpt2_model_load: qntvr   = 0
gpt2_model_load: ggml tensor size    = 320 bytes
gpt2_model_load: backend buffer size = 312.72 MB
ggml_init_cublas: found 1 CUDA devices:
  Device 0: NVIDIA GeForce GTX 1660, compute capability 7.5
gpt2_model_load: using CPU backend
gpt2_model_load: memory size =    72.00 MB, n_mem = 12288
gpt2_model_load: model size  =   239.08 MB
extract_tests_from_file : No test file found.
test_gpt_tokenizer : 0 tests failed out of 0 tests.
main: compute buffer size: 3.26 MB


main: generating 5 sequences ...
main: prompt: 'Hello my name is'
main: number of tokens in prompt = 4, first 8 tokens: 15496 616 1438 318


sequence 0:

Hello my name is John. You can call me any way you want, if you want, but for my very first date, I will be on the phone with you. We're both in our early 20s, but I feel like it's all

sequence 1:

Hello my name is Robert, and I want to say that we're proud to have your company here on the world's largest platform for sharing your stories with us. This is a huge opportunity for our community. We have hundreds of people on this team and

sequence 2:

Hello my name is Jack. I'm the one who created you.

Jack is a boy with a big smile and a big heart. He is a handsome guy. He loves the outdoors and loves the people he meets. He wants to be a

sequence 3:

Hello my name is John. I am a Canadian citizen with a large number of family in Quebec and I am interested in studying. My aim is to take up a post in the Journal of the International Academy of Sciences of Canada which I am currently finishing.

sequence 4:

Hello my name is Dan. I am an entrepreneur. I am a great father. I am a great husband. I am a great husband. I am a great dad. And I am a great husband.

I love my life. I love



main:     load time =   880.80 ms
main:   sample time =    91.43 ms
main:  predict time =  2518.29 ms
main:    total time =  3544.32 ms
```


# `examples/gpt-j/convert-h5-to-ggml.py`

这段代码的作用是将一个名为 GPT-J-6B 的 h5 格式的 transformer 模型转换为 ggml 格式的格式。它主要实现了以下几个功能：

1. 加载输入的 GPT-J-6B 模型，并支持使用 GPTJForCausalLM。
2. 遍历所有变量，并将相关信息写入一个 binary 文件中。
3. 对每个变量，先输出其数量维度，然后输出其名称长度、维度（包括所有维度的大小）和数据。
4. 默认情况下，将大矩阵转换为 16 位浮点数。
5. 支持通过添加 "use-f32" CLI 参数来禁用将大矩阵转换为 16 位浮点数。


```cpp
# Convert GPT-J-6B h5 transformer model to ggml format
#
# Load the model using GPTJForCausalLM.
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

这段代码是一个Python脚本，主要用于读取和处理自然语言文本数据。它具体执行以下操作：

1. 在代码开始位置，定义了一些模型参数和词汇表。

2. 从命令行输入中读取文件内容，并将其存储在两个变量（`lines` 和 `read_lines`）中。

3. 将读取到的文本数据进行预处理，包括去除控制字符（如 space, tab, newline 等）和转换为小写。

4. 将文本数据转换为GPT模型可以处理的格式，并将模型和词汇存储在两个变量（`gpt_model` 和 `gpt_vocab`）中。

5. 使用所选的GPT模型对文本数据进行编码，并将结果存储在另一个变量（`encoded_text`）中。

6. 将编码后的文本数据与另一个文本数据进行比较，以确定它们是否匹配。

7. 如果两个文本数据匹配，则使用词汇表（`gpt_vocab`）中的单词检索模型的输出，并将结果打印出来。

8. 如果两个文本数据不匹配，则返回。


```cpp
# At the start of the ggml file we write the model parameters
# and vocabulary.
#

import sys
import struct
import json
import torch
import numpy as np

from transformers import GPTJForCausalLM

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

这段代码是一个Python脚本，它的作用是转换H5格式的文件到GGML格式的文件。它主要通过两个文件进行输入和输出。

具体来说，当运行脚本时，如果传入的参数数量小于3个，脚本会输出一个使用说明，告诉用户如何正确地运行脚本。当传入的参数中包含`use-f32`参数时，脚本会将输入的文件全部转换为float32数据类型；当传入的参数中包含`use-f16`参数时，脚本会将输入的文件全部转换为float16数据类型。

脚本的具体实现主要依赖于两个文件：`vocab.json`和`added_tokens.json`。这两个文件都是从另外的JSON格式的文件中读取出来的，存储了各个单词对应的Freq（频率）、 Tag（标记）信息。脚本使用`json.load()`函数从这两个文件中读取出来，并赋值给对应的变量`encoder`和`encoder_added`。

最终，脚本会将读取到的文件转换为指定的格式，并输出结果。


```cpp
if len(sys.argv) < 3:
    print("Usage: convert-h5-to-ggml.py dir-model [use-f32]\n")
    print("  ftype == 0 -> float32")
    print("  ftype == 1 -> float16")
    sys.exit(1)

# output in the same directory as the model
dir_model = sys.argv[1]
fname_out = sys.argv[1] + "/ggml-model.bin"

with open(dir_model + "/vocab.json", "r", encoding="utf-8") as f:
    encoder = json.load(f)

with open(dir_model + "/added_tokens.json", "r", encoding="utf-8") as f:
    encoder_added = json.load(f)

```

这段代码的作用是读取一个名为 "config.json" 的JSON文件，并将其内容存储在名为 "hparams" 的变量中。

首先，使用Python内置的 "open" 函数打开一个名为 "dir_model/config.json" 的文件，并将其以只读模式打开。

然后，使用Python内置的 "json" 函数从文件中读取JSON内容，并将其存储在 "hparams" 变量中。

接下来，定义了一个名为 "ftype_str" 的列表，其中包含 ftype 可能的数据类型。

然后，根据用户提供的第二个参数 "ftype"，将其转换为整数类型并将其存储在 "ftype" 变量中。

接着，根据 ftype 的值，从 "ftype_str" 中获取相应的文件名 "fname_out"，并将其存储在文件名 "sys.argv[1]" 和 "ftype" 之间。文件类型为 "f32" 或 "f16"，具体取决于用户提供的参数。

最后，没有做任何其他操作，因为该代码已经完成了它的预期任务。


```cpp
with open(dir_model + "/config.json", "r", encoding="utf-8") as f:
    hparams = json.load(f)

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
    fname_out = sys.argv[1] + "/ggml-model-" + ftype_str[ftype] + ".bin"


```

这段代码的作用是训练一个预训练的语言模型，并将其保存到一个文件中。这个语言模型使用门控循环神经网络（GPT）架构，并且对输入的语言数据进行编码时使用的是条件逻辑门（Log-OR）。此外，它还被训练来支持理解文本中的实体、关系和事件，因此它也被称为CausalLM。

首先，我们使用GPTJForCausalLM.from_pretrained这个函数来加载预训练的语言模型。这个函数的第一个参数是模型在哪个目录中存储，第二个参数是是否要在低CPU内存使用的情况下加载模型。

接着，我们获取模型的状态快照，并将其打印出来。这个状态快照包含了模型的一些元数据，例如模型的输入数据大小、词汇表大小、位置编码数量、嵌入层数量、头数和循环的维度。

然后，我们创建一个文件并写入一个特定的数据类型，即"ggml"。这个数据类型是PyTorch中用于存储序列标记的数据类型，其中的第一个元素是0x67676d6c，它是一个二进制数据，包含一个完整的XML格式的文档，包含了模型的全部架构和参数。

接下来，我们使用write函数将模型的状态快照写入到文件中。在这个过程中，我们使用struct.pack函数来将每个变量打包装入二进制数据中。这个函数需要一个参数，它是一个元组，包含了要打包的变量名称和数据类型。

通过调用这个代码，我们可以将预训练的语言模型下载到一个文件中，并使用不同的数据类型来打开和读取这个文件。这个文件可以在PyTorch中后续地进行解析和修改，以适应我们的具体应用场景。


```cpp
model = GPTJForCausalLM.from_pretrained(dir_model, low_cpu_mem_usage=True)
#print (model)

list_vars = model.state_dict()
#print (list_vars)

fout = open(fname_out, "wb")

fout.write(struct.pack("i", 0x67676d6c)) # magic: ggml in hex
fout.write(struct.pack("i", hparams["vocab_size"]))
fout.write(struct.pack("i", hparams["n_positions"]))
fout.write(struct.pack("i", hparams["n_embd"]))
fout.write(struct.pack("i", hparams["n_head"]))
fout.write(struct.pack("i", hparams["n_layer"]))
fout.write(struct.pack("i", hparams["rotary_dim"]))
```

这段代码的主要目的是将一个名为 "encoder" 的结构体中的两个名为 "key" 和 "value" 的字段编码为字节序列，然后将其写入到 fout 文件中。同时，还处理了一个名为 "encoder_added" 的字典，其中包含了一些额外的键值对，这些键值对对应于 "encoder" 中的 "value" 字段。

具体来说，代码首先定义了一个名为 "byte_encoder" 的函数，该函数将一个名为 "byte_encoder" 的字典编码为字节序列，并返回一个名为 "encoder" 的字节序列。接着，定义了一个名为 "byte_decoder" 的函数，该函数将一个名为 "byte_encoder" 的字典中的键值对映射到一个名为 "text" 的字节序列中。

接下来，代码通过 fout.write() 函数将 "encoder" 和 "len(encoder_added)" 的字节序列写入 fout 文件中。然后，代码遍历 "encoder" 和 "encoder_added" 中的每个键，并将它们对应的 "text" 字节序列写入 fout 文件中。注意，在 "encoder_added" 中的键中，如果 "value" 对应的键已经被 "encoder" 中的某个键 "key" 所对应，那么就不需要再进行编码，直接写入 "text" 即可。

最后，代码通过 fout.write() 函数将所有编码后的字节序列写入 fout 文件中。


```cpp
fout.write(struct.pack("i", ftype))

byte_encoder = bytes_to_unicode()
byte_decoder = {v:k for k, v in byte_encoder.items()}

fout.write(struct.pack("i", len(encoder) + len(encoder_added)))

for key in encoder:
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

for key in encoder_added:
    text = bytearray([byte_decoder[c] for c in key])
    fout.write(struct.pack("i", len(text)))
    fout.write(text)

```

It looks like you are describing a Python function that takes in a list of data in the form of numpy arrays


```cpp
for name in list_vars.keys():
    data = list_vars[name].squeeze().numpy()
    print("Processing variable: " + name + " with shape: ", data.shape)

    # we don't need these
    if name.endswith("attn.masked_bias") or name.endswith(".attn.bias"):
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

    # for efficiency - transpose these matrices:
    # (note - with latest ggml this is no longer more efficient, so disabling it)
    #  "transformer.h.*.mlp.fc_in.weight"
    #  "transformer.h.*.attn.out_proj.weight"
    #  "transformer.h.*.attn.q_proj.weight"
    #  "transformer.h.*.attn.k_proj.weight"
    #  "transformer.h.*.attn.v_proj.weight"
    #if name.endswith(".mlp.fc_in.weight")     or \
    #   name.endswith(".attn.out_proj.weight") or \
    #   name.endswith(".attn.q_proj.weight")   or \
    #   name.endswith(".attn.k_proj.weight")   or \
    #   name.endswith(".attn.v_proj.weight"):
    #    print("  Transposing")
    #    data = data.transpose()

    # header
    str = name.encode('utf-8')
    fout.write(struct.pack("iii", n_dims, len(str), ftype_cur))
    for i in range(n_dims):
        fout.write(struct.pack("i", data.shape[n_dims - 1 - i]))
    fout.write(str);

    # data
    data.tofile(fout)

```

这段代码的主要作用是关闭名为 fout 的文件，并输出一个消息，指出文件已成功保存到 fname_out 命名的文件中。

具体来说，第一行使用 fout.close() 方法关闭了文件 fout，这是在程序打开文件时调用的一行代码，确保文件在程序运行期间不会被打开。

第二行使用 print() 函数输出了一个消息，指出文件已成功保存到 fname_out 命名的文件中。print() 函数是一个内置的 Python 函数，用于在程序中输出消息或信息。在这里，print() 函数的第一个参数是一个字符串和一个空格，表示在字符串中插入一个空格，然后将 fname_out 变量与空格连接，最后输出一个感叹号。

第三行也是使用 print() 函数输出一个空格，用于在字符串中插入一个空格，以便在 fname_out 变量和空格之间插入一个空行。


```cpp
fout.close()

print("Done. Output file: " + fname_out)
print("")

```

# `examples/gpt-j/main.cpp`

这段代码是一个C++文件，其中包含两个头文件和两个函数。它们都是在C++标准中定义的。

第一个头文件是“ggml/ggml.h”，可能是ggml库的头文件。第二个头文件是“common.h”，可能是定义了一些通用的基类的头文件。第三个头文件是“common-ggml.h”，可能是ggml库的定义文件。

这两个头文件中可能会定义一些函数，定义了ggml库和common库的一些基本的数据结构和函数，用于在地图中存储和操作点集。

注意，没有在代码中包含任何函数，因此无法确定ggml库具体是如何工作的。


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

这段代码是一个C++头文件，其中包含了一些C++14和C++17的预处理指令，以及一些语义定义。具体来说，这个头文件的作用是定义了一个gptj_hparams结构体，用于定义一个GPT-J模型的参数。

首先，头文件中包含了一个#pragma warning(disable: 4244 4267)指令，这个指令的作用是告诉编译器不要因为定义了一些未定义的变量而产生警告。这里的4244和4267指的是C++14和C++17中未定义的扩展关键字。

接下来，头文件中定义了一个gptj_hparams结构体，它的各个成员变量分别表示了一个GPT-J模型的参数。其中，n_vocab表示GPT-J模型的词汇数，n_ctx表示GPT-J模型的上下文大小，n_embd表示GPT-J模型的嵌入维度，n_head表示GPT-J模型的头数，n_layer表示GPT-J模型的层数，n_rot表示GPT-J模型的旋转数，fty表示GPT-J模型的类型，eps表示GPT-J模型的 Evaluation Error String。

最后，头文件中包含了一个printf，输出了一些字符串，这些字符串告诉编译器在编译时将给定的gptj_hparams结构体的一些成员变量定义为const，以便在编译时检查编译器的错误。


```cpp
#pragma warning(disable: 4244 4267) // possible loss of data
#endif


// default hparams (GPT-J 6B)
struct gptj_hparams {
    int32_t n_vocab = 50400;
    int32_t n_ctx   = 2048;
    int32_t n_embd  = 4096;
    int32_t n_head  = 16;
    int32_t n_layer = 28;
    int32_t n_rot   = 64;
    int32_t ftype   = 1;
    float   eps     = 1e-5f;
};

```

这段代码定义了一个名为 `gptj_layer` 的结构体，表示文本到序列（Token-to-Sequence）模型中的一个时间步。该结构体包含了一系列与注意力机制相关的张量。

具体来说，这个结构体包含以下内容：

- `ln_1_g` 和 `ln_1_b`：这两个张量表示输入文本的第一个隐藏层中的节点。这些节点通过一个非线性变换（例如 ReLU）进行预处理，然后被输入到注意力机制中。

- `c_attn_q_proj_w`、`c_attn_k_proj_w` 和 `c_attn_v_proj_w`：这三个张量表示注意力机制中的 queries（查询）、keys（键）和 values（值）。这些张量被用来计算注意力权重，从而对输入文本中的不同部分进行加权。

- `c_attn_proj_w`：这个张量表示注意力机制中的 projection（投影），它将查询、键和值张量通过一个固定的维数（例如 2048）进行投影，然后再输入到下一层。

- `c_mlp_fc_w` 和 `c_mlp_fc_b`：这两个张量表示 MLP（多层感知器）中的前馈（feedforward）层。这些张量在模型中用于计算隐藏层的输入，并将其输入到 `c_attn_proj_w` 中进行计算。

- `c_mlp_proj_w` 和 `c_mlp_proj_b`：这两个张量表示 MLP 中的前馈层。这些张量在模型中用于计算隐藏层的输入，并将其输入到 `c_attn_proj_w` 中进行计算。


```cpp
struct gptj_layer {
    // normalization
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    // attention
    struct ggml_tensor * c_attn_q_proj_w;
    struct ggml_tensor * c_attn_k_proj_w;
    struct ggml_tensor * c_attn_v_proj_w;

    struct ggml_tensor * c_attn_proj_w;

    // ff
    struct ggml_tensor * c_mlp_fc_w;
    struct ggml_tensor * c_mlp_fc_b;

    struct ggml_tensor * c_mlp_proj_w;
    struct ggml_tensor * c_mlp_proj_b;
};

```

这段代码定义了一个名为 `gptj_model` 的结构体，用于表示一个基于 Gradient-based预训练的语言模型。

该结构体包含以下几个部分：

1. `hparams`：表示模型超参数，包括 `num_layers`（层数）、`hidden_size`（隐藏层大小）等。
2. `ln_f_g` 和 `ln_f_b`：表示两个浮点数向量，分别用于计算语言模型函数的 f 值和 b 值。这些向量可能会在训练过程中进行更新，以适应不同数据集和预处理技术。
3. `wte`：表示位置嵌入向量，用于表示输入文本中的位置信息。
4. `layers`：是一个 `std::vector<gptj_layer>` 类型的层，用于表示整个模型的结构。其中 `gptj_layer` 是一个自定义的层，可能包含层的一些关键部分，如 `ctx`、`memory_k` 和 `memory_v` 等。
5. `ctx`：表示一个 `ggml_context` 类型的数据结构，用于管理模型上下文。这个上下文可以包含一些有用的信息，如 `tensors`  map，用于存储模型在训练和推理过程中使用的各种数据。
6. `tensors`：是一个 `std::map<std::string, struct ggml_tensor *>` 类型的 map，用于存储模型在训练和推理过程中使用的各种数据。

`gptj_model` 可以在训练和推理过程中使用，以对输入文本进行自然语言处理，并生成相应的输出。它的主要作用是为了解决自然语言处理中的各种问题，如文本分类、命名实体识别、机器翻译等。


```cpp
struct gptj_model {
    gptj_hparams hparams;

    // normalization
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte; // position embedding

    struct ggml_tensor * lmh_g; // language model head
    struct ggml_tensor * lmh_b; // language model bias

    std::vector<gptj_layer> layers;

    // key + value memory
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    //
    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

```

It appears that this function, `__apply_tensor_to_model_file`, takes in a single tensor object and applies a mathematical operation to it and writes the result back to the model file. It does this by first reading in the tensor data from the given file, then applying the operation to the tensor data and writing the result back to the file.

The function takes in an optional parameter `tensor_type`, which specifies the type of tensor to apply the operation to. If this parameter is not provided, the function defaults to `float`.

The function also takes in an optional parameter `bpe`, which specifies the block size for the Gender Mixture Encoder (BMC) encoding algorithm. This is used to ensure that the tensor data is written in a consistent format.

The function contains a number ofprintf statements that are used to print out information about the tensor data, such as the tensor name, the number of elements in the tensor, the data type of the tensor (either "float" or "f16"), and the size of the tensor data in bytes. These printouts are used to debug the tensor data and help the user understand what the function is doing.


```cpp
// load the model's weights from a file
bool gptj_model_load(const std::string & fname, gptj_model & model, gpt_vocab & vocab) {
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
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: n_rot   = %d\n", __func__, hparams.n_rot);
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

        ctx_size += n_embd*n_vocab*ggml_type_sizef(wtype); // wte

        ctx_size += n_embd*n_vocab*ggml_type_sizef(wtype);         // lmh_g
        ctx_size +=        n_vocab*ggml_type_sizef(GGML_TYPE_F32); // lmh_b

        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_g
        ctx_size += n_layer*(n_embd*ggml_type_sizef(GGML_TYPE_F32)); // ln_1_b

        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype)); // c_attn_q_proj_w
        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype)); // c_attn_k_proj_w
        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype)); // c_attn_v_proj_w

        ctx_size += n_layer*(n_embd*n_embd*ggml_type_sizef(wtype)); // c_attn_proj_w

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_fc_w
        ctx_size += n_layer*(       4*n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_fc_b

        ctx_size += n_layer*(4*n_embd*n_embd*ggml_type_sizef(wtype));         // c_mlp_proj_w
        ctx_size += n_layer*(         n_embd*ggml_type_sizef(GGML_TYPE_F32)); // c_mlp_proj_b

        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F16); // memory_k
        ctx_size += n_ctx*n_layer*n_embd*ggml_type_sizef(GGML_TYPE_F16); // memory_v

        ctx_size += (5 + 10*n_layer)*512; // object overhead

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
        model.lmh_b  = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_vocab);

        // map by name
        model.tensors["transformer.wte.weight"] = model.wte;

        model.tensors["transformer.ln_f.weight"] = model.ln_f_g;
        model.tensors["transformer.ln_f.bias"]   = model.ln_f_b;

        model.tensors["lm_head.weight"] = model.lmh_g;
        model.tensors["lm_head.bias"]   = model.lmh_b;

        for (int i = 0; i < n_layer; ++i) {
            auto & layer = model.layers[i];

            layer.ln_1_g          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);
            layer.ln_1_b          = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            layer.c_attn_q_proj_w = ggml_new_tensor_2d(ctx, wtype,           n_embd,   n_embd);
            layer.c_attn_k_proj_w = ggml_new_tensor_2d(ctx, wtype,           n_embd,   n_embd);
            layer.c_attn_v_proj_w = ggml_new_tensor_2d(ctx, wtype,           n_embd,   n_embd);

            layer.c_attn_proj_w   = ggml_new_tensor_2d(ctx, wtype,           n_embd,   n_embd);

            layer.c_mlp_fc_w      = ggml_new_tensor_2d(ctx, wtype,           n_embd, 4*n_embd);
            layer.c_mlp_fc_b      = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, 4*n_embd);

            layer.c_mlp_proj_w    = ggml_new_tensor_2d(ctx, wtype,         4*n_embd,   n_embd);
            layer.c_mlp_proj_b    = ggml_new_tensor_1d(ctx, GGML_TYPE_F32,   n_embd);

            // map by name
            model.tensors["transformer.h." + std::to_string(i) + ".ln_1.weight"]          = layer.ln_1_g;
            model.tensors["transformer.h." + std::to_string(i) + ".ln_1.bias"]            = layer.ln_1_b;

            model.tensors["transformer.h." + std::to_string(i) + ".attn.q_proj.weight"]   = layer.c_attn_q_proj_w;
            model.tensors["transformer.h." + std::to_string(i) + ".attn.k_proj.weight"]   = layer.c_attn_k_proj_w;
            model.tensors["transformer.h." + std::to_string(i) + ".attn.v_proj.weight"]   = layer.c_attn_v_proj_w;

            model.tensors["transformer.h." + std::to_string(i) + ".attn.out_proj.weight"] = layer.c_attn_proj_w;

            model.tensors["transformer.h." + std::to_string(i) + ".mlp.fc_in.weight"]     = layer.c_mlp_fc_w;
            model.tensors["transformer.h." + std::to_string(i) + ".mlp.fc_in.bias"]       = layer.c_mlp_fc_b;

            model.tensors["transformer.h." + std::to_string(i) + ".mlp.fc_out.weight"]    = layer.c_mlp_proj_w;
            model.tensors["transformer.h." + std::to_string(i) + ".mlp.fc_out.bias"]      = layer.c_mlp_proj_b;
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

        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);

        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);

        printf("%s: memory_size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);
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

            //printf("%42s - [%5d, %5d], type = %6s, %6.2f MB\n", name.c_str(), ne[0], ne[1], ttype == 0 ? "float" : "f16", ggml_nbytes(tensor)/1024.0/1024.0);
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

This code appears to be a implementation of the Flask-Based Pre-trained Language Model (FPLM) pre-training algorithm。 The code seems to be using the Glorot还有一种 attention mechanism for the model. It also appears to be using a有很好的饮誉度指标，我觉得这对评估模型好坏是很好的指标。此外，还使用了开放空间策略来避免过拟合。


```cpp
// evaluate the transformer
//
//   - model:     the model
//   - n_threads: number of threads to use
//   - n_past:    the context size so far
//   - embd_inp:  the embeddings of the tokens in the context
//   - embd_w:    the predicted logits for the next token
//
// The GPT-J model requires about 16MB of memory per input token.
//
bool gptj_eval(
        const gptj_model & model,
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

        // norm
        {
            cur = ggml_norm(ctx0, inpL, hparams.eps);

            // cur = ln_1_g*cur + ln_1_b
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0,
                        ggml_repeat(ctx0, model.layers[il].ln_1_g, cur),
                        cur),
                    ggml_repeat(ctx0, model.layers[il].ln_1_b, cur));
        }

        struct ggml_tensor * inpSA = cur;

        // self-attention
        {
            struct ggml_tensor * Qcur = ggml_rope_inplace(ctx0, ggml_reshape_3d(ctx0, ggml_mul_mat(ctx0, model.layers[il].c_attn_q_proj_w, cur), n_embd/n_head, n_head, N), KQ_pos, n_rot, 0, 0);
            struct ggml_tensor * Kcur = ggml_rope_inplace(ctx0, ggml_reshape_3d(ctx0, ggml_mul_mat(ctx0, model.layers[il].c_attn_k_proj_w, cur), n_embd/n_head, n_head, N), KQ_pos, n_rot, 0, 0);

            // store key and value to memory
            {
                struct ggml_tensor * Vcur = ggml_transpose(ctx0, ggml_mul_mat(ctx0, model.layers[il].c_attn_v_proj_w, cur));

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

            // projection (no bias)
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_attn_proj_w,
                    cur);
        }

        struct ggml_tensor * inpFF = cur;

        // feed-forward network
        // this is independent of the self-attention result, so it could be done in parallel to the self-attention
        {
            // note here we pass inpSA instead of cur
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_mlp_fc_w,
                    inpSA);

            cur = ggml_add(ctx0,
                    ggml_repeat(ctx0, model.layers[il].c_mlp_fc_b, cur),
                    cur);

            // GELU activation
            cur = ggml_gelu(ctx0, cur);

            // projection
            // cur = proj_w*cur + proj_b
            cur = ggml_mul_mat(ctx0,
                    model.layers[il].c_mlp_proj_w,
                    cur);

            cur = ggml_add(ctx0,
                    ggml_repeat(ctx0, model.layers[il].c_mlp_proj_b, cur),
                    cur);
        }

        // self-attention + FF
        cur  = ggml_add(ctx0, cur, inpFF);

        // input for next layer
        inpL = ggml_add(ctx0, cur, inpL);
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

        inpL = ggml_add(ctx0,
                ggml_repeat(ctx0, model.lmh_b, inpL),
                inpL);
    }

    // logits -> probs
    //inpL = ggml_soft_max_inplace(ctx0, inpL);

    // run the computation
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-j.dot");
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

This is a C++ implementation of the Text-to-Token (T2T) model training and evaluation code. The T2T model is trained on a limited amount of training data and is used to predict the translation of a given text into a fixed language model.

The main function of the code is to start and run the T2T model. It does this by first loading the model parameters and input data from a text file and a translation dictionary. It then starts the model processing and displays the predicted translation at each step.

The code uses the Deep Learning library (LLTK) to handle the model training and evaluation. It uses LLTK's G饶计函数测量模型的运行时间并打印输出。

The code prints timing information at the end of the input text, such as the time taken to process the entire input and the time taken to process each individual token.

It is important to note that this is a simple example and that in practice, you would need to use more robust tools to parse and translate text files, as well as handle more complex input data.


```cpp
int main(int argc, char ** argv) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;
    params.model = "models/gpt-j-6B/ggml-model.bin";

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
    gptj_model model;

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();

        if (!gptj_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        t_load_us = ggml_time_us() - t_start_us;

        test_gpt_tokenizer(vocab, params.token_test);
    }

    int n_past = 0;

    int64_t t_sample_us  = 0;
    int64_t t_predict_us = 0;

    std::vector<float> logits;

    // tokenize the prompt
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());

    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());
    printf("\n");

    std::vector<gpt_vocab::id> embd;

    // determine the required inference memory per token:
    size_t mem_per_token = 0;
    gptj_eval(model, params.n_threads, 0, { 0, 1, 2, 3 }, logits, mem_per_token);

    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // predict
        if (embd.size() > 0) {
            const int64_t t_start_us = ggml_time_us();

            if (!gptj_eval(model, params.n_threads, n_past, embd, logits, mem_per_token)) {
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
                if (int32_t(embd.size()) > params.n_batch) {
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